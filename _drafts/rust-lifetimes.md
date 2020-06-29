---
layout: post
title: "Rust language: lifetimes"
author: Paulius Gudonis
---

> **Note**: This post is a part of series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and ["borrowing"]({% post_url 2020-04-06-rust-borrowing %}).  
> To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org).

For the last segment on Rust language features, we'll take a look at lifetimes. In the [previous post]({% post_url 2020-04-06-rust-borrowing %}) on borrowing I have few times mentioned term lifetimes when talking about references. Each reference has a lifetime, determining the scope for which lifetime is valid.

Most of the time lifetime of a reference is implicit and borrow checker can do pretty good job determining how long each reference is valid within a function or closure. However, once the code gets more complicated and multiple references are passed between functions, borrow checker cannot determine lifetime and it requires to get explicit annotation to figure out how those references are related.
The aim of lifetimes is to avoid dangling pointers which reference invalid data. As mentioned in the previous post, this type of issue is always exploited when searching for vulnerabilities in software.

```rust
fn load<'a>(xml: &'a String) -> Reader<&'a [u8]> {
  let mut reader = Reader::from_str(&xml);
  reader.trim_text(true);
  return reader;
}
```

The following syntax is used to denote lifetime:

```rust
&'a i32			// a reference with explicit lifetime for type i32
&'a mut i32		// a mutable reference with explicit lifetime for type i32
```

<--- Lifetime Elision ---> 