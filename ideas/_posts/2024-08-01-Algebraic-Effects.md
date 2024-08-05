---
layout: post
author: Li Boxiu
title: Using Generators to Implement Algebraic Effects
---

Recently I've heard `Algebraic Effects` a lot. After some investigation,
I'm very interested in implementing it in Pivot Lang. `Algebraic Effects`
is known for its ability to implement `async/await`, `generators`, `try/catch`
etc. in ease. As I've already implemented `generators` in Pivot Lang,
I immediately become curious about if I can use `generators` to implement
`algebraic effects`? If so, I can implement it in Pivot Lang with
minimal effort.

## What is Algebraic Effects?

Well, basically, `Algebraic Effects` is a way to separate the side effects
from the computation. It's a well-known concept in functional programming.
If you're not familiar with it, I would recommend you to read this
[article](https://overreacted.io/algebraic-effects-for-the-rest-of-us/).

## How to Implement Algebraic Effects with Generators?

Below is a simple example of my experiments, using Pivot Lang's generator:

```Rust
var effect1 = 1;
var effect2 = 2;

gen fn generator1() Iterator<i64> {
    executeGenerator(generator2());
    yield return effect1;
}

gen fn generator2() Iterator<i64> {
    yield return effect2;
}


fn eff1handler(resolve: ||=>void) void {
    println!("effect 1");
    resolve();
    return;
}
fn eff2handler(resolve: ||=>void) void {
    println!("effect 2");
    resolve();
    return;
}


fn executeGenerator(g:Iterator<i64>) void {
    let eff = g.next();
    let resolve = || => {
        executeGenerator(g);
        return;
    };
    match eff {
        i64(1) => {
            eff1handler(resolve);
        }
        i64(2) => {
            eff2handler(resolve);
        }
        _ => {
        }
    }
    return;
}

fn main() void {
    executeGenerator(generator1());
    return;
}

```

In this example, I have two effects `effect1` and `effect2`. I have two
generators `generator1` and `generator2`. `generator1` will yield `effect1`
and `generator2` will yield `effect2`. I have two handlers `eff1handler`
and `eff2handler` to handle the effects. `executeGenerator` will execute
the generator and call the corresponding handler.

If we need to add `effect` to Pivot Lang, every function that may use
the effect should be a generator. This is, of course, a performance
overhead. It's just like making every function `async` in JavaScript.

### The Proposal of Algebraic Effects in Pivot Lang

[see the proposal](./2024-08-05-Algebraic-Effects-Proposal.md)
