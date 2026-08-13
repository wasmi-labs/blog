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

