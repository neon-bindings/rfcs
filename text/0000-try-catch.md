- Feature Name: try_catch
- Start Date: 2020-05-21
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a new `Context` method for catching any JavaScript exceptions that may be thrown during the execution of a computation. The result of the API uses Rust's standard `Result` enum to distinguish between normal return values and exceptional values:
```rust
match cx.try_catch(|cx| { cx.throw(42) }) {
    Ok(v) => { /* ... computation produced a normal result ... */ }
    Err(Catch(v)) => { /* ... computation threw an exception ... */ }
}
```

# Motivation
[motivation]: #motivation

Today, Neon APIs that throw JavaScript exceptions product a Rust `Result` with the `Throw` sentinel value, which simply indicates that the JS VM has entered into a throwing state. But they do not offer a way to model the JavaScript `catch` semantics, which restores the VM to non-throwing state and extracts the exceptional value that was being thrown.

This RFC proposes an idiomatic way to mimic JavaScript's `try`/`catch` syntax with a Neon API, which "tries" a computation encapsulated in a Rust callback and "catches" any exception that gets thrown and reflects that in an idiomatic Rust `Result`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Throwing vs non-throwing state

The JavaScript VM can be in a _throwing state_ or _non-throwing state_. Any API that can trigger the throwing of a JavaScript exception value sets the VM into throwing state. **When the VM is in a throwing state, most Neon APIs are not allowed to be called and will panic.**

Every Neon API is documented as either _safe for throwing state_, meaning that the API can safely be called without panicking even when the VM is in a throwing state, or _unsafe for throwing state_, meaning that it will panic if called when the VM is in a throwing state.

## Catching exceptions

The `try_catch` method runs a callback and intercepts any thrown exception value and ensures the JavaScript VM is restored to non-throwing state.

Example:

```rust
let v = match cx.try_catch(|cx| { cx.throw(v) }) {
    Ok(v) => v,
    Err(Catch(v)) => v
};
```

## Inspecting the exception state

At any time, the VM can be inspected to detect whether it is a throwing or non-throwing state:

```rust
cx.is_throwing()
```

The `is_throwing()` method is safe for throwing state.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `neon::result` module defines a `Catch` type that represents a caught exception value:

```rust
pub struct Catch<'a, T>(Handle<'a, T>);
```

The `neon::context::Context` trait gets two new methods:

```rust
pub trait Context<'a> {
    fn is_throwing(&self) -> bool;
    fn try_catch<T, U, F>(&self, f: F) -> Result<T, Catch<'a, U>>
    where
        T: Value,
        U: Value,
        F: for<'b> FnOnce(CatchContext<'b>) -> JsResult<'b, T>;
}
```

The `is_throwing` method can be implemented with the N-API [`napi_is_exception_pending`](https://nodejs.org/api/n-api.html#n_api_napi_is_exception_pending) method in the upcoming N-API backend, or the underlying V8 API in the legacy backend.

The `try_catch` method can be implemented with the N-API [`napi_get_and_clear_last_exception`](https://nodejs.org/api/n-api.html#n_api_napi_get_and_clear_last_exception), or with [`Nan::TryCatch`](https://github.com/nodejs/nan/blob/master/doc/errors.md#api_nan_try_catch) in the legacy backend.

# Drawbacks
[drawbacks]: #drawbacks

This adds a bit more size to the Neon API but it's a core missing primitive. Note also that it's a primitive in the V8 C++ API, the Nan API, and N-API. It's really just necessary. Otherwise people have to propagate all exceptions back to JS and catch them there.

# Rationale and alternatives
[alternatives]: #alternatives

Alternatives:
- A two-callback mirror of `try`/`catch` (e.g., `cx.try_catch(|cx| { cx.throw(42) }, |v, cx| { /* ... */ }))`)
- A builder pattern mirror of `try`/`catch` (e.g., `cx.try_catch(|cx| { cx.throw(42) }).with(|v, cx| { /* ... */ })`)

I tried these approaches before and they were just awkward and less readable. Note also the more awkward naming and awkward closure arguments.

Alternative naming:
- `cx.r#try()`: feels more correct but the lexical syntax is super unfortunate.
- `cx.catch()`: prettier than `try_catch` but not very accurate naming.

# Unresolved questions
[unresolved]: #unresolved-questions

- We need some implementation experience to get the API signatures exactly right, especially the lifetimes.
- We need some usage experience to see how ergonomic the reflection of exceptions into a `Result` type turns out to be.
