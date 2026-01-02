---
title: "Automated Detection of Performance Regressions in MLIR"
date: 2026-01-02
author: "Poorna Gunathilaka"
image: "/img/cover-wasm.png"
tags: ["mlir", "fuzzer", "stablehlo", "jax", "titanfuzz", "e-graph", "compiler"]
categories: ["Tech"]
draft: false
description: "Automated Detection of Performance Regressions in MLIR Optimizations via Metamorphic Testing"
---

### What is MLIR and Why Does It Matter?

Compiler infrastructure has historically been fragmented. You had one representation for the high-level language (like the TensorFlow graph) and a completely different one for the low-level machine code (like LLVM IR). Moving between these levels often resulted in lost optimization opportunities.

MLIR (Multi-Level Intermediate Representation) solves this by providing a single, unified infrastructure. It allows compilers to define "dialects"—modular levels of abstraction that can mix and match. It serves as the backbone for modern machine learning compilers like XLA (used by TensorFlow, JAX, and PyTorch).

When you write code in JAX, it doesn't go straight to machine code. It gets lowered into the **StableHLO** dialect, then usually to the **Linalg** or **Affine** dialects, and finally to LLVM IR or NVVM for hardware execution.

Because MLIR sits right in the middle of this pipeline, it is the critical path for performance. If an MLIR pass handles a loop inefficiently, the final application runs slowly, regardless of how fast the hardware is.

### The Problem: Correctness vs. Performance

Most compiler testing focuses on **correctness**. We run fuzzers to answer the question: *"Does the compiler crash, or does it produce the wrong number?"*

However, in High-Performance Computing (HPC), a program that produces the right answer but takes 10x longer to run is effectively broken. These are **Silent Performance Regressions**. They are dangerous because they don't throw errors; they just silently degrade the efficiency of the underlying hardware.

Detecting them is hard. To test correctness, you just need to check the output. To test performance, you typically need an "oracle" you need to know exactly how fast the code *should* run. For random, generated test cases, calculating that theoretical speed is impossible.

I recently built a framework to automate performance testing for MLIR without needing a ground-truth oracle.

### The Approach: Metamorphic Testing

Since we can't predict absolute performance for random inputs, we look at relative performance. **Metamorphic Testing** validates the system by checking relationships between different inputs or configurations.

I defined three Metamorphic Relations (MRs) that a healthy MLIR compiler should satisfy:

1.  **Hardware Offloading:** For sufficient workloads, running on a GPU should be faster than a CPU.
2.  **Polyhedral Optimization:** Code optimized with polyhedral techniques (tiling, fusion, affine transforms) should outperform the scalar CPU baseline.
3.  **Semantic Equivalence:** Mathematically simplified code (e.g., applying the distributive property) should run at least as fast as the unsimplified version.

If a test case violates these rules—for example, if the GPU code is slower than the CPU code—we flag it as a regression.

### Generating Test Cases with TitanFuzz

A major bottleneck in compiler testing is generating input programs that are actually useful. Random code generators often produce "spaghetti code" that lacks the loop structures needed to trigger advanced optimizations.

To solve this, I extended **TitanFuzz**, an LLM-based fuzzer originally built for PyTorch. I added a JAX backend and fed it kernels from the **PolyBench** suite. This allowed the LLM to generate syntactically valid JAX programs that utilize the `StableHLO` dialect and contain the nested loops necessary for meaningful performance analysis.

The system works in three steps:
1.  **Generate:** TitanFuzz creates JAX programs based on PolyBench templates.
2.  **Fork:** The program is compiled down three different paths (Baseline CPU, GPU Offload, Polyhedral via Polygeist).
3.  **Verify:** We execute the binaries and check if the performance relations hold.

### Results and Findings

I evaluated the framework on 45 computational kernels. The data highlighted both successes and failures in the current MLIR ecosystem.

**1. Polyhedral Optimizations are Consistent**
The relation `Time(Polygeist) < Time(Baseline)` held up well. Using **Polygeist** to raise code to the Affine dialect consistently improved performance. Speedups ranged from **1.5x** to **8.45x** depending on the kernel's arithmetic intensity.

**2. Semantic Rewrites Work**
We used **DialEgg**, an equality saturation tool, to rewrite expression trees (e.g., factoring out matrix multiplications). This also passed consistently. The tool correctly identified when to simplify operations, leading to speedups between **1.15x** and **2.48x**.

**3. The GPU "Crossover" Issue**
The hardware relation (`Time(GPU) < Time(CPU)`) revealed a significant issue. We generally assume GPUs are faster, but my testing showed this depends entirely on input size.

* **Large Inputs (N=2048):** The GPU path was significantly faster (up to 49x).
* **Small Inputs (N=512):** The GPU path was often **slower** than the CPU.

For example, on the `gesummv` kernel with small inputs, the GPU version ran at **0.67x** the speed of the CPU version. The overhead of kernel launches and PCIe data transfer outweighed the compute benefits. This is a performance regression because the compiler lowered to the GPU without checking if the workload size justified the cost.

### Summary

This project demonstrates that we don't need a theoretical oracle to find performance bugs. By defining simple invariants—like "GPU should beat CPU"—we can automate the detection of inefficient code generation.

For the MLIR ecosystem, the next step is implementing size-aware cost models to prevent the regressions I observed in the GPU offloading path.
