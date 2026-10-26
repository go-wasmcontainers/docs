---
title: "Limitations"
weight: 25
description: "What the engine (v0.8.0) and the playground do not do."
tags: [limits]
---

## Playground

The playground has no stdin, no shell (one `wctr` command per line) and no
networking. Ctrl-C stops waiting for a container but leaves it running. Each
container is bounded: 256 MiB of memory, a 5-minute run, 64 MiB of writes, 4 MiB
of logs, and 64 containers at once.

## Engine

Engine v0.8.0 does **not** have:

- CPU accounting: a WebAssembly guest gives none, so `wctr stats` shows CPU as
  `--`;
- host-path volumes: only named volumes exist, and a host path is refused;

- networking, `exec`, attach and build: no container networking, no `exec`, no
  attaching to a container (so an interactive `run`), and no image build; the
  [roadmap]({{< relref "roadmap" >}}) lists them as planned;
- `tar+zstd` layers: they are not supported yet;
- a distinct quota error: when a container exhausts its storage quota, the
  write surfaces to the module as `EIO`.
