---
layout: post
title: "Rust language: lifetimes"
author: Paulius Gudonis
---

> **Note**: This post is a part of series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and ["borrowing"]({% post_url 2020-04-06-rust-borrowing %}).  
> To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org).

For the last segment on Rust language features, we'll take a look at lifetimes. In the [previous post]({% post_url 2020-04-06-rust-borrowing %}) on borrowing I have few times mentioned term lifetimes when talking about references. Each reference has a lifetime specifying the scope it is valid for.

Most of the time lifetime of a reference is implicit and borrow checker can do pretty good job determining how long each reference is valid within a function or closure. However, once the code gets more complicated and multiple references are passed between functions, borrow checker cannot determine lifetime and it requires to get explicit annotation to figure out how those references are related. The aim of lifetimes is to avoid dangling pointers which reference invalid data. As mentioned in the previous post, [these type of issues](https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/) are always exploited when searching for vulnerabilities in software.

Let's have a look at the following snippet:

```rust
fn main() {
    let result = increment(&mut 2, &mut 4);
    println!("{}", result);
}

fn increment(x: &mut i32, y: &mut i32) -> &i32 {
    *x = *x + *y;
    return x;
}
```

Right of the bat we get the following compiler error:

```rust
eerror[E0106]: missing lifetime specifier
 --> src/main.rs:6:43
  |
6 | fn increment(x: &mut i32, y: &mut i32) -> &i32 {
  |                 --------     --------     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
6 | fn increment<'a>(x: &'a mut i32, y: &'a mut i32) -> &'a i32 {
  |             ^^^^    ^^^^^^^^^^^     ^^^^^^^^^^^     ^^^

error: aborting due to previous error
```

As described previously, Rust borrow checker has a hard time deciding what is a lifetime of a returned value. The suggestion from the error message is pretty helpful: let's try adding named lifetime parameter.

<--- Example of the solution including syntax ---> 


The following syntax is used to denote lifetime:

```rust
&'a i32			// a reference with explicit lifetime for type i32
&'a mut i32		// a mutable reference with explicit lifetime for type i32
```

<--- Lifetime Elision ---> 


<--- Conclusion ---> 