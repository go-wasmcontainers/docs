---
title: "Build"
weight: 24
description: "wctr build: OCI images from a Containerfile or Dockerfile, for WASI. Build v0.2.0; shell-form RUN is in progress."
tags: [build]
---

**`wctr build` builds OCI images from a Containerfile (or a Dockerfile, same
syntax) for the engine.** It is the separate module
[go-wasmcontainers/build](https://github.com/go-wasmcontainers/build) (v0.2.0),
which drives the engine through its API, the way BuildKit sits beside a
container daemon: each step runs in a container made from the previous step's
image, which is then committed. Pure Go, so it also runs in the browser.

**Until it is wired into `wctr`, the command is the standalone `wctr-build`.**
It needs a `wctrd` of v0.12.0 or later.

```console
$ go install github.com/go-wasmcontainers/build/cmd/wctr-build@v0.2.0
$ wctr-build -H "$WCTR_HOST" -t lab/app:v1 .
$ wctr run --rm lab/app:v1
```

## Flags

`wctr build [OPTIONS] PATH | -`

| Flag | |
|---|---|
| `-t, --tag NAME[:TAG]` | name the image (repeatable) |
| `-f, --file FILE` | the Containerfile (`-`: standard input); default `PATH/Containerfile`, else `PATH/Dockerfile` |
| `--build-arg K=V` | a build argument (`K` alone: from the environment) |
| `--target STAGE` | build up to this stage |
| `--no-cache` | run every step |
| `--progress plain` | plain progress, the only kind |
| `-q, --quiet` | print only the image ID |

`PATH` is the build context, sent without what its `.containerignore` (else
`.dockerignore`) excludes; `-` reads the context as a tar on standard input.
Multi-stage builds work.

## What works, and what does not yet

- **Exec-form `RUN ["prog", "arg"]` works now**: it runs a WebAssembly program in
  the step's container.
- **Shell-form `RUN cmd` is in progress.** It needs an image with a shell and an
  engine that spawns processes, which the [shell]({{< relref "shell" >}})
  repository is building. Until then the build stops with a precise error.
- **Not supported, each a precise error and never ignored**: here-documents,
  `RUN --mount`, `ADD` from a URL or Git, `COPY --parents/--exclude`, `--chown`
  by name, symbolic `--chmod`, and xz/zstd archives in `ADD`.

## Cache

A step is reused when its parent image, the configuration it produces, the
instruction and the digest of its inputs are the same. The cache is kept in
`$XDG_CACHE_HOME/wasmcontainers/build-cache.json` (in memory in a browser).

## Speed

A seven-instruction build with one `RUN` took 61.7 ms cold and 4.8 ms with every
step cached. **Measured under heavy load (a load average of 66), so indicative
only**, and not yet compared with BuildKit.
