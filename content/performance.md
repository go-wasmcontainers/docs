---
title: "Performance"
weight: 30
description: "Engine v0.1.0 benchmarks, on one core. Not yet compared to a reference runtime."
tags: [performance]
---

Measured on an Apple M4 Max, one core, median of 3. The benchmarks are
committed in the engine repository.

| Benchmark | What | Result |
|---|---|---|
| pull from a layout | cold pull of the `hello` image (735 kB tar+gzip layer): verify and store | 1.26 ms |
| unpack a gzip layer | apply that layer to an empty tree | 11.5 ms |
| copy-on-write clone | a container's root plus its first write | 2.2 µs |
| cold start | first container of an image: wazero compiles the module and runs it to exit | 610 ms |
| warm start | every later container: compiled module from the per-digest cache | 3.4 ms |

**These numbers are not yet compared to a reference runtime.** No reference
runtime was installed on the measuring machine; the comparison is an open item.
