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

## What does 1.0 mean?

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

Given Wasmi's origins as execution engine for smart contracts on the blockchain and its focus on constraint devices and embedded systems, deterministic execution, safety and security has always been of central importance.

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

A minified graph of external dependencies usually implies a smaller attack surface. In March 2024, Wasmi still had [7 external dependencies](https://crates.io/crates/wasmi/0.32.3/dependencies). Wasmi 1.0 reduced this to just 2:

- [`spin`](https://crates.io/crates/spin): Low-level primitives for locks and mutexes for [`no_std` environments](https://docs.rust-embedded.org/book/intro/no-std.html) which Wasmi supports. Note that this is a trivial dependency when compiling Wasmi with `std` feature enabled.
- [`wasmparser`](https://crates.io/crates/wasmparser): A [Bytecode Alliance](https://bytecodealliance.org/) maintained highly efficient Wasm parser and validator.

There are [plans for an internal `wasmi_parse` crate](https://github.com/wasmi-labs/wasmi/issues/1514) to replace the external `wasmparser` crate.

The removal of external dependencies has also had significant positive impact on Wasmi's compile times. [^5]

## New Features

Wasmi and its feature set is constantly evolving. In this section the most notable new features since the last post in March 2024 are listed.

### C-API Bindings

Support for the "official" [WebAssembly C-API](https://github.com/WebAssembly/wasm-c-api) has been added to Wasmi via [`wasmi_c_api_impl` crate](https://crates.io/crates/wasmi_c_api_impl). With this, any library or language that can interface with C code can now also use Wasmi. Visit [C-API README](https://github.com/wasmi-labs/wasmi/blob/main/crates/c_api/README.md) to learn more about using Wasmi from C.

There are also plans for Python bindings for Wasmi via the popular [PyO3](https://github.com/PyO3/pyo3) library. If that's something you would love to see please [join the discussion](https://github.com/wasmi-labs/wasmi/issues/1436).

### Refueled Resumable Calls

As many other Wasm runtimes, Wasmi has [fuel metering built-in](https://docs.rs/wasmi/0.51.2/wasmi/struct.Config.html#method.consume_fuel). This is a solution to solve the problem of making sure that executions halt eventually. Executions are provided a quantity of fuel and yield back to the host when they run out of it before returning.

> 💡 Wouldn't it be great if the host could then decide to resume the execution later?

This is exactly what is possible with Wasmi today and it comes in very handy when using Wasmi as execution engine to schedule concurrent Wasm jobs similar to how operating systems do.

### Usability Improvements

Wasmi has received more usability improvments than could fit on a single list.
Some of the most relevant are:

- 🔋: WAT is now supported in `Module::new` and `Module::new_unchecked`.
- 🪞: Wasmtime API mirror has been further improved. The ultimate goal is for Wasmi to be a drop-in replacement for most users.
- ⚡: The `Instance::new` API has been added which is more low-level and efficient than the `Linker` API in some cases.
- 📚: An extensive [Usage Guide](https://github.com/wasmi-labs/wasmi/blob/main/docs/usage.md) has been written to provide Wasmi users with the most important knowledge to get the most out of their Wasmi installment.
- 🌳: Via `hash-collections` and `prefer-btree-collections` users can decide if Wasmi uses hash-tables or btree-tables. Why is this important? In some `no_std` environments it is not generally safe to use hash-tables since they require random initialization which environment such as `wasm32` cannot provide. [^6]
- 🔓: Users can compile Wasm modules within called host functions which previously caused a dead-lock.

## Looking Ahead

- Wasmi 2.0 and its new executor.
    - Make use of Rust's `become` once stable, prepare instruction dispatch for it.
    - Direct threaded code, inspired by Stitch for even faster execution performance.
    - Encodings all immediates in bytecode inline - removing the need for function local constants bloating up the stack.
    - Wasm module preprocessing for even faster Wasm module instantiations.
    - Instance related bytecode to speed-up code with interacts heavily with instance data.
    - Accumulator based interpreter architecture for maximum execution performance.
- Full Wasm 3.0 support: Wasm `function-references`, `exception-handling` and `gc` proposal implementation.

## Try It Out!

- Wasmi can now be used as a backend in Wasmer, broadening its applicability in the Wasm ecosystem.

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
- [x] Fixed a dead-lock allowing users to compile Wasm modules in host functions.
- [x] Added support for Wasm C-API bindings via `wasmi_c_api_impl` crate. Visit [C-API README](https://github.com/wasmi-labs/wasmi/blob/main/crates/c_api/README.md).
- [x] Minified Wasmi's dependency graph from 7 external dependencies down to just 2. (`spin` and `wasmparser`)
- [x] Wasmi was added to Google's OSSFuzz.
- [x] Wasmi now provided as backend in Wasmer.
- [x] Batteries included: WAT support in `Module::new` and `Module::new_unchecked`.
- [x] Added support for Wasm function call resumption after running out of fuel.
- Foundational clean-up of the Wasmi translator: simpler and future proofed.
    - Fuel metering no longer an afterthought - nearly free.
    - Inspired by Stitch's translation model.
- [x] Wasmi specific Wast testsuite allowing to catch more Wasmi specific bugs all while being disentangled with implementation details.

[^1]: I verified this using `cargo show-asm` on a Macbook M2 Pro ARM machine at least for many of the `simd` operators.

[^2]: The full list of Wasmi support Wasm proposals can be found [here](https://github.com/wasmi-labs/wasmi?tab=readme-ov-file#webassembly-features).

[^3]: Currently we differentially fuzz against an older, much simpler version of Wasmi itself as well as the mature and battletested [Wasmtime](https://github.com/bytecodealliance/wasmtime) Wasm runtime.

[^4]: Before that, Wasmi's tests were tightly integrated with its translation engine. This necessitated updates to them whenever Wasmi's translation engine was changed. While those tightly integrated tests helped with early development, it is needless to say, that this process wasn't sustainable.

[^5]: Where Wasmi v0.32 required ~10 seconds to compile, we are now down to just ~4.5s. Tested on Macbook M2 Pro.

[^6]: That's why Rust's standard library does not provide `HashMap` in its `alloc` crate. Note that the `hashbrown` crate has `no_std` support but kinda cheats as its random initialization is not applicable to some systems and thus a potential attack vector for safety critical `no_std` systems.
