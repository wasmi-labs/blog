---
date: 2025-08-14T13:16:51+02:00
title: "Wasmi 1.0"
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
draft: false
---

It has been a long time since the last [article about Wasmi](https://wasmi-labs.github.io/blog/posts/wasmi-v0.32/) in March 2024.
Since that time a lot of has happened on the Wasmi project, both technically and non-technically, which culminated in Wasmi 1.0.

Wasmi is an efficient and versatile [WebAssembly (Wasm)](https://webassembly.org/) interpreter with a focus on embedded environments. It is an excellent choice for plugin systems, cloud hosts and as smart contract execution engine.

Before going into all the details, a huge thank you to the [Stellar Development Foundation](https://stellar.org/foundation) which sponsors development of the Wasmi project since October 2024. Without their sponsorship, the Wasmi project wouldn't be where it is today.

## What does 1.0 mean to you?

Over the course of the last major versions a ton of deprecations and fine tuning of [Wasmi's API](https://docs.rs/wasmi/latest/wasmi/) has been implemented, all in preparation of the 1.0 release. With this release, all those deprecated APIs are going to be removed. The 1.0 release marks an important promise of API stability for Wasmi users.

That's it!

## New Wasm Proposals

Since March 2024 a lot of [Wasm proposals](https://github.com/WebAssembly/proposals) have been implemented in Wasmi.

- ✅ The [Wasm `multi-memory` proposal](https://github.com/WebAssembly/multi-memory) allows users to define multiple linear memories within a single Wasm module.
  With this, users can isolate and protect different memories with different purposes from each other.
- ✅ The [Wasm `memory64` proposal](https://github.com/WebAssembly/memory64) allows users to define 64-bit Wasm modules, thus being able to execute Wasm binaries
  that require more than just 4GB of memory.
- ✅ The [Wasm `custom-page-sizes` proposal](https://github.com/WebAssembly/custom-page-sizes) allows to define linear memories with page sizes of 1 byte. By default, linear memories has page sizes of 64KB, thus this allows to execute Wasm on tiny embedded devices with less than 64KB of memory.
- ✅ The [Wasm `simd` proposal](https://github.com/webassembly/simd) defines over 200 new 128-bit SIMD operators. This allows users to optimize certain compute intense workloads. Honestly, a questionable use-case for an interpreter to say the least which is why Wasmi's `simd` support comes [opt-in](https://github.com/wasmi-labs/wasmi/blob/v0.51.2/crates/wasmi/Cargo.toml#L54) so users don't have to take the bloat. To add insult to injury, it was made sure that `simd` operators in Wasmi actually use SIMD machine instructions. [^1]
- ✅ The [Wasm `relaxed-simd` proposal](https://github.com/WebAssembly/relaxed-simd) defines some non-deterministic extension operators on top of the Wasm `simd` proposal which can be more efficient in some cases.
- ✅ The [Wasm `wide-arithmetic` proposal](https://github.com/WebAssembly/wide-arithmetic) defines 128-bit arithmetic operators for `add`, `sub` and `mul`. These operators allow to optimize certain use-cases such as big-integer arithmetic.

With these additions, Wasmi supports all of [Wasm 2.0](https://webassembly.org/news/2025-03-20-wasm-2.0/) and even quite a few proposals from [Wasm 3.0](https://webassembly.org/news/2025-09-17-wasm-3.0/). [^2]

## Engine Optimizations

Countless engine improvements, new optimizations, clean-ups and refactorings have significantly improved Wasmi's translation and execution performance all while lowering its memory footprint.

TODO: graphics, diagrams

## What has happened since last blog post

- [x] ~Wasmi no longer belongs to Parity but is a stand-alone project.~
- [x] Wasmi development financially sponsored by the SDF since October 2024.
- [x] New Wasm proposals support:
    - [x] Wasm `multi-memory`.
    - [x] Wasm `memory64`.
    - [x] Wasm `custom-page-sizes`.
    - [x] Wasm `wide-arithmetic`.
    - [x] Wasm `simd` enabled by `simd` crate feature.
    - [x] Wasm `relaxed-simd` enabled by `simd` crate feature.
- [x] Optimized Wasmi engine internals.
    - Implemented lots of new Wasmi bytecode optimizations and lowering to improve execution performance.
- Improved Wasmtime API mirror.
- Received another security audit conducted by Runtime Verification Inc. sponsored by Stellar Development Foundation.
- Wasmi now allows to inspect Wasm custom sections.
- Fixed a dead-lock allowing users to compile Wasm modules in host functions.
- Added support for Wasm C-API bindings via `wasmi_c_api_impl` crate. Visit [C-API README](https://github.com/wasmi-labs/wasmi/blob/main/crates/c_api/README.md).
- Minified Wasmi's dependency graph from 7 external dependencies down to just 2. (`spin` and `wasmparser`)
- Wasmi was added to Google's OSSFuzz.
- Wasmi now provided as backend in Wasmer.
- Batteries included: WAT support in `Module::new` and `Module::new_unchecked`.
- Added support for Wasm function call resumption after running out of fuel.
- Foundational clean-up of the Wasmi translator: simpler and future proofed.
    - Fuel metering no longer an afterthought - nearly free.
    - Inspired by Stitch's translation model.
- Wasmi specific Wast testsuite allowing to catch more Wasmi specific bugs all while being disentangled with implementation details.

## Future plans and projects

- Wasmi 2.0 and its new executor.
    - Make use of Rust's `become` once stable, prepare instruction dispatch for it.
    - Direct threaded code, inspired by Stitch for even faster execution performance.
    - Encodings all immediates in bytecode inline - removing the need for function local constants bloating up the stack.
    - Wasm module preprocessing for even faster Wasm module instantiations.
    - Instance related bytecode to speed-up code with interacts heavily with instance data.
    - Accumulator based interpreter architecture for maximum execution performance.
- Full Wasm 3.0 support: Wasm `function-references`, `expection-handling` and `gc` proposal implementation.

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

[^1]: I verified this using `cargo show-asm` on a Macbook M2 Pro ARM machine at least for many of the `simd` operators.

[^2]: The full list of Wasmi support Wasm proposals can be found [here](https://github.com/wasmi-labs/wasmi?tab=readme-ov-file#webassembly-features).

