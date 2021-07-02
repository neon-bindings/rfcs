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
    pub const MIN_VALID_VALUE: f64 = -8640000000000000;
    pub const MAX_VALID_VALUE: f64 = 8640000000000000;

    pub fn new<'a, C: Context<'a>, V: Into<f64>>(_: &mut S, value: V) -> JsResult<JsDate>;
    pub fn value<'a, C: Context<'a>>(self, cx: &mut C) -> f64;
    pub fn is_valid<'a, C: Context<'a>>(self, cx: &mut C) -> bool;
}
```

Like [`neon::types::JsNumber`](https://docs.rs/neon/0.4.0/neon/types/struct.JsNumber.html), the constructor accepts any type coercible to `f64`.

The `is_valid()` convenience method checks whether the `value()` of a `Date` object is in the [legal time range](https://www.ecma-international.org/ecma-262/11.0/index.html#sec-time-values-and-time-range) of -8640000000000000 to 8640000000000000 (inclusive).

The [associated constants](https://doc.rust-lang.org/edition-guide/rust-2018/trait-system/associated-constants.html) `MIN_VALID_VALUE` and `MAX_VALID_VALUE` represent the minimum and maximum legal `JsDate` values, respectively.

## Dates are objects

The `JsDate` type is also an object type:

```rust
impl Object for JsDate { /* ... */ }
```

## Context method

We also add a convenience method to the `Context` trait:

```rust
pub trait Context<'a> {
    // ...
    fn date<V: Into<f64>>(&mut self, value: V) -> Handle<'a, JsDate>;
}
```

The behavior of `cx.date(value)` is equivalent to `JsDate::new(&mut cx, value)`.

# Critique
[critique]: #critique

Previous revisions of this RFC offered automatic conversations to and from the standard library's [`std::time::SystemTime`](https://doc.rust-lang.org/std/time/struct.SystemTime.html) type. However, it's not entirely clear that this type is completely equivalent to JavaScript's `Date` API. Given that JavaScript's `Date` is explicitly interconvertible to double-precision floating-point numbers, the most basic primitive we can offer at first is an API based on `f64`. In future we could consider whether there are sensible coercions we can offer between `JsDate` and other Rust date/time APIs.

We could check for whether an `f64` is in the legal time range and either produce a failing `Result` or even throw a JS exception, but this would not fully model the semantics of JS, which allows constructing dates from values outside the legal range. In order to faithfully model JS, we allow all `f64` values.

# Unresolved questions
[unresolved]: #unresolved-questions

None
