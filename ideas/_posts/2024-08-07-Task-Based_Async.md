---
layout: post
author: Li Boxiu
title: Design of Task Based Async in Pivot Lang
---



Defination of `Task`:


```Rust
pub trait Task<T> {
    fn poll(wk:||=>void) Option<T>;
}

```

Await transformation:

```Rust
await doTask();

```

to

```Rust
let task = doTask();
let re = task.poll(wk);
while re is None {
    yield return None;
    re = task.poll(wk);
}
let ret = re as T!;
```


