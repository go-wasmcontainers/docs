---
title: "GPU"
weight: 28
description: "wasi-nn on the CPU and on WebGPU, wasi:webgpu compute and rendering to surfaces, for WASI containers. gpu v0.6.0, with surface input."
tags: [gpu]
---

[go-wasmcontainers/gpu](https://github.com/go-wasmcontainers/gpu) is the module
for GPU access from WASI containers: wasi-nn inference, wasi:webgpu compute and
rendering, on WebGPU in the browser and on the CPU natively. Pure Go. The design
and its references are in
[docs/DESIGN.md](https://github.com/go-wasmcontainers/gpu/blob/main/docs/DESIGN.md).

**Status (v0.6.0): inference, compute, rendering and surface input exist.**

| Part | State |
|---|---|
| `onnx`: ONNX model loader and CPU operators | done (v0.1.0) |
| `nn`: wasi-nn host module (`wasi_ephemeral_nn`), CPU backend; `wasinn` Go guest bindings; the `mnist` example | done (v0.2.0) |
| `nn/webgpu`: wasi-nn on the GPU, WGSL kernels (browser) | done (v0.3.0) |
| wasi:webgpu compute (host and Go guest), `examples/compute` | done (v0.4.0) |
| rendering: `wasi-gfx:surface` and `surface-webgpu`, frames to the page, `examples/triangle` | done (v0.5.0); the dashboard (v0.6.0) shows the frames in its Surfaces view |
| surface input: `on-pointer-up/down/move` and `on-key-up/down` streams for the guest, `webgpu.SurfaceInput` for the page | done (v0.6.0); no dashboard release forwards pane events yet |
| `on-resize` and `on-frame` streams, vertex buffers, depth-stencil | not yet |

A container asks for the GPU with `wctr run --gpu` (engine v0.13.0 or later). The
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
pipelines; a submission still running after 2 seconds has its device destroyed,
and fallback adapters are refused.

On an Apple Metal adapter, `examples/compute` (adding two 1M-float vectors,
multiplying two 256×256 matrices) matches the CPU on every element, with the
matmul in 15 ms on the GPU against 42 ms on the guest's CPU, and the vector add in
69 ms against 3.5 ms (dominated by transfers). That is one run, not a benchmark.

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

## `onnx`

The ONNX loader runs a model on the CPU. Operators: Add, AveragePool, Concat,
Constant, Conv, Div, Dropout (inference), Flatten, Gemm, Identity, MatMul,
MaxPool, Mul, Relu, Reshape, Softmax, Sub and Transpose (default domain, operator
sets 7 and later, float tensors, int64 for shapes). A model using anything else
fails at load, naming every unsupported operator. The model is untrusted input:
its sizes and the work of one inference are bounded, and the decoder is fuzzed.
CPU performance has not been measured yet.
