---
date: 2025-11-27T12:16:51+02:00
title: "Wasmi 1.0"
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
draft: false
---

> ⚠️ No A.I. was ~harmed~ used for the making of this article. [^9]

It has been a long time since the last [article about Wasmi](https://wasmi-labs.github.io/blog/posts/wasmi-v0.32/) in March 2024.
Since that time a lot of has happened on the Wasmi project, both technically and non-technically, which culminated in Wasmi 1.0, [almost 8 years after its initial version](https://crates.io/crates/wasmi/0.0.0).

Wasmi is an efficient and versatile [WebAssembly (Wasm)](https://webassembly.org/) interpreter with a focus on embedded environments. It is an excellent choice for IoT devices, [plugin systems](https://typst.app/docs/reference/foundations/plugin/), cloud hosts, as [smart contract execution engine](https://stellar.org/soroban) and even for your lightweight [game console](https://fireflyzero.com/).

Before going into all the details, a huge thank you to the [Stellar Development Foundation (SDF)](https://stellar.org/foundation) which sponsors development of the Wasmi project since October 2024. Without their sponsorship, the Wasmi project wouldn't be where it is today.

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

Countless execution engine improvements, new optimizations, refinements on Wasmi's internal bytecode, code clean-ups and refactorings have significantly improved Wasmi's translation and especially execution performance all while lowering its memory footprint and supporting new features.

Note though that performance was not the primary focus since the last blog post and Wasmi is still using the same interpreter architecture that has been introduced in March 2024.

### Benchmarks

All the benchmarks shown below were conducted on an Apple M2 Pro using nightly `rustc 1.93.0-nightly (1be6b13be 2025-11-26)`
via the [Wasmi benchmarks](https://github.com/wasmi-labs/wasmi-benchmarks) at [this revision](https://github.com/wasmi-labs/wasmi-benchmarks/tree/b9385cae9bfb8cf84dbb13996d0b948ca5826b53).

| | |
|--|--|
| [![][execute-fib-iterative]][execute-fib-iterative] | [![][execute-fib-recursive]][execute-fib-recursive] |
| [![][execute-fib-tailrec]][execute-fib-tailrec] | [![][execute-argon2]][execute-argon2] |
| [![][execute-primes]][execute-primes] | [![][execute-matmul]][execute-matmul] |

The Coremark benchmark also shows a clear positive progression. [^7]

[![][bench-coremark]][bench-coremark]

[execute-fib-iterative]: ./benches/execute/fib.iterative.svg
[execute-fib-recursive]: ./benches/execute/fib.recursive.svg
[execute-fib-tailrec]: ./benches/execute/fib.tailrec.svg
[execute-argon2]: ./benches/execute/argon2.svg
[execute-primes]: ./benches/execute/primes.svg
[execute-matmul]: ./benches/execute/matmul.svg
[bench-coremark]: ./benches/coremark.svg

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

- 🔋: WAT is now supported in [`Module::new`](https://docs.rs/wasmi/0.51.2/wasmi/struct.Module.html#method.new) and [`Module::new_unchecked`](https://docs.rs/wasmi/0.51.2/wasmi/struct.Module.html#method.new_unchecked) if its [`wat` crate feature](https://github.com/wasmi-labs/wasmi/blob/v0.51.2/Cargo.toml#L44) is enabled. [^8]
- 🪞: [Wasmtime API](https://docs.rs/wasmtime/latest/wasmtime/) mirror has been further improved. The ultimate goal is for Wasmi to be a drop-in replacement for most users.
- ⚡: The [`Instance::new`](https://docs.rs/wasmi/0.51.2/wasmi/struct.Instance.html#method.new) API has been added which is more low-level and efficient than the `Linker` API in some cases.
- 📚: An extensive [Usage Guide](https://github.com/wasmi-labs/wasmi/blob/main/docs/usage.md) has been written to provide Wasmi users with the most important knowledge to get the most out of their Wasmi installment.
- 🌳: Via [`hash-collections` and `prefer-btree-collections`](https://docs.rs/wasmi/0.51.2/wasmi/#crate-features) users can decide if Wasmi uses hash-tables or btree-tables. Why is this important? In some `no_std` environments it is not generally safe to use hash-tables since they require random initialization which environment such as `wasm32` cannot provide. [^6]
- 🔓: Users can compile Wasm modules within called host functions which previously caused a dead-lock.

## Looking Ahead

The look ahead is even more exciting than the look behind!

### The Next-Gen Engine

Wasmi 2.0 and its new executor.

- Make use of Rust's [`become` keyword](https://github.com/rust-lang/rfcs/pull/3407) once stable.
- Make use of Rust's [`#[loop_match]` attribute](https://github.com/rust-lang/rfcs/pull/3720) once stable.
- 4 modes of execution:
    - direct-threaded code: fastest option
    - indirect-threaded code: fast option with lower memory consumption
    - loop-call dispatch: portable, direct-dispatch
    - loop-match dispatch: portable, indirect-dispatch
- Direct threaded code, inspired by Stitch for even faster execution performance.
- Encodings all immediates in bytecode inline - removing the need for function local constants bloating up the stack.
- Wasm module preprocessing for even faster Wasm module instantiations.
- Instance related bytecode to speed-up code with interacts heavily with instance data.
- Accumulator based interpreter architecture for maximum execution performance.

### Full Wasm 3.0 Support

Wasmi already implements many of the Wasm 3.0 proposals, and [even](https://github.com/WebAssembly/custom-page-sizes) [more](https://github.com/WebAssembly/wide-arithmetic):

- ✅ [`memory64`](https://github.com/WebAssembly/memory64)
- ✅ [`multi-memory`](https://github.com/WebAssembly/multi-memory)
- ✅ [`tail-call`](https://github.com/WebAssembly/tail-call)
- ✅ [`relaxed-simd`](https://github.com/WebAssembly/relaxed-simd)
- ✅ [`extended-const`](https://github.com/WebAssembly/extended-const)

However, the following Wasm 3.0 proposals are still missing:

- 🚧 [`function-references`](https://github.com/WebAssembly/function-references)
- 🚧 [`exception-handling`](https://github.com/WebAssembly/exception-handling)
- 🚧 [`gc` (Garbage Collection)](https://github.com/WebAssembly/gc)

Supporting those 3 missing Wasm proposals and thus making Wasmi WebAssembly 3.0 compatible will be a high priority once the next-gen execution engine, described above, has been implemented.

## Try It Out!

Try out and use Wasmi today via various different ways:

- 📚 As library dependency via the [`wasmi` crate](https://crates.io/crates/wasmi).
- 🖥 Using its CLI application by installing the [`wasmi_cli` crate](https://crates.io/crates/wasmi_cli) using `cargo install wasmi_cli`.
- ⚙️ In your C interfacing language using the [Wasmi C-API](https://github.com/wasmi-labs/wasmi/tree/main/crates/c_api#readme).
- 🧩 As [Wasmer](https://wasmer.io/) backend, using its [`wasmi` crate feature](https://github.com/wasmerio/wasmer/blob/v6.1.0/Cargo.toml#L224).
- 📦 Or enjoy Wasmi indirectly by using any of its [big users](https://github.com/wasmi-labs/wasmi#used-by).

## Special Thanks

- Again, a huge thank you to [Stellar Development Foundation (SDF)](https://stellar.org/foundation) for their generous sponsoring and support of the Wasmi project and its development. All this work would have not been possible without the SDF.
- I would like to express my gratitude to [Parity Technologies](https://www.parity.io/) for allowing me to turn Wasmi into a stand-alone [FOSS](https://en.wikipedia.org/wiki/Free_and_open-source_software) project.
- Finally, I would like to thank [E.J.P. Bruel](https://github.com/eddybruel). He is the author of [Stitch](https://github.com/makepad/stitch), an extremely well engineered high-performance Wasm interpreter. We randomly met at [RustWeek2025](https://2025.rustweek.org/) and are having the best nerd talks about Wasm interpreters for which I am really grateful. His marvellous work on Stitch inspired a lot of the techniques that are introduced with Wasmi 2.0.

[^1]: I verified this using `cargo show-asm` on a Macbook M2 Pro ARM machine at least for many of the `simd` operators.

[^2]: The full list of Wasmi support Wasm proposals can be found [here](https://github.com/wasmi-labs/wasmi?tab=readme-ov-file#webassembly-features).

[^3]: Currently we differentially fuzz against an older, much simpler version of Wasmi itself as well as the mature and battletested [Wasmtime](https://github.com/bytecodealliance/wasmtime) Wasm runtime.

[^4]: Before that, Wasmi's tests were tightly integrated with its translation engine. This necessitated updates to them whenever Wasmi's translation engine was changed. While those tightly integrated tests helped with early development, it is needless to say, that this process wasn't sustainable.

[^5]: Where Wasmi v0.32 required ~10 seconds to compile, we are now down to just ~4.5s. Tested on Macbook M2 Pro.

[^6]: That's why Rust's standard library does not provide `HashMap` in its `alloc` crate. Note that the `hashbrown` crate has `no_std` support but kinda cheats as its random initialization is not applicable to some systems and thus a potential attack vector for safety critical `no_std` systems.

[^7]: Note that the Coremark score of Wasmi v0.32 is significantly worse than in the [last blog post](https://wasmi-labs.github.io/blog/posts/wasmi-v0.32/#coremark). The reason for this is likely the different Rust compiler version. It is not uncommon for new compiler versions to drastically change performance metrics, especially if the underlying LLVM version changes.

[^8]: Note that Wasmi's `wat` crate feature is enabled by default.

[^9]: The author of this article is not a native english speaker. All mistakes contained in the article are his. In case of severe issues feel free to open a [pull request](https://github.com/wasmi-labs/blog/pulls).
