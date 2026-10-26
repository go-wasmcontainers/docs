---
title: "Dashboard"
weight: 6
description: "A graphical dashboard for the engine, built with go-widgets, running in the browser."
tags: [dashboard]
---

**A graphical dashboard for the engine — containers, images, volumes and a
`wctr` terminal — built with [go-widgets](https://github.com/go-widgets),
running in the browser on the same engine and API as `wctr`. Every action
shows the `wctr` command it stands for.** This is dashboard v0.2.0, which runs
on engine v0.5.0.

## What it does

- The Start, Stop and Restart buttons use the engine's `/start`, `/stop` and
  `/restart`, and show the matching `wctr start|stop|restart NAME` line in the
  footer and in the terminal. Stop returns once the container has exited.
- The status bar shows the storage, from `GET /system/df` (for example
  `IndexedDB 2MB of 1GB (persistent)`).
- Volumes, live events and stats appear automatically once the engine serves
  them.

Live demo: **<https://go-wasmcontainers.github.io/dashboard/>**. Source:
[go-wasmcontainers/dashboard](https://github.com/go-wasmcontainers/dashboard).

## The terminal widget

The terminal in the dashboard is the `Terminal` widget that go-widgets/toolkit
v0.342.0 adds: a subset of xterm, safe for untrusted output. It has no reply
channel (nothing the output prints is ever sent back), and OSC, DCS and APC
sequences are dropped.

## Limitations

- The Volumes view and the live events appear once the engine serves them; the
  dashboard polls every 2 seconds until the engine serves `/events`.
- The standalone demo keeps its state in memory: nothing persists across a
  reload. Persistence comes when the dashboard becomes the playground's main UI
  (planned).
- Browsers keep Ctrl-W, Ctrl-T and Ctrl-N, so the terminal cannot use them.
- `dashboard.wasm` is 20 MB uncompressed.
- The end-to-end tests have not yet been run in Firefox.

## Next

The dashboard is planned to become the playground's main UI. That has not
happened: the [playground]({{< relref "playground" >}}) is still the terminal
page.
