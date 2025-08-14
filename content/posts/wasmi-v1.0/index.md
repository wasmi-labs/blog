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

---

Over the past year, the Wasmi project has seen significant growth and refinement. With the upcoming 1.0 release, it’s a good moment to reflect on the new capabilities, improvements, and performance optimizations that have been added since our last update.

## New WebAssembly Proposals Support

Wasmi now supports several cutting-edge WebAssembly proposals:

Multi-memory – allowing modules to define and use more than one linear memory.

Memory64 – extending memory addressing to 64-bit, opening up new possibilities for large memory applications.

Custom page sizes – for more flexible memory layouts.

Wide arithmetic – enhanced numeric operations beyond the standard WebAssembly spec.

SIMD and relaxed-SIMD – enabled via the simd crate feature, unlocking faster vectorized operations.

These new capabilities make Wasmi a more complete and modern Wasm interpreter, keeping pace with the evolving WebAssembly ecosystem.

## API Improvements

Better Wasmtime API mirroring – developers familiar with Wasmtime will find it easier to transition or use Wasmi in similar patterns.

C-API bindings – the new wasmi_c_api_impl crate allows integration with C applications. See the README for details.

Performance and Engine Optimizations

Significant work has gone into improving the internal performance of Wasmi:

Extensive bytecode optimizations and lowering for faster execution.

Simplified and future-proofed translator internals, including fuel metering that is now nearly free.

Support for resuming Wasm function calls after running out of fuel, improving control over execution.

Batteries included: WAT support is now available in Module::new and Module::new_unchecked.

## Security and Reliability

Wasmi underwent another security audit conducted by Runtime Verification Inc., sponsored by Stellar Development Foundation, ensuring a higher degree of trustworthiness for critical applications. Additionally:

A deadlock allowing module compilation within host functions was fixed.

Wasmi was added to Google’s OSSFuzz, enabling continuous automated fuzz testing.

## Reduced Dependencies

We trimmed Wasmi’s dependency graph from 7 external crates down to just 2: spin and wasmparser. This makes Wasmi lighter, simpler to integrate, and easier to audit.

## Ecosystem Integration

Wasmi can now be used as a backend in Wasmer, broadening its applicability in the Wasm ecosystem.

You can now inspect custom sections of Wasm modules directly, offering more introspection capabilities.

## Looking Ahead

Wasmi 1.0 represents a year of careful evolution: new WebAssembly features, cleaner internals, better performance, and stronger security. This release positions Wasmi as a robust choice for anyone needing a lightweight, highly compatible, and efficient Wasm interpreter.

We’re excited for developers to try these improvements and contribute to shaping the next phase of Wasmi’s growth. Stay tuned for further updates and community contributions.

---

