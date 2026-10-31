---
title: "Shell"
weight: 27
description: "A POSIX shell for WASI containers, built on mvdan.cc/sh and a WASIX subset. Shell v0.2.0 with the coreutils v0.5.0, a public base image; the engine runs it from v0.17.0."
tags: [shell]
---

**A POSIX shell for WASI containers, in pure Go**, built for
`GOOS=wasip1 GOARCH=wasm`:
[go-wasmcontainers/shell](https://github.com/go-wasmcontainers/shell). It is
what makes shell-form commands work: `RUN a && b` in a Containerfile,
`wctr run IMAGE sh -c '…'`, and `wctr exec -it CONTAINER sh`.

A WebAssembly module cannot start another program: Go's port has no fork or
exec, and WASI preview 1 has no process calls. So the shell uses a small subset
of [WASIX](https://wasix.org/), in WASIX's own import module (`wasix_32v1`):
`proc_spawn3`, `proc_join`, `fd_pipe`, `proc_id`, `proc_parent`, `tty_get` and
`tty_set`, and nothing else (no fork, exec, threads or signals). **The engine
serves that subset from v0.17.0** (`wctrd --wasi engine`, the default): see
[Architecture]({{< relref "architecture" >}}).

**Status: shell v0.2.0 is released.** v0.2.0 adds **a line editor on a terminal**
(arrow keys, history, Ctrl-C, Ctrl-D) and makes **`wasix` a module of its own**
(`github.com/go-wasmcontainers/shell/wasix`, tag `wasix/v0.1.0`) with
**`wasix/exec`**, which mirrors `os/exec` for WASI containers. The base image
documented below is `ghcr.io/go-wasmcontainers/shell:0.1.0`. `sh` runs on the engine, where pipelines
through the coreutils work. The base image is
`ghcr.io/go-wasmcontainers/shell:0.1.0` (`wasip1/wasm`, one layer): `/bin/sh`,
`/bin/coreutils.wasm` (the multicall module of
[wasmdesk/coreutils](https://github.com/wasmdesk/coreutils), v0.4.0 in this image,
with 42 tools as symbolic links; coreutils v0.5.0 is released, see below), a root-only `/etc/passwd`, and the config
`Shell ["/bin/sh","-c"]`, which is what a shell-form `RUN` runs. **The image is
public**; pull it with:

```console
$ wctr pull ghcr.io/go-wasmcontainers/shell:0.1.0
```

- **`wasix`**: `//go:wasmimport` bindings for the subset, with a Go API; a
  module of its own since `wasix/v0.1.0`.
- **`wasix/exec`**: `os/exec` for WASI containers (`Command`, `Cmd`, `Start`,
  `Wait`, `Run`, `Output`, `CombinedOutput`, the stdin, stdout and stderr pipes,
  `ExitError`, `LookPath`). It has no `Kill` or `CommandContext`, because signals
  are not in the WASIX subset. `sh` starts its children through it.
- **`sh`**: the mvdan.cc/sh v3.14.1 interpreter (patched with one change: a pipe
  function, since upstream's `os.Pipe` is `ENOSYS` on wasip1), whose external
  commands start with `proc_spawn3`.

`sh` does pipelines, redirections, here-documents, command substitution, `cd`,
exported variables, `$?`, `set -e/-u/-x/-f/-a`, `exec`, `sh -c`, `sh file` and
an interactive prompt on a TTY, with a line editor since v0.2.0. **Coreutils v0.5.0**
(released; the image documented above is `:0.1.0`) brings `grep` flags (`-F -r -l -c -o -q -w -H -h`, combined flags,
POSIX exit 2), a `tee` that writes its files as the input arrives, POSIX
`ls FILE`, `find` expressions, `sort` keys, `head` and `tail` (`-c`, `+N`,
`tail -f`), `date +FORMAT`, `tr` and a full POSIX `printf`, and adds `hostname`,
`uname`, `id`, `whoami`, `clear` and `which`. By design of the subset it has no signals, no
job control and no process substitution.
