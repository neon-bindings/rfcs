- Feature Name: zero_dot_two
- Start Date: 2018-06-17
- RFC PR: https://github.com/neon-bindings/rfcs/pull/22
- Neon Issue: https://github.com/neon-bindings/neon/pull/323

# Summary
[summary]: #summary

This RFC proposes a complete set of backwards-incompatible changes to Neon for the version 0.2 release.

# Motivation
[motivation]: #motivation

As we've maintained and used Neon over the last year or so, the Neon community has discovered a few issues that should be fixed incompatible on the way to a stable 1.0 release.

While 1.0 will make strong compatibility guarantees, we do intend to have a limited number of incompatible changes in the 0.x series. We aim to avoid either of two extremes: on the one hand, batching too many incompatible changes at once, making it too difficult for users to perform a single 0.x upgrade, and on the other hand, making too many individual one-off incompatible changes, exhausting users with a seemingly endless amount of churn.

So this RFC represents an attempt to batch up a small but ideally manageable number of incompatible changes for the 0.2 release, along with an outline of the work required to update existing code.

# Changes
[changes]: #changes

## Major changes

### VM 2.0

See: [VM 2.0 RFC](https://github.com/neon-bindings/rfcs/pull/14)

#### Migration guide

The primary changes are:

- Neon functions take `cx: CallContext` instead of `call: Call` as their single argument.
- Most APIs take `&mut cx` instead of `call.scope`.
- Locking the VM is `cx.borrow_mut(obj, |contents| { ... })` instead of `obj.grab(|contents| { ... })`.

### `ArrayBuffer` views

See: [`ArrayBuffer` views RFC](https://github.com/neon-bindings/rfcs/blob/master/text/0005-array-buffer-views.md)

#### Migration guide

The primary change is that you no longer need to call `as_slice()` or `as_mut_slice()` on a borrowed `JsArrayBuffer`'s contents, since it will already be a slice.

### Simplified module organization

See: [Simplified module organization RFC](https://github.com/neon-bindings/rfcs/pull/20)

#### Migration guide

Most code should just be able to eliminate all of their `use` declarations that import from Neon and replace them with a single `use neon::prelude::*;` declaration. Beyond that, it should be a mechanical change and the rustc type errors and suggestions should quickly guide the refactor.

## Minor changes

### Infallible string constructor

See: [Infallible string constructor RFC](https://github.com/neon-bindings/rfcs/pull/21)

#### Migration guide

Code that calls `JsString::new()` no longer produces an `Option`, so code that matches or `.unwrap()`s the results should be deleted. Code that calls `JsString::new_or_throw()` can be replaced with `JsString::try_new(...).unwrap_or_throw()` (make sure `JsResultExt` is in scope; it automatically is if you import `neon::prelude::*`).

### Error subtyping

See: [Error subtyping RFC](https://github.com/neon-bindings/rfcs/pull/23)

#### Migration guide

A relatively mechanical change: replace `JsError::new(..., Kind::FooError, ...)` or `JsError::throw(..., Kind::FooError, ...)` with either `cx.foo_error(...)` / `cx.throw_foo_error(...)` or `JsFooError::new(&mut cx, ...)` / `JsFooError::throw(&mut cx, ...)`.

### `JsBuffer::new()` vs `JsBuffer::uninitialized()`

To match modern `Buffer` behavior, and to be safer, the `JsBuffer::new()` constructor now always zero-fills the buffer it returns. The `JsBuffer::uninitialized()` constructor implements the legacy behavior, but is marked `unsafe` since it contains unspecified initial contents.

#### Migration guide

In most cases, code should continue to work unchanged. If you absolutely require the legacy behavior, change `JsBuffer::new()` calls to `JsBuffer::uninitialized()` and place the constructor call inside an `unsafe { }` block.

### Remove `JsInteger`

The `JsInteger` type wasn't very well thought-out: it's an exposure of a V8 C++ class that optimizes some special cases of JavaScript numbers but was never very well documented and is non-standard. We should keep Neon engine-agnostic and in close correspondence with universal JS semantics.

#### Migration guide

Use `JsNumber` instead, and cast from `f64` to `i32` as necessary. (`JsNumber::new()` can accept an i32 without casting.)

### Remove `Variant`

The `Variant` type was an experiment at representing `typeof` dispatch in Rust via pattern-matching. This is an attractive idea but has two fatal flaws:

1. Enums are flat, but even the core `typeof` types include some subclassing (namely: `function` vs `object`). This makes it tempting to want to include very standard class tagging schemes like `Array.isArray` in the core set of attributes.
2. Even the set of `typeof` types is growing over time (most notably with the recent addition of the `symbol` type), but enums cannot acquire new variants without a backwards-incompatible API change.

This just suggests that enums and `typeof` have irreconcilable impedance mismatches, and the trait-based approach that the rest of Neon uses is a better match for JavaScript's extensible subtyping.

#### Migration guide

Patterm matching on a `Variant` can be replaced with conditionals like:
```rust
if let Ok(s) = x.downcast::<JsString>() {
    // ...
else if let Ok(a) = `.downcast::<JsArray>() {
    // ...
} // ...
```

### Remove `ToJsString`

The `ToJsString` type is incorrectly documented in the API docs as an implementation of `\[\[ToString]]`. In fact, it's just a convenience trait for types that are infallibly convertible to `JsString`. It ends up not really being needed in the Neon API, and I don't _think_ it's highly used in the community. The name is awkward, the definition is unclear, and it's otherwise just taking up API space.

#### Migration guide

Change APIs that take `ToJsString` to just directly take `JsString` or `String` or `&str`.

### Rename `Key` methods

Since `Key::get` and `Key::set` have the same method names as `Object::get` and `Object::set`, they confuse the Rust compiler's mechanism for suggesting fixes when Neon users forget to import the `Object` trait. That is, when calling `obj.get(...)` or `obj.set(...)`, rustc suggests importing the `Key` trait, when the right suggestion would be `Object`.

Since nobody ever needs to use the `Key` trait directly, this RFC proposes changing the trait's API to:

```rust
trait PropertyKey {
    unsafe get_from(self, out: &mut raw::Local, obj: raw::Local) -> bool;
    unsafe set_from(self, out: &mut bool, obj: raw::Local, val: raw::Local) -> bool;
}
```

The longer name increases clarity and doesn't affect ergonomics since users never actually need to import it.

#### Migration guide

This should not require any explicit changes to code.

### Remove `.callee()`

We recently discovered that Node 10 dropped support for its C++ API exposing the functionality of `arguments.callee`, and [removed the functionality with a dynamic error in Node 10](https://github.com/neon-bindings/neon/pull/314). This is a dubious feature to begin with. This RFC proposes removing it entirely.

#### Migration guide

If an API needs access to the function being called, it can be passed in as an extra parameter, and a JS facade can be wrapped around the function to give it the right API for JS consumers.

## CLI changes

### Default to `--debug`

#### Migration guide

Build scripts should make sure to select the right build target. They can always be sure to get the target they want with an explicit `--debug` or `--release` argument.

### Eliminate `NEON_NODE_ABI` environment variable

This environment variable is no longer actually used so it shouldn't make a difference.

#### Migration guide

No migration required.

# Critique
[critique]: #critique

We could freeze all past decision and avoid backwards-incompatible changes. Neon is still at 0.1.x and while the community is growing, I think we still have a budget for incompatible changes.

We could make these changes one at a time. This would be frustrating to people who'd rather get their upgrades over with.

We could wait for more incompatible changes before releasing a new minor version. There's only so many changes a user can tolerate at once before they're going to either put off upgrading indefinitely or give up on Neon. Since VM 2.0 is a pretty major change, we should only have a small handful of smaller changes go along with it.

# Unresolved questions
[unresolved]: #unresolved-questions

- What will it take to make `--debug` work in Windows? This is an important question to answer, but it doesn't need to block us changing the default. Expectations from Cargo are strong, and this divergence is just an unnecessary pitfall.
