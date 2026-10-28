---
title: "Roadmap"
weight: 35
description: "What has shipped and what is planned."
tags: [roadmap]
---

## Shipped

| Item | Release |
| --- | --- |
| **Engine M1**: native, wazero executor; `wctr pull`, `images`, `run`, `ps`, `logs`, `rm`, `rmi`, `inspect`, `kill`, `version`; the `wctrd` daemon | engine v0.1.0 |
| **Browser executor** (`runtime/browser`), the shared WASI host (`wasi`), `api/inproc`, the `cli` package, `RegistryOptions.Mirrors` | engine v0.2.0 |
| **`wctr completion`** for bash, zsh, fish and PowerShell; **`cli/lineedit`**, a pure-Go line editor with completion, used by the playground terminal | engine v0.3.0 |
| **Persistence**: containers survive a `wctrd` restart and a page reload (state, exit code, logs, written files; running ones come back exited 137); `wctr system df`, `system prune`, `diff` | engine v0.4.0 |
| **Fix**: a `kill`, `rm -f` or shutdown during start could succeed without killing the process; it is now held until the process attaches | engine v0.4.1 |
| **`wctr start`** (`-a`), **`stop`**, **`restart`** | engine v0.5.0 |
| **`wctr events`** / `GET /events` (NDJSON, `--since`, `--until`, filters) | engine v0.6.0 |
| **`wctr stats`** (memory against its limit, bytes written, layer and log size, pids; CPU not measured) | engine v0.7.0 |
| **Volumes**: `wctr volume create\|ls\|inspect\|rm\|prune`, `run -v NAME:/path[:ro]` | engine v0.8.0 |
| **`tar+zstd` layers** (bounded, fuzzed); WebAssembly exception handling enabled natively | engine v0.9.0 |
| **Networking**: sockets for guests (WasmEdge socket extension V2, preview1 `sock_*`), built-in DNS by container name, the networks `bridge` (10.88.0.0/16) and `none`, `wctr network create\|ls\|inspect\|rm`, `run --network`, a class network through the relay (`wctrd --relay`); `wctrd --wasi engine` is the new default; see [Networking]({{< relref "networking" >}}) | engine v0.10.0 |
| **A class joined at run time**: the engine's network manager can add and remove a class network while running (`AddDriver`, `RemoveDriver`); the playground's "Join class" uses it | engine v0.11.0 |
| **Builder support**: `wctr commit`, `wctr tag`, dangling `<none>` images, `POST /images/load`, container archive `PUT`/`GET` (1 GiB limit on those two routes), the full image config with `diff_ids`, and the `wctr` extension hook that `wctr build` uses | engine v0.12.0 |
| **Playground v0.2.0**, on engine v0.9.0 (with completion, persistence and offline use): `start`, `stop`, `restart`, `diff`, `stats`, `events`, `volume`, `system df\|prune`, `run -v`; the images `lua` 5.4.9, `sqlite3` 3.53.4 and `minigzip` (zlib 1.3.2), C tools built by go-pkgx for wasip1 | playground v0.2.0, live |
| **Playground M1**: `wctr` in a browser tab, no server | playground v0.1.0; live at <https://go-wasmcontainers.github.io/playground/> |
| **Classroom relay** M-relay-1 and M-relay-2: [`wctr-relay`]({{< relref "relay" >}}), the relay and firewall with security groups modelled on OpenStack Neutron, its gRPC control API, TLS, OpenID Connect | relay v0.1.0 |
| **Live flow map** (M-relay-2b): the relay's page drawing the class's flows, refused ones in red | relay v0.2.0 |
| **CORS** for browsers on another origin: `wctr-relay serve --cors-origin` | relay v0.3.0 |
| **`wctr build`**: multi-stage builds from a Containerfile or Dockerfile, the build cache, `.containerignore`; exec-form `RUN` ([Build]({{< relref "build" >}})) | build v0.2.0 (standalone `wctr-build`, needs wctrd v0.12.0 or later) |
| **`wctr compose`**: Compose files in YAML, JSON and HCL2; `up`, `down`, `ps`, `logs`, `start`, `stop`, `restart`, `rm`, `pull`, `images`, `config` ([Compose]({{< relref "compose" >}})) | compose v0.1.0 |
| **ONNX loader and CPU operators** ([GPU]({{< relref "gpu" >}})) | gpu v0.1.0 |

What the engine does not do is on [Limitations]({{< relref "limitations" >}}).

## Planned

Nothing below exists yet.

| Item | What it is |
| --- | --- |
| Playground: stdin, a shell, networking | the playground has none of the three yet |
| Networking in the playground | the networking above, in the browser: the next milestone, **not live yet** |
| Published ports | `run -p`, and `network connect` |
| Classroom relay, the rest | students' containers reaching each other through the relay from the playground: the engine's network driver and the browser path (M-relay-3, -4), ICMP (M-relay-5), high availability across relays (M-relay-HA); see the [design]({{< relref "design/classroom-network" >}}) |
| Shell and `exec` | `wctr exec`, attach, and an interactive shell, through a WASIX subset: the guest side (`sh`, the `wasix` package) exists in [shell]({{< relref "shell" >}}); the engine's process spawning and a base image are **in progress** |
| Shell-form `RUN` | needs the shell above; exec-form `RUN` works now |
| `wctr build` and `wctr compose` inside `wctr` | today they are the standalone `wctr-build` and `wctr-compose` |
| GPU | wasi-nn host module (next), wasi:webgpu compute and rendering (planned); see [GPU]({{< relref "gpu" >}}) |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles. go-pkgx builds C tools for wasip1 (lua 5.4.9, sqlite 3.53.4, zlib 1.3.2 with minigzip), which are now in the playground's catalogue; a base image plus tools is still planned |
| A reference comparison | the benchmarks set against another WebAssembly runtime |

An item moves to "shipped" with the release that carries it.
