---
title: 'Wasmi 2.0 - Engineering of the Fastest Wasm Interpreters'
description: 'The optimizations behind Wasmi 2.0, and how they compare to Wasm3 and Stitch.'
date: 2026-08-10T17:18:33+02:00
author: 'Robin Freyler'
authorURL: 'https://github.com/robbepop'
draft: true
---

In my last post about [Wasmi v1.0](/blog/posts/wasmi-v1.0/) I promised a [fundamental engine overhaul](/blog/posts/wasmi-v1.0/#the-next-gen-engine) for the future Wasmi version. The future is now! [^intro]

Wasmi is an efficient and feature-rich [WebAssembly (Wasm)](https://webassembly.org/) interpreter.
It is an excellent choice for IoT devices, plugin systems ([Typst], [Zellij], [Josh]), cloud hosts, smart contracts ([Soroban], [Ripple]) and even for your lightweight game consoles ([Firefly Zero]).

Before going into all the details, a huge thank you to the [Stellar Development Foundation (SDF)](https://stellar.org/foundation) that has been sponsoring the Wasmi project since October 2024. Without their sponsorship, the Wasmi project wouldn’t be where it is today.
Also special thanks to [Felix Kutzner](https://github.com/fkutzner) for proofreading the article and suggesting many improvements.

## Wasmi 2.0 Release

Today, I am happy to announce that after eight months of focused work, Wasmi 2.0 is finally done and ready to use.

This release focuses on execution performance: Wasmi 2.0 runs ~2.2x faster than Wasmi 1.0 in geometric mean across the [`wasmi-benchmarks`] suite on an Apple M2 Pro.

Wasmi 2.0 also ships new knobs, such as the `validate` crate feature, that significantly reduce its binary artifact size. [^cargo-bloat-show]
Some user-requested features, such as stable fuel metering [^explain-stable-metering], support for [WebAssembly's deterministic profile][wasm-deterministic-profile] and an improved [Wasmi CLI] tool, also made it into this release.

- [**View Changelog**](https://github.com/wasmi-labs/wasmi/releases/tag/v2.0.0)
- [**View Migration Guide**](https://github.com/wasmi-labs/wasmi/tree/v2.0.0/docs/migration-v1-to-v2.md)

[wasm-deterministic-profile]: https://github.com/WebAssembly/profiles/blob/main/proposals/profiles/Overview.md
[Wasmi CLI]: https://crates.io/crates/wasmi_cli

## Where Wasmi 2.0 Landed

2.2x faster than Wasmi 1.0 is great, but how does Wasmi 2.0 fare against its competition?

For this, I have benchmarked Wasmi 2.0 against some of the fastest portable Wasm interpreters: [^benches-runtimes]

- [Wasm3](https://github.com/wasm3/wasm3)
- [WAMR fast-interpreter](https://github.com/wasm-micro-runtime/wasm-micro-runtime)
- [Wasmtime Pulley](https://github.com/bytecodealliance/wasmtime/blob/main/pulley/README.md)
- [Makepad Stitch](https://github.com/makepad/stitch)
- [Wasmi 1.0](https://crates.io/crates/wasmi/1.0.9)

> **Note:** Wasmi 2.0 was inspired by all the interpreter above!

The benchmarks were conducted using the [`wasmi-benchmarks`] project which should make it possible
to easily reproduce them on your own machine.

I ran the benchmarks on three different hardware setups to make interpreter preferences visible:

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

### Geometric Mean

The following two plots show the [geometric mean] across all `execute` and all `startup`
benchmarks of the [`wasmi-benchmarks`] suite.

[geometric mean]: https://en.wikipedia.org/wiki/Geometric_mean

![][execute-geomean-apple-m2-pro]

![][startup-geomean-apple-m2-pro]

> **Note:** I had to use logarithmic scaling for `startup` because Wasmtime Pulley is quite an outlier. [^why-pulley-outlier]

Despite the focus on execution performance in Wasmi 2.0,
its startup performance is still outstanding and mostly on par with its previous version. [^wasmi-configs]

[execute-geomean-apple-m2-pro]: ./resources/geomean/execute.svg
[startup-geomean-apple-m2-pro]: ./resources/geomean/startup.svg

### Conclusion: Benchmarks

It is fair to say that Wasmi 2.0 clearly belongs to the category of the fastest portable Wasm interpreters.

In a follow-up article I will present all the results and findings of the [`wasmi-benchmarks`] suite
and put each of its many supported Wasm runtimes into the spotlight it deserves.

## What made Wasmi 2.0 so fast?

> **Note:** This section assumes a basic understanding of Wasm and interpreters.

### New Modes of Instruction Dispatch

As promised in the original Wasmi 1.0 blog post, Wasmi 2.0 now has four different modes of dispatching instructions:

| Mode | Description | Crate Features |
|:----:|:------------|:-----:|
| Direct-Threaded Code | The fastest configuration that is used by both Wasm3 and Stitch. It embeds the function pointers directly into the internal IR of the interpreter and uses tail calls to jump from one instruction handler to the next. | - |
| Indirect-Threaded Code | Very similar to Direct-Threaded Code, but embeds op-codes into the internal IR and uses a jump table to map an op-code to its instruction handler's function pointer upon dispatch. Roughly 10-15% slower than Direct-Threaded Code, but uses significantly less memory for its IR. | `indirect-dispatch` |
| Switch-Loop | This is the technique used in Wasmi 1.0. It is the naive way to build interpreters using a loop and a switch (or match). Unfortunately, it leaves a lot of performance on the table, especially on Apple Silicon. | `portable-dispatch` + `indirect-dispatch` |
| Call-Loop | This calls the next instruction handler within a loop without tail calls. Unfortunately, it is very slow and not memory efficient, therefore I cannot recommend using it. It exists only because `portable-dispatch` and `indirect-dispatch` are independent crate features, so this combination simply falls out of the configuration matrix. | `portable-dispatch` |

Wasmi users should use

- **Direct-Threaded Code:** if they want to maximize interpreter performance.
- **Indirect-Threaded Code:** for a good balance between interpreter performance and memory usage.
- **Switch-Loop:** for running on platforms that do not support tail calls.

Wasmi 2.0 ships the `auto-dispatch` crate feature that automatically uses [threaded-code]-based configurations where possible. [^nightly-become]

[threaded-code]: https://en.wikipedia.org/wiki/Threaded_code

#### How Do Instruction Dispatch Modes Perform?

![][configs-apple-m2-pro]

[configs-apple-m2-pro]: ./resources/coremark/configs/apple-m2-pro.svg

> **Note:** CoreMark results for Direct-Threaded Code do not perfectly
>           match the ones from above since it was a different run and
>           we used the [`wasm-coremark-rs`] project instead.

[`wasm-coremark-rs`]: https://github.com/wasmi-labs/wasm-coremark-rs

Despite these extreme differences in performance, all of these instruction dispatching
modes share the same interpreter execution logic and architecture under the hood.

If you are interested in how the instruction dispatch selection in Wasmi works in detail,
you can find the code here: [Wasmi Dispatch Selection]

[Wasmi Dispatch Selection]: https://github.com/wasmi-labs/wasmi/tree/v2.0.0/crates/wasmi/src/engine/executor/handler/dispatch

### Execution Handler Signature

Before execution, Wasmi translates the Wasm bytecode to Wasmi IR.

Each Wasmi IR instruction has its own instruction handler (or execution handler)
which defines how the instruction is executed.

In Wasmi 2.0, all instruction handlers share the same signature:

```rust
fn(
    store: &mut PrunedStore, // A reference to the `Store<T>` that is associated to the execution.
    ip: Ip,                  // The instruction pointer.
    sp: Sp,                  // The stack pointer.
    mem0: Mem0Ptr,           // The pointer to the data of the default linear memory: `(memory 0)`
    mem0_len: Mem0Len,       // The number of bytes of the default linear memory.
    instance: Inst,          // A pointer to the Wasm instance that is used by the currently executed function.
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
  tables, memories, data and element segments. We will go into greater details later in the post.
- The `ireg`, `freg32` and `freg64` arguments are so-called accumulator registers which are
  used to efficiently store intermediate results between instructions.
  We will go into greater details later in the post.
- The `Done` result is just a bit pattern that tells the executor why execution halted.
  More detailed information is communicated via the `store` for later retrieval.

#### Problem: Calling Conventions

7 out of the 9 arguments in Wasmi's instruction handlers require passing their values in
general-purpose registers (GPRs), namely `store`, `ip`, `sp`, `mem0`, `mem0_len`, `instance` and `ireg`.

However, common calling conventions such as `sysv64` only provide up to 6 GPRs for integer arguments.
A 7th integer argument would trash performance because it would have to be spilled to the stack on every dispatch.
Both Stitch and Wasm3 circumvent this issue by using only 6 and 4 GPRs respectively.

The simple solution is to turn one of Wasmi's GPR arguments into a floating-point value where necessary.
The `instance` argument was chosen since it is used only for relatively expensive operations anyway.

Benchmarks show that the integer-to-float register domain move isn't a big deal.

> **Note:** the currently unstable [`preserve_none`] ABI might be able to improve this situation in the future once it becomes stable and available on more platforms.

[`preserve_none`]: https://github.com/rust-lang/rust/issues/151401

### Accumulator Registers

#### How Wasmi 1.0 Worked

Wasmi 1.0 pervasively uses stack offsets (stack slots) for operands and results of IR instructions.

A simplified `i64.add` instruction handler computing `res = lhs + rhs` is shown below, where
`res` and `lhs` are stack slots, and `rhs` is an immediate `i64` value:

```rust
fn i64_add(ip: Ip, sp: Sp, ..) -> Done {
    let res: Slot = decode_slot(ip);
    let lhs: Slot = decode_slot(ip);
    let rhs:  i64 = decode_i64(ip);
    let lhs:  i64 = lhs.load(sp);
    let sum:  i64 = lhs + rhs;
    res.store(sp, sum);
    ip.offset(encode_size::<i64_add>);
    next!(ip, sp, ..)
}
```

1. Decode the result `Slot` from `ip`.
2. Decode the left-hand side `lhs` operand `Slot` from `ip`.
3. Decode the right-hand side `rhs: i64` operand from `ip`.
4. Load the value from `lhs`. (`sp[lhs]`)
5. Compute the sum `sp[lhs] + rhs`.
6. Store the sum into `sp[result]`.
7. Offset `ip` to point to the next instruction handler.
8. Execute the next instruction handler. [^why-next-macro]

#### How Wasmi 2.0 Works

Wasmi 2.0 introduced the three new accumulator registers: `ireg`, `freg32` and `freg64`.
This allows Wasmi 2.0 to load and store instruction operands and results from and to actual hardware registers.

A simplified `i64.add` example that computes `res = lhs + rhs` where `res` and `lhs` refer to the `ireg` accumulator
and `rhs` is a `i64` immediate value would look like this:

```rust
fn i64_add(ip: Ip, sp: Sp, ireg: i64, ..) -> Done {
    let rhs: i64 = decode_i64(ip);
    ireg = ireg + rhs;
    ip.offset(encode_size::<i64_add>);
    next!(ip, ireg, ..)
}
```

1. Decode the right-hand side `rhs: i64` operand from `ip`.
2. Compute the sum `ireg + rhs`.
3. Store the sum into `ireg`.
4. Offset `ip` to point to the next instruction handler.
5. Execute the next instruction handler.

This is notably simpler and more efficient since the accumulator registers are always
implicit and need no costly decoding, loading or storing of values.

This is what it looks like compiled to `aarch64` assembly with `x1 = ip` and `x6 = ireg`:

```asm
i64_add_rri:
    ldr x7, [x1, #16]!   ; bump ip by 16 bytes and load next handler
    ldur x8, [x1, #-8]   ; fetch rhs immediate operand from ip
    add x6, x8, x6       ; ireg = ireg + rhs
    br x7                ; tail-call next handler
```

### Copy Instructions

Instructions in Wasmi 2.0 usually store their result into the implicit accumulator register.

The drawback is that this design requires copy instructions that the old design did not need.

#### Example: `local.set` & `local.tee`

```lisp
local.get 0   ;; push (local 0)
i32.const 10  ;; push 10
i32.add       ;; pop 2 operands, push their sum
local.set 1   ;; pop sum, store into (local 1)
```

This Wasm sequence adds `(local 0) + 10` and stores the result into `(local 1)`.
Wasmi 1.0 could do all of this in a single Wasmi 1.0 IR instruction:

```lisp
i32_add_ssi 1 0 10
```

> **Note:** we use the suffixes `s` for stack slots, `r` for accumulator registers and `i` for immediates.

Wasmi 2.0 requires two instructions for the same job:

```lisp
i32_add_rsi 0 10 ;; ireg = (local 0) + 10
u64_copy_sr 1    ;; (local 1) = ireg
```

#### Example: Register Preservation

```lisp
local.get 0   ;; push (local 0)
i32.const 10  ;; push 10
i32.add       ;; pop 2 operands, push their sum
local.get 1   ;; push (local 1) on top of the pending sum
i32.const 20  ;; push 20
i32.mul       ;; pop 2 operands, push their product
```

Wasmi 2.0 has to translate this to roughly the following Wasmi 2.0 IR:

```lisp
i32_add_rsi 0 10 ;; ireg = (local 0) + 10
u64_copy_sr A    ;; (slot A) = ireg
i32_mul_rsi 1 20 ;; ireg = (local 1) * 20
```

The `u64_copy_sr A` instruction is required because we overwrite `ireg` in the next instruction
without actually using it, thus we need to preserve the previous value of `ireg`.

#### Solution 1: More Efficient Copies

As demonstrated, Wasmi 2.0 requires many more copy instructions due to this design.
There are some effective ways to reduce their number, and for the remaining copies, some patterns emerged.

Wasmi 2.0 introduced optimized IR instructions for common situations:

- `u64_copy_sNr`: copies `ireg` to a fixed `(local N)` where `N = 0..10`
- `f32_copy_sNr`: copies `freg32` to a fixed `(local N)` where `N = 0..10`
- `f64_copy_sNr`: copies `freg64` to a fixed `(local N)` where `N = 0..10`
- `u64_copy_sNsM`: copies `(local M)` to `(local N)` where `N,M = 0..5` and `N != M`
    - Variants for `freg32` and `freg64` are not required since stack slots are always treated as generic 64-bit patterns.

#### Solution 2: Op-Code Fusion

For some reason, Wasm produces lots of `add` and `load` instructions that are immediately followed by `local.set` or `local.tee`.

This is so common that it was worth introducing special
fused variants which store their results not only in the
accumulator register but also into a stack slot so that
Wasmi 2.0 can represent those use-cases with a single IR instruction.

### Accumulators Across Control Flow Boundaries

For control flow, WebAssembly uses `block`, `loop` and `if` frames.

A `br` (branch), `br_if` (conditional branch), or `br_table` (branch table) instruction
can be used to either jump to the end of a `block` or `if` or to continue a `loop`.

Control flow in WebAssembly is organized as a stack,
so branches use a stack depth to which they jump where a depth of 0
jumps to the label of their direct parent.

Additionally, control flow can have parameters and results, and Wasmi 2.0
carries accumulator registers across those boundaries if possible.

- If a `block` for example has the result types `(i32 f32)`, Wasmi 2.0
  uses both `ireg` and `freg32` to return its results.
- If a `block` has `(i32, i32, i32)` result types, Wasmi 2.0 only puts the
  last result in `ireg` and returns the other results in stack slots.
- For technical reasons, accumulator registers are only used for the tail of results.
  For example, a `block` with results `(f32 i32 i32)` only puts the
  last `i32` into `ireg` and all other results into stack slots.

The same rules apply to `if` results and `loop` parameters.

For `loop`, this may allow induction variables to stay in accumulator registers.
An example for this can be seen in the `execute/counter-param` test case of [`wasmi-benchmarks`].

> **Note:** We experimented with applying the same rules to calls but
unfortunately this led to performance regressions, so it was not merged.

### Instance Object Access

#### How Wasmi 1.0 Worked

Wasmi 1.0 uses a naive way to model the internal object representation of Wasm module instances.
An instance (or `InstanceEntity`) uses one heap allocation per type of instance object, e.g. for
memories, tables, functions, globals, data or element segments. An `InstanceEntity` only holds
handles which need to be fetched from the store to retrieve the underlying object internals,
such as a global's value and type.

![][instance-entity-wasmi-1.0]

The `global.get g` instruction in Wasmi 1.0 does three things:
read `instance.globals` to get the boxed slice,
load the handle `h` at index `g`, then resolve `h` in the `store`'s globals table.
Each load depends on the previous one and is very costly.

This procedure is so slow that Wasmi 1.0 ships special handling for `(global 0)` to speed up
the common case of accessing the global at Wasm index 0, which is commonly used as the pointer to
the shadow stack in C code that was compiled to Wasm.

#### How Wasmi 2.0 Works

Wasmi 2.0 acknowledges the fact that all Wasm instances of the same Wasm module
share the same object layout. An instance object's address is a property of the Wasm module.

![][instance-entity-wasmi-2.0]

The aforementioned `instance: Inst` parameter of all Wasmi 2.0 instruction handlers
is a thin-pointer to the `InstanceEntity`, which now is a dynamically sized type.

`InstanceEntity` consists of the fixed-size `InstanceEntityHeader` with common information
as well as the dynamically sized `handles` buffer.
The `handles` buffer contains all instance objects in a single contiguous allocation.

The ordering is `memories`, `globals`, `tables`, `funcs`, `elems`, `datas`.

- `memories` are first so that the commonly used default memory `(memory 0)` remains at address 0.
  Additionally, this allows Wasmi to define its memory addresses as 16-bit values.
- `datas` must be last since the `data count` section is not guaranteed to be available at Wasm module
  creation time, so it isn't known how many data segments exist.
- `InstanceEntityHeader` contains a `table0` field to allow fast access to the commonly
  used default table `(table 0)`, e.g. for `call_indirect`. [^explain-table0]
- The order of the remaining `globals`, `tables`, `funcs` and `elems` regions was chosen in relation
  to how common those object accesses are in Wasm executions.

The `handles` buffer contains the `handle` alongside an `entity` cache which
is a pointer to the actual instance object owned by the store. [^why-handles] These `entity` pointers
are initialized during Wasm instantiation so that the execution can rely on them.

To this end, the store also had to be slightly re-designed in that its containers, previously `Arena`,
now guarantee stable addresses for their owned objects, for which the `StableArena` type was created.

With all this, Wasmi 2.0 IR now uses instance addresses instead of Wasm indices for accessing
instance objects, and accessing an instance object is just one pointer offset away from `instance`
and thus extremely fast.

#### Instance Access: Comparison With Wasm3 & Stitch

Both Wasm3 and Stitch use instance-related bytecode.
This allows each instance to embed pointers to instance objects into its bytecode,
but requires each instance of the same Wasm module to store its own unique bytecode.

Wasmi uses module-related bytecode, which means that all Wasm instances of the same Wasm module
share the same underlying Wasmi IR bytecode, which improves memory consumption significantly.

Thanks to the re-design of the `InstanceEntity`, Wasmi 2.0 achieves Wasm3- and Stitch-level
performance for instance object access despite its inability to embed instance object pointers
into its bytecode.

The `counter-global` benchmark from [`wasmi-benchmarks`] counts a large number down to zero using
only a global variable. This strains the instance object access of Wasm interpreters quite a bit. [^why-not-faster]

![][counter-global-0]

The new Wasmi 2.0 design dissolves the need for a `(global 0)` cache as it performs great in both cases.
Wasmi 1.0 regresses significantly with `(global 1)` since its `(global 0)` cache is no longer used.

![][counter-global-1]

[instance-entity-wasmi-1.0]: ./resources/instance-access/instance-entity-wasmi-1.0-v6.svg
[instance-entity-wasmi-2.0]: ./resources/instance-access/instance-entity-wasmi-2.0-v3.svg
[counter-global-0]: ./resources/bench/counter-global/counter-global-0.svg
[counter-global-1]: ./resources/bench/counter-global/counter-global-1.svg

With the above instance-layout optimizations, Wasmi 2.0's `global_get_u64_r` instruction handler looks like this:

```asm
global_get_u64_r:
    ldr x7, [x1, #16]!     ; bump ip by 16 bytes and store next handler
    ldur w8, [x1, #-8]     ; fetch global address operand from ip
    add x8, x5, x8, lsl #4 ; compute instance[address]
    ldr x8, [x8, #64]      ; offset instance[address] by constant handles offset
    ldr x6, [x8]           ; load raw global value into ireg
    br x7                  ; tail-call next handler
```

### Lock-Free `CodeMap`

Wasmi's `CodeMap` is part of Wasmi's `Engine` and stores all the Wasmi IR function bodies.
The remaining function information (`FuncEntity`) on the other hand lives in the Wasmi store.

Due to their instance-related bytecode, both Wasm3 and Stitch can treat Wasm functions as
just another type of instance object and thus embed pointers to Wasm functions directly into
the encoded stream of IR instructions - and that is exactly what they do to make calls fast.

#### How Wasmi 1.0 Worked

The whole `CodeMap` was one mutex-guarded `Vec`-like arena data structure.
So every internal Wasm call had to take the mutex and then index into the `Vec`-like arena.
Needless to say, this was a very costly and inefficient procedure.

```rust
pub struct CodeMap {
    funcs: Mutex<Arena<EngineFunc, FuncEntity>>,
    features: WasmFeatures,
}
```

#### How Wasmi 2.0 Works

Because all instances share one Wasmi IR translation, a single engine-level address for a function
is valid for every instance of the same Wasm module.

Similar to Stitch and Wasm3, Wasmi can therefore bake pointers to the Wasmi IR function bodies
into its bytecode.

A baked pointer is only useful as long as the function it points to never moves.
The `CodeMap` however keeps growing, since every newly translated Wasm module appends
its functions to it.
A `Vec`-like arena as used by Wasmi 1.0 reallocates as it grows and moves all of its entries,
invalidating every pointer baked into the bytecode so far.
On top of that, the engine is shared across stores and modules, so this growth can
happen while other threads are already executing.

Wasmi 2.0 therefore stores functions in append-only buckets which never reallocate or move for
the lifetime of the engine.
Allocating new functions to the `CodeMap` (e.g. via `Module::new`) is serial whereas
accessing allocated functions is lock-free and concurrent with minimal synchronization overhead.

![][code-map]

Each `FuncEntry` carries its own atomic state, so the hot path of a call just checks if a
`FuncEntry` has already been compiled and returns its function body internals.

This allows for the lazy compilation of `FuncEntry` which is considered the cold path as
it only ever happens at most once successfully per `FuncEntry`.

A `call_internal` instruction handler in Wasmi 2.0 performs zero look-ups as its `FuncEntry`
address (pointer) is encoded directly into its bytecode as one of its operands, similar to
Stitch and Wasm3.

#### Calls: Comparison With Wasm3 & Stitch

The `fibonacci-rec` benchmark from [`wasmi-benchmarks`] stresses Wasm-to-Wasm calls:

![][fibonacci-rec]

The new `CodeMap` design closes the same gap for calls: Wasmi 2.0 keeps up with Stitch and Wasm3
despite its module-related bytecode and the need for shared atomic loads.
The reason why Wasmi even outperforms Stitch and Wasm3 is due to other technical differences:

- Both Stitch and Wasm3 use merged call and value stacks, which causes more copy overhead,
whereas Wasmi uses two different stacks to avoid exactly that.
- Furthermore, Wasm3 uses a constant pool per function to avoid the need for immediate operands, which
results in even more copy overhead per function call.

[code-map]: ./resources/code-map/code-map-wasmi-2.0-v2.svg
[fibonacci-rec]: ./resources/bench/fibonacci-rec.svg

### Fixed 64-Bit Cells

The Wasmi executor organizes values on the stack into so-called stack slots or cells.
In Wasmi 1.0 cells are either 64-bit wide, or 128-bit wide if the `simd` crate feature is enabled.
By default the `simd` feature is disabled, but users who require Wasm `simd` proposal support enable it.

This widening of cells from 64-bit to 128-bit not only causes increased memory consumption
but also regresses performance by roughly 5-10% due to more memory traffic,
worse cache utilization and wider copies at call boundaries.

Wasmi 2.0 on the other hand fixes cell width to 64 bits always.
For `simd` values it simply uses two adjacent cells instead.
Work to determine which values are assigned which cells is performed in the translator
so there is nothing to do in the executor.

Wasm `simd` instructions themselves are not slower:

- Wasmi 1.0 already stored its 128-bit cells as two 64-bit halves,
  so the same number of 64-bit words is moved either way.
- Wasmi 2.0 simply stops imposing that width on all non-`v128` values on the stack.

Enabling `simd` in Wasmi 2.0 no longer increases memory consumption or regresses performance. [^why-simd-not-default]

![][coremark-simd]

[coremark-simd]: ./resources/coremark/simd/apple-m2-pro.svg

The above diagram shows the effect on CoreMark results from Wasmi 1.0 and Wasmi 2.0
with their `simd` features enabled (+simd) and disabled.

As can be seen Wasmi 1.0 regresses by roughly 8% whereas
Wasmi 2.0 with `simd` remains just as fast as Wasmi 2.0 with `simd` disabled within noise levels.

## Accidental Rust Deoptimization

While working on [`wasmi-benchmarks`] and benchmarking other Wasm runtimes,
I noticed that [Stitch](https://github.com/makepad/stitch) performed significantly worse than in past measurements.

Its CoreMark score dropped by roughly 30% from over 3000 points to just ~2200 on my Apple M2 Pro.
The regression happened between Rust 1.91 and 1.92.

Further investigation found the culprit: `DestinationPropagation`, a MIR optimization that
[Rust 1.92 enabled by default][rust-#142915]. This pass merges MIR locals holding the same value.
The effect is that the two dispatch paths of a conditional branch handler collapse into just one:
a `csel` feeding a single branch site.
A CPU's branch predictor now only sees one entry with mixed history, which complicates its job.

[rust-#142915]: https://github.com/rust-lang/rust/pull/142915

With [the fix applied to Stitch](https://github.com/Robbepop/stitch/commit/3280ff672c861a1e73107c9b1d393b06127e27ad),
its CoreMark score went back up to over 3000 points again. Success!

Using [`cargo-show-asm`](https://crates.io/crates/cargo-show-asm) I looked at Wasmi 2.0's
own `i32.lt` instruction handler and ... oh boy! It suffered from the same underlying issue: [^wasmi-not-deoptimized]

```asm
branch_i32_lt_ri:
    ldp w8, w9, [x1, #8] ; fetch branch offset and rhs immediate from ip
    sxtw x8, w8          ; sign-extend the branch offset
    add x10, x1, #16     ; compute the fall-through ip
    add x8, x1, x8       ; compute the branch target ip
    cmp w9, w6           ; branch is taken if ireg < rhs
    csel x1, x8, x10, gt ; csel picks the target
    ldr x7, [x1]         ; load the handler at the chosen ip
    br x7                ; unified branch site to the chosen target
```

After [applying the fix to Wasmi 2.0][#2027] its CoreMark score rose from ~2800 to over 4200.
That's a ~50% improvement with this singular fix which made it the single most important "optimization" for Wasmi 2.0.

The considerably faster assembly of `i32.lt` now looks like this:

```asm
branch_i32_lt_ri:
    ldr w8, [x1, #12]  ; fetch rhs immediate operand from ip
    cmp w8, w6         ; branch is taken if ireg < rhs
    b.le LBB1242_2     ; not taken: continue with the next instruction
    ldrsw x8, [x1, #8] ; fetch the sign-extended branch offset from ip
    add x1, x1, x8     ; ip = branch target
    ldr x7, [x1]       ; load the handler at the branch target
    br x7              ; branch site if branch is taken
LBB1242_2:
    ldr x7, [x1, #16]! ; bump ip by 16 bytes and load next handler
    br x7              ; branch site if branch is not taken
```

> **Note:** Interestingly only the tail-call-based instruction dispatch configurations
>           of Wasmi 2.0 see performance improvements due to the fix above.

## What's Next

With the next major version Wasmi 3.0, we aim to support all of [WebAssembly 3.0]
which requires implementing the following Wasm proposals still missing from Wasmi 2.0:

- 🚧 [`function-references`]
- 🚧 [`exception-handling`]
- 🚧 [`gc`] (Garbage Collection)

[WebAssembly 3.0]: https://webassembly.org/news/2025-09-17-wasm-3.0/

## Try It Out!

Try out and use Wasmi today in various ways:

- 📚 As a library dependency via the [`wasmi` crate](https://crates.io/crates/wasmi).
- 🖥 Using its CLI application by installing the [`wasmi_cli` crate](https://crates.io/crates/wasmi_cli) using `cargo install wasmi_cli` or any of its pre-build release artifacts.
- ⚙️ In your C-interfacing language using the [Wasmi C-API](https://github.com/wasmi-labs/wasmi/tree/main/crates/c_api#readme).
- 😈 Play [Doom in your browser](https://wasmi-labs.github.io/wasmi-doom/) powered by Wasmi 2.0. [^🤖]
- 📦 Or enjoy Wasmi indirectly by using any of its [major known users](https://github.com/wasmi-labs/wasmi#used-by).

## Personal Note

Without the sponsorship of the [Stellar Development Foundation][SDF], Wasmi 2.0 would not exist today.
This funding allowed me to work on this open source project full-time for two years which is a rare opportunity
for which I am deeply grateful.

That sponsorship ends in October 2026. I intend to keep working on Wasmi past that point
and am looking for ways to make that possible: another sponsorship, or a role that leaves room for
further Wasmi development at least part-time.

If that is something you or your company could be interested in, contact me at <robin.freyler@gmail.com>.

## Footnotes

[^intro]: I am not a native English speaker and this article is hand-written. All mistakes contained in the article are mine. In case of severe issues, feel free to open a [pull request](https://github.com/wasmi-labs/blog/pulls).
[^benches-runtimes]: Wasmtime's Pulley and WAMR's fast-interpreter are shown in benchmarks throughout the article since they also provide
respectable performance despite their differences in interpreter architecture compared to Wasm3, Stitch and Wasmi 2.0.
[^wasmi-configs]: The configs such as `eager`, `lazy` and `lazy-translation` are explained
    [here](https://github.com/wasmi-labs/wasmi-benchmarks#configuration-explanation).
[^why-pulley-outlier]: Pulley sits behind Cranelift's expensive optimization pipeline,
  which makes it an optimizing Wasm interpreter that would easily take the number one spot on unoptimized Wasm inputs.
  In practice, however, Wasm is essentially always optimized before it ships.
[^explain-table0]: We could also have a `global0` pointer for example for faster access to `(global 0)` in `InstanceEntity`, but we decided against it for now since global access is already quite speedy and accessing globals is usually not on the hot execution path anyway.
[^why-handles]: We still keep the `handle` in the `AnyHandleAndEntity` type for non-performance critical usage outside the Wasmi executor.
[^why-not-faster]: Wasmi 2.0 is even faster than both Wasm3 and Stitch in `execute/counter-global`. However, that is likely due to other technical differences between the interpreters. For example, Wasm3 puts `global.get` results into stack slots whereas Wasmi 2.0 uses accumulator registers which is beneficial for this benchmark test case.
[^wasmi-not-deoptimized]: In contrast to Stitch, Wasmi 2.0's performance did not regress between Rust 1.91 and 1.92 so the source
of its collapsed branch sites was different, but it still suffered from the same consequences and it was possible to fix it using
the same trivial fix.
[^🤖]: Be aware that Wasmi Doom was created using AI.
[^cargo-bloat-show]: Using the [`cargo-bloat-show`](https://crates.io/crates/cargo-bloat-show) tool, it was quite easy to explore the parts that caused the most bloat and eliminate them.
[^explain-stable-metering]: "Stable fuel metering" does not mean that the feature has been stabilized (it already was) but that the metered fuel per unit of execution stays the same across Wasmi versions.
[^why-next-macro]: We use a `next!` macro instead of a function call, `become` or `return` since that allows Wasmi to use different instruction dispatch modes using the same underlying code. The `next!` macro simply expands to slightly different code depending on the chosen configuration.
[^why-simd-not-default]: Users of Wasm interpreters usually have no need for the Wasm `simd` proposal and it still adds bloat to binary artifact size, compilation time and, very minimally, to the translation of Wasm bytecode to Wasmi IR at runtime. Therefore `simd` is not part of Wasmi's default set of enabled crate features.
[^nightly-become]: When Wasmi's default `stable` crate feature is disabled and its `unstable` feature is enabled, Wasmi makes use of Rust's unstable [`become` keyword] for its threaded-code dispatch.

[`become` keyword]: https://doc.rust-lang.org/std/keyword.become.html

[#2027]: https://github.com/wasmi-labs/wasmi/pull/2027
[Typst]: https://typst.app/docs/reference/foundations/plugin/
[Zellij]: https://github.com/zellij-org/zellij
[Josh]: https://github.com/josh-project/josh
[Soroban]: https://stellar.org/soroban
[Firefly Zero]: https://fireflyzero.com/
[SDF]: https://stellar.org/foundation
[Ripple]: https://ripple.com/
[`function-references`]: https://github.com/WebAssembly/function-references
[`exception-handling`]: https://github.com/WebAssembly/exception-handling
[`gc`]: https://github.com/WebAssembly/gc
