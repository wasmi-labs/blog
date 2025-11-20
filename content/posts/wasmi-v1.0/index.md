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

Wasmi 1.0 represents years of careful evolution: new WebAssembly features, cleaner internals, better performance, and stronger security. This release positions Wasmi as a robust choice for anyone needing a lightweight, highly compatible, versatile, and efficient Wasm interpreter.

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

## Safety & Security Improvements

Given Wasmi's origins in the blockchain space and its focus on embedded systems, safety and security has always been of central importance.

### Security Audit

The [Stellar Development Foundation](https://stellar.org/foundation) sponsored an [extensive security audit for Wasmi](https://github.com/wasmi-labs/wasmi/blob/main/resources/audit-2024-11-27.pdf) conducted by [Runtime Verification Inc.](https://runtimeverification.com/).

Over the course of several months a whole team of security professionals inspected the Wasmi codebase, analysed its architecture and wrote down their assessment in the document linked above. During that process they were able to find some issues and concerns which were quickly fixed.

### Fuzzing Infrastructure

Wasmi has received its own fuzzing infrastructure consisting of 3 different fuzzing targets:

- 🔧 `translate`: Continuously translates Wasm blobs and tries to find bugs in Wasmi's translation engine.
- 🏃 `execute`: Executes Wasm blobs with various inputs and tries to find bugs in Wasmi's execution engine.
- ⚖️ `differential`: Compares the execution of Wasmi with other Wasm runtimes and tries to find examples of mismatching semantics. [^3]

Especially the `differential` fuzzing target finds quite a lot of issues before they land in new releases.
And conversely, on Wasmtime's own fuzzing infrastructure [Wasmi acts as an oracle](https://github.com/bytecodealliance/wasmtime/blob/v38.0.4/crates/fuzzing/src/oracles/diff_wasmi.rs) for its differential fuzzing.

### OSS-Fuzz

On top of that [Wasmi has been added to Google's OSS-Fuzz](https://github.com/google/oss-fuzz/pull/12665) and is now continuously fuzzed. This has already found some intricate bugs and issues and is a valuable addition that helps to keep Wasmi safe and secure.

### Wast Testsuite

Wasmi now has its own external [Wasmi testsuite](https://github.com/wasmi-labs/wasmi-tests) alongside the official Wasm spec testsuite.
This [WebAssembly Script (Wast)](https://github.com/WebAssembly/spec/tree/wg-3.0/interpreter#scripts) based testsuite features Wasmi specific testcases and is continuously extended to assert semantic behavior in even more cases. [^4]

### Minified Dependency Graph

A minified graph of external dependencies usually implies a smaller attack surface. In March 2024, Wasmi still had 7 external dependencies. And today, with Wasmi 1.0 we are down to just 2:

- [`spin`](https://crates.io/crates/spin): Low-level primitives for locks and mutexes for `no_std` environments which Wasmi supports. Note that this is a trivial dependency when compiling Wasmi with `std` feature enabled.
- [`wasmparser`](https://crates.io/crates/wasmparser): A Bytecode Alliance maintained highly efficient Wasm parser and validator.

There are [plans for an internal `wasmi_parse` crate](https://github.com/wasmi-labs/wasmi/issues/1514) that would replace the external `wasmparser` crate, eventually dropping to just 1 lightweight external dependency.

The removal of external dependencies has also had significant positive impact on Wasmi's compile times. [^5]

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
- [x] Received another security audit conducted by Runtime Verification Inc. sponsored by Stellar Development Foundation.
- Wasmi now allows to inspect Wasm custom sections.
- Fixed a dead-lock allowing users to compile Wasm modules in host functions.
- Added support for Wasm C-API bindings via `wasmi_c_api_impl` crate. Visit [C-API README](https://github.com/wasmi-labs/wasmi/blob/main/crates/c_api/README.md).
- [x] Minified Wasmi's dependency graph from 7 external dependencies down to just 2. (`spin` and `wasmparser`)
- [x] Wasmi was added to Google's OSSFuzz.
- Wasmi now provided as backend in Wasmer.
- Batteries included: WAT support in `Module::new` and `Module::new_unchecked`.
- Added support for Wasm function call resumption after running out of fuel.
- Foundational clean-up of the Wasmi translator: simpler and future proofed.
    - Fuel metering no longer an afterthought - nearly free.
    - Inspired by Stitch's translation model.
- [x] Wasmi specific Wast testsuite allowing to catch more Wasmi specific bugs all while being disentangled with implementation details.

## Future plans and projects

- Wasmi 2.0 and its new executor.
    - Make use of Rust's `become` once stable, prepare instruction dispatch for it.
    - Direct threaded code, inspired by Stitch for even faster execution performance.
    - Encodings all immediates in bytecode inline - removing the need for function local constants bloating up the stack.
    - Wasm module preprocessing for even faster Wasm module instantiations.
    - Instance related bytecode to speed-up code with interacts heavily with instance data.
    - Accumulator based interpreter architecture for maximum execution performance.
- Full Wasm 3.0 support: Wasm `function-references`, `exception-handling` and `gc` proposal implementation.

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

Wasmi 1.0 represents years of careful evolution: new WebAssembly features, cleaner internals, better performance, and stronger security. This release positions Wasmi as a robust choice for anyone needing a lightweight, highly compatible, and efficient Wasm interpreter.

We’re excited for developers to try these improvements and contribute to shaping the next phase of Wasmi’s growth. Stay tuned for further updates and community contributions.

---

[^1]: I verified this using `cargo show-asm` on a Macbook M2 Pro ARM machine at least for many of the `simd` operators.

[^2]: The full list of Wasmi support Wasm proposals can be found [here](https://github.com/wasmi-labs/wasmi?tab=readme-ov-file#webassembly-features).

[^3]: Currently we differentially fuzz against an older, much simpler version of Wasmi itself as well as the mature and battletested [Wasmtime](https://github.com/bytecodealliance/wasmtime) Wasm runtime.

[^4]: Before that, Wasmi's tests were tightly integrated with its translation engine. This necessitated updates to them whenever Wasmi's translation engine was changed. While those tightly integrated tests helped with early development, it is needless to say, that this process wasn't sustainable.

[^5]: Where Wasmi v0.32 required ~10 seconds to compile, we are now down to just ~4.5s. Tested on Macbook M2 Pro.
