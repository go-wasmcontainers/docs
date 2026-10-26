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
**Engine v0.9.0 is released** (`tar+zstd` layers, WebAssembly exception
handling, named volumes, `wctr events` and `wctr stats`,
containers that survive a restart, `wctr start`, `stop` and `restart`, shell
completion), and **the playground is live**:
<https://go-wasmcontainers.github.io/playground/> runs `wctr` in a browser tab,
with no server, and works offline. The engine also runs natively on Linux and
macOS: `wctr pull`, `images`, `run`, `ps`, `logs`, `rm`, `rmi`, `inspect`,
`kill`, `start`, `stop`, `restart`, `diff`, `events`, `stats`, `volume`,
`system df`, `system prune` and `version`. No networking, `exec`, attach or build yet: see
[Limitations]({{< relref "limitations" >}}). Everything on the
[roadmap]({{< relref "roadmap" >}}) is **planned**. These pages describe
engine v0.9.0.
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
- [Commands]({{< relref "commands" >}}): `wctr`, `wctrd`, exit codes and limits
- [Architecture]({{< relref "architecture" >}}): the engine, its API, the executors, the images
- [Performance]({{< relref "performance" >}}): what was measured
- [Limitations]({{< relref "limitations" >}}): what the engine does not do
- [Relay]({{< relref "relay" >}}): `wctr-relay`, the classroom relay and firewall (v0.1.0)
- [Design]({{< relref "design" >}}): design documents for planned parts, such as the [classroom network]({{< relref "design/classroom-network" >}})
- [Roadmap]({{< relref "roadmap" >}}): what is planned
- [FAQ]({{< relref "faq" >}}): why `wctr`, and not `docker` or `ctr`
- [References]({{< relref "references" >}}): the specifications the code is written against

## Repositories

| Repo | Status |
| --- | --- |
| [engine](https://github.com/go-wasmcontainers/engine) | v0.9.0 released |
| [relay](https://github.com/go-wasmcontainers/relay) | v0.1.0 released (M-relay-1, M-relay-2) |
| [playground](https://github.com/go-wasmcontainers/playground) | live, in the browser |
| [dashboard](https://github.com/go-wasmcontainers/dashboard) | v0.1.0 released; [live demo](https://go-wasmcontainers.github.io/dashboard/) |
| [docs](https://github.com/go-wasmcontainers/docs) | this site |
| [go-wasmcontainers.github.io](https://github.com/go-wasmcontainers/go-wasmcontainers.github.io) | the landing page |
| [brand](https://github.com/go-wasmcontainers/brand) | logo, icons and social images |
