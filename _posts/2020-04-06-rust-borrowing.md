---
layout: post
title: "Rust language: borrowing"
author: Paulius Gudonis
---

> **Note**: This post is a part of series on Rust language features, including ["ownership"]({% post_url 2020-03-23-rust-ownership %}) and ["lifetimes"]({% post_url 2020-04-13-rust-lifetimes %}).  
> To execute code snippets in this post without any prior setup, try [Rust playground](https://play.rust-lang.org).

Next important Rust language concept which is part of ownership is called borrowing. Let's jump straight into the following example:

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

As discussed in the [previous post]({% post_url 2020-03-23-rust-ownership %}) on ownership, this example won't compile and errors about moved values and missing `Copy` traits will be shown. Question is, can we do something about it and the answer is yes - take advantage of borrowing. Let's have a look at another example:

```rust
fn main() {
	let text = String::from("Hello 2020");
	let vec = vec![3, 42];
	let result = compute(&text, &vec);
	println!("{}, {:?}, {}",text, vec, result);
}

fn compute(str: &String, vec: &Vec<i32>) -> String {
	return format!("{} {}", str, &vec.into_iter().map(|v| v.to_string()).collect::<String>());
}
```

After building and running, the following output will appear: `Hello 2020, [3, 42], Hello 2020 342`

The subtle difference is that instead of passing value of `String` and `Vec` as a parameter to `compute` function, the references are provided (notice `&` character next to function parameter type definition `&String` and `&Vec<i32>`). In Rust terminology it is called "borrowing ownership". It allows to access value without affecting its ownership. This mechanism unlike moving values, does not deallocate the resource when it goes out of scope. That's why even after passing references to `compute` function in the last code snippet, it was still possible to print values of `text` and `vec` directly within `println!` macro in `main` function.

This brings us to the rules of borrowing:
* There can be more than one reference to a resource
* There can be only one mutable reference to a resource at the time
* Any borrow may not last longer than a scope of the owner

The second rule talks about mutable references. These type of references allow to mutate values that the reference is borrowing. Regular references, as you probably suspected, are by default immutable and that's the reason you can have more than one without losing memory safety. On the other hand, mutable reference can be only one at the time and is denoted in the following syntax `&mut T`. For example:

```rust
fn main() {
	let mut vec = vec![42];
	mutate_value(&mut vec);
	println!("{:?}", vec);
}

fn mutate_value(vec: &mut Vec<i32>) {
	vec.push(0);
}
```

The latter code snippet will compile, run and will output: `[42, 0]`

As you noticed `mutate_value` function signature contains parameter with type `&mut Vec<i32>` which indicates that the caller has to provide mutable reference of vector type. This allows to append value to vector and later print updated value within `main` function. Now let's see what rust compiler will do if multiple mutable references are introduced:

```rust
fn main() {
	let mut vec = vec![0];

	let reference1 = &mut vec;
	reference1.push(1);
	println!("{:?}", reference1);

	let reference2 = &mut vec;
	reference2.push(2);
	println!("{:?}", reference2);
}
```

Weirdly enough, the code compiles, runs and outputs two lines: `[0, 1]` and `[0, 1, 2]`

It turns out, Rust compiler is pretty smart and can determine whether two multiple references are simultaneous. In this case `reference1` and `reference2` are not simultaneous and because reference scope starts when it is declared and ends with the last usage, Rust compiler allows the code. Try visualizing it in scopes:

```rust
fn main() {
	let mut vec = vec![0];
	{									// scope starts
		let reference1 = &mut vec;		// reference1 binding to mutable vector is created
		reference1.push(1);
		println!("{:?}", reference1);		// reference1	 last usage
	}									// scope end end everything is deallocated
	let reference2 = &mut vec;
	reference2.push(2);
	println!("{:?}", reference2);
}
```

The latter code snippet is equivalent to the previous one, but with a nice curly bracket visual to emphasize `reference1` and `reference2` lifetimes. However, if we rearrange the code to something like this:

```rust
fn main() {
	let mut vec = vec![0];

	let reference1 = &mut vec;
	let reference2 = &mut vec;
	
	reference1.push(1);
	println!("{:?}", reference1);

	reference2.push(2);
	println!("{:?}", reference2);
}
```

The following error messages will appear as expected as no two mutable references are allowed:

```html
error: src/main.rs:5: cannot borrow `vec` as mutable more than once at a time
error: src/main.rs:5: second mutable borrow occurs here
error: src/main.rs:4: first mutable borrow occurs here
error: src/main.rs:7: first borrow later used here
```

Exact same applies if there's a mix of mutable/immutable references:

```rust
fn main() {
	let mut vec = vec![0];

	let reference1 = &vec;
	let reference2 = &mut vec;
	reference2.push(2);
	
	println!("{:?}, {:?}", reference1, reference2);
}
```

Rust does not allow having mutable and immutable references at the same time and it makes sense - would immutable reference hold true if the value actually is mutated underneath. All these constraints help to prevent data races at compile time: no two pointers will ever write and read values at the same time.

Finally, according to the last rule, any borrow may not last longer than a scope of the owner or in other words no reference will ever outlive data it references. Rust compiler will ensure no dangling pointers (dangling pointer - a pointer that references a location in memory that may have been deallocated or given to someone else). These kind of issues are often source of vulnerabilities and exploits in languages such as C. For example:

```c
#include<stdio.h>  
int main()  
{  
    char *str;  
    {  
        char a = 'A';  
        str = &a;  
    }  
    // a falls out of scope   
    // str is now a dangling pointer   
    printf("%s", *str);  
}  
```

So how does Rust language prevents it? Let's take a look at Rust equivalent version:

```rust
fn main() {
	let y;
	{
		let x = 42;
		y = &x;
	}
	println!("{}", y);
}
```

Compiling the latter code snippet will fail and the following error message will be shown:

```html
error: src/main.rs:5: `x` does not live long enough
error: src/main.rs:5: borrowed value does not live long enough
error: src/main.rs:6: `x` dropped here while still borrowed
error: src/main.rs:7: borrow later used here
```

Rust immediately pinpoints the issue - value `x` is dropped within line 6 and has a lifetime shorter than variable y, rendering it inaccessible on line 7.

In conclusion, Rust lives up to expectations of being memory safe and data-race free. Borrow checker does a magnificent job enforcing the rules and managing borrowing references. Furthermore, improved Rust compiler error messages describes issues pretty well, providing a better feedback on what went wrong.

In the next and last post on Rust ownership, we'll get into lifetimes.