---
title: "Compose"
weight: 26
description: "wctr compose: multi-container applications from Compose files in YAML, JSON or HCL2, with published ports, restart policies and health checks. Compose v0.6.0, built into wctr."
tags: [compose]
---

**`wctr compose` runs a multi-container application described by the Compose
Specification, through the engine's API.** It is the separate module
[go-wasmcontainers/compose](https://github.com/go-wasmcontainers/compose)
(v0.6.0, status M1; it needs engine v0.14.0 or later, **engine v0.22.0 or later for published ports**, **engine v0.25.0 or later for restart policies** and **engine v0.28.0 or later for health checks**), and it is **built into `wctr`** since `cmd/wctr` v0.1.0 (see
[Getting started]({{< relref "getting-started" >}})); the standalone
`wctr-compose` still exists. Compose files are accepted in **three formats, YAML,
JSON and HCL2**, which load to the same model; files of different formats merge
(`-f compose.yaml -f override.hcl`). The HCL2 mapping is specified in
[docs/HCL.md](https://github.com/go-wasmcontainers/compose/blob/main/docs/HCL.md).

## Commands

`wctr compose [-f FILE]... [-p NAME] [--profile P]... COMMAND`

| Command | Flags |
|---|---|
| `up [SERVICE...]` | `-d`, `--force-recreate`, `--build`, `--remove-orphans`, `--wait-timeout` (how long a `service_healthy` dependency may stay starting; default 5 minutes) |
| `down` | `-v` |
| `ps [SERVICE...]` | `-a`, `-q` |
| `logs [SERVICE...]` | `-f` |
| `start`, `stop`, `restart` | dependencies first; `stop` also stops a container waiting to restart, and ends its restart policy until the next start |
| `rm [SERVICE...]` | `-s` |
| `pull`, `images` | |
| `build` | builds the services' images with [`wctr build`]({{< relref "build" >}}) |
| `config` | `--format yaml\|json\|hcl`, `-q`, `--services` |

Loading is complete: discovery, interpolation, `.env`, merging, `extends`,
`include`, profiles, and validation against the specification's schema, with
every error carrying `file:line:column`. The Compose files are untrusted input
and every reader is bounded and fuzzed.

## Networking (v0.3.0)

Services **reach each other by service name and aliases**, on as many networks as
they declare. This needs engine v0.14.0 or later (it is built against v0.18.0).
Builds (`up --build`, `build`) use `wctr build` (compose v0.2.0).

## Published ports (v0.4.0)

A service's `ports` (short and long syntax) and `expose` are applied, through the
engine's published ports (engine v0.22.0 or later; see
[Networking]({{< relref "networking" >}})). **One difference from the
specification, on purpose**: a port that names no host address (`"8080:80"`, or the
long syntax without `host_ip`) is published on **127.0.0.1**, the engine's
default, not on every interface as the specification says, so that a lab
container does not become a service on the machine's network by accident. Write
`"0.0.0.0:8080:80"` (or `host_ip: 0.0.0.0`) when that is meant.

## Restart policies (v0.5.0)

A service's `restart` (`no`, `always`, `on-failure[:N]`, `unless-stopped`) and
`deploy.restart_policy` are applied, through the engine's restart policies
(engine v0.25.0 or later; see [Commands]({{< relref "commands" >}})).
In `deploy.restart_policy`, `condition` `none`, `on-failure` and `any` map to `no`,
`on-failure` and `always`, and `max_attempts` is the retry count. When both are
written, **`deploy.restart_policy` wins over `restart`**. `delay` and `window` are
not applied: the engine's own backoff (100 ms, doubling up to a minute) applies, and
compose warns about them. `compose stop` also stops a container that is waiting to
restart, and ends its restart policy until the next start.

## Health checks (v0.6.0)

A service's `healthcheck` is applied, through the engine's health checks (engine
v0.28.0 or later; see [Commands]({{< relref "commands" >}})): `test` as `CMD`,
`CMD-SHELL` or `NONE`, the durations, and `retries`. `disable: true` is `NONE`.
What a `healthcheck` leaves unset, even `test`, comes from the image's own check,
which the engine merges field by field, as Docker does. **`CMD-SHELL` needs a
`/bin/sh` in the image**, for example `ghcr.io/go-wasmcontainers/shell`
([Shell]({{< relref "shell" >}})).

`depends_on` with `condition: service_healthy` **really waits**. The wait follows
the engine's events, and `up` fails when the dependency reports unhealthy (with the
last probe's exit code and output), when it stops first, when it has no health
check, and after `--wait-timeout` (default 5 minutes) when it is still starting. A
dependency with `required: false` that fails any of these is a warning, and `up`
carries on. The engine's minimum interval is 100 ms.

With this, every engine feature compose waited for is applied: aliases and several
networks, label filters, ports, restart policies and health checks.

## Limits

What the engine cannot do yet, and compose therefore does not:

- **Build attributes** such as remote contexts, `cache_from` and `secrets` are
  accepted with a warning and not applied; what the builder itself does not do is
  on the [Build]({{< relref "build" >}}) page.
- **No host bind mounts**, and no file outside the project directory.
