- Feature Name: date_api
- Start Date: 2018-03-21
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This proposal defines an API for creating and inspecting JavaScript `Date` objects.

# Motivation
[motivation]: #motivation

The `Date` type is part of the standard JavaScript library and should be exposed to Rust.

Example:

```rust
let date: Handle<JsDate> = JsDate::new(1431673200000)?;
let value: f64 = date.value(); // 1431673200000.0
```

# Pedagogy
[pedagogy]: #pedagogy

This is a relatively straightforward API and can be explained with the API docs.

# Details
[details]: #details

## `JsDate`

This API adds a `JsDate` type to `neon::types`:

```rust
impl JsDate {
    pub fn new<'a, S: Scope<'a>, V: Into<f64>>(_: &mut S, value: V) -> JsResult<JsDate>;
    pub fn value(self) -> f64;
}
```

Like [`neon::types::JsNumber`](https://docs.rs/neon/0.4.0/neon/types/struct.JsNumber.html), the constructor accepts any type coercible to `f64`.

## Dates are objects

The `JsDate` type is also an object type:

```rust
impl Object for JsDate { /* ... */ }
```

# Critique
[critique]: #critique

Previous revisions of this RFC offered automatic conversations to and from the standard library's [`std::time::SystemTime`](https://doc.rust-lang.org/std/time/struct.SystemTime.html) type. However, it's not entirely clear that this type is completely equivalent to JavaScript's `Date` API. Given that JavaScript's `Date` is explicitly interconvertible to double-precision floating-point numbers, the most basic primitive we can offer at first is an API based on `f64`. In future we could consider whether there are sensible coercions we can offer between `JsDate` and other Rust date/time APIs.

# Unresolved questions
[unresolved]: #unresolved-questions

None
