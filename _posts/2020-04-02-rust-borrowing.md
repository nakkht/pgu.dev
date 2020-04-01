---
layout: post
title: "Rust language: borrowing"
author: Paulius Gudonis
---

Another important Rust language concept which caters ownership is called borrowing.

> **Note**: This post is part of a series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and lifetimes ["lifetimes[in the works]"]().

To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org)
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

As described in the [previous post]({% post_url 2020-03-23-rust-ownership %}) on ownership, the example won't compile and errors about moved values and missing `Copy` traits will be shown. Question is, can we do something about it and the answer is yes. Let's take a look at the next example:

```rust
fn main() {
	let text = String::from("Hello 2020");
	let vec = vec![3, 42];
	let result = compute(&text, &vec);

	println!("{}, {:?}, {}",text, vec, result);
}

fn compute(str: &String, vec: &Vec<i32>) -> String {
	return format!("{} {}", str.to_owned(), &vec.into_iter().map(|v| v.to_string()).collect::<String>());
}
```

After building and running, similar output will appear: "Hello 2020, [3, 42], Hello 2020 342"
The subtle difference is that instead of taking value of `String` and `Vec` as an argument, the actual references are taken (notice `&` character next to parameter type definition). In Rust terminology it is called "borrowing ownership". This mechanism unlike moving values, does not deallocate the resource when it goes out of scope. That's why even after passing references to `compute` function in the last code snippet, it was still possible to print values of `text` and `vec` in the next line.