---
title: "Playground"
weight: 5
description: "wctr in a browser tab: pull and run WASI containers with no server."
tags: [playground]
---

**<https://go-wasmcontainers.github.io/playground/>** is a terminal in a web
page running `wctr` against WASI containers. There is no server: the engine
runs in the page (the browser executor, `runtime/browser`, new in engine
v0.2.0). This is playground v0.1.0.

## What you can type

`wctr pull`, `run`, `ps`, `logs`, `rm`, `rmi`, `images`, `inspect` and `kill`,
plus `help` and `clear`. `run` takes `--rm`, `-d`, `--name`, `-e`, `-w`, `-m`
and `--entrypoint`. Exit codes are the native engine's: the container's own,
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

Images and pulls survive a reload: they are kept in the browser's IndexedDB.

## Limits per container

256 MiB of memory, a 5-minute run, 64 MiB of writes, 4 MiB of logs, and 64
containers at once.

## Not yet

No stdin, no shell, no networking. Ctrl-C stops waiting for a container but
leaves it running. See [Limitations]({{< relref "limitations" >}}) and the
[roadmap]({{< relref "roadmap" >}}).

## Speed

Measured in one CI run, the page is ready in 0.46 s in Chromium and 1.0 s in
Firefox, and the first `hello` runs in 81 ms and 291 ms. **Cold start is slower
than wasmtime** (23 ms natively). The table is on
[Performance]({{< relref "performance" >}}).
