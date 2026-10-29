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
                                      │           └─ browser: runtime/browser (v0.2.0)
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
- **Browser** (`runtime/browser`, engine v0.2.0): the same executor contract
  implemented over the browser's WebAssembly, with the WASI host shared with the
  native side (`wasi`) and the API served in-process (`api/inproc`), so the
  [playground]({{< relref "playground" >}}) needs no server that runs
  containers.

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

### Processes

With the engine's WASI host (`wctrd --wasi engine`, the default), a guest can
start other programs of its container through a small subset of
[WASIX](https://wasix.org/), in WASIX's own import module `wasix_32v1`:
`proc_spawn2`, `proc_spawn3`, `proc_join`, `proc_id`, `proc_parent`, `fd_pipe`,
`tty_get` and `tty_set` (no fork, exec, threads or signals), since engine v0.17.0.
A child is a new instance of a module of the same root filesystem; a container's
processes form one group (at most 64 at once, 32 spawns per second), killed
together with the container. This is what runs `sh` and its pipelines (the
[shell]({{< relref "shell" >}})), shell-form `RUN` in a build, and `wctr exec`
(engine v0.20.0).

### Published ports

A container has no socket on the host, so a published port (`run -p`, engine
v0.22.0) is a listener that `wctrd` opens on the host and proxies into the
container's network endpoint. It listens on `127.0.0.1` unless the binding names an
address. See [Networking]({{< relref "networking" >}}).

## What is not in this design yet

A mini-Kubernetes is on the [roadmap]({{< relref "roadmap" >}}) and does not
exist. Published ports in the browser (in progress) and `exec` in the playground
(the browser executor has it since engine v0.26.0; the playground is not rebuilt yet)
are also still to come: see [Limitations]({{< relref "limitations" >}}).
