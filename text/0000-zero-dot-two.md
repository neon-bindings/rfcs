- Feature Name: zero_dot_two
- Start Date: 2018-06-17
- RFC PR: 
- Neon Issue: 

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

#### Upgrade requirements

TODO

### `ArrayBuffer` views

See: [`ArrayBuffer` views RFC](https://github.com/neon-bindings/rfcs/blob/master/text/0005-array-buffer-views.md)

#### Upgrade requirements

TODO

### Simplified module organization

See: [Simplified module organization RFC](https://github.com/neon-bindings/rfcs/pull/20)

#### Upgrade requirements

TODO

## Minor changes

### Infallible string constructor

See: [Infallible string constructor RFC](https://github.com/neon-bindings/rfcs/pull/21)

#### Upgrade requirements

TODO

### Remove `JsInteger`

The `JsInteger` type wasn't very well thought-out: it's an exposure of a V8 C++ class that optimizes some special cases of JavaScript numbers but was never very well documented and is non-standard. We should keep Neon engine-agnostic and in close correspondence with universal JS semantics.

#### Upgrade requirements

TODO

### Remove `Variant`

The `Variant` type was an experiment at representing `typeof` dispatch in Rust via pattern-matching. This is an attractive idea but has two fatal flaws:

1. Enums are flat, but even the core `typeof` types include some subclassing (namely: `function` vs `object`). This makes it tempting to want to include very standard class tagging schemes like `Array.isArray` in the core set of attributes.
2. Even the set of `typeof` types is growing over time (most notably with the recent addition of the `symbol` type), but enums cannot acquire new variants without a backwards-incompatible API change.

This just suggests that enums and `typeof` have irreconcilable impedance mismatches, and the trait-based approach that the rest of Neon uses is a better match for JavaScript's extensible subtyping.

#### Upgrade requirements

TODO

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

#### Upgrade requirements

TODO

### Remove `.callee()`

We recently discovered that Node 10 dropped support for its C++ API exposing the functionality of `arguments.callee`, and [removed the functionality with a dynamic error in Node 10](https://github.com/neon-bindings/neon/pull/314). This is a dubious feature to begin with. This RFC proposes removing it entirely.

#### Upgrade requirements

TODO

## CLI changes

### Default to `--debug`

#### Upgrade requirements

TODO

### Eliminate `NEON_ABI` environment variable

#### Upgrade requirements

TODO

# Critique
[critique]: #critique

We could freeze all past decision and avoid backwards-incompatible changes. TODO

We could make these changes one at a time. TODO

We could wait for more incompatible changes before releasing a new minor version. TODO

# Unresolved questions
[unresolved]: #unresolved-questions

- What will it take to make `--debug` work in Windows? This is an important question to answer, but it doesn't need to block us changing the default. Expectations from Cargo are strong, and this divergence is just an unnecessary pitfall.
