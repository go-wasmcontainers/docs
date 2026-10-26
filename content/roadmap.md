---
title: "Roadmap"
weight: 30
description: "What is in progress and what is planned. Nothing here has shipped."
tags: [roadmap]
---

Nothing on this page has shipped. It is the list of what the engine will not do
on day one, so nobody has to guess.

## In progress

| Item | Status |
| --- | --- |
| **Engine M1**: native, wazero executor; `wctr pull`, `run`, `ps`, `logs`, `rm`, `rmi`, `inspect`, `images` | in progress |

## Planned

| Item | What it is |
| --- | --- |
| Browser executor | the executor contract in a browser Web Worker, which the playground needs |
| Playground | an in-browser terminal running `wctr` against WASI containers, for cloud and container training labs (the `playground` repository exists; no code yet) |
| Networking | containers that listen and connect, through the WasmEdge socket ABI |
| Classroom relay | students' browsers reach each other's containers through a relay on the teacher's laptop that is also a firewall with security groups modelled on OpenStack Neutron, managed through a gRPC control API and the `wctr-relay` command; see the [design]({{< relref "design/classroom-network" >}}) (milestones M-relay-1 to M-relay-5) |
| Shell and `exec` | `wctr exec` and an interactive shell, through a WASIX subset |
| Image build | building an image from a Dockerfile |
| Compose | several containers described in one file |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles, once go-pkgx has its `wasip1/wasm` target |

An item moves to "in progress" when its code is on a branch, and is removed from
this page when it ships, with the release that carries it.
