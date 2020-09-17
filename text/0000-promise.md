- Feature Name: `JsPromise`
- Start Date: 2020-09-17
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Provide an API for creating, resolving and rejecting JavaScript Promises.

```rust
fn return_a_promise(cx: FunctionContext) -> JsResult<JsPromise> {
    let (promise, deferred) = cx.promise();
    let msg = cx.string("Hello, World!");

    deferred.resolve(&cx, msg);

    Ok(promise)
}
```

# Motivation
[motivation]: #motivation

JavaScript Promise support has been a [long requested feature](https://github.com/neon-bindings/neon/issues/73). Promises are desirable for several reasons:

* Considered best practices in idiomatic JavaScript
* Enable `await` syntax for asynchronous operations
* More easily map to asynchronous operations in native code

Additionally, they can be combined with the [`EventQueue`](https://github.com/neon-bindings/rfcs/pull/32) API for very simple asynchronous threaded usage.

```rust
fn real_threads(cx: FunctionContext) -> JsResult<JsPromise> {
    let queue = cx.queue();
    let (promise, deferred) = cx.promise();

    std::thread::spawn(move || {
        let result = perform_complex_operation();
    
        queue.send(move || {
            let result = cx.number(result);

            deferred.resolve(&cx, result);
        });
    });

    Ok(promise)
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Before `Promise`, callbacks of the form `function (err, value)` were very common in JavaScript. Neon has excellent for for these "node style" callbacks in the `Task` trait.

```rust
fn fibonacci_async(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let n = cx.argument::<JsNumber>(0)?.value() as usize;
    let cb = cx.argument::<JsFunction>(1)?;

    FibonacciTask::new(n).schedule(cb);

    Ok(cx.undefined())
}
```

However, in idiomatic JavaScript, this should method should return a promise. This can be solved with some glue code in JavaScript:

```js
const util = require('util');
const native = require('../native');

export const fibonacciAsync = util.promisify(native.fibonacci_async);
```

Alternatively, a `Promise` can be constructed directly in Neon. Unlike an asynchronous method that accepts a callback, a method that returns a promise requires two parts:

* A `Promise` to return from the method
* A hook to resolve or reject the `Promise`

In JavaScript, this looks like the following:

```js
function asyncMethod() {
    return new Promise((resolve, reject) => {
        if (Math.random() > 0.5) {
            resolve();
        } else {
            reject();
        }
    });
}
```

In `Neon`, when a `JsPromise` is constructed, a `Deferred` object is also provided. It can be thought of as the following pattern in JavaScript:

```js
function deferred() {
    const deferred = {};
    const promise = new Promise((resolve, reject) => {
        deferred.resolve = resolve;
        deferred.reject = reject;
    });

    return [promise, deferred];
}

function asyncMethod() {
    const [promise, d] = deferred();

    setTimeout(() => d.resolve(), 5000);

    return promise;
}
```

This could be written in Neon with the following:

```rust
fn async_method(cx: FunctionContext) -> JsResult<JsPromise> {
    let queue = cx.queue();
    let (promise, d) = cx.promise();

    std::thread::spawn(move || {
        std::thread::sleep(std::time::Duration::from_millis(5000));

        queue.send(|cx| d.resolve(cx.undefined()));
    });

    Ok(promise)
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `JsPromise` API consists of two structs:

* `JsPromise`. Opaque value; only useful for passing back to JavaScript.
* `Deferred`. Handle for resolving and rejecting the related `Promise`.

They may only be created with the `deferred` method on `Context`.

```rust
trait Context {
    fn deferred(&mut self) -> (Handle<JsPromise>, Deferred);
}
```

## `JsPromise`

Opaque handle that represents a JavaScript `Promise`. It is not threadsafe.

```rust
/// A JavaScript `Promise`
#[repr(C)]
#[derive(Clone, Copy)]
pub struct JsPromise(raw::Local);

impl Value for JsPromise {}

impl Managed for JsPromise {
    fn to_raw(self) -> raw::Local { self.0 }
    fn from_raw(h: raw::Local) -> Self { JsPromise(h) }
}

impl ValueInternal for JsPromise {
    fn name() -> String { "promise".to_string() }
    fn is_typeof<Other: Value>(env: Env, other: Other) -> bool {
        unsafe { neon_runtime::tag::is_promise(env.to_raw(), other.to_raw()) }
    }
}

unsafe impl This for JsPromise {
    fn as_this(_env: Env, h: raw::Local) -> Self {
        JsPromise(h)
    }
}

impl JsPromise {
    pub(crate) fn new_internal<'a>(value: raw::Local) -> Handle<'a, JsPromise> {
        Handle::new_internal(JsPromise(value))
    }
}
```

## `Deferred`

`Send` handle for resolving or rejecting a `JsPromise`.

```rust
pub struct Deferred(*mut c_void);

unsafe impl Send for Deferred {}

impl Deferred {
    /// Resolves a deferred Promise with the value
    fn resolve<'a, C: Context<'a>, T: Value>(self, cx: &mut C, v: Handle<T>);

    /// Rejects a deferred Promise with the error
    fn reject<'a, C: Context<'a>, E: Value>(self, cx: &mut C, err: Handle<E>);

    /// Resolves or rejects a deferred Promise with the contents of a `Result`
    fn complete<'a, C: Context<'a>, T: Value, E: Value>(self, cx: &mut C, result: Result<T, E>);
}
```

Similar to [`Persistent`](https://github.com/neon-bindings/rfcs/pull/32), if a `Deferred` is not resolved or rejected, the `Promise` will leak. To help catch this and guide users towards a correct implementation, `Deferred` should `panic` on `Drop` if not used.

```rust
impl std::ops::Drop for Deferred {
    fn drop(&mut self) {
        panic!("JsPromise leaked. Deferred must be used.");
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

None? :grin:

# Rationale and alternatives
[alternatives]: #alternatives

## High Level Promise Tasks

Using `JsPromise` along with other async features requires careful and verbose usage of several features (`JsPromise`, `EventQueue` and `try_catch`). Neon could exclusively provide a high-level API similar to `Task` or the proposed `TaskBuilder`.

```rust
fn async_task(cx: FunctionContext) -> JsResult<JsPromise> {
    let promise = cx.task(|| /* perform async task */)
        .complete(|cx| /* convert to js types */);

    Ok(promise)
}
```

This API is very ergonomic, but removes a considerable amount of flexibility and power from the user. Instead, the initial implementation will focus on orthogonal core primitives like `Persistent`, `EventQueue`, `JsBox` and `JsPromise` that can later be combined into high-level APIs.

## Common `Deferred` pattern

Most promise libraries that provide a `Deferred` object provide the `promise` as part of that object. Neon might have a similar approach:

```rust
struct Deferred<'a> {
    handle: *mut c_void,
    promise: Handle<'a, JsPromise>,
}
```

The issue with this approach is that `Deferred` is `Send` and `JsPromise` is *not*. We would also need to provide a getter for the resolve/reject handle. That getter would need to _consume_ self because it cannot be used multiple times. This would result in worse ergonomics.

Instead a tuple is returned, similar to `(sender, receiver)` returned from `std::sync::mpsc::channel`.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should there be a constructor method on `JsPromise`? This is slightly awkward because it's impossible to create a `Promise` without also creating a `Deferred`.
