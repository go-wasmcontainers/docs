---
title: "Limitations"
weight: 25
description: "What engine v0.1.0 does not do."
tags: [limits]
---

Engine v0.1.0 (milestone M1) does **not** have:

- networking, `exec`, attach and build: no container networking, no `exec`, no
  attaching to a container (so an interactive `run`), and no image build; the
  [roadmap]({{< relref "roadmap" >}}) lists them as planned;
- `tar+zstd` layers: they are not supported yet;
- containers that survive a `wctrd` restart: containers live in the daemon's
  memory and are lost when it restarts (images are kept);
- a distinct quota error: when a container exhausts its storage quota, the
  write surfaces to the module as `EIO`.
