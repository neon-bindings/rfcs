- Feature Name: date_api
- Start Date: 2018-03-21
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This proposal defines an API for creating and inspecting JavaScript `Date` objects.

# Motivation
[motivation]: #motivation

The JavaScript `Date` API is effectively quite similar to the Rust `std::time::SystemTime` API: a non-monotonic view of the system clock. This type should be reflected into Rust, and it should be possibly to conveniently interoperate between the corresponding JavaScript and Rust APIs.

Example:

```rust
let date: Handle<JsDate> = JsDate::now(scope)?;
let time: SystemTime = date.as_time();
```

# Pedagogy
[pedagogy]: #pedagogy

This is a relatively straightforward API and can be explained with the API docs.

The API docs should make sure to suggest the [`chrono`](https://github.com/chronotope/chrono) crate as a convenient way to work with date values.

# Details
[details]: #details

## `JsDate`

This API adds a `JsDate` type to `neon::js`:

```rust
impl JsDate {
    pub fn new<'a, S: Scope<'a>, V: AsTimeValue>(_: &mut S, value: V) -> JsResult<JsDate>;
    pub fn now<'a, S: Scope<'a>>(_: &mut S) -> JsResult<JsDate>;
    pub fn value(self) -> f64;
    pub fn as_time(self) -> SystemTime;
}
```

The `JsDate::new()` method takes any implementation of `AsTimeValue`, described below.

## `AsTimeValue`

The `AsTimeValue` trait is defined for value types (implementations of `Copy`) that can represent [time values](https://www.ecma-international.org/ecma-262/8.0/index.html#sec-time-values-and-time-range), which can be used to construct `Date` objects.

```rust
trait AsTimeValue: Copy {
    fn as_time_value(self) -> f64;
}
```

There are three implementors of `AsTimeValue`: `i32`, `f64`, and `SystemTime`.

```rust
impl AsTimeValue for i32 { /* ... */ }
impl AsTimeValue for SystemTime { /* ... */ }
impl AsTimeValue for f64 { /* ... */ }
```

The implementation of `AsTimeValue` for `SystemTime` determines the number of milliseconds since the Unix epoch by calling
```rust
SystemTime::now().duration_since(UNIX_EPOCH)
```
and then using `as_secs()` and `subsec_nanos()` to compute the right number.

## Dates are objects

The `JsDate` type is also an object type:

```rust
impl Object for JsDate { /* ... */ }
```

# Critique
[critique]: #critique

We could support other date/time libraries out of the box, such as the [`time`](https://github.com/rust-lang-deprecated/time) crate, or [`chrono`](https://github.com/chronotope/chrono). However, `time` is deprecated, and `chrono` defines full-fidelity coercions from the stdlib types to its richer API. So it's better just to stick to the standard library, which is the most stable option, and then recommend `chrono` in the API docs. Client code would look like:

```rust
let date: DateTime<Local> = js_date.as_time().into();
```

We could have explicit `from_XXX` methods instead of the `AsTimeValue` trait, but `JsDate::new()` is a nicer API and doesn't require clients to import the trait in order to use the constructor. So it's no less convenient and also more extensible.

# Unresolved questions
[unresolved]: #unresolved-questions

Are there any hidden impedance mismatches between `Date` and `SystemTime`? It seems like both represent non-monotonic time and can be computed as the elapsed time since the Unix epoch. `SystemTime` is higher-resolution but that seems OK, unless I'm missing some subtleties.
