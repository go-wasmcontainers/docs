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
| **Host modules and `--gpu`**: typed imports beside `wasi_snapshot_preview1`, per-process state, `cabi_realloc`, `wctr run --gpu` | engine v0.13.0 |
| **Networks, more**: aliases, several networks per container, `network connect`, `disconnect`, `prune`, label filters, `HEAD` on the archive route | engine v0.14.0 |
| **Process primitives**: spawn, descriptor tables, process groups, pipes | engine v0.15.0 |
| **`wctr` is a module of its own**, with `wctr build` and `wctr compose` built in; install with two `go install` commands | engine v0.16.0, `cmd/wctr` v0.1.0 |
| **The WASIX process subset** as a host module (`wasix_32v1`): spawn, join, pipes, tty; `sh`, pipelines and shell-form `RUN` can run | engine v0.17.0 |
| **List filters** on the Go client (`Containers`, `Networks`, `Volumes`) | engine v0.18.0 |
| **`Config.Label`** names a process's container (the seam for process-level features) | engine v0.19.0 |
| **`wctr exec`** and the exec endpoints, with a terminal line discipline | engine v0.20.0 |
| **The Go client's old list signatures restored** (v0.18.0 had broken the dashboard); filtering through `ContainersFiltered`, `NetworksFiltered`, `VolumesFiltered`; a CI job compiles downstream modules | engine v0.21.0 |
| **Published ports** natively: `run -p`, `-P`, `--expose`, `wctr port`, a PORTS column in `ps`, `NetworkSettings.Ports`; default bind `127.0.0.1` with a bounded proxy; a refused start records exit 128 and `--rm` removes the container | engine v0.22.0 |
| **Step output streams live** in `wctr build` | build v0.4.0 |
| **Rendering on surfaces** (`wasi-gfx:surface` with `surface-webgpu`, frames to the page), shown in the dashboard's Surfaces view | gpu v0.5.0 |
| **Playground v0.3.0**, on engine v0.14.0: the dashboard as the main UI, networking in the browser, Join class (relay), wasi-nn with the `mnist` image | playground v0.3.0, live |
| **Playground v0.2.0**, on engine v0.9.0 (with completion, persistence and offline use): `start`, `stop`, `restart`, `diff`, `stats`, `events`, `volume`, `system df\|prune`, `run -v`; the images `lua` 5.4.9, `sqlite3` 3.53.4 and `minigzip` (zlib 1.3.2), C tools built by go-pkgx for wasip1 | playground v0.2.0, live |
| **Playground M1**: `wctr` in a browser tab, no server | playground v0.1.0; live at <https://go-wasmcontainers.github.io/playground/> |
| **Classroom relay** M-relay-1 and M-relay-2: [`wctr-relay`]({{< relref "relay" >}}), the relay and firewall with security groups modelled on OpenStack Neutron, its gRPC control API, TLS, OpenID Connect | relay v0.1.0 |
| **Live flow map** (M-relay-2b): the relay's page drawing the class's flows, refused ones in red | relay v0.2.0 |
| **CORS** for browsers on another origin: `wctr-relay serve --cors-origin` | relay v0.3.0 |
| **The POSIX shell** (`sh`, the `wasix` package) and its public base image with the coreutils (`ghcr.io/go-wasmcontainers/shell:0.1.0`) | shell v0.1.0, coreutils v0.4.0 |
| **Compose on engine v0.21.0** | compose v0.3.2 |
| **Relay fix**: `golang.org/x/net` updated for two advisories | relay v0.3.1 |
| **`wctr build`**: multi-stage builds from a Containerfile or Dockerfile, the build cache, `.containerignore`; exec-form `RUN` ([Build]({{< relref "build" >}})) | build v0.2.0 |
| **Shell-form `RUN`** on an image with a shell (engine v0.17.0 or later) | build v0.3.0 |
| **`wctr compose`**: Compose files in YAML, JSON and HCL2; `up`, `down`, `ps`, `logs`, `start`, `stop`, `restart`, `rm`, `pull`, `images`, `config` ([Compose]({{< relref "compose" >}})) | compose v0.1.0 |
| **Compose builds** (`up --build`, `build`) | compose v0.2.0 |
| **Service-name networking**, aliases and several networks (engine v0.14.0 or later), label-filtered lists | compose v0.3.0, v0.3.1 |
| **GPU**: ONNX loader and CPU operators (v0.1.0); wasi-nn host module and `mnist` (v0.2.0); wasi-nn on WebGPU, proven on a real Apple GPU in CI (v0.3.0); wasi:webgpu compute (v0.4.0) ([GPU]({{< relref "gpu" >}})) | gpu v0.1.0 to v0.4.0 |
| **Surfaces view** for GPU frames; Networks view and Join class | dashboard v0.6.0, v0.5.0 |

What the engine does not do is on [Limitations]({{< relref "limitations" >}}).

## Planned

Nothing below exists yet.

| Item | What it is |
| --- | --- |
| Playground: stdin, a shell, `exec` | the playground has none of the three yet |
| Published ports in the browser | `run -p` in the playground: **in progress**, through a second origin; natively it shipped in engine v0.22.0 |
| Classroom relay, the rest | students' containers reaching each other through the relay from the playground: the engine's network driver and the browser path (M-relay-3, -4), ICMP (M-relay-5), high availability across relays (M-relay-HA); see the [design]({{< relref "design/classroom-network" >}}) |
| Attach and `run -it` | a terminal and stdin for a container's first process |
| GPU surface input, vertex buffers, depth-stencil | rendering shipped (gpu v0.5.0); these are not served yet; see [GPU]({{< relref "gpu" >}}) |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles. go-pkgx builds C tools for wasip1 (lua 5.4.9, sqlite 3.53.4, zlib 1.3.2 with minigzip), which are now in the playground's catalogue; a base image plus tools is still planned |
| A reference comparison | the benchmarks set against another WebAssembly runtime |

An item moves to "shipped" with the release that carries it.
