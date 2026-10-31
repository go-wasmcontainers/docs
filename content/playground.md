---
title: "Playground"
weight: 5
description: "A dashboard of WASI containers with a wctr terminal, in a browser tab, with no server. Playground v0.5.0: published ports (-p) in a tab of their own, run -it, GPU surfaces, a shell, WASIX, networking, Join class, wasi-nn."
tags: [playground]
---

**<https://go-wasmcontainers.github.io/playground/>** is a dashboard of WASI
containers, images and volumes with a terminal running `wctr`, in a web page.
There is no server: the engine runs in the page (the browser executor,
`runtime/browser`, new in engine v0.2.0). This is **playground v0.5.0**, on engine
v0.30.0.

## What is new in v0.5.0

- **`-p` in the browser**: `wctr run -d -p 8080:80 net serve`, then **open :8080**
  in the bar above the terminal. The container's pages open in a tab of their own,
  on the usercontent origin (see [Published ports](#published-ports--p)).
- **Every pkgx image comes from the signed catalogue**, and Pages rebuilds daily, so
  new pkgx bottles appear without a release.

## What was new in v0.4.1

- **Engine v0.30.0**: `run -i/-t`, `attach` and `start -ai` in the page. A line
  whose own `wctr` flags say `-i` (`run`, `start`, `exec`; also `-dit`) gets what
  you type; the end-to-end test runs `wctr run --rm -it shell` and `exec -it c sh`
  and checks the exit status.
- **The usercontent shell forgets requests the playground never answers**
  (`shell.js` and `sw.js`): a pending-request leak fixed.

## What was new in v0.4.0

- **GPU surfaces and compute**: the `compute` and `triangle` images (copies of
  [gpu]({{< relref "gpu" >}}) v0.5.1's examples) run with `--gpu`; a surface is a
  pane under the dashboard's **Surfaces** view. A container without `--gpu`, or a
  browser without WebGPU, gets no adapter: these images print `no GPU adapter` and
  exit 2.
- **WASIX**: programs started by a shell run as WASIX processes in the page.
- **The `shell` image**, `ghcr.io/go-wasmcontainers/shell:0.1.0`, mirrored verbatim
  and pinned by its digest: `wctr run --rm shell sh -c 'echo a | cat'` and
  `wctr exec -it CONTAINER sh` work in the page.
- **Typing into `exec -i`**: a line whose own `wctr` flags say `-i` gets what you
  type, Ctrl-C and Ctrl-D included.
- **`wctr build` and `wctr compose`** are wired into the page's `wctr`, as in
  `cmd/wctr`.
- **The shell of a second origin** for containers' pages
  (<https://go-wasmcontainers-usercontent.github.io/>) and the route policy: the
  first half of `-p` in the browser. The second half came in v0.5.0.

## What was new in v0.3.0

- **The dashboard is the main UI**: the [dashboard]({{< relref "dashboard" >}})
  (Containers, Images, Volumes, Networks, Terminal) runs on the persistent engine
  worker. The terminal alone, on xterm.js, stays at `?ui=terminal`.
- **Networking in the browser**: containers talk over TCP and UDP, by name (see
  [Networking]({{< relref "networking" >}})). The `net` image has `serve`, `get`,
  `bench`, `udp-echo`, `udp`, `lookup` and `dial`.
- **Join class**: with a teacher's [relay]({{< relref "relay" >}}), the page joins
  the class network (the dashboard's Networks view, **Join class…**). The relay
  must run with `--cors-origin https://go-wasmcontainers.github.io` (relay v0.3.0 or
  later). The join code is typed in a password field, posted once to the engine
  worker, and never stored.
- **wasi-nn in the browser**: the `mnist` image reads a handwritten digit (see
  [GPU]({{< relref "gpu" >}})).
- It still works offline and keeps its images and containers across a reload.

## Published ports (`-p`)

```console
$ wctr run -d -p 8080:80 net serve
```

The bar above the terminal then shows **open :8080**, which opens the
container's web pages in a tab of their own. They are served on another site,
<https://go-wasmcontainers-usercontent.github.io/>, never on the playground's
origin: a container's page there could otherwise read the playground's storage and
script it.

- The engine publishes the port in memory, and gives each opened port a
  **session**: a random id that names its pages (`/s/<session>/8080/`), bound to
  that one port.
- The view tab relays each request of the container's page to the playground tab,
  which runs the engine, and the answer back. Nothing leaves the browser.
- **Cookies work, but not as you might expect.** The engine keeps one cookie jar
  per session. The container's `Set-Cookie` never reaches the browser, and
  `document.cookie` in its page does not see them.
- **Closing the view tab ends the session.** The playground tab must stay open, as
  it runs the engine.

Measured in Chromium: 8 MiB from a container to its page, through every hop, in
32 ms (about 265 MB/s).

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
`start`, `stop`, `restart`, `diff`, `stats`, `events`, `volume`, `network` and
`system df|prune`, plus `help` and `clear`. `run` takes `--rm`, `-d`, `--name`,
`-e`, `-w`, `-m`, `--entrypoint`, `-v NAME:/path[:ro]`, `--network NAME` and
`-p HOST:CONTAINER` (see [Published ports](#published-ports--p)). Exit
codes are the native engine's: the container's own, or 125, 126, 127, 134 and 137;
see [Commands]({{< relref "commands" >}}).

```console
$ wctr run --rm hello
$ wctr run --rm -e NAME=you greet
```

## Images

| Image | What it is |
|---|---|
| `hello`, `greet` | the small demonstration modules |
| `lab` | a module with subcommands: `echo`, `env`, `pwd`, `ls`, `cat`, `write`, `count`, `sleep`, `spin`, `hog`, `bench`, `fs`, `exit`, `esc` |
| `net` | a networking probe: `serve`, `get`, `bench`, `udp-echo`, `udp`, `lookup`, `dial` |
| `mnist` | classifies a handwritten digit with wasi-nn |
| every [pkgx](https://github.com/go-pkgx) `wasip1/wasm` bottle: 43 images from 41 bottles today (`yq`, `gojq`, `jj`, `shfmt`, `actionlint`, `age`, `lua`, `luac`, `sqlite3`, `zstd`, `simdjson-minify` and more; `wctr images` lists them all) | derived from pkgx's signed catalogue at build time, see below |

### pkgx images are derived, not listed

The site build (`cmd/site build --pkgx`) reads pkgx's signed `wasip1/wasm`
catalogue (`go-pkgx.dev/catalog`; the signature is verified) and mirrors the
newest version of every project that has one published. No list is typed in the
playground: a bottle published tomorrow is an image after the next build, and
`wctr images` and the dashboard's Images view show whatever `catalog.json`
holds, each entry with its project, version and layer size.

- **Version**: the highest by numeric components, then rebuild counter, then
  raw string; the catalogue's order is not trusted.
- **Names**: one image per `bin/<name>.wasm` in the bottle (`lua` gives `lua`
  and `luac`), at most 8 per bottle; all of a bottle's images share its layer.
  The catalogue's `provides` is only a hint (on wasip1/wasm it is copied from
  the host recipe and wrong both ways): a name without a module and a module it
  does not list are reported as findings. The image's entrypoint is the bare
  binary name, so argv[0] is `lua`. A name taken by two projects goes to the one
  whose last path segment equals it, else the smallest project name; the other is
  published as `<project-with-dashes>-<name>`.
- **Dependencies are ignored**: a `wasip1` module is statically linked.
- **Limits**: 400 projects, 900 images, 8 images per bottle, 100 MiB per
  compressed layer, 512 MiB and 20 000 entries unpacked, 3 minutes for the
  catalogue and 30 in all. An unverifiable or empty catalogue stops the build;
  there is no built-in list. A bottle over a limit, or whose tar has an absolute
  path, a `..` or a link leaving the root, is refused and listed in
  `findings.json`.
- **Checked**: each `catalog.json` entry has `checked`: `true` when the build ran
  the image under the engine and under wasmtime (probes with real input for
  `yq`, `gojq`, `jj`, `tomljson` and others) and nothing failed or differed
  (40 of 43 today), `false` when it is only mirrored.
- **Site size**: GitHub Pages refuses a site over 1 GB. With 43 images the mirror
  is 112 MB (about 2.6 MB per image, each image stored separately) and the whole
  site 167 MB. The build stops if the layers exceed 700 MiB; the plan beyond that
  is to drop the largest, then to shard across a second Pages site.
- **Page cost**: one `catalog.json` of about 330 bytes per image, read once;
  an image's blobs are fetched only when a command uses it.
- **Smoke test**: CI runs every derived image with `--version` and `--help` (or a
  named probe such as `lua -v`) under the engine and under wasmtime and compares
  exit status, stdout and stderr byte for byte; a failure or difference that is
  not named, with its reason, fails the job.

Lua's errors, which use WebAssembly exception handling, work in Chromium,
Firefox and natively. The first `lua` run took 69–108 ms in Chromium and 170 ms
in Firefox, measured in CI.

Images and pulls survive a reload: they are kept in the browser's IndexedDB.
So do the containers (engine v0.4.0): state, exit code, logs and written files.
A container that was running comes back exited with code 137, and a `--rm` one
is removed.

## Offline and updates

The playground works offline (`dashboard.wasm` is in the service-worker cache). A
new version installs in the background and applies when you choose "Reload to
update". Only one tab runs the engine at a time; another tab can take over with
"Use it here". The engine was ready in 130–190 ms on a first visit in Chromium,
80–115 ms after a reload, and 75 ms offline.

## Limits per container

256 MiB of memory, a 5-minute run, 64 MiB of writes, 4 MiB of logs, and 64
containers at once.

## Not yet

One `wctr` command per line (pipes need the `shell` image's `sh`); the dashboard in the
page is v0.6.0, so GPU panes send no input yet; published ports are HTTP only, shown
in a tab, with no UDP and no raw TCP from outside the page. Networks stay inside the page: there is no route to the Internet. Ctrl-C
stops waiting for a container but leaves it running. See
[Limitations]({{< relref "limitations" >}}) and the [roadmap]({{< relref "roadmap" >}}).

## Speed

Measured in one CI run, the page is ready in 0.46 s in Chromium and 1.0 s in
Firefox, and the first `hello` runs in 81 ms and 291 ms. **Cold start is slower
than wasmtime** (23 ms natively). The table is on
[Performance]({{< relref "performance" >}}).
