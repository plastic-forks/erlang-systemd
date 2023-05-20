# systemd

[![Hex.pm](https://img.shields.io/hexpm/v/systemd?style=flat-square)](https://hex.pm/packages/systemd)
[![HexDocs](https://img.shields.io/badge/HexDocs-docs-blue?style=flat-square)](https://hexdocs.pm/systemd/)
[![Hex.pm License](https://img.shields.io/hexpm/l/systemd?style=flat-square)](<https://tldrlegal.com/license/apache-license-2.0-(apache-2.0)>)
[![GitHub Workflow Status](https://img.shields.io/github/workflow/status/hauleth/erlang-systemd/Erlang%20CI?style=flat-square)](https://github.com/hauleth/erlang-systemd/actions)
[![Codecov](https://img.shields.io/codecov/c/gh/hauleth/erlang-systemd?style=flat-square)](https://codecov.io/gh/hauleth/erlang-systemd)

Systemd utilities for Erlang applications.

## Features

- [Notify Socket](https://www.freedesktop.org/software/systemd/man/sd_notify.html) which allows applications to send notifications to systemd.
- [Watchdog](https://www.freedesktop.org/software/systemd/man/sd_watchdog_enabled.html) which allows applications to integrate with watchdog of systemd.
- [Listen FDs](https://www.freedesktop.org/software/systemd/man/sd_listen_fds.html) which helps applications to use socket activation with ease.
- Logger handler and formatters for [`journald`](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html).

## Installation

In case of Rebar project, add this to your `rebar.config`:

```erlang
{deps, [systemd]}.
```

### Non-systemd systems

In non-systemd systems, all functionalities in this library will simply work as (almost) no-ops, which are safe to use.

## Usage - starting point

Suppose that the systemd unit file is `my_app.service`, and the initial content of this file is:

```ini
[Unit]
Description=My App

[Service]
User=appuser
Group=appgroup

# Application need to start in foreground instead of forking into background,
# otherwise it may be not correctly detected and system will try to start it
# again.
ExecStart=/path/to/my_app start

[Install]
WantedBy=multi-user.target
```

> In the following sections, above systemd unit file will be updated as required.

## Usage - Notify Socket

This is used to notify systemd the state of applications.

### Why you need this?

> In short, it is useful for signaling readiness.

When VM starts, applications rarely are ready to work. They often require some extra setup, such as establishing DB connections, setting up an HTTP server, etc. This setup process may take some noteworthy time.

By leveraging this feature, applications can be explicitly notify systemd that they are ready to work.

### Setup

This feature requires setup on both systemd side and application side.

On systemd side, the type of systemd unit should be changed to `Type=notify`:

```ini
[Unit]
Description=My App

[Service]
User=appuser
Group=appgroup

# added
Type=notify

ExecStart=/path/to/my_app start

[Install]
WantedBy=multi-user.target
```

On application side, you need notify systemd the state of your application.

Let's say, you want to notify systemd that the application is ready, then you can use `systemd:notify/1`:

```erlang
systemd:notify(ready).
```

To simplify ready notification, you can also utilize the `systemd:ready/0` function, which returns child specs that can be used as part of your supervision tree to indicate when the application is ready:

```erlang
-module(my_app_sup).

-behaviour(supervisor).

-export([start_link/1,
         init/1]).

start_link(Opts) ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, Opts).

init(_Opts) ->
    SupFlags = #{
      strategy => one_for_one
    },
    Children = [
      my_app_db:child_spec(),
      my_app_webserver:child_spec(),
      systemd:ready(),
      my_app_periodic_job:child_spec()
    ],

    {ok, {SupFlags, Children}}.
```

Read the doc of `systemd:notify/1` for more details.

## Usage - Watchdog

Watchdog is heart-beat subsystem of systemd. It's used for monitoring the state of applications and taking appropriate action in the event of a fault.

### Why you need this?

> In short, it is useful for probing liveness.

Watchdog can help to improve the reliability and availability of applications. For example, if a application becomes non-responsive, but it refuses to die, the watchdog can detect it, and automatically handle it without the need for manual intervention.

### Setup

This feature requires setup on both systemd side and application side.

On systemd side, the `WatchdogSec` parameter of systemd unit should be set:

```ini
[Unit]
Description=My App

[Service]
User=appuser
Group=appgroup

Type=notify

ExecStart=/path/to/my_app start

# added
#
# Enable the watchdog on the systemd side - expect notifications sent from application
# within a certain time interval.
WatchdogSec=10s
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

On application side, the watchdog will be setup (sending keep-alive notifications) automatically. But, you still have options to do that manually:

- send a keep-alive notification to systemd watchdog via `systemd:watchdog(ping)`
- trigger an action specified by `Restart=` via `systemd:watchdog(trigger)`

Read the doc of `systemd:watchdog/1` for more details.

## Usage - Listen FDs

### Why you need this?

> In short, it makes the deployment process more graceful.

This is a required feature for implementing socket activation. And, socket activation is useful when you want:

- starting application only when there is incoming data in socket.
- keeping sockets opened between restarts.
- ...

### Setup

This feature requires setup on both systemd side and application side.

On systemd side, a socket unit file - `my_app.socket` should be created:

```ini
[Unit]
Description=Listening socket
Requires=sockets.target

[Socket]
ListenStream=8080
ReusePort=true
NoDelay=true
```

On application side, you can use the function related to this feature - `systemd:listen_fds/0`, which returns all file descriptors passed from systemd to the service unit.

Read the source code in [examples/](./examples/) for more examples.

Read the doc of `systemd:listen_fds/0` for more details.

## Usage - Logger handler and formatters

To handle logs, you have 2 possible options:

- Output data to standard output or error with special prefixes. This approach
  is much simpler and straightforward, however it doesn't support structured logging
  and multiline messages.
- Use datagram socket with special communication protocol. This requires a
  little bit more effort to set up, but seamlessly supports structured logging
  and multiline messages.

This library supports both formats, and it is up to you which one (or both?) your application will decide to use.

### Erlang

#### Standard output or error

There is `systemd_kmsg_formatter` which formats data using `kmsg`-like level
prefixes can be used with any logger that outputs to standard output or
standard error if this is attached to the journal. By default `systemd` library
will update all handlers that use `logger_std_h` with type `standard_io` or
`standard_error` that are attached to the journal (it is automatically detected
via `JOURNAL_STREAM` environment variable). You can disable that behaviour by
setting:

```erlang
[
  {systemd, [{auto_formatter, false}]}
].
```

For custom loggers you can use this formatter by adding new option `parent` to
the formatter options that will be used as "upstream" formatter, for example:

```erlang
logger:add_handler(example_handler, logger_disk_log_h, #{
  formatter => {systemd_kmsg_formatter, #{parent => logger_formatter,
                                          template => [msg]},
  config => #{
    file => "/var/log/my_app.log"
  }
}).
```

#### Datagram socket

This one requires `systemd` application to be started to spawn some processes
required for handling sockets, so the best way to handle it is to add predefined
`systemd` handlers after your application starts:

```erlang
logger:add_handlers(systemd),
logger:remove_handler(default).
```

Be aware that this one is **not** guaranteed to work on non-systemd systems, so
if you aren't sure if that application will be ran on systemd-enabled systems then
you shouldn't use it as an only logger solution in your application or you can
end with no logger attached at all.

This handler **should not** be used with `systemd_kmsg_formatter` as this will
result with pointless `kmsg`-like prefixes in the log messages.

You can also "manually" configure handler if you want to configure formatter:

```erlang
logger:add_handler(my_handler, systemd_journal_h, #{
  formatter => {my_formatter, FormatterOpts}
}),
logger:remove_handler(default).
```

### Elixir

This assumes Elixir 1.10+, as earlier versions do not use Erlang's `logger`
module for dispatching logs.

#### Standard output and error

`systemd` has Erlang's `logger` backend, which mean that you have 2 ways of
achieving what is needed:

1. Disable Elixir's backends and just rely on default Erlang's handler:

```elixir
# config/config.exs
config :logger,
  backends: [],
  handle_otp_reports: false,
  handle_sasl_reports: false
```

And then allow `systemd` to make its magic that is used in "regular" Erlang
code.

2. "Manually" add handler that will use `systemd_kmsg_formatter`:

```elixir
# In application start/2 callback
:ok = :logger.add_handler(
  :my_handler,
  :logger_std_h,
  %{formatter: {:systemd_kmsg_formatter, %{}}}
)
Logger.remove_backend(:console)
```

However remember, that currently (as Elixir 1.11) there is no "Elixir formatter"
for Erlang's `logger` implementation, so you can end with Erlang-style
formatting of the metadata in the logs.

##### Datagram socket

You can use Erlang-like approach, which is:

```elixir
# In application start/2 callback
:logger.add_handlers(:systemd)
Logger.remove_backend(:console)
```

Or you can manually configure the handler:

```elixir
# In application start/2 callback
:logger.add_handler(
  :my_handler,
  :systemd_journal_h,
  %{formatter: {MyFormatter, formatter_opts}}
)
Logger.remove_backend(:console)
```

Be aware that this one is **not** guaranteed to work on non-systemd systems, so
if you aren't sure if that application will be ran on systemd-enabled systems then
you shouldn't use it as an only logger solution in your application or you can
end with no logger attached at all.

This handler **should not** be used with `:systemd_kmsg_formatter` as this will
result with pointless `kmsg`-like prefixes in the log messages.

## More

- [Who Watches Watchmen? - Part 1](https://hauleth.dev/post/who-watches-watchmen-i)

## License

See [LICENSE](LICENSE).
