---
title: "GPU"
weight: 28
description: "wasi-nn on the CPU and on WebGPU, wasi:webgpu compute and rendering to surfaces, for WASI containers. gpu v0.9.0, with surface input, the batched command stream, loud WGSL errors, and a language model in a container."
tags: [gpu]
---

[go-wasmcontainers/gpu](https://github.com/go-wasmcontainers/gpu) is the module
for GPU access from WASI containers: wasi-nn inference, wasi:webgpu compute and
rendering, on WebGPU in the browser and on the CPU natively. Pure Go. The design
and its references are in
[docs/DESIGN.md](https://github.com/go-wasmcontainers/gpu/blob/main/docs/DESIGN.md).

**Status (v0.7.0): inference, compute, rendering, surface input and the batched command stream exist.**

| Part | State |
|---|---|
| `onnx`: ONNX model loader and CPU operators | done (v0.1.0) |
| `nn`: wasi-nn host module (`wasi_ephemeral_nn`), CPU backend; `wasinn` Go guest bindings; the `mnist` example | done (v0.2.0) |
| `nn/webgpu`: wasi-nn on the GPU, WGSL kernels (browser) | done (v0.3.0) |
| wasi:webgpu compute (host and Go guest), `examples/compute` | done (v0.4.0) |
| rendering: `wasi-gfx:surface` and `surface-webgpu`, frames to the page, `examples/triangle` | done (v0.5.0); the dashboard (v0.6.0) shows the frames in its Surfaces view |
| surface input: `on-pointer-up/down/move` and `on-key-up/down` streams for the guest, `webgpu.SurfaceInput` for the page | done (v0.6.0); the dashboard forwards pane events since its v0.8.0 |
| `gpubatch` and `wasigpu/batchsink`: the batched command stream (`wasi_webgpu_batch`), one host call per submit, with read-back; `examples/batchcompute` | done (v0.7.0), see [Batched commands](#batched-commands-wasi_webgpu_batch) |
| GPU time for the batched stream: fair scheduling, a per-submit budget with a sanction (revoked grant, `gpu-sanction` event), a compute quota | done (v0.10.0), plus a loop budget that cuts runaway shader loops (v0.11.0) and refuses what it cannot bound (v0.12.0), see [GPU time](#gpu-time-scheduling-budget-sanction-quota) |
| `on-resize` and `on-frame` streams, vertex buffers, depth-stencil | not yet |

A container asks for the GPU with `wctr run --gpu` (engine v0.13.0 or later);
`--gpu-memory=SIZE` (engine v0.32.0) sets the GPU memory quota of its batched
contexts, 256 MiB by default and at most 1.5 GiB. `--gpu-memory` without `--gpu`
is refused. The
execution target is checked: `gpu` needs that capability **and** a GPU backend,
and otherwise fails with `unsupported_operation`, never running on the CPU in its
place. A process without `--gpu` gets no adapter.

## wasi-nn, on the CPU and on WebGPU

`wasinn` gives Go guests `Load`, `NewContext`, `SetInputF32`, `Compute` and
`OutputF32`. In the browser, each ONNX node becomes one WGSL compute dispatch on a
WebGPU device of the graph's own. The WGSL kernels are checked twice: natively,
against all 122 vendored ONNX node tests, with each kernel's index arithmetic run
in Go (no GPU needed); and in a browser, on the real WGSL. **On an Apple Metal
adapter (Chromium 143), 123 of 123 pass, MNIST-12 within 3.1e-5 of the
reference.** CI has a macOS lane that must get a GPU adapter or fail, so the GPU
path is proven on a real Apple GPU; the Linux lane runs on SwiftShader, a
software adapter, and is labelled as proving the WGSL and not a GPU.

The [playground]({{< relref "playground" >}}) has an `mnist` image that runs
wasi-nn in the browser.

## wasi:webgpu compute

Containers are WASI preview 1 core modules and the engine has no component-model
runtime, so the gpu module parses the WIT and implements the Canonical ABI for a
32-bit memory by hand. The host serves every function of the `wasi:webgpu`
interface; in the browser it maps them to WebGPU in the engine's worker. Quotas
per process: 4096 handles, 256 MiB of buffers, 1 MiB per WGSL module and 256
pipelines; a submission still running after 2 seconds has its device destroyed
(the `wasi_webgpu_batch` module below has the scheduled, budgeted version),
and fallback adapters are refused.

On an Apple Metal adapter, `examples/compute` (adding two 1M-float vectors,
multiplying two 256×256 matrices) matches the CPU on every element, with the
matmul in 15 ms on the GPU against 42 ms on the guest's CPU, and the vector add in
69 ms against 3.5 ms (dominated by transfers). That is one run, not a benchmark.

## Batched commands: `wasi_webgpu_batch`

`wasi:webgpu` costs one host call per WebGPU call. The batched path costs one per
submit: a Go guest (`wasigpu/batchsink`, `GOOS=wasip1`) records
[go-gfx/webgpu](https://github.com/go-gfx/webgpu) work with a remote device, the
sink packs creations, destructions and the recorded stream into one frame, and
the host module `wasi_webgpu_batch` (`gpubatch`) replays it on the container's
WebGPU device and fills a read window before returning (`caps`,
`context_create`, `submit`, `context_destroy`). It is the host ABI for running
a small language model inside a container; the
[design](https://github.com/go-wasmcontainers/gpu/blob/docs/llm-design/docs/llm-design.md)
has the plan.

The guest is hostile and the replay is the trust boundary: the frame is copied
out of guest memory before it is decoded; guest ids are private to a context
and remapped; descriptors, streams and buffer ranges (writes, copies, indirect
arguments, dispatch sizes) are checked before WebGPU sees them; a refused frame
loses its context and releases what it held; guest memory is written only
inside the read window; GPU time is scheduled and budgeted (see
[GPU time](#gpu-time-scheduling-budget-sanction-quota)).
Without `--gpu` every call answers *denied*; with no GPU backend, or only a
software adapter, *unsupported*. A container has at most 4 contexts and one
GPU memory quota, `--gpu-memory` (256 MiB, at most 1.5 GiB).

[`examples/batchcompute`](https://github.com/go-wasmcontainers/gpu/tree/main/examples/batchcompute)
records a compute shader that writes known values and reads 1 MiB and 4 KiB
back, one crossing each. **Run as real containers in the engine's browser
executor on Chromium and the Apple GPU (no fallback adapter), every element
matches**, and the refusals are checked there: no `--gpu`, an out-of-range
write (refused by the host before WebGPU), a read outside the window, an
oversized, truncated or bad-magic frame, an allocation past the quota, a
smaller `--gpu-memory`, and a software adapter. Measured from the guest on an
Apple M4 Max at a load below 8 (medians of 30 interleaved rounds): a read-back
submit takes 315 to 375 µs at 4 KiB and 895 to 1 105 µs at 1 MiB, against 124
and 155 µs host-only.

## GPU time: scheduling, budget, sanction, quota

WebGPU has no way to cancel or preempt work already submitted (the spec's
`GPUDevice.destroy()` loses the device and fails outstanding asynchronous
operations; its denial-of-service section only allows a browser watchdog, and
Chromium's kills the whole GPU process). So a container whose shader runs for
seconds keeps the GPU busy after the host destroys its device, and the next
container queues behind it (go-wasmcontainers/gpu#26). The module (gpu v0.10.0)
does what a host can do about it:

- **Fair scheduling.** One submit is in flight in the whole module; waiting
  submits of different containers are served round-robin. A submit's clock
  starts when it gets its turn, not when it was queued.
- **A budget and a sanction.** A frame that only streams commands has
  **500 ms**; a frame that creates objects (shader and pipeline compilation) has
  the **2 s** hard cap. On an overrun the device is destroyed and the
  container's GPU grant is **revoked for the rest of its life**: the submit
  answers *timeout* (errno 8) and every later call except `context_destroy`
  *revoked* (errno 12). The engine (v0.34.0) publishes the event `gpu-sanction`
  (`wctr events`) and `wctr inspect` shows it as `State.Sanction`. The host then
  waits for the destroyed device's work to leave the GPU before it serves the
  next container, so an innocent container is never charged for it. A container
  can stall the GPU at most once. The longest legitimate submit measured is
  226 ms (Firefox 159, measured when its round trip had a 200 ms floor, see
  the Firefox read-back note in [A language model in a container](#a-language-model-in-a-container)) and 43 ms (Chromium 151, the language
  model); that is why the default is 500 ms and not 250 ms.
- **A compute quota.** Before anything is replayed, a dispatch is refused above
  2^26 invocations and a stream above 2^30 (workgroup counts times the device's
  workgroup bound; go-gfx/webgpu v0.11.0, fuzzed).

- **A loop budget (gpu v0.11.0, nothing unbounded runs since v0.12.0).** What the layers
  above cannot bound is the **first** stall: the victim waits for the hostile
  shader's remaining length. So the host rewrites every guest compute shader
  before it is compiled (go-gfx/wgsl `transform.LoopBudget`, parse only). Each
  loop stops after `Work / invocations` iterations of its dispatch (2^36 work,
  between 1024 and 2^24), with one counter per loop depth, and sets a flag the
  host reads after the stream. A flagged submit answers *truncated* (errno 14,
  the partial result is never returned) and the container is revoked like for a
  timeout; the victim's worst latency fell from 4,574 ms to 41 ms (Chromium 151)
  and from 2,176 ms to 278 ms (Firefox 159, whose own floor is 200 ms), 22 to
  69 ms on Safari 27.2. A module the host cannot bound (WGSL its parser does
  not accept, a loop in a vertex or fragment module, a rewrite the browser
  refuses) is refused with *unbounded* (errno 15), never run as written; the
  `wasi:webgpu` module and the wasi-nn GPU kernels go through the same guard.
  Bind group 3 is the host's: a guest that uses it gets *reserved* (errno 13).
  A submit that never completes because the browser destroyed another
  container's device (Safari's own watchdog does, after about 3 s) is not an
  overrun: the device is reset, the guest answers *context lost* and nobody is
  sanctioned. SmolLM2-135M and Qwen2.5-0.5B run with zero refusals and zero
  truncations; decode costs about 9 % of its tokens per second. The note,
[docs/gpu-scheduling.md](https://github.com/go-wasmcontainers/gpu/blob/main/docs/gpu-scheduling.md),
has the threat model (a student's own tab, or, natively, users sharing a GPU),
the sources and the measurements.

## Rendering: surfaces

In the browser each `wasi-gfx:surface` is an `OffscreenCanvas` in the engine's
worker; `present` turns it into an `ImageBitmap` handed to the page, which draws it
(in the playground, a dashboard pane). The page acknowledges each frame: at most two
are unacknowledged, a page that stops acknowledging slows the guest to a frame per
second without stopping it, frames are paced to 60 per second, and surfaces are at
most 1920×1080. Render pipelines take their vertices from the shader (no vertex
buffers yet) and have no depth-stencil. **Surface input** (v0.6.0): pointer and key
events reach the guest as `wasi-gfx:surface` streams, which the host serves for
core modules per the Canonical ABI. The page's events are untrusted and bounded:
finite coordinates up to 2^16, key text of at most 64 bytes of valid UTF-8, 256
queued events per stream and 64 streams per surface.

`examples/triangle` draws a spinning triangle. On an Apple Metal adapter, the end to
end test checks 60 frames in order, the clear colour, the triangle, frames that
change, and a rate within the pacing (one run: 51 frames per second).

**Testing on headless Linux (fixed in v0.5.1).** Before v0.5.1, on Linux headless
Chromium 143 with SwiftShader (the CI fallback lane), every frame reached the page
fully transparent. Headless Linux Chromium can back a WebGPU canvas only with Skia
on Vulkan: launch it with `--enable-features=Vulkan --use-vulkan=swiftshader
--use-angle=swiftshader` (or a real Vulkan driver); otherwise the WebGPU instance is
torn down at the first canvas configure and frames are blank. With those flags the
Linux SwiftShader lane passes the same checks as macOS, pixels included, and the
pixel checks are strict. `go run ./internal/cmd/gpuprobe --canvas` follows one
canvas frame from the GPU to the page, to check a browser.

### Quake

The `quake` image of the [playground]({{< relref "playground" >}}) runs the
[go-quake1/engine](https://github.com/go-quake1/engine) guest (`cmd/quake-wctr`,
engine v0.10.0) on a surface with `--gpu`:

```sh
wctr run --rm --gpu -m 512m quake
```

It draws every frame with the engine's software renderer and presents it on a
`wasi-gfx:surface`. The 8-bit frame and its palette go to the GPU as storage
buffers, and a loop-free WGSL shader looks up the colours. Keys come from the
surface's input stream. There is no audio yet, and the game boots straight into
`e1m1` of LibreQuake (data from [go-quake1/freepak](https://github.com/go-quake1/freepak)
v0.2.0). The image is GPL-3.0-or-later; its source is named in
`/usr/share/doc/quake/copyright`.

The playground's end-to-end test, `TestQuake`, runs on a real GPU in CI (GitHub's
macOS runner, Apple adapter). For fixed view 4 (640x400) the frame the page receives
has RGBA sha256 `f84421f86941b51dbb5afafd0eb2cb6a608314e3064b07100ff83c2fb2b3c8df`,
the figure the guest computed on the CPU; 150 frames reach the page, 140 of them
distinct. One CI run measured 58 ms per frame on that runner, one run and not a
benchmark. The world drawn on the GPU is not done yet.

## `onnx`

The ONNX loader runs a model on the CPU. Operators: Add, AveragePool, Concat,
Constant, Conv, Div, Dropout (inference), Flatten, Gemm, Identity, MatMul,
MaxPool, Mul, Relu, Reshape, Softmax, Sub and Transpose (default domain, operator
sets 7 and later, float tensors, int64 for shapes). A model using anything else
fails at load, naming every unsupported operator. The model is untrusted input:
its sizes and the work of one inference are bounded, and the decoder is fuzzed.
CPU performance has not been measured yet.

## Invalid WGSL fails loudly (gpu v0.9.0)

A shader module, layout, pipeline or bind group the GPU refuses used to do
nothing: the kernels simply never ran. Now the `submit` that carries it fails
with errno **11, invalid shader**, the context is lost, and the fifth import
`last_error(ctx, buf, len, out)` returns the browser compiler's message (at
most 1 KiB, WGSL line and column included; on Firefox it comes from
`getCompilationInfo`). `batchsink.Error.Msg` carries it:

```text
submit: invalid shader or object (the GPU refused it): ...: 'meta' is a reserved keyword
```

`meta` is a reserved WGSL word, which is how the defect was found. The
scoping is in go-gfx/webgpu v0.10.0 (`batch.ErrInvalid`,
`ShaderModule.CompilationMessages`). `examples/batchcompute shaders` runs a
reserved word, a syntax error and a valid control in a container.

## A language model in a container

[go-wasmcontainers/llm](https://github.com/go-wasmcontainers/llm) (v0.5.0)
runs a GGUF llama-family model, SmolLM2-135M Q4_0 first and Qwen2.5-0.5B
Q4_0 since v0.4.0, inside a container: the
module reads `/models/NAME.gguf` and generates on the GPU through
`wasi_webgpu_batch`, eight tokens per submit with one read-back of the
sampled tokens (argmax runs on the GPU).

```sh
wctr run --gpu --gpu-memory=512m --model smollm2-135m-instruct-q4_0 IMAGE run "The capital of France is"
wctr run --gpu --gpu-memory=512m --model qwen2.5-0.5b-instruct-q4_0 IMAGE run -model /models/qwen2.5-0.5b-instruct-q4_0.gguf "The capital of France is"
```

- **The model mount** (engine v0.33.0): `--model NAME` names an entry of the
  engine's registry, a Hugging Face repository pinned by commit and sha256.
  The size (at most 2 GiB) and then the sha256 are checked before the
  container exists; the file is read-only and range-readable, never copied
  into memory, and a container can neither write to it nor name a URL.
  Natively it is the cache file `<sha256>.gguf`; in a browser it is fetched
  by 16 MiB ranges into the origin's OPFS and read through a synchronous
  handle. Only huggingface.co and us.aws.cdn.hf.co are allowed, at every
  redirect.
- **Firefox read-back** (gpu v0.11.1, go-gfx/webgpu v0.12.1). Firefox 159
  resolves `mapAsync` and `onSubmittedWorkDone` on the tick of a 100 ms
  device-poll timer ([Bugzilla 1870699](https://bugzilla.mozilla.org/show_bug.cgi?id=1870699),
  `POLL_TIME_MS = 100` in `WebGPUParent.cpp`), so a submit with a read-back
  cost 200 ms (a work-done wait, then a map) whatever its size. While the host
  waits it now kicks the queue with an empty `submit([])` (the first after
  500 us, then at doubling gaps up to 8 ms); wgpu-core polls at the end of a
  submit, so the completion arrives as soon as the GPU has it. Measured on
  Firefox 159, Apple M4 Max, load 5: 200 ms before, 2.6 ms after (Chromium
  0.35 to 1 ms, Safari 0.34 to 0.94 ms). A language model (llm v0.3.1,
  SmolLM2-135M Q4_0) decodes at about 290 tokens/s on Firefox and about
  550 on Chromium; the default decode reads back once per eight tokens, so the
  fix does not move it. The control is `internal/cmd/ffwait` in the gpu
  repository (a bare page, no engine); the kick stays harmless when Firefox
  polls on demand.
- **Kernels** are f32 WGSL (Q4_0, Q4_1, Q8_0), with no subgroups and no f16;
  the workgroup size and grids come from the adapter's limits. Only Apple Metal
  has been run.
- **Correctness**: against llama.cpp running the same weights dequantised to
  F32 (its own `llama-quantize`), the container gives the same 32 greedy
  tokens on all four test prompts, in Chromium on the Apple GPU. llama.cpp's own
  Q4_0 CPU run differs by 0.1 to 0.4 nats because it quantises activations to
  Q8_0, which amplifies rounding about a million times; the repository's README
  explains the measurement.
- **Speed** (v0.3.0): 580 tokens/s decoding and 5359 prefill (128-token
  prompt), against 514 and 1268 for llama.cpp on one CPU core in the same
  session (v0.2.0: 154 and 312). A layer is five fused dispatches, eight tokens
  are computed per submit and prompt chunks of up to 64 tokens go through the
  layers together. Firefox and non-Apple adapters are not measured; the
  profile, each change and the limits are in the repository's
  [docs/perf/2026-11-04.md](https://github.com/go-wasmcontainers/llm/blob/main/docs/perf/2026-11-04.md).
- **Qwen2.5-0.5B Q4_0** (428,730,208 bytes, engine v0.38.0 registry entry
  `qwen2.5-0.5b-instruct-q4_0`): 350 tokens/s decoding and 1908 prefill against
  161 and 426 for llama.cpp on one CPU core, the same strict oracle (32 of 32
  greedy tokens on four prompts, CPU path and GPU), and 480 MB of GPU memory at
  context 512, so `--gpu-memory=512m` suffices; v0.5.0 runs its Q8_0 output
  matrix through the shared `kernels/q4` kernel of go-gfx/webgpu v0.15.0 and
  decodes 1.19 times faster than v0.4.0 (325 to 387 tokens/s, interleaved,
  load at most 6.6; the guard of gpu v0.12.0 refuses and truncates nothing)
  ([details](https://github.com/go-wasmcontainers/llm/blob/main/docs/perf/2026-11-04-qwen2.5-0.5b.md)).
