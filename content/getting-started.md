---
title: "Getting started"
weight: 10
description: "Try wctr in your browser, or install wctr and wctrd and run a WebAssembly container."
tags: [start]
---

## In your browser

Open <https://go-wasmcontainers.github.io/playground/> and type
`wctr run --rm hello`. Nothing to install, no server: see the
[Playground]({{< relref "playground" >}}) page.

## Install

Go 1.27.3 or later, no cgo:

```sh
go install github.com/go-wasmcontainers/engine/cmd/wctr@v0.5.0
go install github.com/go-wasmcontainers/engine/cmd/wctrd@v0.5.0
```

Shell completion for `wctr` (bash, zsh, fish or PowerShell), for example in
zsh:

```sh
wctr completion zsh > "${fpath[1]}/_wctr"
```

## A session

This is the output of the engine's acceptance test at v0.1.0
(`TestAcceptance` in `cmd/wctrd`), run against the real binaries in CI; nothing is edited. The
default socket, `/var/run/wctr.sock`, needs write access to `/var/run`, so the
session uses a socket in the home directory:

```console
$ export WCTR_HOST=unix://$HOME/wctr.sock
$ wctrd -H "$WCTR_HOST" &
wctrd v0.1.0 listening on unix:///home/me/wctr.sock (data root /home/me/.local/share/wasmcontainers)
```

With an OCI image layout `./layout` holding three images (`hello`, `greet`,
`probe`):

```console
$ wctr pull oci:./layout:hello
Pulling from oci:/home/me/layout:hello
f0beefb89ff1: Pulling fs layer
f0beefb89ff1: Pull complete
Digest: sha256:da3eead2a4f462a53bebdee2f9dc5fcb94cc5a97ebe1680477f0ce0328ff67f0
Status: Downloaded newer image for hello:latest
$ wctr run --rm hello
Hello from a WebAssembly container!
$ wctr run --rm -e NAME=x greet
Hello, x!
$ wctr run --name failing probe exit 42
bye
$ echo $?
42
$ wctr ps -a
CONTAINER ID   IMAGE   COMMAND                  CREATED        STATUS                               PORTS   NAMES
2105748c99ce   probe   "/bin/probe.wasm exi…"   1 second ago   Exited (42) Less than a second ago           failing
$ wctr logs failing
bye
$ wctr rm failing
failing
$ wctr images
REPOSITORY   TAG      IMAGE ID       CREATED        SIZE
greet        latest   068beee4f451   12 hours ago   2.45MB
hello        latest   85c424cd2beb   12 hours ago   735kB
probe        latest   ae7fb9d7cefb   12 hours ago   811kB
```

Registries work the same way (`wctr pull localhost:5000/lab/hello:v1`, or
Docker Hub names such as `user/app:tag`); `wctr run` pulls an image it does not
have. The full list of commands and flags is on the
[Commands]({{< relref "commands" >}}) page; what v0.1.0 does not do is on
[Limitations]({{< relref "limitations" >}}).
