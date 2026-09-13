+++
title = "Matching newtypes in function parameters"
description = "A small Rust trick: destructure newtypes like UserId(u64) directly in function parameters to avoid scattering .0 and .inner() calls across your code."
date = "2025-02-20"
keywords = ["newtype","Rust","trick"]

[extra]
repo_view = false
comment = false
+++

Here's a little "trick" I learned way too late which polluted a lot of my code with ``.0``, ``.inner()``, etc.
The [newtype](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) pattern is quite common in rust to guarantee that the right type is used.

E.g. this example prevents the possibility of adding a ``UserId`` to an ``OrderId`` even though adding two ``u64`` is totally fine, the semantics make no sense.

```rust,
struct UserId(u64);
struct OrderId(u64);
```

However, now you have to choose between writing ``id.0`` all the time, creating a new function that returns the inner type like ``id.inner()``, or implementing the ``Deref``/``DerefMut`` traits. 

Writing ``.0`` or ``.inner()`` all the time gets annoying very quickly and does not make the code look any better:

```rust,
fn work_with_id(id: UserId) {
    println!("The id is: {}", id.0);
    
    if id.0 > 123 {
        let id2 = id.0 + 1;
    }
    
    match id.0 {
        _ => 2
    };
}
```

However, implementing ``Deref`` would be considered bad practice as it's not designed for that use case but rather exclusively for [smart pointers](https://doc.rust-lang.org/std/ops/trait.Deref.html), because the *deref coercion* can be unexpected and it exposes all of the underlying type's methods and fields. 

Does it make sense to have a method ``rotate_right()`` for a ``u64``? Yes. Does it make sense for a ``UserId``? Probably not so much.

So what can we do?
It turns out pattern matching does not only work for ``if-let`` and ``match`` statements but also in function parameters. This is an easy way to destructure the newtype already in the parameters, so there is no more ``.0`` inside the function body:

```rust,
fn print_id(UserId(id): UserId) {
    println!("The id is: {}", id);
}
```

However, this of course only makes sense if the newtype cannot be confused with another type in the function's body. If we destructure the ``UserId`` and ``OrderId`` in a function's parameter list and then handle both types in the function body, we basically worked in a circle as we simply undo the newtype's benefits.

