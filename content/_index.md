---
title: "go-wasmcontainers documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A Docker-compatible container engine whose containers are WASI modules, in pure Go (CGO=0), with an in-browser playground planned for cloud and container training labs."
---

**go-wasmcontainers runs WASI modules as Docker containers.** A container here
is a `wasip1/wasm` module: the image is an OCI image whose platform is
`os=wasip1`, `arch=wasm`, the `docker` command line talks to the engine over
`/var/run/docker.sock`, and the module runs in [wazero](https://wazero.io), a
pure-Go WebAssembly runtime, instead of in a Linux kernel.

{{< callout type="warning" >}}
**Work in progress; nothing is released.** The engine's first milestone, M1
(native, wazero executor: `pull`, `run`, `ps`, `logs`, `rm`, `inspect`), is
being written. Everything else is **planned** and is marked so on every page.
These pages describe the `main` branch.
{{< /callout >}}

## Why

Teaching containers usually means a Linux machine per learner. A WASI module
needs no kernel, so the same engine can run natively today (M1) and, in the
plan, in a browser worker — which would make a training lab a web page. The
learner still types the `docker` commands they will type at work.

## Pages

- [Architecture]({{< relref "architecture" >}}): the engine, the Docker API, the executors, the images
- [Getting started]({{< relref "getting-started" >}}): what exists today, and what M1 will give you
- [Roadmap]({{< relref "roadmap" >}}): what is in progress and what is planned
- [References]({{< relref "references" >}}): the specifications the code is written against

## Repositories

| Repo | Status |
| --- | --- |
| [engine](https://github.com/go-wasmcontainers/engine) | in progress (M1) |
| [playground](https://github.com/go-wasmcontainers/playground) | planned; not started |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
