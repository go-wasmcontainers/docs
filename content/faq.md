---
title: "FAQ"
weight: 35
description: "Why the command is wctr, and not docker or ctr."
tags: [faq]
---

## Why `wctr`?

`wctr` is the command line and `wctrd` the daemon that serves it. The name
follows the convention of the container tools that came before it, `ctr`,
`nerdctl` and `crictl`: a short name ending in `ctr`, for container. The `w` is
for **wasm**: `wctr` reads as *wasm container*, and says that this is the
WebAssembly variant.

### Not `docker`

Docker is a product and a trademark of another organisation. This engine is
not Docker and does not ship under its name: it is **compatible with the Docker
Engine API**, which is a statement of compatibility and nothing more. That is
why its socket is `/var/run/wctr.sock` and the address variable is
`WCTR_HOST`.

### Not `ctr`

`ctr` is the command line that containerd ships. Trainees meet it on
Kubernetes nodes, where it means something different, so a lab teaching a
`ctr` of its own would teach the wrong tool in the place they will use it.

## Does anything work with existing Docker clients?

The API is meant to be compatible with the Docker Engine API (v1.47 is the
reference the code is written against), so a client of that API can be pointed at
`/var/run/wctr.sock`. Only the subset M1 needs is planned first; see the
[roadmap]({{< relref "roadmap" >}}).
