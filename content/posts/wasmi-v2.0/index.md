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

