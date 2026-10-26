---
title: "Performance"
weight: 30
description: "Native benchmarks (engine v0.1.0) and the playground in two browsers. Cold start is slower than wasmtime."
tags: [performance]
---

## Native (engine v0.1.0)

Measured on an Apple M4 Max, one core, median of 3. The benchmarks are
committed in the engine repository.

| Benchmark | What | Result |
|---|---|---|
| pull from a layout | cold pull of the `hello` image (735 kB tar+gzip layer): verify and store | 1.26 ms |
| unpack a gzip layer | apply that layer to an empty tree | 11.5 ms |
| copy-on-write clone | a container's root plus its first write | 2.2 µs |
| cold start | first container of an image: wazero compiles the module and runs it to exit | 610 ms |
| warm start | every later container: compiled module from the per-digest cache | 3.4 ms |

These native numbers were measured without a reference runtime on the machine.

## In the browser (the playground)

One run on a CI `ubuntu-latest` runner, in Chromium and Firefox, with the
native wazero and wasmtime 50 on the same runner for comparison:

| What | Chromium | Firefox | wazero native | wasmtime 50 |
|---|---|---|---|---|
| page ready | 0.46 s | 1.0 s | | |
| first run of `hello` | 81 ms | 291 ms | 825 ms | 23 ms |
| next run | 47 ms | 231 ms | 5.4 ms | |
| one WASI call | 97 µs | 354 µs | | |

**Cold start in the browser is slower than wasmtime** (81 ms against 23 ms in
Chromium, 291 ms in Firefox). These are single runs on one CI machine, not
medians, so they show orders of magnitude rather than precise ratios.
