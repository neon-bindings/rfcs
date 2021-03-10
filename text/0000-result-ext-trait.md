- Feature Name: `ResultExt`
- Start Date: 2021-03-10
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary

[summary]: #summary

This RFC proposes a new trait `ResultExt` with an implementation for `std::result::Result`. The trait includes a method named `or_throw` which works similarly to that of `JsResultExt`, except that it does not require the success value to implement `neon::types::Value`.

# Motivation

[motivation]: #motivation

This feature would allow developers using Neon to call functions from `std` or third party libraries which return `Result`, and properly handle errors which may occur by converting these to `NeonResult` without needing to write their own boilerplate code, or repetitive code in function bodies.

Currently, developers using Neon with non-Neon `Result`-returning functions either have to implement a similar trait themselves, or write something like the following on a line-by-line basis, which is rather terse:

```rust
do_something().or_else(|e| cx.throw_error(e.to_string()))?
```

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Suppose we want to create a new file and write some text into it, from within a Rust function exported to JavaScript. We might first try writing the following:

```rust
use neon::prelude::*;
use std::fs::File;
use std::io::prelude::*;

fn create_file(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let mut file = File::create("hello.txt")?;
    file.write_all(b"Hello, world!")?;
    Ok(cx.undefined())
}
```

However, this would result in the following error:

```
error[E0277]: `?` couldn't convert the error to `neon::result::Throw`
```

File methods return `std::io::Result`, which has the error type of `std::io::Error`. As the error message states, this error can't automatically be converted to `neon::result::Throw`, the error type which `JsResult` uses.

To fix this, we can use `or_else` as follows:

```rust
use neon::prelude::*;
use std::fs::File;
use std::io::prelude::*;

fn create_file(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let mut file = File::create("hello.txt")
        .or_else(|e| cx.throw_error(e.to_string()))?;
    file.write_all(b"Hello, world!")
        .or_else(|e| cx.throw_error(e.to_string()))?;
    Ok(cx.undefined())
}
```

Using `or_throw`, we can achieve the same as the previous example using a shorter syntax:

```rust
use neon::prelude::*;
use std::fs::File;
use std::io::prelude::*;

fn create_file(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let mut file = File::create("hello.txt").or_throw(&mut cx)?;
    file.write_all(b"Hello, world!").or_throw(&mut cx)?;
    Ok(cx.undefined())
}
```

In this example, `or_throw` automatically creates a JavaScript error if either of the file methods fail, including the original error text.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This could be added to `result/mod.rs` in the following manner:

```rust
/// An extension trait for `Result` values that can be converted into `NeonResult` values by throwing a JavaScript
/// exception in the error case.
pub trait ResultExt<V> {
    fn or_throw<'a, C: Context<'a>>(self, cx: &mut C) -> NeonResult<V>;
}

impl<V, E: Display> ResultExt<V> for Result<V, E> {
    fn or_throw<'a, C: Context<'a>>(self, cx: &mut C) -> NeonResult<V> {
        match self {
            Ok(v) => Ok(v),
            Err(e) => cx.throw_error(e.to_string()),
        }
    }
}
```

It would also be advisable to include this in `prelude.rs` so that users don't have to `use` it explicitly.

In this proposed code, the trait is implemented for `Result` types where the error variant implements `Display`, which means it should automatically work for `std` and third-party library functions which return a `Result`. Advanced users could implement the trait for custom result types whose errors may not implement `Display`.

> This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

In the second example under [Guide-level explanation], we create a new `Result` if either of the `File::create` or `file.write_all` functions return an error. We call `to_string` on the original error and pass the string to `cx.throw_error` method, so that this text is available in the `Error` object thrown in JavaScript. `or_throw` achieves the same result with less repetitive code.

# Drawbacks

[drawbacks]: #drawbacks

- Developers using Neon could trivially implement this themselves
- Isn't that much shorter than using `or_else`
- Isn't useful if a developer wants to write a custom error message
- Only supports JS `Error` (see [Unresolved questions])

# Rationale and alternatives

[alternatives]: #alternatives

> Why is this design the best in the space of possible designs?

I believe this is the most elegant way to 'translate' Rust errors to JavaScript errors as it does not require repetitive code.

As I understand, it is not possible to automatically convert these errors because the context object is needed to create a JavaScript error, although this would be ideal if possible.

> What other designs have been considered and what is the rationale for not choosing them?

As of yet I have not considered any alternatives as various other ways of converting errors are already possible - suggestions welcome!

> What is the impact of not doing this?

Developers can continue to use `or_else` or other techniques to handle `Result` values that aren't compatible with `NeonResult`, however these do not seem as elegant.

# Unresolved questions

[unresolved]: #unresolved-questions

- Are there any 'more elegant' ways to convert errors that I haven't considered?
- Should this trait also be implemented for other `Result` types, e.g. where `E` doesn't implement `Display`?
- Should similar methods be written to throw other types of JavaScript errors, e.g. `TypeError`?
  - Should there be an extra parameter to specify the JS error type?
