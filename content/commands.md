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
| `run IMAGE [COMMAND] [ARG...]` | `--rm`, `-d/--detach`, `--name`, `-e/--env KEY=value` (or `KEY` from the shell), `--entrypoint`, `-w/--workdir`, `-m/--memory`, `-v NAME:/path[:ro]` (v0.8.0), `--network NAME` (v0.10.0), `--network-alias NAME` (v0.14.0), `--gpu` (v0.13.0) |
| `ps` | `-a/--all`, `-q` |
| `logs CONTAINER` | `-f/--follow` |
| `rm CONTAINER...` | `-f/--force` |
| `rmi IMAGE...` | `-f/--force` |
| `commit CONTAINER [REPO[:TAG]]` | `-m/--message`, `-a/--author`, `-c/--change` (`CMD`, `ENTRYPOINT`, `ENV`, `EXPOSE`, `LABEL`, `ONBUILD`, `STOPSIGNAL`, `USER`, `VOLUME`, `WORKDIR`): the container's changes as one OCI layer on the parent's, with a history entry; without a name, a `<none>` image addressable by ID (v0.12.0) |
| `tag SOURCE TARGET` | another name for an image; a tag that moves leaves the old image as `<none>` (v0.12.0) |
| `inspect NAME\|ID...` | containers, then images |
| `build`, `compose` | [`wctr build`]({{< relref "build" >}}) and [`wctr compose`]({{< relref "compose" >}}), built into `wctr` since `cmd/wctr` v0.1.0; they use wctr's `-H`, which goes before the subcommand |
| `exec [-d] [-i] [-t] [-e K=V] [-w DIR] CONTAINER CMD...` | a further process in a running container, in its process group; needs `--wasi engine`; exits with the command's status; `-t` gives it a terminal (the engine does the line editing and echo); `-u` accepts only `root` or `0` (engine v0.20.0; not in the browser executor yet) |
| `kill CONTAINER...` | |
| `start CONTAINER...` | `-a/--all` (v0.5.0) |
| `stop CONTAINER...` | `-t`, accepted and ignored (v0.5.0) |
| `restart CONTAINER...` | `-t`, accepted and ignored (v0.5.0) |
| `diff CONTAINER` | (v0.4.0) |
| `events` | `--since`, `--until`, filters on type, event, container, image, volume and label (v0.6.0) |
| `stats` | (v0.7.0) |
| `volume create\|ls\|inspect\|rm\|prune` | (v0.8.0) |
| `network create\|ls\|inspect\|rm` | (v0.10.0) |
| `network connect [--alias NAME] NETWORK CONTAINER` | a running container at once (v0.14.0) |
| `network disconnect [-f] NETWORK CONTAINER` | (v0.14.0) |
| `network prune [-f] [--filter label=K[=V]]` | unused networks (v0.14.0) |
| `system df` | `-v` (v0.4.0) |
| `system prune` | `-a`, `-f` (v0.4.0) |
| `version` | |
| `completion bash\|zsh\|fish\|powershell` | prints the completion script for that shell (v0.3.0) |

`wctr run` pulls an image it does not have.

A WebAssembly guest has no signals, so `wctr stop` kills the container at once:
`-t` is accepted for compatibility and ignored. In the API, `/attach` is a plain
streamed response, not a hijacked connection.

## For builders (v0.12.0, API only for now)

- `POST /images/load` takes an OCI image layout as a tar; each `index.json` entry
  is named by `io.containerd.image.name` or `org.opencontainers.image.ref.name`,
  else `<none>`. It is checked as a pull is, and nothing is read from the host.
- `PUT /containers/{id}/archive?path=DIR` extracts a tar into a container
  (relative clean names only, no devices, paths resolved inside the container, its
  quota applies), and `GET` returns a tar of a path. These two routes accept
  uploads up to **1 GiB** by default; every other request is limited to 1 MiB.
- `GET /images/{name}/json` shows the full image config, the history, and
  `RootFS.Layers` as `diff_ids`. Dangling images appear as `<none>`.
- `HEAD` on the archive route returns only the path's stat, without building a tar
  (v0.14.0).
- Since v0.18.0 the Go client's `Containers`, `Networks` and `Volumes` take
  optional filters (label filters among them). That changed their signatures;
  **engine v0.21.0 is planned to restore the old signatures**, which is not
  released yet.
- Other modules add commands to `wctr` through an extension hook,
  `cli.Options.Extra`; [`wctr build`]({{< relref "build" >}}) and
  [`wctr compose`]({{< relref "compose" >}}) use it. **`wctr build` needs a
  `wctrd` of v0.12.0 or later** (v0.17.0 for shell-form `RUN`).

## Events (v0.6.0)

`wctr events` and `GET /events` stream NDJSON. `--since` and `--until` bound
the range, and the filters are type, event, container, image, volume, network and label.
The last 1024 events are kept, and a reader that is too slow is cut off.

## Stats (v0.7.0)

`wctr stats` and the stats endpoint report memory against its limit, bytes
written, layer and log size, and pids. **CPU is not measured**: a WebAssembly
guest gives no CPU accounting, so it is shown as `--`.

## Volumes (v0.8.0)

`wctr volume create|ls|inspect|rm|prune`, and `wctr run -v NAME:/path[:ro]`.
Only named volumes exist: a host path is refused. A volume is shared between
containers, `:ro` is enforced, and volumes survive a restart and a page reload.
They appear in `system df` and in `inspect`.

## Persistence (v0.4.0)

Containers survive a `wctrd` restart: they are kept under `--data-root` in
`containers/` (state, exit code, logs and written files). A container that was
running comes back `exited` with code 137, and a `--rm` container is removed. An
entry that is corrupted is dropped and reported.

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
