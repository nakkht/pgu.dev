---
layout: post
title: "Rust language: lifetime elision"
author: Paulius Gudonis
---

While writing previous post on ["Rust lifetimes"]({% post_url 2020-07-13-rust-lifetimes %}), for brevity reasons I've decided to exclude details about lifetime elision and rather keep it for the next post.

If you followed post on ["Rust borrowing"]({%  post_url 2020-04-06-rust-borrowing %}), you most likely noticed that there were functions that accepted multiple references as parameters but did not require to explicitly write out what lifetimes were expected. In fact, the same applies to returned references. To assure you, lifetimes are there. Rust does not magically drop lifetimes but rather allows to omit syntax when it is obvious what they are. This is very similar to type inference in strongly typed languages like Swift or Kotlin.

> To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org).

In very simple, deterministic cases, where function takes reference as en argument but never returns one as such:

```rust
fn main() {
    let result = sum(&40, 2);
    println!("{}", result);
}

fn sum(x: &i32, y: i32) -> i32 {
    return *x + y;
}
```

Then you don't need to explicitly write out lifetimes for parameters (also called _input lifetime_). Rust does that for you behind the scenes. Equivalent `sum` signature would be:

```rust
fn sum<'a>(x: &'a i32, y: i32) -> i32 {
    return *x + y;
}
```