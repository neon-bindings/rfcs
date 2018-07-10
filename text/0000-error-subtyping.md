- Feature Name: error_subtyping
- Start Date: 2018-07-09
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes a backwards-incompatible change to the construction of `JsError` values. It makes the following changes:
- Replaces the `ErrorKind` enum with simple methods for constructing various error types, to make it possible to add new error types compatibly in the future.
- Reduces the set of error types to `Error`, `RangeError`, and `TypeError`, since these are currently the only error types supported by N-API.

# Motivation
[motivation]: #motivation

The `ErrorKind` enum has a fixed number of subtypes of JavaScript's `Error` class. But this excludes some types like `URIError`, and makes it impossible to add new subtypes in the future without breaking backwards compatibility (since you can't add variants to an enum without breaking semver compatibility).

Moreover, the ergonomics of creating and throwing an error with this API are not ideal:

```rust
JsError::throw(&mut cx, ErrorKind::TypeError, "undefined is not a function")
```

With this RFC, the above examples simplifies to:

```rust
cx::throw_type_error("undefined is not a function")
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This simplifies the API documentation, since each type clearly corresponds to its analogously named global JavaScript function (e.g., `JsError::type_error()`, `cx.type_error()`, and `cx.throw_type_error()` correspond to `TypeError`, etc).


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Each of the error types has the same constructor signature: a single string reference:

```rust
JsError::error(&mut cx, "...")
JsError::type_error(&mut cx, "...")
JsError::range_error(&mut cx, "...")
```

The `Context` trait acquires convenience methods for constructing and throwing each of the error types:

```rust
cx.error("...")
cx.throw_error("...")

cx.type_error("...")
cx.throw_type_error("...")

cx.range_error("...")
cx.throw_range_error("...")
```

# Critique
[critique]: #critique

It could be argued this is polluting the Neon API surface area, but this just naturally corresponds to the global API surface of JavaScript.

The convenience methods are a bit of combinatorial explosion for the `Context` trait but I think the convenience outweighs the cost of the added messiness.

At first I thought we'd have different subtypes for each error type, e.g. `JsRangeError` and `JsTypeError`. But these don't have distinct tags to test for type membership at runtime.

We could even eliminate the `JsError` type, since there's no standard JS `Error.isError` tag test (although it's been proposed in TC39 before). So the error types would just return `JsObject` instances instead of `JsError` instances. However, since N-API has standardized on [`napi_is_error`](https://nodejs.org/api/n-api.html#n_api_napi_is_error_1), it seems engines are committing to exposing this API to Node plugins. So we should support it as a standard primitive of the Node platform.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
