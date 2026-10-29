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
pure-Go WebAssembly runtime, instead of in a Linux kernel. `wctrd` serves a
subset of the Docker Engine API.

{{< callout type="info" >}}
**Engine v0.27.0 is released**: restart policies (`run --restart`), published ports (`run -p`, `wctr port`), `wctr exec` (in the browser executor too, and `exec -t` follows window resizes), the WASIX process subset (`sh`,
pipelines, shell-form `RUN`), networks with aliases and several networks per
container, `wctr commit` and `tag`, `wctr build` and `wctr compose` built into
`wctr`, networking ([Networking]({{< relref "networking" >}})), `tar+zstd` layers,
named volumes, `wctr events` and `wctr stats`, containers that survive a restart,
and shell completion. **The playground is live** (v0.3.0, with networking, Join
class and wasi-nn): <https://go-wasmcontainers.github.io/playground/> runs a
dashboard and `wctr` in a browser tab, with no server, and works offline. No
attach and no `-p` in the browser yet (in progress), and the playground (v0.3.0) does not carry the new `exec` until it is rebuilt: see
[Limitations]({{< relref "limitations" >}}). Everything on the
[roadmap]({{< relref "roadmap" >}}) under "Planned" is **planned**. These pages
describe engine v0.27.0.
{{< /callout >}}

## Why

Teaching containers usually means a Linux machine per learner. A WASI module
needs no kernel, so the same engine runs natively and in a browser tab — which
makes a training lab a web page. The learner types container commands the way
they will at work.

## Pages

- [Playground]({{< relref "playground" >}}): `wctr` in a browser tab
- [Dashboard]({{< relref "dashboard" >}}): a graphical dashboard for the engine (v0.6.0)
- [Getting started]({{< relref "getting-started" >}}): install, and a real session
- [Networking]({{< relref "networking" >}}): sockets, DNS by container name and networks
- [Build]({{< relref "build" >}}): `wctr build` from a Containerfile (build v0.4.0)
- [Compose]({{< relref "compose" >}}): multi-container applications, YAML, JSON or HCL2, with ports (compose v0.4.0)
- [Shell]({{< relref "shell" >}}): a POSIX shell for WASI containers (shell v0.1.0, coreutils v0.4.0)
- [GPU]({{< relref "gpu" >}}): wasi-nn on the CPU and WebGPU, wasi:webgpu compute and rendering to surfaces (gpu v0.5.1)
- [Commands]({{< relref "commands" >}}): `wctr`, `wctrd`, exit codes and limits
- [Architecture]({{< relref "architecture" >}}): the engine, its API, the executors, the images
- [Performance]({{< relref "performance" >}}): what was measured
- [Limitations]({{< relref "limitations" >}}): what the engine does not do
- [Relay]({{< relref "relay" >}}): `wctr-relay`, the classroom relay and firewall (v0.3.1)
- [Design]({{< relref "design" >}}): design documents for planned parts, such as the [classroom network]({{< relref "design/classroom-network" >}})
- [Roadmap]({{< relref "roadmap" >}}): what is planned
- [FAQ]({{< relref "faq" >}}): why `wctr`, and not `docker` or `ctr`
- [References]({{< relref "references" >}}): the specifications the code is written against

## Repositories

| Repo | Status |
| --- | --- |
| [engine](https://github.com/go-wasmcontainers/engine) | v0.27.0 released (`cmd/wctr` v0.1.0) |
| [relay](https://github.com/go-wasmcontainers/relay) | v0.3.1 released |
| [build](https://github.com/go-wasmcontainers/build) | v0.4.0 released (built into `wctr`) |
| [compose](https://github.com/go-wasmcontainers/compose) | v0.4.0 released (built into `wctr`; ports need engine v0.22.0 or later) |
| [shell](https://github.com/go-wasmcontainers/shell) | v0.1.0 released (runs on engine v0.17.0 or later; public base image `ghcr.io/go-wasmcontainers/shell:0.1.0`) |
| [gpu](https://github.com/go-wasmcontainers/gpu) | v0.5.1 released (wasi-nn, wasi:webgpu compute, rendering to surfaces; the Linux SwiftShader lane renders) |
| [playground](https://github.com/go-wasmcontainers/playground) | v0.3.0 released, live in the browser |
| [dashboard](https://github.com/go-wasmcontainers/dashboard) | v0.6.0 released; [live demo](https://go-wasmcontainers.github.io/dashboard/) |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
