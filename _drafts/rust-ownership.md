---
layout: post
title: Rust and ownership
author: Paulius Gudonis
---
One of the key rust language concepts is ownership. It is important to understand it early on, as ownership rules are heavily enforced by the compiler.

> **Note**: This post is part of a series on Rust language features, including [borrowing]() and [lifetimes]().

There are three main rules:
* Value/instance belong to owner
* Every value has a single owner
* Owner determines lifetime of the value

The last part also means that if the owner is freed or as in rust terminology - dropped, owned values are freed as well.<br>
For example, if we run the following code snippet:

```rust
fn main() {
	let value = 42;
	println!("{}", value)
}
```

We will get value 42 printed in the terminal.