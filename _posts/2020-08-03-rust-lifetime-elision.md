---
layout: post
title: "Rust language: lifetime elision"
author: Paulius Gudonis
---

While writing previous post on ["Rust lifetimes"]({% post_url 2020-07-13-rust-lifetimes %}), for brevity reasons I've decided to exclude details about lifetime elision and rather keep it for the next post.

If you followed post on ["Rust borrowing"]({%  post_url 2020-04-06-rust-borrowing %}), you most likely noticed that there were functions that accepted multiple references as parameters but did not require to explicitly write out what lifetimes were expected. To assure you, lifetimes are there. Rust does not magically drop lifetimes but rather allows to omit syntax when it is obvious what they are. This is very similar to type inference in strongly typed languages like Swift or Kotlin.

In very simple, deterministic cases, where function takes reference as a parameter but never returns one as such:

```rust
fn sum(x: &i32) -> i32
```

Then you don't need to explicitly write out lifetimes for parameters. Rust does that for you behind the scenes by providing each reference its own input lifetime parameter. Equivalent `sum` function would be:

```rust
fn sum<'a>(x: &'a i32) -> i32

// In case there are multiple references passed in the function signature, it would look something like this behind the scenes
fn sum<'a, 'b>(x: &'a i32, y: &'b i32) -> i32
```

In other unambiguous cases, where function takes a single reference as a parameter and returns a reference as such:

```rust
fn sum(x: &i32) -> &i32
```

Rust can determine and apply the rules for both: input and output lifetime parameters. It would look something like this behind the scenes for `sum` function:

```rust
fn sum<'a>(x: &'a i32) -> &'a i32
```

So far it is all nice and dandy, but what will happen if we try multiple reference parameters and return reference as well as such:

```rust
fn sum(x: &i32, y: &i32) -> &i32
```

You most likely guested it: at this point Rust cannot unambiguously determine lifetimes of each reference. Rust lifetime elision sets input lifetime parameters as follows:

```rust
fn sum<'a, 'b>(x: &'a i32, y: &'b i32) -> &i32
```

However, it is impossible to figure out the lifetime of the returned reference. That's why Rust compiler would complain and ask to annotate lifetimes explicitly. 

There's one caveat though. So far we only covered functions but in case there is an instance method with multiple parameters, Rust will automatically assign instance lifetime for the returned value, thus making the following method signature valid without explicit lifetime annotations:

```rust
impl Foo {
	fn bar(&self, x: &i32, y: &i32) -> &i32 {}

	// works for `&mut self` as well 
	fn bar(&mut self, x: &i32, y: &i32) -> &i32 {}
}   
```