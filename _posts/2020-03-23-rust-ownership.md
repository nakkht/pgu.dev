---
layout: post
title: Rust and ownership
author: Paulius Gudonis
---

One of rust language's key concepts is ownership. It is important to understand it early on, as ownership rules are heavily enforced by the compiler.

> **Note**: This post is part of a series on Rust language features, including [lifetimes]({% post_url 2020-03-30-rust-lifetimes %}) and [borrowing[in-the-works]]().

There are three main rules:
* Value/instance belongs to a variable called owner
* Every value has a single owner
* Owner determines lifetime of the value

The last point ultimately means that if the owner is freed/deallocated from the memory or as in rust terminology - dropped, any owned values are promptly dropped as well.

To execute code snippets in this post without any prior setup, try [rust playground](https://play.rust-lang.org)

Let us start with the following example:

```rust
{						// closure starts
	let x: i32 = 42;		// value 42 assigned to variable x (x is the owner)
	println!("{}", x)		// value gets printed
}						// closure scope ends and all variables are dropped - variable x is no longer valid
```

The snippet will get compiled and print value 42 after execution in the terminal as expected.

Now let's take a look at a different example:

```rust
{								// closure starts
	let x = String::from("42");		// string of value "42" assigned to variable x (x is the owner)
	let y = x;						// value which belongs to x is moved to variable y (y is the owner of value "42")
	println!("{}", x)				// ERROR! borrow of moved value: `x`
}											
```

If we try to compile and run the snippet, besides getting a warning about unused variable `y`, you will notice that the program didn't even compile and spit out the error messages similar to the ones shown below:

```html
error: src/main.rs:4: borrow of moved value: `x`
error: src/main.rs:4: value borrowed here after move
error: src/main.rs:3: value moved here
error: src/main.rs:2: move occurs because `x` has type `std::string::String`, which does not implement the `Copy` trait
```

Let's investigate error messages.

First two lines indicate that on line 4 in `main.rs` file, we try to borrow value `x` (by calling `println!` macro) which was moved and is inaccessible. The third line instructively tells us that the line 3 in `main.rs` is where the move happened: variable `y` was assigned value of `x` and is now the 'owner' of the value "42". As you probably already noticed, the last line of the error messages already describes the underling cause - variable `x` has a type `std::string::String`, which does not implement `Copy` trait. To understand that, let's take a look at another example:

```rust
{								// closure starts
	let x: i32 = 42;				// value 42 assigned to variable x (x is the owner)
	let y = x;						// value which belongs to x is 'copied' to variable y (y is the owner of distinct value 42)
	println!("x: {} y: {}", x, y)		// value of x and y are printed
}								// closure scope ends and all variables are dropped - variable x and y are no longer valid		
```

To our surprise, the following snippet compiled, ran and printed out "x: 42 y: 42".	 So what happened? The answer is `Copy` type/trait. `String::from("42")` produces a String (not to be confused with primitive [string literal](https://doc.rust-lang.org/1.7.0/book/strings.html)) which does not conform to `Copy` trait and thus its value is moved from variable `x` to `y` whereas value 42 is type of `i32` which conforms to `Copy` trait and its value is copied rather than moved. Due to the following, the latter example is valid rust code without breaking any rules stated at the beginning. You could say that the `Copy` trait is an exception to 'move' semantics.

So far rules declared at the beginning of the post seemed to hold up. To ensure memory safety rust language brings ownership concept which is heavily enforced at compile time. This allows to avoid some those serious 'dangling pointer' bugs coming from the C language world. 