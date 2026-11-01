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
**Engine v0.30.0 is released**: `run -i` and `-t`, `wctr attach` and `start -ai` (a container's own stdin and terminal, detach keys, resizes), health checks (`run --health-cmd`, `State.Health`, `health_status` events), restart policies (`run --restart`), published ports (`run -p`, `wctr port`), `wctr exec` (in the browser executor too, and `exec -t` follows window resizes), the WASIX process subset (`sh`,
pipelines, shell-form `RUN`), networks with aliases and several networks per
container, `wctr commit` and `tag`, `wctr build` and `wctr compose` built into
`wctr`, networking ([Networking]({{< relref "networking" >}})), `tar+zstd` layers,
named volumes, `wctr events` and `wctr stats`, containers that survive a restart,
and shell completion. **The playground is live** (v0.6.0, on engine v0.30.0, with networking, Join
class, wasi-nn, GPU surfaces, WASIX, a shell image, `run -it`, `-p` in a tab and a WebDAV file server, `fileshare`):
<https://go-wasmcontainers.github.io/playground/> runs a dashboard and `wctr` in
a browser tab, with no server, and works offline. A second origin for containers'
pages, <https://go-wasmcontainers-usercontent.github.io/>, is deployed, but
**`-p` in the browser is not live yet** (the rest of it is in progress): see
[Limitations]({{< relref "limitations" >}}). Everything on the
[roadmap]({{< relref "roadmap" >}}) under "Planned" is **planned**. These pages
describe engine v0.30.0.
{{< /callout >}}

## Why

Teaching containers usually means a Linux machine per learner. A WASI module
needs no kernel, so the same engine runs natively and in a browser tab — which
makes a training lab a web page. The learner types container commands the way
they will at work.

## Pages

- [Playground]({{< relref "playground" >}}): `wctr` in a browser tab
- [Dashboard]({{< relref "dashboard" >}}): a graphical dashboard for the engine (v0.8.0)
- [Getting started]({{< relref "getting-started" >}}): install, and a real session
- [Networking]({{< relref "networking" >}}): sockets, DNS by container name and networks
- [Build]({{< relref "build" >}}): `wctr build` from a Containerfile (build v0.4.0)
- [Compose]({{< relref "compose" >}}): multi-container applications, YAML, JSON or HCL2, with ports, restart policies and health checks (compose v0.6.0)
- [Shell]({{< relref "shell" >}}): a POSIX shell for WASI containers (shell v0.3.0, coreutils v0.6.0)
- [GPU]({{< relref "gpu" >}}): wasi-nn on the CPU and WebGPU, wasi:webgpu compute, rendering to surfaces and surface input (gpu v0.6.0)
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
| [engine](https://github.com/go-wasmcontainers/engine) | v0.30.0 released (`cmd/wctr` v0.2.0) |
| [relay](https://github.com/go-wasmcontainers/relay) | v0.3.1 released |
| [build](https://github.com/go-wasmcontainers/build) | v0.4.0 released (built into `wctr`) |
| [compose](https://github.com/go-wasmcontainers/compose) | v0.6.0 released (built into `wctr`; ports need engine v0.22.0 or later, restart policies v0.25.0, health checks v0.28.0) |
| [shell](https://github.com/go-wasmcontainers/shell) | v0.3.0 released (a line editor on a terminal; `wasix/v0.1.0` with `wasix/exec`; runs on engine v0.17.0 or later; public base image `ghcr.io/go-wasmcontainers/shell:0.3.0`, with coreutils v0.6.0) |
| [gpu](https://github.com/go-wasmcontainers/gpu) | v0.6.0 released (wasi-nn, wasi:webgpu compute, rendering to surfaces, surface input) |
| [playground](https://github.com/go-wasmcontainers/playground) | v0.6.0 released, live in the browser |
| [dashboard](https://github.com/go-wasmcontainers/dashboard) | v0.8.0 released (Open shell; surface input for GPU panes); [live demo](https://go-wasmcontainers.github.io/dashboard/) |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
