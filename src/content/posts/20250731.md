---
title: A Rustacean's Guide to PGO
published: 2025-07-31
description: "Understand the why and how of Profile-Guided Optimization in the Rust ecosystem."
image: ""
tags: ["Rust"]
category: "Rust"
draft: false
lang: ""
---

## What is Profile-Guided Optimization (PGO)?

**Profile-Guided Optimization (PGO)** is a technique that improves program performance by analyzing how it actually runs.

Instead of making general assumptions, the compiler collects real-world data about the program (like which branches are taken most often), and uses that data to guide optimizations such as :

- Function inlining
- Code layout for better cache locality
- Register allocation

## How does Rust implement PGO?

Rust's compiler, `rustc`, uses **LLVM** for PGO.

LLVM supports multiple approaches, e.g.,

- Sampling-based PGO (via `perf`)
- GCOV-style profiling (code coverage-based)
- **IR-level instrumentation** (LLVM adds profiling hooks itself)

Rust only supports **IR-level instrumentation**, where LLVM inserts profiling instructions directly into the Intermediate Representation (IR) during compilation.

PGO in Rust works in two stages :

1. Compile-time instrumentation : Build a special binary with profiling hooks
2. Runtime instrumentation : Run the instrumented binary to collect `.profraw` files

### Over workflow

1. Compile the program with intrumentation enabled. E.g.,
   ```bash
   rust -C profile-generate main.rs
   ```
2. Run the intrumented program (e.g, `./main`) which generates a `default-<id>.profraw` file
3. Convert the `.profraw` file into a `.profdata` file using LLVM's `llvm-profdata` tool
4. Compile the program again, this time making use of the profiling data. E.g.,
   ```bash
   rust -C profile-use=merged.profdata main`
   ```

## Compile-time

When we talked earlier, we ran `rustc -C profile-generate main.rs`, which tells LLVM to insert profiling hooks into the generated binary.

`rustc` instructs LLMV to do so by [setting flags](https://github.com/rust-lang/rust/blob/1.34.1/src/rustllvm/PassWrapper.cpp#L412-L416) :

```cpp
// `PMBR` is an `LLVMPassManagerBuilderRef`
unwrap(PMBR)->EnablePGOInstrGen = true;
// Instrumented binaries have a default output path for the `.profraw` file
// hard-coded into them:
unwrap(PMBR)->PGOInstrGen = PGOGenPath;
```

When compiling binaries with optimizations that utilizing profiling data, `rustc` again delegates most of the work to LLVM.

Essentially, it [informs](https://github.com/rust-lang/rust/blob/1.34.1/src/rustllvm/PassWrapper.cpp#L417-L420) LLVM where the profiling data is located :

```cpp
unwrap(PMBR)->PGOInstrUse = PGOUsePath;
```

## Runtime

Instrumentation-based PGO isn't jsut about how we compile code, it also requires runtime support. After all, once we've built an instrumented binary, we need to run it to gather real-world profiling data.

In LLVM, the runtime components responsible for collection profile data are part of a seperate project called `compiler-rt`. These components are **statically linked** into every instrumented binary and take care of :

- Recording branch hits, function calls, and counters
- Writing the `.profraw` files to disk

Rust doesn't reinvent this wheel. Instead, it reuses LLVM's runtime by bundling its C code into a Rust crate called `profiler_builtins`. This crate wraps the profiling logic from `compile-rt` and makes it available in Rust builds.

To make this work, you must explicitly enable it during compiler builds by `bootstrap.toml` settings :

```toml
profiler = true
```
