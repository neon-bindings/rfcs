- Feature Name: error_subtyping
- Start Date: 2018-07-09
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

To allow for the addition of new error types in the future, as well as to improve the ergonomics of creating and throwing error objects, this RFC replaces the `ErrorKind` enum with a set of new `Js___Error` types that are "subtypes" of `JsError` (using the protocol defined by Neon's `SuperType` trait).

# Motivation
[motivation]: #motivation

The `ErrorKind` enum has a fixed number of subtypes of JavaScript's `Error` class. But this excludes some types like `URIError`, and makes it impossible to add new subtypes in the future without breaking backwards compatibility (since you can't add variants to an enum without breaking semver compatibility).

Moreover, the ergonomics of creating and throwing an error with this API are not ideal:

```rust
JsError::throw(&mut cx, ErrorKind::TypeError, "undefined is not a function")
```

With this RFC, the above examples simplifies to:

```rust
JsTypeError::throw(&mut cx, "undefined is not a function")
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This simplifies the API documentation, since each type clearly corresponds to its analogously named global JavaScript function (e.g., `JsTypeError` corresponds to `TypeError`, etc).


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Each of the error types has the same constructor signature: a single string reference:

```rust
JsError::new(&mut cx, "...")
JsTypeError::new(&mut cx, "...")
JsReferenceError::new(&mut cx, "...")
JsRangeError::new(&mut cx, "...")
JsSyntaxError::new(&mut cx, "...")
```

Each error type also implements a `throw` convenience method:

```rust
JsError::throw(&mut cx, "...")
JsTypeError::throw(&mut cx, "...")
JsReferenceError::throw(&mut cx, "...")
JsRangeError::throw(&mut cx, "...")
JsSyntaxError::throw(&mut cx, "...")
```

The `Context` trait acquires convenience methods for constructing and throwing each of the error types:

```rust
cx.error("...")
cx.throw_error("...")

cx.type_error("...")
cx.throw_type_error("...")

cx.reference_error("...")
cx.throw_reference_error("...")

cx.range_error("...")
cx.throw_range_error("...")

cx.syntax_error("...")
cx.throw_syntax_error("...")
```

# Critique
[critique]: #critique

It could be argued this is polluting the Neon API surface area, but this just naturally corresponds to the global API surface of JavaScript.

The convenience methods are a bit of combinatorial explosion for the `Context` trait but I think the convenience outweighs the cost of the added messiness.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
