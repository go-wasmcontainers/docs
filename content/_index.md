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
**Engine v0.10.0 is released** (networking: sockets for guests, DNS by container
name and networks, see [Networking]({{< relref "networking" >}}); `tar+zstd`
layers, WebAssembly exception handling, named volumes, `wctr events` and `wctr stats`,
containers that survive a restart, `wctr start`, `stop` and `restart`, shell
completion), and **the playground is live**:
<https://go-wasmcontainers.github.io/playground/> runs `wctr` in a browser tab,
with no server, and works offline. The engine also runs natively on Linux and
macOS: `wctr pull`, `images`, `run`, `ps`, `logs`, `rm`, `rmi`, `inspect`,
`kill`, `start`, `stop`, `restart`, `diff`, `events`, `stats`, `volume`,
`network`, `system df`, `system prune` and `version`. No `exec`, attach or build yet, and networking is not in the playground yet: see
[Limitations]({{< relref "limitations" >}}). Everything on the
[roadmap]({{< relref "roadmap" >}}) is **planned**. These pages describe
engine v0.10.0.
{{< /callout >}}

## Why

Teaching containers usually means a Linux machine per learner. A WASI module
needs no kernel, so the same engine runs natively and in a browser tab — which
makes a training lab a web page. The learner types container commands the way
they will at work.

## Pages

- [Playground]({{< relref "playground" >}}): `wctr` in a browser tab
- [Dashboard]({{< relref "dashboard" >}}): a graphical dashboard for the engine (v0.1.0)
- [Getting started]({{< relref "getting-started" >}}): install, and a real session
- [Networking]({{< relref "networking" >}}): sockets, DNS by container name and networks (engine v0.10.0)
- [Build]({{< relref "build" >}}): `wctr build` from a Containerfile (build v0.2.0)
- [Compose]({{< relref "compose" >}}): multi-container applications, YAML, JSON or HCL2 (compose v0.1.0)
- [Shell]({{< relref "shell" >}}): a POSIX shell for WASI containers (in progress)
- [GPU]({{< relref "gpu" >}}): ONNX models on the CPU (gpu v0.1.0); GPU compute and rendering planned
- [Commands]({{< relref "commands" >}}): `wctr`, `wctrd`, exit codes and limits
- [Architecture]({{< relref "architecture" >}}): the engine, its API, the executors, the images
- [Performance]({{< relref "performance" >}}): what was measured
- [Limitations]({{< relref "limitations" >}}): what the engine does not do
- [Relay]({{< relref "relay" >}}): `wctr-relay`, the classroom relay and firewall (v0.3.0)
- [Design]({{< relref "design" >}}): design documents for planned parts, such as the [classroom network]({{< relref "design/classroom-network" >}})
- [Roadmap]({{< relref "roadmap" >}}): what is planned
- [FAQ]({{< relref "faq" >}}): why `wctr`, and not `docker` or `ctr`
- [References]({{< relref "references" >}}): the specifications the code is written against

## Repositories

| Repo | Status |
| --- | --- |
| [engine](https://github.com/go-wasmcontainers/engine) | v0.10.0 released |
| [relay](https://github.com/go-wasmcontainers/relay) | v0.3.0 released |
| [build](https://github.com/go-wasmcontainers/build) | v0.2.0 released (standalone `wctr-build`) |
| [compose](https://github.com/go-wasmcontainers/compose) | v0.1.0 released (M1) |
| [shell](https://github.com/go-wasmcontainers/shell) | in progress (guest side) |
| [gpu](https://github.com/go-wasmcontainers/gpu) | v0.1.0 released (ONNX on the CPU) |
| [playground](https://github.com/go-wasmcontainers/playground) | live, in the browser |
| [dashboard](https://github.com/go-wasmcontainers/dashboard) | v0.1.0 released; [live demo](https://go-wasmcontainers.github.io/dashboard/) |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
