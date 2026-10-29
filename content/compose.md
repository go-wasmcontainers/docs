---
title: "Compose"
weight: 26
description: "wctr compose: multi-container applications from Compose files in YAML, JSON or HCL2. Compose v0.3.2, built into wctr."
tags: [compose]
---

**`wctr compose` runs a multi-container application described by the Compose
Specification, through the engine's API.** It is the separate module
[go-wasmcontainers/compose](https://github.com/go-wasmcontainers/compose)
(v0.3.2, status M1; it needs engine v0.14.0 or later and is built against v0.21.0; published ports are coming in compose v0.4.0), and it is **built into `wctr`** since `cmd/wctr` v0.1.0 (see
[Getting started]({{< relref "getting-started" >}})); the standalone
`wctr-compose` still exists. Compose files are accepted in **three formats, YAML,
JSON and HCL2**, which load to the same model; files of different formats merge
(`-f compose.yaml -f override.hcl`). The HCL2 mapping is specified in
[docs/HCL.md](https://github.com/go-wasmcontainers/compose/blob/main/docs/HCL.md).

## Commands

`wctr compose [-f FILE]... [-p NAME] [--profile P]... COMMAND`

| Command | Flags |
|---|---|
| `up [SERVICE...]` | `-d`, `--force-recreate`, `--build`, `--remove-orphans` |
| `down` | `-v` |
| `ps [SERVICE...]` | `-a`, `-q` |
| `logs [SERVICE...]` | `-f` |
| `start`, `stop`, `restart` | dependencies first |
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

## Limits

What the engine cannot do yet, and compose therefore does not:

- **`ports` and `restart` are accepted with a warning**, not applied (no
  published ports, no restart policies).
- **`service_healthy` fails clearly**: compose does not run health checks.
- **Build attributes** such as remote contexts, `cache_from` and `secrets` are
  accepted with a warning and not applied; what the builder itself does not do is
  on the [Build]({{< relref "build" >}}) page.
- **No host bind mounts**, and no file outside the project directory.
