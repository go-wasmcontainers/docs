---
title: "Dashboard"
weight: 6
description: "A graphical dashboard for the engine, built with go-widgets, running in the browser. Dashboard v0.8.0, with surface input, Open shell and a Surfaces view for GPU frames."
tags: [dashboard]
---

**A graphical dashboard for the engine — containers, images, volumes, networks
and a `wctr` terminal — built with [go-widgets](https://github.com/go-widgets),
running in the browser on the same engine and API as `wctr`. Every action
shows the `wctr` command it stands for.** This is dashboard v0.8.0.

## What it does

- The Start, Stop and Restart buttons use the engine's `/start`, `/stop` and
  `/restart`, and show the matching `wctr start|stop|restart NAME` line in the
  footer and in the terminal. Stop returns once the container has exited.
- The status bar shows the storage, from `GET /system/df` (for example
  `IndexedDB 2MB of 1GB (persistent)`).
- It follows `GET /events` instead of polling. When the stream drops, it
  refreshes and reconnects from the last event.
- The Volumes view shows each volume's size and use count, and can create,
  inspect, remove (with force) and prune; each action shows its
  `wctr volume …` line.
- A container's detail shows its mounts and a Stats tab: a memory gauge against
  the limit, a sparkline of recent readings, bytes written, log size and pids.
  CPU is shown as `--` because it is not measured.
- The run dialog accepts `-v NAME:/path[:ro]`.
- **Networks view and Join class** (v0.5.0): the networks, the networks of each
  container, and a **Join class…** dialog for a teacher's
  [relay]({{< relref "relay" >}}).
- **Surfaces view** (v0.6.0): one pane per GPU surface a container opens
  (`wasi-gfx:surface`, rendered by the [gpu]({{< relref "gpu" >}}) module in the
  engine's worker), titled with the container and the surface number, with its
  size and frame rate. A closed surface keeps its last frame, greyed, until you
  dismiss it.
- **Surface input** (v0.8.0): a pane forwards pointer events (down, up, move, in
  surface pixels, scaled from the letterboxed frame; events outside the frame are
  dropped) and key events (the `KeyboardEvent.code`, the text when it is one
  character, the modifiers) to the guest, through gpu v0.6.0's
  `webgpu.SurfaceInput`. A press on a pane's frame gives it the focus (drawn in the
  accent colour); only the focused pane sends keys, and it takes every key except
  F6, which leaves it. Pointer moves are coalesced to one per surface per frame. A
  closed surface takes no input. The playground (v0.4.1) still uses dashboard v0.6.0.
- **Open shell** (v0.7.0): an "Open shell" toolbar action and a Shell tab in a
  container's detail, the equivalent of `wctr exec -it CONTAINER sh`. It starts a
  TTY exec of `/bin/sh` at the terminal's size, sends the keys as raw bytes,
  follows resizes, and shows the exit code when the shell ends. Request bodies of
  unknown length are streamed over the bridge, which carries the shell's stdin.
  The container's image needs a `/bin/sh`.
- It was tested live in Chromium and Firefox.

Live demo: **<https://go-wasmcontainers.github.io/dashboard/>**. Source:
[go-wasmcontainers/dashboard](https://github.com/go-wasmcontainers/dashboard).

## The terminal widget

The terminal in the dashboard is the `Terminal` widget that go-widgets/toolkit
v0.342.0 adds: a subset of xterm, safe for untrusted output. It has no reply
channel (nothing the output prints is ever sent back), and OSC, DCS and APC
sequences are dropped.

## Limitations

- The standalone demo keeps its state in memory: nothing persists across a
  reload. In the [playground]({{< relref "playground" >}}) the dashboard runs
  on the persistent engine worker instead.
- Browsers keep Ctrl-W, Ctrl-T and Ctrl-N, so the terminal cannot use them.
- `dashboard.wasm` was 20.45 MB raw and 5.32 MB gzipped when measured. Dashboard
  v0.4.2 made it smaller (go-widgets toolkit v0.345.1), and **no newer figure has
  been measured**.

## In the playground

The dashboard is the [playground]({{< relref "playground" >}})'s default UI since
playground v0.3.0 (Containers, Images, Volumes, Networks, Terminal). The
terminal-only page stays at `?ui=terminal`.
