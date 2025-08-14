---
date: 2025-08-14T13:16:51+02:00
title: "Wasmi 1.0"
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
draft: false
---

## What has happened since last blog post

- New Wasm proposals support:
    - [x] Wasm `multi-memory`.
    - [x] Wasm `memory64`.
    - [x] Wasm `custom-page-sizes`.
    - [x] Wasm `wide-arithmetic`.
    - [x] Wasm `simd` enabled by `simd` crate feature.
    - [x] Wasm `relaxed-simd` enabled by `simd` crate feature.
- Improved Wasmtime API mirror.
- Optimized Wasmi engine internals.
- Received another security audit conducted by Runtime Verification Inc. sponsored by Stellar Development Foundation.
- Wasmi now allows to inspect Wasm custom sections.
- Fixed a dead-lock allowing users to compile Wasm modules in host functions.
- Added support for Wasm C-API bindings via `wasmi_c_api_impl` crate. Visit [C-API README](https://github.com/wasmi-labs/wasmi/blob/main/crates/c_api/README.md).
- Minified Wasmi's dependency graph from 7 external dependencies down to just 2. (`spin` and `wasmparser`)
- Wasmi was added to Google's OSSFuzz.
- Implemented lots of new Wasmi bytecode optimizations and lowering to improve execution performance.
- Wasmi now provided as backend in Wasmer.
- Batteries included: WAT support in `Module::new` and `Module::new_unchecked`.
- Added support for Wasm function call resumption after running out of fuel.
- Foundational clean-up of the Wasmi translator: simpler and future proofed.
    - Fuel metering no longer an afterthought - nearly free.
    - Inspired by Stitch's translation model.

## Future plans and projects

- Make use of Rust's `become` once stable, prepare instruction dispatch for it.
- Direct threaded code, inspired by Stitch for even faster execution performance.
- Encodings all immediates in bytecode inline - removing the need for function local constants bloating up the stack.
- Wasm `function-references` and `expection-handling` proposals support.

