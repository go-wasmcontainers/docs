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
| **Engine M1**: native, wazero executor; `pull`, `run`, `ps`, `logs`, `rm`, `inspect` | in progress |

## Planned

| Item | What it is |
| --- | --- |
| Browser executor | the executor contract in a browser Web Worker, which the playground needs |
| Playground | an in-browser terminal running `docker` against WASI containers, for cloud and container training labs (the `playground` repository exists; no code yet) |
| Networking | containers that listen and connect, through the WasmEdge socket ABI |
| Shell and `exec` | `docker exec` and an interactive shell, through a WASIX subset |
| `docker build` | building an image from a Dockerfile |
| Compose | several containers described in one file |
| Mini-Kubernetes | enough of the orchestrator's API to teach its concepts |
| apt-like images | a base image plus tools, from [pkgx](https://github.com/go-pkgx) `wasip1` bottles, once go-pkgx has its `wasip1/wasm` target |

An item moves to "in progress" when its code is on a branch, and is removed from
this page when it ships, with the release that carries it.
