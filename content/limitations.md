---
title: "Limitations"
weight: 25
description: "What the engine (v0.30.0) and the playground do not do."
tags: [limits]
---

## Playground

The playground (v0.4.1, built on engine v0.30.0) takes one `wctr` command per
line, with no pipes in that line. It has `exec`, a shell (the `shell` image),
typing into `exec -i`, and `run -it` and `start -ai` (its end-to-end test runs
`wctr run --rm -it shell`). Its dashboard is v0.6.0, so GPU panes do not send
input there yet. There is
**no `-p` in the browser** yet: the second origin for containers' pages is
deployed, but the rest of it is in progress and not live. Networks
stay inside the page: no Internet, no published ports. Ctrl-C stops waiting for a container but leaves it running.
Each container is bounded: 256 MiB of memory, a 5-minute run, 64 MiB of writes,
4 MiB of logs, and 64 containers at once.

The default UI, the dashboard, makes the page slower to be ready than the
terminal page (728 ms against 1070 ms in Chromium on the live site), because
`dashboard.wasm` was 20.45 MB raw (5.32 MB gzipped) when measured; dashboard
v0.4.2 made it smaller and no newer figure has been measured. The terminal-only
page is at `?ui=terminal`.

## Engine

Engine v0.30.0 does **not** have:

- CPU accounting: a WebAssembly guest gives none, so `wctr stats` shows CPU as
  `--`;
- host-path volumes: only named volumes exist, and a host path is refused;
- signals: processes are the WASIX subset, a child is stopped only with its
  container, and Ctrl-C in a terminal erases the line (a container's first
  process has its own stdin and terminal since v0.30.0: `run -it`, `attach`);
- **published ports in the browser** (`-p` works natively since v0.22.0; in the
  browser it is in progress, through a second origin, with the groundwork `portmap.Virtual` in v0.24.0; natively there is no SCTP,
  and the source address the guest sees is its own). A container on several networks binds to its
  primary address or to all (`0.0.0.0`), not to a secondary address alone;
  aliases work on in-engine networks only; no IPv6 on in-engine networks; the DNS
  shadows a container's own UDP server on port 53 at the resolver addresses (see
  [Networking]({{< relref "networking" >}}));
- a Kubernetes API, and volume drivers or tmpfs;
- a distinct quota error: when a container exhausts its storage quota, the
  write surfaces to the module as `EIO`.

`wctr build` runs shell-form `RUN` from engine v0.17.0 on the shell base image
(shell v0.1.0, `ghcr.io/go-wasmcontainers/shell:0.1.0`, public). See
[Build]({{< relref "build" >}}) and [Shell]({{< relref "shell" >}}).
