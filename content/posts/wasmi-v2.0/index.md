---
title: 'Wasmi 2.0 - Engineering of the Fastest Wasm Interpreters'
description: 'The architecture underlying Wasm3, Stitch and Wasmi 2.0.'
date: 2026-08-10T17:18:33+02:00
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
draft: true
---

In my last post about [Wasmi v1.0](/blog/posts/wasmi-v1.0/) I promised a [fundamental engine overhaul](/blog/posts/wasmi-v1.0/#the-next-gen-engine) for the future Wasmi version. The future is now!

Wasmi is an efficient and feature-rich [WebAssembly (Wasm)](https://webassembly.org/) interpreter.
It is an excellent choice for IoT devices, plugin systems ([Typst], [Zellij], [Josh]), cloud hosts, smart contracts ([Soroban], [Ripple]) and even for your lightweight game consoles ([Firefly Zero]).

Before going into all the details, a huge thank you to the [Stellar Development Foundation (SDF)](https://stellar.org/foundation) that sponsors the development of the Wasmi project since October 2024. Without their sponsorship, the Wasmi project wouldn’t be where it is today.

## Where Wasmi 2.0 Landed

But was all the work in the last 8 months worth the immense effort?

For this, I have benchmarked some of the fastest portable Wasm interpreters:

- [Wasm3](https://github.com/wasm3/wasm3)
- [WAMR's fast-interpreter](https://github.com/wasm-micro-runtime/wasm-micro-runtime)
- [Wasmtime's Pulley](https://github.com/bytecodealliance/wasmtime/blob/main/pulley/README.md)
- [Makepad's Stitch](https://github.com/makepad/stitch)
- [Wasmi 1.0](https://crates.io/crates/wasmi/1.0.9)

The benchmarks were conducted using the [`wasmi-benchmarks`] project which should make it possible
to easily reproduce them on your own machine.

I ran the benchmarks on 3 different hardware setups to make interpreter preferences visible:

### Apple M2 Pro

![][coremark-apple-m2-pro]

### AMD EPYC 7763

![][coremark-amd-epyc-7763]

### Intel Xeon Platinum 8370C

![][coremark-intel-xeon-platinum-8370c]

[coremark-apple-m2-pro]: ./resources/coremark/runtimes/apple-m2-pro.svg
[coremark-amd-epyc-7763]: ./resources/coremark/runtimes/amd-epyc-7763.svg
[coremark-intel-xeon-platinum-8370c]: ./resources/coremark/runtimes/intel-xeon-platinum-8370c.svg
[`wasmi-benchmarks`]: https://github.com/wasmi-labs/wasmi-benchmarks

Note that this is just a peek of the total benchmarks and runtimes supported
by the [`wasmi-benchmarks`] project but it provides a good overview.

It is fair to say that Wasmi 2.0 clearly belongs to the category of the fastest portable Wasm interpreters.
But there is even more to uncover in an upcoming blog post - stay tuned!

## What made Wasmi 2.0 so fast?

### New Modes of Instruction Dispatch

As promised in the original Wasmi 1.0 blog post Wasmi 2.0 now has 4 different modes of dispatching instructions:

| Mode | Description | Crate Features |
|:----:|:------------|:-----:|
| Direct-Threaded Code | The fastest configuration that is used by both, Wasm3 and Stitch. It embeds the function pointers directly into the internal IR of the interpreter and uses tail-calls to jump from instruction handler to the next. | - |
| Indirect-Threaded Code | Very similar to Direct-threaded Code but embeds op-codes into the internal IR and a jump table that maps those to function pointers upon jumping to the next instruction handler. Roughly 10-15% slower than Direct-Threaded Code but uses way less memory for its IR. | `indirect-dispatch` |
| Switch-Loop | This is the technique that Wasmi 1.0 used so far. It is the naive way to build interpreters using a loop and a switch (or match). Unfortunately, it leaves a lot of performance on the table, especially on Apple Silicon. | `portable-dispatch` + `indirect-dispatch` |
| Call-Loop | This calls the next instruction handler within a loop without tail-calls. Unforunately it is very slow and not memory efficient, therefore I cannot recommend using it. It just fell out of the configuration matrix. | `portable-dispatch` |

Wasmi users should use

- **Direct-Threaded Code:** if they want to maximize interpreter performance.
- **Indirect-Threaded-Code"** for a good balance between interpreter performance and memory usage.
- **Switch-Loop:** for running on platforms that do not support tail-calls.

Wasmi 2.0 ships the `auto-dispatch` that automatically uses tail-call based configurations where possible.

#### How Do Instruction Dispatch Modes Perform?

![][configs-apple-m2-pro]

[configs-apple-m2-pro]: ./resources/coremark/configs/apple-m2-pro.svg

> **Note:** CoreMark results for Direct-Threaded Code do not perfectly
>           match the ones from above since it was a different run and
>           we used the [`wasmi-coremark`] project instead.

[`wasmi-coremark`]: https://github.com/wasmi-labs/wasm-coremark-rs

Despite these extreme differences in performance all of these instruction dispatching
modes share the same interpreter execution logic and architecture under the hood.

If you are interested in how the instruction dispatch selection in Wasmi works in detail,
you can find the code here: [Wasmi Dispatch Selection]

[Wasmi Dispatch Selection]: https://github.com/wasmi-labs/wasmi/tree/v2.0.0-beta.10/crates/wasmi/src/engine/executor/handler/dispatch

### Execution Handler Signature

Before execution Wasmi translates the Wasm bytecode to Wasmi IR.

Each Wasmi IR instruction has its own unique instruction handler (or execution handler)
which defines how this particular instruction is executed when run.

In Wasmi 2.0 all instruction handlers share the same signature:

```rust
fn(
    store: &mut PrunedStore, // A reference to the `Store<T>` that is associated to the execution.
    ip: Ip,                  // The instruction pointer.
    sp: Sp,                  // The stack pointer.
    mem0: Mem0Ptr,           // The pointer to the data of the default linear memory: `(memory 0)`
    mem0_len: Mem0Len,       // The number of bytes of the default linear memory.
    instance: Inst,          // A pointer to the Wasm instance that is used by the function that is currently being executed.
    ireg: Ireg,              // Accumulator register for integer and reference values.
    freg32: Freg32,          // Accumulator register for `f32` values.
    freg64: Freg64,          // Accumulator register for `f64` values.
) -> Done;                   // State used to signal traps or successful halts.
```

- The `store` argument is basically a `Store<T>` that was pruned by its `T` type.
  This is important since instruction handlers are not allowed to be generic.
  The `store` is used for fuel metering, host calls, `memory.grow` and `table.grow` operations.
- The `ip` argument is the instruction pointer which tells the executor where in the stream
  of encoded instructions it is and which instruction it has to decode and execute.
- The `sp` argument is the position of the currently executed function within the value stack.
- The `mem0` and `mem0_len` arguments are used for optimized access to the default memory
  `(memory 0)`. This is very common in Wasm even when using the Wasm `multi-memory` proposal.
- The `instance` argument is used to load Wasm instance related objects such as globals, functions,
  tables, memories, data- and element segments. We will go into greater details later in the post.
- The `ireg`, `freg32` and `freg64` arguments are so-called accumulator registers which are
  used to store intermediate results in between instructions very efficiently.
  We will go into greater details later in the post.
- The `Done` result is just a bit pattern that tells the executor why execution halted.
  More detailed information is communicated via the `store` for later retrieval.

#### Problem: Calling Conventions

7 out of the 9 arguments in Wasmi's instruction handlers require integer registers (GPRs),
namely `store`, `ip`, `sp`, `mem0`, `mem0_len`, `instance` and `ireg`.

However, common calling conventions such as `sysv64` only provide up to 6.
A 7th integer argument would trash performance due to having to spill to the stack on every dispatch.
Both, Stitch and Wasm3 circumvent this issue by using only 6 and 4 GPRs respectively.

The simple solution is to turn one of Wasmi's GPR arguments into a floating point value.
The `instance` argument was chosen since it is used only for relatively expensive operations anyway.

Benchmarks show that the integer to float register domain move isn't a big deal.

> **Note:** the currently unstable [`preserve_none`] ABI might be able to improve this situation in the future once it becomes stable and available on more platforms.

[`preserve_none`]: https://github.com/rust-lang/rust/issues/151401

