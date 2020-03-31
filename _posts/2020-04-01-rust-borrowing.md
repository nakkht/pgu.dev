---
layout: post
title: Rust and borrowing
author: Paulius Gudonis
---

Another important rust language concept which caters ownership is called borrowing.

> **Note**: This post is part of a series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and lifetimes ["lifetimes[in the works]"]().

Firstly, let's take a look at the following example:

```rust
fn main() {
	let text = String::from("Hello 2020");
	let vec = vec![3, 42];
	compute(text, vec);

	print!("{:?}, {:?}",text, vec)
}

fn compute(str: String, vec: Vec<i32>) -> (String, Vec<i32>) {
	(str, vec)
}
```

As from the previous post on ownership, the example won't compile and errors about moved values and `Copy` traits missing will be shown.