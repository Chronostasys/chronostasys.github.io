---
layout: post
author: Li Boxiu
title: Proposal of Algebraic Effects in Pivot Lang
---

This post is a proposal of implementing `Algebraic Effects` in Pivot Lang. It will cover the
propose syntax and transformation rules to translate the `effects` into `generators`.

## Syntax Proposal

### Define an Effect Type

A effect type should be defined just like normal function type.

```Rust
effect effect1: ||i32|=>void, u32| => void;
```

The effect type `effect1` takes two parameters, the first one is
a resolver that takes no `i32` and returns `void`, the resolver
is something like `return` statement in a function, and its argument
is the value that the effect will return, it's return type is always `void`.
From the second parameter, they are the arguments that the effect will take.

The effect type can have generic parameters.

```Rust
effect effect2<T>: ||T|=>void| => void;
```

### Use an Effect

When a function uses effects is called, the caller should set up
handlers for the effects in advance.

```Rust

effect effect1: ||i64|=>void, i64| => void;
fn fnUsingEff1() void {
    let re = fnUsingEff2();

    perform effect1(re);
    return;
}

fn fnUsingEff2() i64 {
    perform effect1(1);
    return 100;
}

fn eff1Handler(resolve: ||i64|=>void, a:i64) void {
    resolve(a+1);
    return;
}

fn main() void {
    handler effect1(a:i64) {
        eff1Handler(a);
    } handle {
        let a = fnUsingEff1();
        println(a);
    }
    return;
}

```

### Transform Effects into Generators

The effect system can be transformed into a generator system.

The above code can be transformed into the following code:

```Rust

var effect1 = 1;
var eff_ret = 10086;


fn main() void {
    executeGenerator(fnUsingEff1());
    return;
}

gen fn fnUsingEff1() Iterator<(i64, |*||=>void,*||=>void| =>void)> {
    let retgen2 = 0;
    let generator = fnUsingEff2();
    while true {
        let nx = generator.next();
        if nx is None {
            break;
        }
        let nx_v = nx as (i64, |*||=>void,*||=>void| =>void)!;
        match nx_v {
            (10086, f) => {
                let wrapped_f = |iter:*||=>void,handler:*||=>void| => {
                    let real_handler = unsafe_cast<|||=>void|=>void>(handler);
                    let ret_handler = |resolve:||=>void, ret|=>{
                        retgen2 = ret;
                        (*real_handler)(resolve);
                        return;
                    };
                    let clp = &ret_handler;
                    let arg2 = unsafe_cast<||=>void>(clp);
                    f(iter,arg2);

                    return;
                };
                let yr= (eff_ret,wrapped_f);
                yield return yr;
                break;
            }
            yr => {
                yield return yr;
            }
        }
    }



    // executeGenerator(fnUsingEff2());
    let a = retgen2;
    let re = 0;

    let f = |next:*||=>void,handler:*||=>void| => {
        let real_handler = unsafe_cast<||i64|=>void,i64|=>void>(handler);
        let resolve = |ret|=> {
            re = ret;
            (*next)();
            return;
        };
        (*real_handler)(resolve,a);

        return;
    };
    let yr= (effect1,f);
    yield return yr;
}


gen fn fnUsingEff2() Iterator<(i64, |*||=>void,*||=>void| =>void)> {
    // executeGenerator(fnUsingEff2());
    let a = 1;
    let re = 0;

    let f = |next:*||=>void,handler:*||=>void| => {
        let real_handler = unsafe_cast<||i64|=>void,i64|=>void>(handler);
        let resolve = |ret|=> {
            re = ret;
            (*next)();
            return;
        };
        (*real_handler)(resolve,a);

        return;
    };
    let yr= (effect1,f);
    yield return yr;

    f = |next:*||=>void,handler:*||=>void| => {
        let real_handler = unsafe_cast<|||=>void,i64|=>void>(handler);
        let resolve = ||=> {
            (*next)();
            return;
        };
        (*real_handler)(resolve,100);

        return;
    };
    // return
    yr= (eff_ret,f);
    yield return yr;
}


fn executeGenerator(g:Iterator<(i64, |*||=>void,*||=>void| =>void)>) void {
    let eff = g.next();
    if eff is None {
        return;
    }
    match eff as (i64, |*||=>void,*||=>void| =>void)! {
        (1,r) => {
            let cl =|resolve, a| => {
                eff1handler(resolve, a);
                return;
            };
            let next = & ||=>{
                executeGenerator(g);
                return;
            };
            let clp = &cl;
            let arg2 = unsafe_cast<||=>void>(clp);
            r(next,arg2);
        }
        (10086,r) => {
            let cl =|resolve:||=>void| => {
                resolve();
                return;
            };
            let next = & ||=>{
                executeGenerator(g);
                return;
            };
            let clp = &cl;
            let arg2 = unsafe_cast<||=>void>(clp);
            r(next,arg2);
        }
        (n, f) => {
            panic!("no handle", n);
        }
    }
    return;
}


fn eff1handler(resolve: |i64|=>void,a:i64) void {
    resolve(a+1);
    return;
}

```

The above code is runable in Pivot Lang. [Reference](https://github.com/Pivot-Studio/pivot-lang/pull/428)
