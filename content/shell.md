---
title: "Shell"
weight: 27
description: "A POSIX shell for WASI containers, built on mvdan.cc/sh and a WASIX subset. The guest side exists; the engine's process spawning is in progress."
tags: [shell]
---

**A POSIX shell for WASI containers, in pure Go**, built for
`GOOS=wasip1 GOARCH=wasm`:
[go-wasmcontainers/shell](https://github.com/go-wasmcontainers/shell). It is
what will make shell-form commands work: `RUN a && b` in a Containerfile,
`wctr run IMAGE sh -c '…'`, and `wctr exec -it CONTAINER sh`.

**Status: the guest side exists; the engine side is in progress.** A WebAssembly
module cannot start another program: Go's port has no fork or exec, and WASI
preview 1 has no process calls. So the shell uses a small subset of
[WASIX](https://wasix.org/), in WASIX's own import module (`wasix_32v1`):
`proc_spawn3`, `proc_join`, `fd_pipe`, `proc_id`, `proc_parent`, `tty_get` and
`tty_set`, and nothing else (no fork, exec, threads or signals). A host that
implements the subset is needed: wasmer, which the tests use, or this engine
once its host side is done. **Nothing here runs in `wctr` yet**, and `wctr exec`
does not exist yet; a base image is also in progress.

- **`wasix`**: `//go:wasmimport` bindings for the subset, with a Go API.
- **`sh`**: the mvdan.cc/sh v3.14.1 interpreter (patched with one change: a pipe
  function, since upstream's `os.Pipe` is `ENOSYS` on wasip1), whose external
  commands start with `proc_spawn3`.

`sh` does pipelines, redirections, here-documents, command substitution, `cd`,
exported variables, `$?`, `set -e/-u/-x/-f/-a`, `exec`, `sh -c`, `sh file` and
an interactive prompt on a TTY. By design of the subset it has no signals, no
job control and no process substitution.
