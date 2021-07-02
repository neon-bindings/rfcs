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
    Err(v) => { /* ... computation threw an exception ... */ }
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
    Err(v) => v
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

The `neon::context::Context` trait gets two new methods:

```rust
pub trait Context<'a> {
    // N-API only
    fn is_throwing(&self) -> bool;

    fn try_catch<T, F>(&mut self, f: F) -> Result<T, Handle<'a, JsValue>>
    where
        T: Sized,
        F: FnOnce(&mut Self) -> JsResult<'a, T>;
}
```

The `is_throwing` method can be implemented with the N-API [`napi_is_exception_pending`](https://nodejs.org/api/n-api.html#n_api_napi_is_exception_pending) method in the upcoming N-API backend. If the implementation would be too complicated, this method may not be supported with the legacy backend.

The `try_catch` method can be implemented with the N-API [`napi_get_and_clear_last_exception`](https://nodejs.org/api/n-api.html#n_api_napi_get_and_clear_last_exception), or with [`Nan::TryCatch`](https://github.com/nodejs/nan/blob/master/doc/errors.md#api_nan_try_catch) in the legacy backend.

If the callback to `try_catch` produces `Err(Throw)` but the VM is not actually in a throwing state, the `try_catch` function panics.

# Drawbacks
[drawbacks]: #drawbacks

This adds a bit more size to the Neon API but it's a core missing primitive. Note also that it's a primitive in the V8 C++ API, the Nan API, and N-API. It's really just necessary. Otherwise people have to propagate all exceptions back to JS and catch them there.

# Rationale and alternatives
[alternatives]: #alternatives

Alternatives:
- A two-callback mirror of `try`/`catch` (e.g., `cx.try_catch(|cx| { cx.throw(42) }, |v, cx| { /* ... */ }))`)
- A builder pattern mirror of `try`/`catch` (e.g., `cx.try_catch(|cx| { cx.throw(42) }).with(|v, cx| { /* ... */ })`)

I tried these approaches before and they were just awkward and less readable. Note also the more awkward naming and awkward closure arguments.

We also tried removing the context argument from the callback, since the API simply passes `self` down to the callback, but Rust's type rules for `&mut` references prevent the inner callback from referencing the context since calling `cx.try_catch()` effectively locks `cx`.

We considered making the type of `try_catch` generic in the return type as well, but this would require doing a downcast, and the goal of `try_catch` is to be infallible so that the `Result` return type only represents a caught JS exception.

Alternative naming:
- `cx.r#try()`: feels more correct but the lexical syntax is super unfortunate.
- `cx.catch()`: prettier than `try_catch` but not very accurate naming.

# Unresolved questions
[unresolved]: #unresolved-questions

- We need some usage experience to see how ergonomic the reflection of exceptions into a `Result` type turns out to be.
