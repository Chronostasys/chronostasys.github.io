---
layout: post
author: Li Boxiu
title: JIT, Optimization, GC and LLVM
---

## Prologue

JIT, Optimization, GC and LLVM, each of them is a huge topic,
and is pretty hard to implement correctly.
It's even harder to make them work together.

In this article, I will share a mistake I made when combining
JIT, optimization and GC in Pivot Lang.

## The System

Pivot Lang is a new programming language I'm working on. It's
a statically typed language with a LLVM-based JIT engine (although it also supports AOT compilation).
It has a mark-sweep garbage collector, which is implemented in Rust. The collect algorithm is called
`Immix`, which may move things around during collection, so some extra care is needed when
generating IR. For simplicity and performance reason, I'm using [`LLVM`'s experimental
`safepoint` instruction](https://llvm.org/docs/Statepoints.html) to handle relocations.
As the experimental instructions are complicated, I'm using the `RS4GC` pass to
add the instructions for me. Detail of the technique is out of the topic, if you are interested,
you can read the [blog post](https://dangui.site/blog/llvm/llvm-and-gc/) I wrote before.

Generally, the `RS4GC` pass is expected to run at the end of the optimization pipeline, as it
can prevent some optimizations from happening, as documented in the `LLVM`'s official document:

> In practice, RewriteStatepointsForGC should be run much later in the pass pipeline, after most optimization is already done. This helps to improve the quality of the generated code when compiled with garbage collection support.

Well, the document is right, but it's missing something important: running `RS4GC` too early
may not only prevent some optimizations from happening, but also break the stackmap generated,
which would lead to misbehavior of the GC.

## The Problem

Well, everything seems fine as long as we put the `RS4GC` pass at the end of the optimization pipeline
isn't it? This is exactly what happened when the code is AOI-compiled, and it works well on all platforms. However, when the code is JIT-compiled, there's an issue.

As we know, the JIT engine is expected to optimize the code on-the-fly, and that's exactly the point why people love it. A well-designed JIT engine can optimize/regenerate the code many times, and make the code run faster and faster. However, the `RS4GC` pass is not expecting any optimization to happen after it. The problem is, I'm using the same IR generated for AOT compilation for JIT compilation, which means the IR is already optimized by the `RS4GC` pass. When the JIT engine tries to optimize the code again, it may break the stackmap.

## The Solution

The solution is simple: I disabled the optimization for the JIT engine.

In the future, I may need to generate different IR for JIT and AOT compilation, the IR for JIT compilation should be less optimized, and the `RS4GC` pass should be run, thus the JIT engine
can optimize the code as it wants.
