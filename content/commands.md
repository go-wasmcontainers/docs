---
title: "Commands"
weight: 20
description: "The wctr client, the wctrd daemon, exit codes and the limits the daemon enforces."
tags: [reference]
---

## `wctr`

The client. It finds the daemon at `-H unix:///path`, else `$WCTR_HOST`, else
`unix:///var/run/wctr.sock`.

| Command | Flags |
|---|---|
| `pull IMAGE` | `IMAGE` is a registry reference or `oci:<dir>[:<ref.name>]` |
| `images` | `-q` |
| `run IMAGE [COMMAND] [ARG...]` | `--rm`, `-d/--detach`, `--name`, `-e/--env KEY=value` (or `KEY` from the shell), `--entrypoint`, `-w/--workdir`, `-m/--memory` |
| `ps` | `-a/--all`, `-q` |
| `logs CONTAINER` | `-f/--follow` |
| `rm CONTAINER...` | `-f/--force` |
| `rmi IMAGE...` | `-f/--force` |
| `inspect NAME\|ID...` | containers, then images |
| `kill CONTAINER...` | |
| `version` | |
| `completion bash\|zsh\|fish\|powershell` | prints the completion script for that shell (v0.3.0) |

`wctr run` pulls an image it does not have.

### Exit status of `wctr run`

The container's own, or:

| Code | Meaning |
|---|---|
| 125 | the CLI or the daemon failed |
| 126 | not a WebAssembly module |
| 127 | module not found |
| 134 | the module trapped |
| 137 | killed, or the wall-clock timeout |

## `wctrd`

| Flag | Meaning | Default |
|---|---|---|
| `-H`, `--host unix:///path` | the socket to serve (unix only; TCP is refused, the API has no authentication) | `unix:///var/run/wctr.sock` |
| `--data-root` | where images are kept | `$XDG_DATA_HOME/wasmcontainers`, else `~/.local/share/wasmcontainers` |
| `--memory` | linear-memory limit per container | `256m` |
| `--run-timeout` | wall-clock limit per run (`0` = none) | `10m` |
| `--storage-quota` | bytes a container may write | `256m` |
| `--max-containers` | containers at once | `256` |
| `--insecure-registry host:port` | allow plain HTTP to that registry (loopback registries are always plain HTTP) | none |

### Bounds on untrusted content

| What | Bound |
|---|---|
| blob | 512 MiB |
| image | 1 GiB, 128 layers |
| unpacked layers | 1 GiB, 131072 files |
| gzip expansion ratio | 200:1 |
| logs kept | 8 MiB per container |
