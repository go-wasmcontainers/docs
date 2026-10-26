---
title: "Architecture"
weight: 10
description: "The wctrd daemon, the API it serves, the executors that run a module, and the OCI images it pulls."
tags: [design]
---

This page describes the design. Where a part is not written yet it says
**planned**.

## The pieces

```text
wctr CLI ──► /var/run/wctr.sock ──► wctrd ──► executor ──► a wasip1 module
              (Docker Engine API)     │           │
                                      │           ├─ native: wazero         (v0.1.0)
                                      │           └─ browser worker         (planned)
                                      └─ image store, root filesystem
```

### The daemon, `wctrd`

A Go program, pure Go, no cgo. It owns the images, the containers and their
state, and serves an HTTP API on the Unix socket `/var/run/wctr.sock`. The API
is compatible with the Docker Engine API: the reference it is written against
is Docker Engine API v1.47, and the subset M1 covers is the one the commands
`pull`, `run`, `ps`, `logs`, `rm`, `rmi`, `inspect` and `images` need:
`GET /_ping`, `POST /images/create`, the image list and delete endpoints, and the
container create, start, list, inspect, logs, wait and delete endpoints. Logs of
a container without a TTY use the API's multiplexed stream framing.

### The command line, `wctr`

The client. It finds the daemon at `/var/run/wctr.sock`, or at the address in
the `WCTR_HOST` environment variable. Because the API is Docker-Engine
compatible, other clients of that API can talk to `wctrd` as well; `wctr` is the
command the labs teach.

### The executors

An executor runs one module with a root filesystem, arguments, an
environment and a working directory.

- **wazero** (released in v0.1.0): the native executor. wazero is a pure-Go
  WebAssembly runtime with a WASI preview1 implementation, so the engine stays
  a single static binary. The container's root filesystem is adapted to wazero's
  writable filesystem interface; the image's `WorkingDir` is passed as `PWD`,
  which is where a Go `wasip1` program takes its initial directory from.
- **Browser worker** (planned): the same executor contract implemented over the
  browser's WebAssembly, in a Web Worker, so the playground needs no server
  that runs containers.

### The images

An image is an OCI image whose manifest or index entry has `os=wasip1` and
`arch=wasm`. The engine pulls from an OCI registry (the Distribution API, with
the registry's bearer-token authentication), selects the `wasip1/wasm` entry of
an index, verifies digests, and unpacks the layers in order — whiteouts
included — into the container's root filesystem. The Wasm OCI artifact layout of
the CNCF TAG-Runtime (`application/wasm` layer,
`application/vnd.wasm.config.v0+json` config) is also read.

Images are meant to be **apt-like**: a small base plus the tools a lab needs,
built from [pkgx](https://github.com/go-pkgx) `wasip1` bottles. go-pkgx is
gaining a `wasip1/wasm` target for this; until it ships, images come from
whatever registry holds `wasip1` images.

## What is not in this design yet

Networking (the WasmEdge socket ABI), `wctr exec` and a shell (a WASIX subset),
building an image from a Dockerfile, Compose and a mini-Kubernetes are on the
[roadmap]({{< relref "roadmap" >}}); none of them exists.
