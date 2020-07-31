---
layout: post
title: "Rust language: lifetime elision"
author: Paulius Gudonis
---

While writing previous post on ["Rust lifetimes"]({% post_url 2020-07-13-rust-lifetimes %}), for brevity reasons I've decided to exclude details about lifetime elision and rather keep it for the next post.

If you followed post on ["Rust borrowing"]({%  post_url 2020-04-06-rust-borrowing %}), you most likely noticed that in some code snippets there were functions that accepted multiple references as parameters but did not require to explicitly annotate lifetimes. To assure you, lifetimes are there. Rust never forgets about lifetimes but rather allows to omit syntax when it is obvious what they are. Rust has something called _lifetime elision_ rules. It is somewhat similar to type inference in strongly typed languages like Swift or Kotlin but with some limitations.

In very simple, deterministic cases, where function takes arbitrary number of references as parameters but does not return one as such:

```rust
fn sum(x: &i32) -> i32
// or
fn sum(x: &i32, y: &i32) -> i32
// and so on
```

Then you don't need to explicitly annotate parameter lifetimes. Rust does that for you behind the scenes by providing each reference its own input lifetime parameter. Equivalent `sum` function would look like:

```rust
fn sum<'a>(x: &'a i32) -> i32
// And in case there are multiple references
fn sum<'a, 'b>(x: &'a i32, y: &'b i32) -> i32
// and so on
```

In other unambiguous cases, where function takes a single reference as a parameter and returns a reference as such:

```rust
fn sum(x: &i32) -> &i32
```

Rust assigns the same lifetime for both: input and output references (namely _input lifetime_ and _output lifetime_). It will look something like this behind the scenes for `sum` function:

```rust
fn sum<'a>(x: &'a i32) -> &'a i32
```

So far it is all nice and dandy, but what will happen if we try multiple reference parameters and return reference as well, as such:

```rust
fn sum(x: &i32, y: &i32) -> &i32
```

You most likely guested it: at this point Rust cannot unambiguously determine lifetimes of each reference. Rust lifetime elision will set input lifetime parameters as follows:

```rust
fn sum<'a, 'b>(x: &'a i32, y: &'b i32) -> &i32
```

However, it is impossible for Rust compiler to figure out the lifetime of the returned reference. That's why compiler would complain and ask to annotate lifetimes explicitly. 

There's one more interesting rule. So far we only addressed functions but in case there is an instance method (`fn` within `impl` blocks) with multiple parameters, lifetime elision will automatically assign instance lifetime for the returned value reference, given `&self` or `&mut self` is provided, thus making the following method signature valid without explicit lifetime annotations:

```rust
impl Foo {
	fn bar(&self, x: &i32, y: &i32) -> &i32
	// works for `&mut self` as well 
	fn bar(&mut self, x: &i32, y: &i32) -> &i32
}   
```

This drastically reduces amount of explicit annotations and repetitive code being written again and again. In addition, it is important to know that if _lifetime elision_ does not behave as you wanted, you can always add annotations explicitly.

---
Lifetime elision rules is a great initiative and effort from Rust team to make code less verbose. Even if it doesn't provide full lifetime inference as one might expect, it brings confidence that when more deterministic patterns appear, they could be added to Rust compiler. 