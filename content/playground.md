---
title: "Playground"
weight: 5
description: "wctr in a browser tab: pull and run WASI containers with no server."
tags: [playground]
---

**<https://go-wasmcontainers.github.io/playground/>** is a terminal in a web
page running `wctr` against WASI containers. There is no server: the engine
runs in the page (the browser executor, `runtime/browser`, new in engine
v0.2.0). This is **playground v0.2.0**: completion and line editing (below),
persistence and offline use, the new commands and the images `lua`, `sqlite3`
and `minigzip`, on engine v0.9.0.

## The dashboard is the default UI

On the live site, the playground now opens the [dashboard]({{< relref "dashboard" >}})
(Containers, Images, Volumes, Terminal), running on the persistent engine worker.
The terminal-only page stays at `?ui=terminal`. It works offline: `dashboard.wasm`
is in the service-worker cache. This is deployed on the playground's main branch
but is not in a playground tag yet (the latest tag is v0.2.0). The terminal
sections below describe the terminal page.

## Completion and editing

The terminal completes with Tab, as a shell does: subcommands, flags, `-m`
sizes, local images (for `run` and `rmi`), the catalogue of images to pull,
container names and IDs (`logs`, `rm`, `inspect`, and only running containers
for `kill`) and image arguments. Editing is bash-style: Up and Down for the
history, Ctrl-R to search it, Ctrl-A, Ctrl-E, Ctrl-B and Ctrl-F to move,
Ctrl-U, Ctrl-K and Ctrl-W to cut, Ctrl-D, and Ctrl-L to clear. The history is
kept per tab (in `sessionStorage`). From a keypress to the line on screen took
7 to 14 ms in Chromium and 23 ms in Firefox, measured in CI.

The line editor and its completion are the package `cli/lineedit` of the
engine: pure Go, usable by anyone who embeds the engine's CLI.

## What you can type

`wctr pull`, `run`, `ps`, `logs`, `rm`, `rmi`, `images`, `inspect` and `kill`,
and, with engine v0.9.0, `start`, `stop`, `restart`, `diff`, `stats`, `events`,
`volume` and `system df|prune`, plus `help` and `clear`. `run` takes `--rm`,
`-d`, `--name`, `-e`, `-w`, `-m`, `--entrypoint` and `-v NAME:/path[:ro]`. Exit codes are the native engine's: the container's own,
or 125, 126, 127, 134 and 137; see [Commands]({{< relref "commands" >}}).

```console
$ wctr run --rm hello
$ wctr run --rm -e NAME=you greet
```

## Images

| Image | What it is |
|---|---|
| `hello`, `greet` | the small demonstration modules |
| `lab` | a module with subcommands: `echo`, `env`, `pwd`, `ls`, `cat`, `write`, `count`, `sleep`, `spin`, `hog`, `bench`, `fs`, `exit`, `esc` |
| `age:1.3.2`, `go-md2man:2.0.7`, `shfmt:3.14.1` | [pkgx](https://github.com/go-pkgx) `wasip1` bottles |
| `lua` 5.4.9, `sqlite3` 3.53.4, `minigzip` (zlib 1.3.2) | C tools built by [go-pkgx](https://github.com/go-pkgx) for wasip1, live with engine v0.9.0 |

Lua's errors, which use WebAssembly exception handling, work in Chromium,
Firefox and natively. The first `lua` run took 69–108 ms in Chromium and 170 ms
in Firefox, measured in CI.

Images and pulls survive a reload: they are kept in the browser's IndexedDB.
So do the containers (engine v0.4.0): state, exit code, logs and written files.
A container that was running comes back exited with code 137, and a `--rm` one
is removed.

## Offline and updates

The playground works offline. A new version installs in the background and
applies when you choose "Reload to update". Only one tab runs the engine at a
time; another tab can take over with "Use it here". The engine was ready in
130–190 ms on a first visit in Chromium, 80–115 ms after a reload, and 75 ms
offline.

## Limits per container

256 MiB of memory, a 5-minute run, 64 MiB of writes, 4 MiB of logs, and 64
containers at once.

## Not yet

No stdin, no shell, and **no networking: engine v0.10.0's networking is not live
in the playground yet** (the next milestone). Ctrl-C stops waiting for a container but
leaves it running. See [Limitations]({{< relref "limitations" >}}) and the
[roadmap]({{< relref "roadmap" >}}).

## Speed

Measured in one CI run, the page is ready in 0.46 s in Chromium and 1.0 s in
Firefox, and the first `hello` runs in 81 ms and 291 ms. **Cold start is slower
than wasmtime** (23 ms natively). The table is on
[Performance]({{< relref "performance" >}}).
