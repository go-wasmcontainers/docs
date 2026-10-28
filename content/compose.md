---
title: "Compose"
weight: 26
description: "wctr compose: multi-container applications from Compose files in YAML, JSON or HCL2. Compose v0.1.0."
tags: [compose]
---

**`wctr compose` runs a multi-container application described by the Compose
Specification, through the engine's API.** It is the separate module
[go-wasmcontainers/compose](https://github.com/go-wasmcontainers/compose)
(v0.1.0, status M1). Compose files are accepted in **three formats, YAML, JSON
and HCL2**, which load to the same model; files of different formats merge
(`-f compose.yaml -f override.hcl`). The HCL2 mapping is specified in
[docs/HCL.md](https://github.com/go-wasmcontainers/compose/blob/main/docs/HCL.md).

The command on its own is `wctr-compose`, which takes `-H` like `wctr`:

```console
$ go install github.com/go-wasmcontainers/compose/cmd/wctr-compose@v0.1.0
```

## Commands

`wctr compose [-f FILE]... [-p NAME] [--profile P]... COMMAND`

| Command | Flags |
|---|---|
| `up [SERVICE...]` | `-d`, `--force-recreate`, `--build` (refused until build is wired to `wctr build`), `--remove-orphans` |
| `down` | `-v` |
| `ps [SERVICE...]` | `-a`, `-q` |
| `logs [SERVICE...]` | `-f` |
| `start`, `stop`, `restart` | dependencies first |
| `rm [SERVICE...]` | `-s` |
| `pull`, `images` | |
| `config` | `--format yaml\|json\|hcl`, `-q`, `--services` |

Loading is complete: discovery, interpolation, `.env`, merging, `extends`,
`include`, profiles, and validation against the specification's schema, with
every error carrying `file:line:column`. The Compose files are untrusted input
and every reader is bounded and fuzzed.

## Limits of v0.1.0

What the engine cannot do yet, and compose therefore does not:

- A service joins **one network** (the first by name, with a warning); **network
  aliases are not applied yet**: reach a container by its name (`lab-web-1`),
  not its service's name.
- **`ports` and `restart` are recorded with a warning**, not applied.
- **`build:` is refused** until compose is wired to `wctr build`.
- **`service_healthy` fails clearly**: the engine cannot run a command in a
  container yet (no `exec`).
- **No host bind mounts**, and no file outside the project directory.
