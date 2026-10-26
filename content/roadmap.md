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

What v0.1.0 does not do is on [Limitations]({{< relref "limitations" >}}).

## Planned

Nothing below exists yet.

| Item | What it is |
| --- | --- |
| Browser executor | the executor contract in a browser Web Worker, which the playground needs |
| Playground | an in-browser terminal running `wctr` against WASI containers, for cloud and container training labs (the `playground` repository exists; no code yet) |
| Networking | containers that listen and connect, through the WasmEdge socket ABI |
| Classroom relay | students' browsers reach each other's containers through a relay on the teacher's laptop that is also a firewall with security groups modelled on OpenStack Neutron, managed through a gRPC control API and the `wctr-relay` command; see the [design]({{< relref "design/classroom-network" >}}) (milestones M-relay-1 to M-relay-5) |
| Shell and `exec` | `wctr exec`, attach, and an interactive shell, through a WASIX subset |
| Image build | building an image from a Dockerfile |
| Compose | several containers described in one file |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles, once go-pkgx has its `wasip1/wasm` target |
| `tar+zstd` layers | image layers compressed with zstd |
| Persistent containers | containers that survive a `wctrd` restart |
| A reference comparison | the benchmarks set against another WebAssembly runtime |

An item moves to "shipped" with the release that carries it.
