- Feature Name: string_constructor
- Start Date: 2018-06-15
- RFC PR: https://github.com/neon-bindings/rfcs/pull/21
- Neon Issue: https://github.com/neon-bindings/neon/pull/322

# Summary
[summary]: #summary

It's almost impossible to fail to create a JS string from a Rust UTF-8 string, so the main Neon API for constructing `JsString` should be infallible, and panic on the (rare) failure cases. This RFC details a backwards-incompatible change to the `JsString` API, as well as `neon::vm::Context` shorthand methods to be considered a modification to the [VM 2.0](https://github.com/neon-bindings/rfcs/pull/14) RFC.

# Motivation
[motivation]: #motivation

The `JsString::new()` method currently returns an `Option` type, which almost never produces `None`. This is unnecessarily inconvenient. The only case where this can fail is when the input string exceeds the maximum string size the JS engine can handle. In recent V8 releases, this maximum size is [2^30 - 25 characters](https://github.com/nodejs/help/issues/712#issuecomment-396823718), which is large enough that for many use cases it's simply an internal bug if the maximum size is exceeded.

For cases where bounds on the length can't be predicted (such as strings coming from the user, for example from very large text files), there should be an opt-in version of the API that makes the error cases explicit with a `Result` type. The idiomatic Rust naming convention for this is `JsString::try_new()`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This API modification makes common string construction cases easier to learn and teach, since you don't have to explain why the `Option` type exists or how to handle it.

The `try_new` variant can be mentioned as useful for cases where strings might get very large. The error type `StringOverflow` makes it more self-explanatory why the failure occurred, so learners of the API are less likely to be confused why it's there or when they need to care about it.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Here's the detailed API:

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord, Clone, Debug)]
pub struct StringOverflow(usize);

pub type StringResult<'a> = Result<Handle<'a, JsString>, StringOverflow>;

impl<'a> JsResultExt<'a, JsString> for StringOverflow {
    fn unwrap_or_throw<'b, C: Context<'b>>(self, cx: &mut C) -> JsResult<'a, JsString> {
        match self {
            Ok(v) => Ok(v),
            Err(e) => JsError::throw(cx, Kind::RangeError, &format!(/* ... */))
        }
    }
}

impl JsString {
    pub fn new<'a, C: Context<'a>, S: AsRef<str>>(cx: &mut C, val: S) -> Handle<'a, JsString> {
        JsString::try_new(cx, val).expect(/* .. */)
    }

    pub fn try_new<'a, C: Context<'a>, S: AsRef<str>>(cx: &mut C, val: S) -> StringResult<'a>;
}

trait Context<'a> {
    ...
    fn string<S: AsRef<str>>(&mut self, val: S) -> Handle<'a, JsString>;
    fn try_string<S: AsRef<str>>(&mut self, val: S) -> StringResult<'a>;
}
```

# Critique
[critique]: #critique

We could make the allocation API always require explicit error handling. I don't believe the loss of convenience is worth it, because a) there are too many cases where it's just statically obvious an overflow isn't possible (e.g., calling it with a string literal), and b) this will only encourage developers to abuse `.unwrap()`.

We could skip the `try_new()` method, but this would make it too hard to deal with the cases where string lengths can't be statically bounded. And the paired `foo`/`try_foo` idiom has many strong precedents in Rust (e.g., the `Mutex` API in `std`).

We could try to expose the maximum string length in the API, so that programmers could check this limit themselves. But this is not specified in any spec, and in fact V8 has changed its limits over the years. So it's better just to let the underlying engine decide when to fail. If they need to know the historical string length limits, programmers can also encode them in a table exposed by a third-party Rust crate or even JS package, and then look up the current engine version at runtime via `process.versions` and use that as a key into the table of historical data.

# Unresolved questions
[unresolved]: #unresolved-questions

- Do we know for sure that [string construction can't throw a JS exception](https://twitter.com/littlecalculist/status/1008113770097303552)? I'm still trying to verify this.
