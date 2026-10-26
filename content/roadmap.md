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
| **Playground M1**: `wctr` in a browser tab, no server | playground v0.1.0, live at <https://go-wasmcontainers.github.io/playground/> |
| **Classroom relay** M-relay-1 and M-relay-2: [`wctr-relay`]({{< relref "relay" >}}), the relay and firewall with security groups modelled on OpenStack Neutron, its gRPC control API, TLS, OpenID Connect | relay v0.1.0 |
| **Live flow map** (M-relay-2b): the relay's page drawing the class's flows, refused ones in red | relay v0.2.0 |

What the engine does not do is on [Limitations]({{< relref "limitations" >}}).

## Planned

Nothing below exists yet.

| Item | What it is |
| --- | --- |
| Playground: stdin, a shell, networking | the playground has none of the three yet |
| Networking | containers that listen and connect, through the WasmEdge socket ABI |
| Classroom relay, the rest | students' containers reaching each other through the relay from the playground: the engine's network driver and the browser path (M-relay-3, -4), ICMP (M-relay-5), high availability across relays (M-relay-HA); see the [design]({{< relref "design/classroom-network" >}}) |
| Shell and `exec` | `wctr exec`, attach, and an interactive shell, through a WASIX subset |
| Image build | building an image from a Dockerfile |
| Compose | several containers described in one file |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles, once go-pkgx has its `wasip1/wasm` target |
| `tar+zstd` layers | image layers compressed with zstd |
| A reference comparison | the benchmarks set against another WebAssembly runtime |

An item moves to "shipped" with the release that carries it.
