+++
title = "Not in 'a lifetime"
date = "2024-11-18"

[extra]
repo_view = false
comment = false
+++

<!-- https://www.youtube.com/watch?v=iVYWDIW71jk -->
<!-- https://rufflewind.com/img/rust-move-copy-borrow.png -->
<!-- https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md -->
<!-- https://www.youtube.com/watch?v=8-KLX1PGg8Q -->

# What is 'a lifetime?

Graphic memory over time

example: &'a &'b T where 'a is longer than 'b

```rust,
fn main() {
    let a = 123;
    let b = 312;
}
```

<center>
Unrelated Lifetimes
</center>

```rust,
fn main() {
    let a = 123;
    let b = &a;
}
```

<center>
Related Lifetimes
</center>

```rust,
fn main() {
    let r;
    {
        let x = 42;
        r = &x;
    }
    println!("r: {r}"); // x does not live long enough
}
```

<center>
Lifetime error
</center>

```rust,
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

<center>
Multiple Lifetimes as arguments
</center>

```rust,
fn main() {
    let s = String::new();
    let x: &'static str = "Hello world";
    let mut y = &s;
    y = x; // works because of variance (covariance)
}
```

<center>
Subtyping / Variance / Covariants
</center>

Three types of variance: Covariance, contravariance, invariance.
Most things are covariant.

```rust,
fn foo(&'a str) {}

fn main() {
    foo(&'a str)
    foo(&'static str)

    // foo<'a>(&'static str) NOT GENERIC

    let x: &'a str;
    x = &'a str;
    x = &'static str;
}
```

<center>
Subtyping / Variance / Covariants
</center>

```rust,
fn main() {
    fn example<'short>(_: &'short str) {}
    let f: for<'long> fn(&'long str) = example;
}
```

<center>
Contravariance
</center>

# Unsoundness

https://github.com/Speykious/cve-rs
https://github.com/rust-lang/rust/issues/25860

# Stuff

"Can I be wrong using lifetimes?" -> not in regard to memory safety. But yes in regards to program semantics
Compiler: Liveness -> A variable is live if its current value may be used late rin the program. From creation to last use.
'a is a name for the lifetime of a memory object a reference points to
'a : 'b -> implicitly used everywhere
lifetimes are easy as a concept. Its hard to map the rust syntax sugar on this model
rust hides a lot of lifetime mumbo jumbo... good for everyday programming, bad for building a mind model
Struct Foo<'a, 'b>
add permissions to model? mutable pointer to same object can only occurr after another. Immutable can be there simultaniously
Its possible to invalidate a references and re validate it again before its used
in simple programs lifetime == scope. But in more complex apps its not

# end

with all CVE stuff you can now write C in Rust... congratz
