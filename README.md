<img src="https://git.pleroma.social/pleroma/pleroma/uploads/8cec84f5a084d887339f57deeb8a293e/pleroma-banner-vector-nopad-notext.svg" width="300px" />

## About 

Pleroma is a microblogging server software that can federate (= exchange messages with) other servers that support ActivityPub. What that means is that you can host a server for yourself or your friends and stay in control of your online identity, but still exchange messages with people on larger servers. Pleroma will federate with all servers that implement ActivityPub, like Friendica, GNU Social, Hubzilla, Mastodon, Misskey, Peertube, and Pixelfed.

Pleroma is written in Elixir and uses PostgresSQL for data storage. It's efficient enough to be ran on low-power devices like Raspberry Pi (though we wouldn't recommend storing the database on the internal SD card ;) but can scale well when ran on more powerful hardware (albeit only single-node for now).

For clients it supports the [Mastodon client API](https://docs.joinmastodon.org/api/guidelines/) with Pleroma extensions (see the API section on <https://docs-develop.pleroma.social>).

- [Client Applications for Pleroma](https://docs-develop.pleroma.social/backend/clients/)

### OpenTelemetry patchset

_ivucica, 2025-07-27_

This is just the minimal, clumsy patchset done for my local instance, mainly to trace requests and database queries
as they happen, using OpenTelemetry tracing.

SQL queries are annotated with sqlcommenter-style comments, so the spans in PostgreSQL can be mapped to traces and
spans created in nginx and Pleroma.

`pg_tracing` extension for PostgreSQL and Pleroma have both been set up to push traces into my local
`otelcol-contrib` install, which then happens to push things onwards into Google Cloud Trace (or into null, since
most of the time I don't feel like pushing data into tracing, but I also don't feel like disabling tracing in
everything that creates traces and spans).

There is currently an issue where the parent span ID is the same as parent trace ID even if
`opentelemetry-sqlcommenter.ex` includes the correct parent span ID. `pg_tracing` extension seems to have hiccups
with the format sent by Ecto and sqlcommenter.ex, possibly related to no whitespace before the SQL comment begins.
This is an unconfirmed theory though.

It would be nice to clean up new deps in `mix.exs`, and perhaps make them optional. Especially the logger changes
could be cleaned up since pre-1.15 awareness already exists in mix.exs. All this is a future problem, though.

Patchset is provided as-is, as it's just a *surprisingly* minor addition of dependencies and a reconfiguration.

#### prod.secret.exs changes

While the rest of the changes are in the patches, a few manual changes are needed to the config,
e.g. in `config/prod.secret.exs`.

```elixir
# https://last9.io/blog/opentelemetry-with-elixir/
config :opentelemetry, :processors,
  otel_batch_processor: %{
    exporter: {:opentelemetry_exporter,
      %{
        # your.real.hostname:4137 has otelcol-contrib's receivers/otlp/protocols/grpc/endpoint value set,
        # with http:// prefix.
        endpoints: [System.get_env("OTEL_EXPORTER_OTLP_ENDPOINT") || "http://your.real.hostname:4317"],
        headers: [{"Authorization", System.get_env("OTEL_EXPORTER_OTLP_AUTH_HEADER") || "Some InvalidHeader"}]
      }}
  }


# https://docs.dynatrace.com/docs/ingest-from/opentelemetry/walkthroughs/elixir
#text_map_propagators: [:baggage, :trace_context]
# unclear how to actually add it, possibly in config for opentelemetry like so?
config :opentelemetry, :text_map_propagators, [:baggage, :trace_context]


config :opentelemetry,
  resource: [service: %{name: System.get_env("OTEL_SERVICE_NAME") || "Pleroma"}],
  span_processor: :batch,
  traces_exporter: :otlp,
  resource_detectors: [
    :otel_resource_app_env,
    :otel_resource_env_var #,
    # ExtraMetadata # https://docs.dynatrace.com/docs/ingest-from/opentelemetry/walkthroughs/elixir <-- we don't really have metadata we can't add in collector, but we could create a subfile to do so
  ]


config :opentelemetry_ecto, :tracer,
  repos: [:pleroma, :repo]  # Add your repositories for tracing


config :opentelemetry_phoenix, :tracer,
  service_name: System.get_env("OTEL_SERVICE_NAME" || "Pleroma")


# https://opentelemetry.io/docs/languages/erlang/exporters/
# how does it conflict with batch_processor above? does it?
# this is receivers/otlp/protocols/http/endpoint value from /etc/otelcol-contrib/config.yaml
config :opentelemetry_exporter,
  otlp_protocol: :http_protobuf,
  otlp_endpoint: "http://your.real.hostname:4318"
```

The following is supposed to log formatting is supposed to work with `logger` -- but that's a 1.15-or-later variant:

```elixir
# https://hexdocs.pm/logger_json/LoggerJSON.html
#config :logger, :default_handler,
#  formatter: {LoggerJSON.Formatters.GoogleCloud, metadata: [:request_id, :otel_trace_id, :otel_span_id, :otel_trace_flags, :pid, :time, :level, :file, :line], project_id: "ivucica-host"}
```

Unfortunately, on the VPS where this code is being run there is no 1.15, so:

```elixir
# https://hexdocs.pm/logger_formatter_json/readme.html
# 1. it might be possible to remove entire "template: []" section
# {keys, gcp} can be inserted inside [] as well
# 2. setting mildly differently due to elixir <15, per docs above
#config :logger, :default_handler,
#  formatter: {
config :foo, :logger_formatter_config, {
    :logger_formatter_json,
    %{
      template: [
        :msg,
        :time,
        :level,
        :file,
        :line,
        # :mfa,
        :pid,
        :request_id,
        :otel_trace_id,
        :otel_span_id,
        :otel_trace_flags
      ]
    }
  }
```

Note how it's setting up a fake app, `foo`. There are complaints on the CLI about this, but it's fine.
It would be nice to attach this to a more sensible 'app', but this should be done by someone that acutally
knows Elixir and Erlang and the ecosystem.


## Installation

### OTP releases (Recommended)
If you are running Linux (glibc or musl) on x86/arm, the recommended way to install Pleroma is by using OTP releases. OTP releases are as close as you can get to binary releases with Erlang/Elixir. The release is self-contained, and provides everything needed to boot it. The installation instructions are available [here](https://docs-develop.pleroma.social/backend/installation/otp_en/).

### From Source
If your platform is not supported, or you just want to be able to edit the source code easily, you may install Pleroma from source.

- [Alpine Linux](https://docs-develop.pleroma.social/backend/installation/alpine_linux_en/)
- [Arch Linux](https://docs-develop.pleroma.social/backend/installation/arch_linux_en/)
- [CentOS 7](https://docs-develop.pleroma.social/backend/installation/centos7_en/)
- [Debian-based](https://docs-develop.pleroma.social/backend/installation/debian_based_en/)
- [Debian-based (jp)](https://docs-develop.pleroma.social/backend/installation/debian_based_jp/)
- [FreeBSD](https://docs-develop.pleroma.social/backend/installation/freebsd_en/)
- [Gentoo Linux](https://docs-develop.pleroma.social/backend/installation/gentoo_en/)
- [NetBSD](https://docs-develop.pleroma.social/backend/installation/netbsd_en/)
- [OpenBSD](https://docs-develop.pleroma.social/backend/installation/openbsd_en/)
- [OpenBSD (fi)](https://docs-develop.pleroma.social/backend/installation/openbsd_fi/)

### OS/Distro packages
Currently Pleroma is packaged for [YunoHost](https://yunohost.org), [NixOS](https://nixos.org), [Gentoo through GURU](https://gentoo.org/) and [Archlinux through AUR](https://aur.archlinux.org/packages/pleroma). You may find more at <https://repology.org/project/pleroma/versions>.  
If you want to package Pleroma for any OS/Distros, we can guide you through the process on our [community channels](#community-channels). If you want to change default options in your Pleroma package, please **discuss it with us first**.

### Docker
While we don’t provide docker files, other people have written very good ones. Take a look at <https://github.com/angristan/docker-pleroma> or <https://glitch.sh/sn0w/pleroma-docker>.

### Raspberry Pi
Community maintained Raspberry Pi image that you can flash and run Pleroma on your Raspberry Pi. Available here <https://github.com/guysoft/PleromaPi>.

### Compilation Troubleshooting
If you ever encounter compilation issues during the updating of Pleroma, you can try these commands and see if they fix things:

- `mix deps.clean --all`
- `mix local.rebar`
- `mix local.hex`
- `rm -r _build`

If you are not developing Pleroma, it is better to use the OTP release, which comes with everything precompiled.

## Documentation
- Latest Released revision: <https://docs.pleroma.social>
- Latest Git revision: <https://docs-develop.pleroma.social>

## Community Channels
* IRC: **#pleroma** and **#pleroma-dev** on libera.chat, webchat is available at <https://irc.pleroma.social>
* Matrix: [#pleroma:libera.chat](https://matrix.to/#/#pleroma:libera.chat) and [#pleroma-dev:libera.chat](https://matrix.to/#/#pleroma-dev:libera.chat)
