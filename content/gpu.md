---
title: "GPU"
weight: 28
description: "ONNX models on the CPU in pure Go (gpu v0.1.0). GPU inference, wasi:webgpu compute and rendering are planned."
tags: [gpu]
---

[go-wasmcontainers/gpu](https://github.com/go-wasmcontainers/gpu) is the module
for GPU access from WASI containers: wasi-nn inference, wasi:webgpu compute and
rendering, on WebGPU in the browser and on the CPU natively. Pure Go. The
design and its references are in
[docs/DESIGN.md](https://github.com/go-wasmcontainers/gpu/blob/main/docs/DESIGN.md).

**Status (v0.1.0): only the ONNX loader and its CPU operators exist.**
Everything that uses a GPU is planned.

| Part | State |
|---|---|
| `onnx`: ONNX model loader and CPU operators | **done** (v0.1.0) |
| wasi-nn host module (`wasi_ephemeral_nn`) and guest bindings | next |
| wasi:webgpu compute (browser) | planned |
| rendering to a dashboard pane | planned |

## `onnx`

It loads an ONNX model with a bounded protobuf reader and runs it **on the CPU**.
Operators: Add, AveragePool, Concat, Constant, Conv, Div, Dropout (inference),
Flatten, Gemm, Identity, MatMul, MaxPool, Mul, Relu, Reshape, Softmax, Sub and
Transpose (default domain, operator sets 7 and later, float tensors, int64 for
shapes).

- **It refuses precisely**: a model using anything else fails at load, naming
  every unsupported operator and its node count.
- **It is checked op by op** against the ONNX node tests (onnx v1.22.0): the 122
  tests whose models load here pass with the ONNX backend's tolerances, and
  MNIST-12 from the ONNX model zoo passes on its own test data.
- **The model is untrusted input**: sizes, node count, rank, elements and the
  multiply-accumulates of one inference are bounded, and the decoder and runner
  are fuzzed.

Performance has not been measured yet.
