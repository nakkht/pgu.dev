---
layout: post
title: "Rust language: lifetimes"
author: Paulius Gudonis
---

> **Note**: This post is a part of series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and ["borrowing"]({% post_url 2020-04-06-rust-borrowing %}).  
> To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org).

For the last segment on Rust language features, we'll take a look at lifetimes. In the [previous post]({% post_url 2020-04-06-rust-borrowing %}) on borrowing I have few times mentioned term lifetimes when talking about references. Each reference has a lifetime specifying the scope it is valid for.

Most of the time lifetime of a reference is implicit and borrow checker can do pretty good job determining how long each reference is valid within a function or closure. However, once the code gets more complicated and multiple references are passed between functions, borrow checker cannot determine lifetime of each reference and it requires to get explicit annotations to figure out how those references are related. The aim of lifetimes is to prevent dangling pointers which reference invalid data. As mentioned in the previous post, [these type of issues](https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/) are always exploited when searching for software vulnerabilities.

Let's start with a look at the following snippet:

```rust
fn main() {
    let mut x = 2;
    let y = 4;
    let result = increment(&mut x, &y);
    println!("{}", result);
}

fn increment(x: &mut i32, y: &i32) -> &i32 {
    *x = *x + *y;
    return x;
}
```

Quick rundown of the code: 
* We have `main` function which calls `increment` function with values 2 and 4 and prints out the result
* `increment` function simply accepts two values, sums them and assigns result to `x` of which reference is returned right after
* Result is printed out using `println!` macro

In practice, you will definitely won't have such over-complicated code for a simple operation as such, but for demonstration purposes let's assume its the case.
Right of the bat after trying to compile the code, we get the following compiler error:

```rust
error[E0106]: missing lifetime specifier
 --> src/main.rs:6:39
  |
6 | fn increment(x: &mut i32, y: &i32) -> &i32 {
  |                 --------     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
6 | fn increment<'a>(x: &'a mut i32, y: &'a i32) -> &'a i32 {
  |             ^^^^    ^^^^^^^^^^^     ^^^^^^^     ^^^
```

As mentioned previously, Rust borrow checker has a hard time deciding what is a lifetime of a returned value in conjunction with passed multiple references (of which we have 2). The explanation from the error message is pretty helpful though: returned value is borrowed, but it is unclear from which argument - `x` or `y`. Let's try and help borrow checker to help us. As a matter of fact, error message already shows possible solution, let's try and apply it:

```rust
fn main() {
    let mut x = 2;
    let y = 4;
    let result = increment(&mut x, &y);
    println!("{}", result);
}

fn increment<'a>(x: &'a mut i32, y: &'a i32) -> &'a i32 {
    *x = *x + *y;
    return x;
}
```

Voilà, the code compiles and result `6` is printed out. The only difference from the previous code is that we added mysterious `'a`. It is called "lifetime annotation syntax", example as follows:

```rust
&'a i32     // a reference to value of type i32 with an explicit lifetime
&'a mut i32 // a mutable reference to value of type i32 with an explicit lifetime
```

Lifetime annotations have an unusual syntax: the names of lifetime parameters must start with an apostrophe `'` followed by generic name, e.g. `'a`, `'b`. It is very common to use short names as well.
Angle brackets in `<'a>` syntax simply defines lifetime using [generics syntax](https://doc.rust-lang.org/book/ch10-01-syntax.html#in-function-definitions). It is important to remember that annotations themselves don't change any lifetimes of references. It only helps borrow checker to understand how multiple references are related to each other.

Let's have one more example when working with structs containing references:

```rust
fn main() {
    let first_name = "John";
    let last_name = "Doe";
    let user = User { first_name: &first_name, last_name: &last_name };
    println!("Full name: {} {}", user.first_name, user.last_name);
}

struct User {
    first_name: &str,
    last_name: &str,
}
```

Quick rundown of the code:
* We have a `User` struct containing string references for first and last name
* In `main` function we create initial string values for first and last names and pass references to created `User` struct
* Values are printed using `println!` macro

As with previous example, we get similar compile time error about expected lifetime parameter and handy suggested solution:

```rust
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:17
  |
9 |     first_name: &str,
  |                 ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
8 | struct User<'a> {
9 |     first_name: &'a str,
  |

error[E0106]: missing lifetime specifier
  --> src/main.rs:10:16
   |
10 |     last_name: &str,
   |                ^ expected named lifetime parameter
   |
help: consider introducing a named lifetime parameter
   |
8  | struct User<'a> {
9  |     first_name: &str,
10 |     last_name: &'a str,
   |
```

What happens, is that both `first_name` and `last_name` fields are references and Rust cannot determine their lifetimes in relation to `User` struct, thus requiring explicit annotations for it as such:

```rust
fn main() {
    let first_name = "John";
    let last_name = "Doe";
    let user = User { first_name: &first_name, last_name: &last_name };
    println!("Full name: {} {}", user.first_name, user.last_name);
}

struct User<'a> {
    first_name: &'a str,
    last_name: &'a str,
}
```

The latter solution fixes the issue and we get values printed as expected: `Full name: John Doe`

Initially, lifetime annotations might appear intimidating and complicated but don't get discouraged. The sooner you become confident in using them, the sooner it will feel natural and part of your development flow. If you get stuck, feel free to have a look at official [Rust book](https://doc.rust-lang.org/book/) for more advanced content or even ask people in friendly communities on [discord](https://discord.com/invite/rust-lang) or [/r/rust](https://www.reddit.com/r/rust/).