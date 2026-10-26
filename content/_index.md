---
title: "go-wasmcontainers documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A container engine whose containers are WASI modules, compatible with the Docker Engine API, in pure Go (CGO=0), with an in-browser playground planned for cloud and container training labs."
---

**go-wasmcontainers runs WASI modules as containers.** A container here is a
`wasip1/wasm` module: the image is an OCI image whose platform is `os=wasip1`,
`arch=wasm`, the `wctr` command line talks to the `wctrd` daemon over
`/var/run/wctr.sock`, and the module runs in [wazero](https://wazero.io), a
pure-Go WebAssembly runtime, instead of in a Linux kernel. `wctrd` is
compatible with the Docker Engine API.

{{< callout type="warning" >}}
**Work in progress; nothing is released.** The engine's first milestone, M1
(native, wazero executor: `wctr pull`, `run`, `ps`, `logs`, `rm`, `rmi`,
`inspect`, `images`), is being written. Everything else is **planned** and is
marked so on every page. These pages describe the `main` branch.
{{< /callout >}}

## Why

Teaching containers usually means a Linux machine per learner. A WASI module
needs no kernel, so the same engine can run natively today (M1) and, in the
plan, in a browser worker — which would make a training lab a web page. The
learner types container commands the way they will at work.

## Pages

- [Architecture]({{< relref "architecture" >}}): the engine, its API, the executors, the images
- [Getting started]({{< relref "getting-started" >}}): what exists today, and what M1 will give you
- [Design]({{< relref "design" >}}): design documents for planned parts, such as the [classroom network]({{< relref "design/classroom-network" >}})
- [Roadmap]({{< relref "roadmap" >}}): what is in progress and what is planned
- [FAQ]({{< relref "faq" >}}): why `wctr`, and not `docker` or `ctr`
- [References]({{< relref "references" >}}): the specifications the code is written against

## Repositories

| Repo | Status |
| --- | --- |
| [engine](https://github.com/go-wasmcontainers/engine) | in progress (M1) |
| [playground](https://github.com/go-wasmcontainers/playground) | planned; not started |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
