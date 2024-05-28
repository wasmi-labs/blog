---
date: "2024-05-28T09:00:00+01:00"
title: "Wasmi's New Execution Engine - Faster Than Ever"
draft: false
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
---

After many months of research, development and QA, Wasmi's most significant update ever is finally ready for production use.

[Wasmi] is an efficient and versatile [WebAssembly] (Wasm) interpreter with a focus on embedded environments. It is an excellent choice for plugin systems, cloud hosts and as smart contract execution engine.

[Wasmi]: https://github.com/wasmi-labs/wasmi
[WebAssembly]: https://webassembly.org/

Wasmi intentionally mirrors the Wasmtime API on a best-effort basis, making it an ideal drop-in replacement or prototyping runtime.

> Install [Wasmi's CLI tool] via `cargo install wasmi_cli` or use it as library via the [`wasmi` crate].

[`wasmi` crate]: https://crates.io/crates/wasmi
[Wasmi's CLI tool]: https://crates.io/crates/wasmi_cli

Wasmi v0.32 comes with a new execution engine that utilizes register-based bytecode enhancing its execution performance by a factor of up to 5.
Additionally, its startup performance has been improved by several orders of magnitudes thanks to lazy compilation and other new techniques.

The [changelog for v0.32] is huge and the following sections will present the most significant changes.

[changelog for v0.32]: https://github.com/wasmi-labs/wasmi/releases/tag/v0.32.0

# Startup Performance

Wasmi is a *rewriting interpreter*, meaning that it rewrites the incoming WebAssembly bytecode into Wasmi's own internal bytecode that is geared towards efficient execution performance.

This re-writing is what we call *compilation* or *translation* in an interpreter. This is not to be confused with compiling the Wasmi interpreter itself.

### Why is translation speed important for Wasmi?

Fast translation enables a fast startup time which is the time spent until the first instruction is executed.

As an interpreter, Wasmi is naturally optimized for fast startup times, making it well-suited for translation-intensive workloads where the time required to translate a Wasm binary exceeds the time needed to execute it.  
Conversely, compute-intensive workloads, where execution time surpasses translation time, are better handled by JIT-based Wasm runtimes such as [Wasmtime], [WAMR], or [Wasmer]. [^10]

[Wasmtime]: https://github.com/bytecodealliance/wasmtime
[WAMR]: https://github.com/bytecodealliance/wasm-micro-runtime
[Wasmer]: https://github.com/wasmerio/wasmer
[Wasmi]: https://github.com/wasmi-labs/wasmi
[Wasm3]: https://github.com/wasm3/wasm3
[Wizard]: https://github.com/titzer/wizard-engine

### Lazy Translation

Translation can be costly, especially with the new register-based bytecode. To address this, lazy translation has been implemented, translating only the parts of the Wasm binary necessary for execution.

Wasmi supports 3 different modes of translation:

- `Eager`: Code is eagerly validated and eagerly translated ahead of time.
    - **Note:** This is the default mode for Wasmi v0.32.
- `Lazy`: Code is lazily translated and lazily validated.
    - **Note:** One downside is that this allows for partially validated Wasm modules which are controversial within the wider Wasm community. [^5] [^13]
- `LazyTranslation`: Code is lazily translated but eagerly validated.
    - **Note:** While slower than `Lazy` this fixes the problem with partially validated Wasm modules.

#### Usage as Library

```rust
let mut c = wasmi::Config::default();
c.compilation_mode(wasmi::CompilationMode::Lazy);
```

#### Usage in Wasmi's CLI

Wasmi CLI now supports the command-line option `--compilation-mode=<mode>` where `<mode>` is one of `eager`, `lazy`, or `lazy-translation`.

### Unchecked Translation

Wasmi validates the Wasm binary which accounts for roughly 20-40% of the total time spent during the startup phase.
However, some users might want to skip Wasm validation altogether since they know ahead of time that used Wasm binaries are pre-validated. This is now possible via the `unsafe fn Module::new_unchecked` API.

### Non-streaming Translation

Wasmi v0.31 and earlier always used streaming translation to process their Wasm input. However, in practice most users never even made use of this, so in v0.32 Wasmi uses non-streaming translation by default which gives it yet another nice performance win.

> Users who actually want to use streaming translation simply can use the new `Module::new_streaming` API for their needs.

### Linker Caching

The Wasmi `Linker` is used to define the set of host functions that a Wasm binary can use to communicate with the host. Oftentimes, dozens of host functions are defined, which can quickly become costly.

To address this, Wasmi now offers a `LinkerBuilder`, which allows to efficiently instantiate new `Linker`s after the initial setup. [^7]

Benchmarks with 50 defined host functions have demonstrated a 120x speedup using this approach.

## Benchmarks

By combining all of the techniques above it is possible to speed up the startup time of Wasmi by several orders of magnitudes compared to the previous Wasmi v0.31.

The newest versions of all Wasm runtimes have been used at the time of writing this article. [^9].

> Currently, Winch only supports `x86_64` platforms and therefore was only tested on those systems.

### ERC-20 - 7KB

| | |
|-|-|
| [![][compile-erc20-epyc]][compile-erc20-epyc] | [![][compile-erc20-tr]][compile-erc20-tr] |
| [![][compile-erc20-m2]][compile-erc20-m2] | [![][compile-erc20-i7]][compile-erc20-i7] |

[compile-erc20-epyc]: ./benches/linux-amd-epyc-7763/compile/erc20.svg
[compile-erc20-tr]: ./benches/linux-amd-threadripper-3990x/compile/erc20.svg
[compile-erc20-m2]: ./benches/macos-m2/compile/erc20.svg
[compile-erc20-i7]: ./benches/windows-intel-i7-14700k/compile/erc20.svg

### Argon2 - 61KB

| | |
|-|-|
| [![][compile-argon2-epyc]][compile-argon2-epyc] | [![][compile-argon2-tr]][compile-argon2-tr] |
| [![][compile-argon2-m2]][compile-argon2-m2] | [![][compile-argon2-i7]][compile-argon2-i7] |

[compile-argon2-epyc]: ./benches/linux-amd-epyc-7763/compile/argon2.svg
[compile-argon2-tr]: ./benches/linux-amd-threadripper-3990x/compile/argon2.svg
[compile-argon2-m2]: ./benches/macos-m2/compile/argon2.svg
[compile-argon2-i7]: ./benches/windows-intel-i7-14700k/compile/argon2.svg

### BZ - 147KB

| | |
|-|-|
| [![][compile-bz2-epyc]][compile-bz2-epyc] | [![][compile-bz2-tr]][compile-bz2-tr] |
| [![][compile-bz2-m2]][compile-bz2-m2] | [![][compile-bz2-i7]][compile-bz2-i7] |

[compile-bz2-epyc]: ./benches/linux-amd-epyc-7763/compile/bz2.svg
[compile-bz2-tr]: ./benches/linux-amd-threadripper-3990x/compile/bz2.svg
[compile-bz2-m2]: ./benches/macos-m2/compile/bz2.svg
[compile-bz2-i7]: ./benches/windows-intel-i7-14700k/compile/bz2.svg

### Pulldown-Cmark - 1.6MB

| | |
|-|-|
| [![][compile-pulldown-epyc]][compile-pulldown-epyc] | [![][compile-pulldown-tr]][compile-pulldown-tr] |
| [![][compile-pulldown-m2]][compile-pulldown-m2] | [![][compile-pulldown-i7]][compile-pulldown-i7] |

[compile-pulldown-epyc]: ./benches/linux-amd-epyc-7763/compile/pulldown-cmark.svg
[compile-pulldown-tr]: ./benches/linux-amd-threadripper-3990x/compile/pulldown-cmark.svg
[compile-pulldown-m2]: ./benches/macos-m2/compile/pulldown-cmark.svg
[compile-pulldown-i7]: ./benches/windows-intel-i7-14700k/compile/pulldown-cmark.svg

### Spidermonkey - 4.2MB

| | |
|-|-|
| [![][compile-spidermonkey-epyc]][compile-spidermonkey-epyc] | [![][compile-spidermonkey-tr]][compile-spidermonkey-tr] |
| [![][compile-spidermonkey-m2]][compile-spidermonkey-m2] | [![][compile-spidermonkey-i7]][compile-spidermonkey-i7] |

[compile-spidermonkey-epyc]: ./benches/linux-amd-epyc-7763/compile/spidermonkey.svg
[compile-spidermonkey-tr]: ./benches/linux-amd-threadripper-3990x/compile/spidermonkey.svg
[compile-spidermonkey-m2]: ./benches/macos-m2/compile/spidermonkey.svg
[compile-spidermonkey-i7]: ./benches/windows-intel-i7-14700k/compile/spidermonkey.svg

### FFMPEG - 19.3MB

> **Note:** Wasmtime (Cranelift) timed out and
> Stitch failed to compile `ffmpeg.wasm`.

| | |
|-|-|
| [![][compile-ffmpeg-epyc]][compile-ffmpeg-epyc] | [![][compile-ffmpeg-tr]][compile-ffmpeg-tr] |
| [![][compile-ffmpeg-m2]][compile-ffmpeg-m2] | [![][compile-ffmpeg-i7]][compile-ffmpeg-i7] |

[compile-ffmpeg-epyc]: ./benches/linux-amd-epyc-7763/compile/ffmpeg.svg
[compile-ffmpeg-tr]: ./benches/linux-amd-threadripper-3990x/compile/ffmpeg.svg
[compile-ffmpeg-m2]: ./benches/macos-m2/compile/ffmpeg.svg
[compile-ffmpeg-i7]: ./benches/windows-intel-i7-14700k/compile/ffmpeg.svg

### Translation Benchmarks: Conclusion

Wasmi and Wasm3 perform best by far due to their lazy compilation capabilities. As expected, optimizing JIT-based Wasm runtimes like Wasmtime and Wasmer perform worse in this context. Single-pass JITs, which are designed for fast startup, such as [Winch] and [Wasmer Singlepass], are also significantly slower. Despite also using lazy translation, [Stitch]'s translation performance is not ideal. However, it is important to note that both Winch and Stitch are still in an experimental phase of their development and improvements are to be expected.

[Stitch]: https://github.com/makepad/stitch
[Winch]: https://crates.io/crates/wasmtime-winch
[Wasmer Singlepass]: https://crates.io/crates/wasmer-compiler-singlepass

## Execution Speed

For an execution engine, the speed of computation is naturally of paramount importance. Unfortunately, the old Wasmi v0.31 left much to be desired in this regard.

## Register-Based Bytecode

The old Wasmi v0.31 internally uses a stack-based intermediate representation (IR) to drive execution. This IR is similar to WebAssembly bytecode and thus allows for fast translation times.

Stack-based IRs generally use more instructions to represent the same problem as register-based IRs. [^4] However, the performance of interpreters is mostly dictated by the dispatch of instructions. Hence, it is usually a good tradeoff to execute fewer instructions even if every executed instruction is more complex. [^6]

This is why starting with version 0.32 Wasmi now uses a register-based IR to drive its execution.

## Memory Consumption

The new register-based IR was carefully designed to enhance execution performance and to minimize memory usage. As the vast majority of a Wasm binary is comprised of encoded instructions, this substantially decreases memory usage and enhances cache efficiency when executing Wasm through Wasmi. [^8]

## Benchmarks

The newest versions of all Wasm runtimes have been used at the time of writing this article. [^9].

### Fibonacci (Iterative) - Compute Intense

| | |
|-|-|
| [![][execute-fib.iter-epyc]][execute-fib.iter-epyc] | [![][execute-fib.iter-tr]][execute-fib.iter-tr] |
| [![][execute-fib.iter-m2]][execute-fib.iter-m2] | [![][execute-fib.iter-i7]][execute-fib.iter-i7] |

[execute-fib.iter-epyc]: ./benches/linux-amd-epyc-7763/execute/fib.iterative.svg
[execute-fib.iter-tr]: ./benches/linux-amd-threadripper-3990x/execute/fib.iterative.svg
[execute-fib.iter-m2]: ./benches/macos-m2/execute/fib.iterative.svg
[execute-fib.iter-i7]: ./benches/windows-intel-i7-14700k/execute/fib.iterative.svg

### Fibonacci (Recursive) - Call Intense

| | |
|-|-|
| [![][execute-fib.rec-epyc]][execute-fib.rec-epyc] | [![][execute-fib.rec-tr]][execute-fib.rec-tr] |
| [![][execute-fib.rec-m2]][execute-fib.rec-m2] | [![][execute-fib.rec-i7]][execute-fib.rec-i7] |

[execute-fib.rec-epyc]: ./benches/linux-amd-epyc-7763/execute/fib.recursive.svg
[execute-fib.rec-tr]: ./benches/linux-amd-threadripper-3990x/execute/fib.recursive.svg
[execute-fib.rec-m2]: ./benches/macos-m2/execute/fib.recursive.svg
[execute-fib.rec-i7]: ./benches/windows-intel-i7-14700k/execute/fib.recursive.svg

Wasmi v0.32 has not significantly improved over v0.31 in this test case. This is partly because Wasmi v0.31 was already comparatively fast and that the new register-based bytecode favors compute-intense workloads over call-intense ones. This usually is a good tradeoff since most Wasm producers (such as LLVM) produce compute intense workloads due to aggressive inlining.

### Primes - Balanced

| | |
|-|-|
| [![][execute-primes-epyc]][execute-primes-epyc] | [![][execute-primes-tr]][execute-primes-tr] |
| [![][execute-primes-m2]][execute-primes-m2] | [![][execute-primes-i7]][execute-primes-i7] |

[execute-primes-epyc]: ./benches/linux-amd-epyc-7763/execute/primes.svg
[execute-primes-tr]: ./benches/linux-amd-threadripper-3990x/execute/primes.svg
[execute-primes-m2]: ./benches/macos-m2/execute/primes.svg
[execute-primes-i7]: ./benches/windows-intel-i7-14700k/execute/primes.svg

### Matrix Multiplication - Memory Intense

| | |
|-|-|
| [![][execute-matmul-epyc]][execute-matmul-epyc] | [![][execute-matmul-tr]][execute-matmul-tr] |
| [![][execute-matmul-m2]][execute-matmul-m2] | [![][execute-matmul-i7]][execute-matmul-i7] |

[execute-matmul-epyc]: ./benches/linux-amd-epyc-7763/execute/matmul.svg
[execute-matmul-tr]: ./benches/linux-amd-threadripper-3990x/execute/matmul.svg
[execute-matmul-m2]: ./benches/macos-m2/execute/matmul.svg
[execute-matmul-i7]: ./benches/windows-intel-i7-14700k/execute/matmul.svg

Interestingly, Wasmer (Singlepass) seems to have some trouble on Apple silicon being even slower than some of the interpreters.

### Argon - Compute Hash

> **Note:** Stitch and Winch could not execute the `argon2.wasm` test case.

| | |
|-|-|
| [![][execute-argon2-epyc]][execute-argon2-epyc] | [![][execute-argon2-tr]][execute-argon2-tr] |
| [![][execute-argon2-m2]][execute-argon2-m2] | [![][execute-argon2-i7]][execute-argon2-i7] |

[execute-argon2-epyc]: ./benches/linux-amd-epyc-7763/execute/argon2.svg
[execute-argon2-tr]: ./benches/linux-amd-threadripper-3990x/execute/argon2.svg
[execute-argon2-m2]: ./benches/macos-m2/execute/argon2.svg
[execute-argon2-i7]: ./benches/windows-intel-i7-14700k/execute/argon2.svg

### Coremark

The following table shows Coremark scores for the Wasm interpreters by CPU. [^12]

| | AMD Epyc 7763 | AMD Threadripper 3990x | Apple M2 Pro | Intel i7 14700K |
|-:|-:|-:|-:|-:|
| Wasmi v0.31 | 657 | 944 | 884 | 1759 |
| Wasmi v0.32 | **1457** | 1779 | 1577 | 2979 |
| Tinywasm | 235 | 339 | 592 | 772 |
| Wasm3 | 1309 | 1999 | 2931 | 3831 |
| Stitch | 1390 | **2187** | **3056** | **4892** |

### Execution Benchmarks: Conclusion

Wasmi is especially strong on AMD server chips and lacks behind on Apple silicon. An explanation for this could be the difference in the technique for instruction dispatch being used. [^1]

The Stitch interpreter performs really well. The reason likely is that Stitch encourages the LLVM optimizer to produce tail calls for its instruction dispatch, despite Rust not supporting them. Due to various downsides this design decision was discussed and dismissed during the development of Wasmi v0.32. [^2] [^3] Given Stitch's impressive execution performance especially on Apple silicon and Windows platforms those decisions should be reevaluated again.

Confusingly the great results for Wasmi on the test cases for the Intel i7 14700K are not reflected by its Coremark score. This probably is because every test case, including Coremark, is biased towards some kinds of workloads to some degree.

## Benchmark Suite

The benchmarks and plots above have been gathered and generated using the [`wasmi-benchmarks`] repository.
The reader is encouraged to run the benchmarks and plot the results on their own computer to confirm or disprove the claims. Usage instructions can be found in the [`wasmi-benchmarks`'s `README.md`].

[`wasmi-benchmarks`'s `README.md`]: https://github.com/wasmi-labs/wasmi-benchmarks?tab=readme-ov-file#plotting

Contributions adding more Wasm runtimes, improving the plots or to add new test cases are welcome!

[`wasmi-benchmarks`]: https://github.com/wasmi-labs/wasmi-benchmarks

# Summary & Outlook

This article displayed the highlights of the new Wasmi version 0.32 and demonstrated the significant improvements in both startup and execution performance through various test cases.

With this new major update Wasmi now has a solid foundation for future development.
Many [WebAssembly proposals] such as the `multi-memory`, `simd` and `gc` proposals that have been put on hold for the development of Wasmi v0.32 are awaiting their implementation.

[WebAssembly proposals]: https://github.com/WebAssembly/proposals

The promising results especially on the AMD server chips are a decent indicator that
Wasmi has great potential.
The performance of Wasmi on Apple silicon will be improved in future releases.

Plans are underway to implement the [Wasm C-API], enabling various ecosystems that can interface with C to use Wasmi as a library.

[Wasm C-API]: https://github.com/WebAssembly/wasm-c-api

Wasmi will continue to solidify its position as an efficient and versatile Wasm interpreter
with a fantastic startup performance and low memory consumption especially suited for embedded environments.

# Special Thanks

- First and foremost, I want to thank [Parity Technologies] for financing and supporting the development of Wasmi for such a long time and for allowing Wasmi to become an independent project.

[Parity Technologies]: https://www.parity.io/

- I want to commend the members of the [Bytecode Alliance] for their outstanding efforts in shaping the WebAssembly specification and ecosystem. Their contributions, among others, include runtimes such as [Wasmtime] and [WAMR] as well as [advanced WebAssembly tooling].

[advanced WebAssembly tooling]: https://github.com/bytecodealliance/wasm-tools

[Bytecode Alliance]: https://bytecodealliance.org/

- Additionally, I want to extend my gratitude to [OLUWAMUYIWA], who dedicated their time and effort to implement WASI preview1 support for Wasmi — which was absolutely amazing!

[OLUWAMUYIWA]: https://github.com/OLUWAMUYIWA

- Furthermore, I want to thank [yamt], who inspired me with their Wasm runtime benchmarking platform. I highly recommend checking out their [toywasm] Wasm interpreter!

[yamt]: https://github.com/yamt
[toywasm]: https://github.com/yamt/toywasm

- Finally, I would like to acknowledge [Neopallium] for the thought-provoking discussions and experiments we shared about efficient interpreter dispatching techniques in Rust. I highly recommend checking out one of his Wasm experiments, [s1vm].

[Neopallium]: https://github.com/Neopallium
[s1vm]: https://github.com/Neopallium/s1vm

[^1]: An explanation for Wasmi's inferior performance on Apple silicon is the loop-switch dispatch that is a black box concerning the generated machine code and heavily depends on heuristics in the optimizer. Recently, Apple announced enhancements in their branch prediction for their latest M4 chips which could significantly affect Wasmi's performance since efficient interpreters heavily rely on well-tuned processor branch prediction. [^11]

[^11]: [The Structure and Performance of Efficient Interpreters](https://jilp.org/vol5/v5paper12.pdf) by Ertl. et. al.

[^2]: The preferred solution is to finally implement [explicit tail calls for Rust]. This has been an ongoing topic of discussion for years and was proposed multiple times. It is more than evident that explicit tail calls, while niche, allow for specific performance critical program designs.

[explicit tail calls for Rust]: https://github.com/rust-lang/rfcs/pull/3407

[^3]: The downsides of Stitch's approach are well documented in its own README. The main issue is its reliance on LLVM's optimizer to produce the correct code on all platforms which is likely but not guaranteed. If LLVM does not produce the correct code, Wasmi will be slow but Stitch will not even work.

[^4]:
    A simple example for this is the translation of the following Wasm bytecode:

    ```wasm
    local.get 0
    local.get 1
    i32.add
    local.set 0
    ```

    Which adds locals at index 0 and 1 and stores the result back into the local at index 0.
    Wasmi v0.32 translates this bytecode to a single Wasmi IR instruction:

    ```wasm
    0 <- i32.add 0 1
    ```
    
    Thus reducing the amount of instructions needed to be executed from 4 down to 1.

[^5]: For more information see [this GitHub issue](https://github.com/WebAssembly/design/issues/1464).

[^6]: The new translation from stack-based to register-based bytecode is a complex and interesting topic that might warrant its own article if there is enough interest in it.

[^7]:
    An example code snippet for how to use the new `LinkerBuilder` is the following:
    ```rust
    fn test() {
        let mut builder = <Linker<()>>::build();
        // Populate the linker with the desired host functionality:
        builder
            .func_wrap("env", "foo", |_caller: Caller<()>| println!("called foo))
            .unwrap();
        builder
            .func_wrap("env", "bar", |_caller: Caller<()>| println!("called bar))
            .unwrap();
        let builder = builder.finish();
    }
    ```
    Now `builder` can be used to quickly spawn new `Linker`'s with the predefined
    set of host functions.
    ```rust
    let engine = Engine::default();
    let linker = builder.create(&engine); // FAST!
    ```

[^8]:
    A benchmark for startup and memory consumption, albeit somewhat outdated, can be found in the [toywasm benchmarks]. Significant improvements have been made to Wasmi since those benchmarks were conducted, so one should to take those numbers with a grain of salt.

    [toywasm benchmarks]: https://github.com/yamt/toywasm/blob/master/benchmark/startup.md

[^9]:
    The following versions have been used for the tested Wasm runtimes:

    | Runtime | Version |
    |:-|:-|
    | Wasmi v0.31 | v0.31.2 |
    | Wasmi v0.32 | v0.32.0-beta.18 |
    | Tinywasm | v0.7.0 |
    | Wasm3 | v0.5.0 |
    | Stitch | v0.1.0 |
    | Wasmtime / Winch | v20.0 |
    | Wasmer | v4.3 |

[^10]:
    There are basically two kinds of Wasm workloads:

    1. **Compute-Intense:** The time to execute the Wasm binary exceeds the time to translate it.
        - This use case is best covered by a JIT based Wasm runtime such as [Wasmtime], [WAMR] or [Wasmer].
    2. **Translation-Intense:** The time to translate the Wasm binary exceeds the time to execute it.
        - This use case is best covered by a Wasm runtime that optimizes for fast startup times such as [Wasmi], [Wasm3] or [Wizard].
    
    If it is unclear whether a workload is translation-intensive or compute-intensive, it may be beneficial to use a Wasm runtime that balances both types of workloads. Examples include [Wasmtime's Winch] or [Wasmer's Singlepass] JIT, which are designed to handle a mix of translation and execution demands effectively.

    [Wasmtime's Winch]: https://crates.io/crates/wasmtime-winch
    [Wasmer's Singlepass]: https://crates.io/crates/wasmer-compiler-singlepass

[^12]:
    For Wasmi and Wasm3 both `lazy` and `eager` modes resulted in nearly the same scores. This is because Coremark is a long-running task where the impact of lazy translation is greatly reduced. The table displays the higher score among the different modes.

[^13]: Another downside is that some Wasm runtime limitations, for example the maximum number of bytes per Wasm function, may not be checked when using lazy function translation.
