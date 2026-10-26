---
title: "References"
weight: 40
description: "The specifications and sources the engine is written against."
tags: [references]
---

Every behaviour of the engine that follows an external format or protocol is
written against one of the documents below. The engine repository's own
`docs/REFERENCES.md` says, for each, which version was opened and which file it
governs.

| Reference | What it decides |
| --- | --- |
| [OCI Image Format Specification](https://github.com/opencontainers/image-spec) (v1.1.1) | digests, manifests, image indexes, image config, layers and whiteouts, image layout |
| [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec) | pulling manifests and blobs from a registry |
| [Docker Registry token authentication](https://github.com/distribution/distribution/blob/main/docs/content/spec/auth/token.md) | the bearer-token challenge and token request |
| [Docker Engine API](https://docs.docker.com/reference/api/engine/) (v1.47) | the endpoints and schemas `wctrd` is compatible with |
| [Wasm OCI artifact layout](https://tag-runtime.cncf.io/wgs/wasm/deliverables/wasm-oci-artifact/) (CNCF TAG-Runtime) | media types and config fields of a Wasm image |
| [wazero](https://github.com/tetratelabs/wazero) (v1.12.0) | the runtime, its WASI preview1 imports and writable filesystem interface |
| [WASI preview1](https://github.com/WebAssembly/WASI/blob/wasi-0.1/preview1/docs.md) | the system interface a `wasip1` module is written against |
