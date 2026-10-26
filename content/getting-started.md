---
title: "Getting started"
weight: 20
description: "What exists today, and the session milestone M1 is being written to serve."
tags: [start]
---

{{< callout type="warning" >}}
**There is nothing to install yet.** The engine has no release and no command
to run: its image and root-filesystem packages are being written. This page
says what M1 will let you do, and is updated when it does.
{{< /callout >}}

## Building from source (today)

The engine is a Go module that needs Go 1.27.3 or later:

```sh
git clone https://github.com/go-wasmcontainers/engine
cd engine
go build ./...
```

That compiles the packages; there is no `wctrd` or `wctr` command to start until
the daemon and its API are written.

## The session M1 is written to serve

**Planned for M1 (native, wazero executor).** Once `wctrd` is running and
listening on `/var/run/wctr.sock`, `wctr` finds it by itself; `WCTR_HOST` points
it elsewhere:

```sh
wctr pull <image>        # an OCI image with os=wasip1, arch=wasm
wctr run <image>         # runs the module under wazero
wctr ps                  # lists the containers
wctr logs <container>    # its standard output and error
wctr inspect <container> # its configuration and state
wctr images              # lists the pulled images
wctr rm <container>
wctr rmi <image>
```

Anything else (`wctr exec`, building an image, Compose, published ports) is
**planned, not part of M1**; see the [roadmap]({{< relref "roadmap" >}}).

## Where to follow it

The engine's progress is in
[go-wasmcontainers/engine](https://github.com/go-wasmcontainers/engine); the
page you are reading describes its `main` branch.
