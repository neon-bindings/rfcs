- Feature Name: closures
- Start Date: 2017-12-08
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Allow `JsFunction` to be initialized from Rust closures, allowing for more advanced interactions between Node.js and Rust.

# Motivation
[motivation]: #motivation

Currently, Rust functions that can be called from Node.js are very limited. Any data that the function can interact with must be explicitly passed to the function via arguments or `this` binding.

Oftentimes, an API may wich to make use of a continuation passing style to allow asynchronous operations. Consider the following example:

```rust
doWork(someData, (continue) => {
  doSomethingAsynchronously(otherData, (result) => { continue(result) });
});
```

In this example, the `doWork` function requires a callback parameter. This callback parameter is later invoked, passing another callback named `continue` to be invoked when the sub-operation is complete. A Rust implementation of `continue` cannot have any reference to the context in which it was created. The user of the API would be responsible for passing in all context information explicitly. This would create a clunky and needlessly complex API.

```rust
// Problem
fn bar(call: Call) -> JsResult<JsValue> {
    //Cannot reference world without explicit pass in
    let msg = foo + "bar";
    JsString::new(call.scope, msg)
}

let foo = String::from("foo");
let bar_func = JsFunction::new(scope, world);
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When creating a `JsFunction` instance, one simply passes in a boxed `FnMut` function, instead of a static function pointer. Consider the following example:

```rust
let foo = String::from("foo");
let bar_func = JsFunction::new(scope, Box::new(move |call| {
   let msg = foo + "bar";
   JsString::new(call.scope, msg)
}));
```

Any rust data can be moved into the closure, allowing for more advanced, context-aware functions that can be handed off to Node.js.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In order to implement this design, the static pointer held inside of a JsFunction would be replaced with a `Box` containing an `FnMut` reference. This "owned pointer" would ensure that only one reference to the data exists, and that it will be deallocated when the `JsFunction` is destroyed.

```rust
let foo = String::from("foo");
let bar_func = JsFunction::new(scope, Box::new(move |call| {
   let msg = foo + "bar";
   JsString::new(call.scope, msg)
}));
```

In this example, `JsFunction` will take ownership of the closure as well as all data moved into it. This ensures that Rust's safety guarantees are maintained and the resources are properly deallocated.

# Drawbacks
[drawbacks]: #drawbacks

Implementing this will add an extra layer of indirection internally to `JsFunction`, potentially leading to lower performance.

Additionally, the current implementation of `JsFunction` uses `catch_unwind` internally. Moving data inside of the `catch_unwind` closure requires the data to be marked as `UnwindSafe`. Please see the [UnwindSafe trait documentation](https://doc.rust-lang.org/std/panic/trait.UnwindSafe.html).

# Rationale and alternatives
[alternatives]: #alternatives

The only alternative to this design is to wrap all context information in a JavaScript object and pass it explicitly through arguments or `this` binding. This adds unecessary boilerplate and makes designing APIs much harder.

# Unresolved questions
[unresolved]: #unresolved-questions

The modified signature of the `JsFunction::new` method must still be looked at. During prototyping, this author could not determine a way to pass in `FnMut` closures directly without getting error messages about type sizes. Wrapping the closure in a `Box` resolved the errors mut makes for an ugly API. 

Additionally, should the module export API be modified to accept closures, or should it only allow static functions?
