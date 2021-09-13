- Feature Name: `JsPromise` and `TaskBuilder`
- Start Date: 2020-09-17
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

## Summary
[summary]: #summary

This RFC provides for two related, but distinct features: `JsPromise` and `TaskBuilder`. Since users will frequently be interacting with both, designing the two in tandem will result in a better design.

### `JsPromise`

Provide an API for creating, resolving and rejecting JavaScript Promises.

```rust
fn return_a_promise(cx: FunctionContext) -> JsResult<JsPromise> {
    let (deferred, promise) = cx.promise();
    let msg = cx.string("Hello, World!");

    deferred.resolve(&mut cx, msg);

    Ok(promise)
}
```

### `TaskBuilder`

Provide an API for executing Rust closures on the libuv threadpool.

```rust
cx.task(|| { /* Executes asynchronously on the threadpool */ })
    .and_then(|cx, result| {
        // Executes on the main JavaScript thread
        Ok(())
    });
```

## Motivation
[motivation]: #motivation

### `JsPromise`

JavaScript Promise support has been a [long requested feature](https://github.com/neon-bindings/neon/issues/73). Promises are desirable for several reasons:

* Considered best practices in idiomatic JavaScript
* Enable `await` syntax for asynchronous operations
* More easily map to asynchronous operations in native code

### `TaskBuilder`

The legacy backend for Neon provided the [`Task`](https://docs.rs/neon/0.8.3/neon/task/trait.Task.html) trait for executing tasks on the libuv threadpool. However, boilerplate heavy implementations and ergonomic issues with ownership inspired the [`TaskBuilder` RFC](https://github.com/neon-bindings/rfcs/pull/30). This RFC is a successor, targeting only the Node-API backend with a heavy emphasis on promises.

Originally, it was thought that `Channel` could replace the need for running tasks on the libuv pool; however, there are many benefits to providing this functionality as well:

* Better CPU scheduling with a single task queue
* Easier integration for users that do not have a preferred threadpool

More details are available in the Node.js [guides](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/#don-t-block-the-worker-pool).

## Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### `JsPromise`

Before `Promise`, callbacks of the form `function (err, value)` were very common in JavaScript. These callbacks are the predominant form of continuation in Neon.

```rust
fn task(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let name = cx.argument::<JsString>(0)?.value(&mut cx);
    let cb = cx.argument::<JsFunction>(1)?.root(&mut cx);
    let channel = cx.channel();

    std::thread::spawn(move || {
        let greeting = format!("Hello, {}!", name);

        channel.send(move |mut cx| {
            let cb = cb.into_inner(&mut cx);
            let this = cx.undefined();
            let args = [cx.null().upcast::<JsValue>(), cx.string(greeting).upcast()];

            cb.call(&mut cx, this, args)?;

            Ok(())
        });
    });

    Ok(cx.undefined())
}
```

However, in idiomatic JavaScript, this method should return a promise. This can be solved with some glue code in JavaScript:

```js
const util = require('util');
const native = require('../native');

export const taskAsync = util.promisify(native.task);
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

    return [deferred, promise];
}

function asyncMethod() {
    const [d, promise] = deferred();

    setTimeout(() => d.resolve(), 5000);

    return promise;
}
```

This could be written in Neon with the following:

```rust
fn async_method(cx: FunctionContext) -> JsResult<JsPromise> {
    let channel = cx.channel();
    let (deferred, promise) = cx.promise();

    std::thread::spawn(move || {
        std::thread::sleep(std::time::Duration::from_millis(5000));

        channel.send(move |mut cx| {
            let value = cx.undefined();
            deferred.resolve(&mut cx, value);
            Ok(())
        });
    });

    Ok(promise)
}
```

However, there are still some subtle gotchas. What if there is a JavaScript exception? What if I don't resolve the `Deferred`? To help, `Deferred::settle_with` is provided as a helper that combines `cx.try_catch(...)` with `Deferred` resolution or rejection:

```rust
channel.send(move |mut cx| {
    deferred.settle_with(&mut cx, |cx| Ok(cx.undefined()));
});
```

### `TaskBuilder`

Tasks are asynchronously executed units of work performed on the libuv threadpool. A task consists of two parts:

* An `execute` callback that runs on the libuv threadpool
* A `complete` callback that receives the result of `execute` and runs on the main JavaScript thread

The `execute` code is typically the slow _native_ operation asynchronously executed, while `and_then` handles converting back to JavaScript values and resuming JavaScript execution with the result.

For example, consider the previous `JsPromise` example that executes on a Rust thread. A similar result can be achieved by running on the libuv threadpool instead:

```rust
fn async_method(cx: FunctionContext) -> JsResult<JsPromise> {
    let (deferred, promise) = cx.promise();

    cx.task(|| std::thread::sleep(std::time::Duration::from_millis(5000)))
        .and_then(move |cx, _result| {
            let value = cx.undefined();
            deferred.resolve(&mut cx, value);
            Ok(())
        });

    Ok(promise)
}
```

However, since most of the time tasks will complete by settling a promise, a convenience method is provided to handle creating and settling a promise:

```rust
fn async_method(cx: FunctionContext) -> JsResult<JsPromise> {
    let promise = cx
        .task(|| std::thread::sleep(std::time::Duration::from_millis(5000)))
        .promise(|cx, _result| Ok(cx.undefined()));

    Ok(promise)
}
```

`TaskBuilder::and_then` and `TaskBuilder::promise` are both thunks that consume the builder, creating and queueing the task.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `JsPromise`

The `JsPromise` API consists of two structs:

* `JsPromise`. Opaque value; only useful for passing back to JavaScript.
* `Deferred`. Handle for resolving and rejecting the related `Promise`.

`JsPromise` may only be constructed directly with `cx.promise()` instead of with `JsPromise::new` or `Deferred::new`. This is because they may not be constructed independently and it follows the convetion of `std::sync::mpsc::channel`.

```rust
trait Context {
    fn promise(&mut self) -> (Deferred, Handle<JsPromise>);
}
```

#### `JsPromise`

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

impl Object for JsPromise {}
```

#### `Deferred`

`Send` handle for resolving or rejecting a `JsPromise`.

Resolving a `Deferred` always takes ownership of `self` and consumes the `Deferred` to prevent double settlement.

```rust
pub struct Deferred(*mut c_void);

unsafe impl Send for Deferred {}

impl Deferred {
    /// Resolves a deferred Promise with the value
    pub fn resolve<'a, T: Value, C: Context<'a>>(self, cx: &mut C, v: Handle<T>);

    /// Rejects a deferred Promise with the error
    pub fn reject<'a, E: Value, C: Context<'a>>(self, cx: &mut C, err: Handle<E>);

    /// Resolves or rejects a Promise with the result of a closure
    pub fn settle_with<'a, T, C, F>(self, cx: &mut C, f: F)
    where
        T: Value,
        C: Context<'a>,
        F: FnOnce(&mut C) -> JsResult<'a, T>;
}
```

Similar to [`Root`](https://github.com/neon-bindings/rfcs/pull/32), if a `Deferred` is not resolved or rejected, the `Promise` chain will leak. To help catch this and guide users towards a correct implementation, `Deferred` should `panic` on `Drop` if not used on Node-API < 4 and be dropped on a global drop queue in Node-API >= 4.

```rust
impl std::ops::Drop for Deferred {
    fn drop(&mut self) {
        panic!("JsPromise leaked. Deferred must be used.");
    }
}
```

#### `Channel` extension

A new method `Channel::settle_with` is added to reduce the boilerplate of nested closures from `Channel::send` and `Deferred::settle_with`. Example without the convenience method:

```rust
channel.send(move |mut cx| {
    deferred.settle_with(&mut cx, |cx| Ok(cx.undefined()));
    Ok(())
});
```

Example with `Channel::settle_with`:

```rust
channel.settle_with(deferred, |cx| Ok(cx.undefined()));
```

```rust
impl Channel {
    /// Settle a deferred promise with the value returned from a closure or the
    /// exception thrown
    ///
    /// Panics if sending fails
    pub fn settle_with<V, F>(&self, f: F)
    where
        V: Value,
        for<'a> F: FnOnce(&mut TaskContext<'a>) -> JsResult<'a, Value> + Send + 'static,
    {
        todo!()
    }

    /// Settle a deferred promise with the value returned from a closure or the
    /// exception thrown
    pub fn try_settle_with<V, F>(&self, f: F) -> Result<(), SendError>
        where
            V: Value,
            for<'a> F: FnOnce(&mut TaskContext<'a>) -> JsResult<'a, Value> + Send + 'static,
    {
        todo!()
    }
}
```

### `TaskBuilder`

`TaskBuilder` follows the [builder pattern](https://doc.rust-lang.org/1.0.0/style/ownership/builders.html) for constructing and scheduling a task.

```rust
pub struct TaskBuilder<'cx, C, E> {
    // Hold a reference to `Context` so thunks do not require it as an argument
    cx: &'cx mut C,
    // Callback to execute on the libuv threadpool; complete is provided as part
    // of the thunk
    execute: E,
}

impl<'a: 'cx, 'cx, C, O, E> TaskBuilder<'cx, C, E>
where
    C: Context<'a>,
    O: Send + 'static,
    E: FnOnce() -> O + Send + 'static,
{
    /// Constructs a new `TaskBuilder`
    ///
    /// See [`Context::task`] for more idiomatic usage
    pub fn new(cx: &'cx mut C, execute: E) -> Self {
        Self { cx, execute }
    }

    /// Creates a promise and schedules the task to be executed, resolving
    /// the promise with the result of the callback
    pub fn promise<V, F>(self, complete: F) -> Handle<'a, JsPromise>
        where
            V: Value,
            for<'b> F: FnOnce(&mut TaskContext<'b>, O) -> JsResult<'b, V> + Send + 'static,
    {
        todo!()
    }

    /// Schedules the task to be executed, invoking the `and_then` callback from
    /// the JavaScript main thread with the result of `execute`
    pub fn and_then<F>(self, complete: F)
        where
            F: FnOnce(&mut TaskContext, O) -> NeonResult<()> + Send + 'static,
    {
        todo!()
    }
}

trait Context<'a> {
    fn task<'cx, O, E>(&'cx mut self, execute: E) -> TaskBuilder<Self, E>
        where
            'a: 'cx,
            O: Send + 'static,
            E: FnOnce() -> O + Send + 'static,
    {
        TaskBuilder::new(self, execute)
    }
}
```

Internally, `TaskBuilder` uses [`napi_async_work`](https://nodejs.org/api/n-api.html#n_api_simple_asynchronous_operations) for creating and scheduling tasks.

## Drawbacks
[drawbacks]: #drawbacks

The most significant drawback is that the user is presented with many more options for asynchronous programming without clear direction on which to use. Neon will need to lean on documentation to guide users to the correct APIs for various situations. 

## Rationale and alternatives
[alternatives]: #alternatives

### Common `Deferred` pattern

Most promise libraries that provide a `Deferred` object provide the `promise` as part of that object. Neon might have a similar approach:

```rust
struct Deferred<'a> {
    handle: *mut c_void,
    promise: Handle<'a, JsPromise>,
}
```

The issue with this approach is that `Deferred` is `Send` and `JsPromise` is *not*. We would also need to provide a getter for the resolve/reject handle. That getter would need to _consume_ self because it cannot be used multiple times. This would result in worse ergonomics.

Instead a tuple is returned, similar to `(sender, receiver)` returned from `std::sync::mpsc::channel`.

### Low level async work

Neon could provide a lower level `AsyncWork` or `trait` that only accepts static function pointers and `&mut data` instead of consuming data. However, this would push complexity down for threading consumable state, try/catch and several other subtle issues. It is also boilerplate heavy.

### Catching panics

A `panic` in a `TaskBuilder::promise` will drop the promise without resolution. We could catch the unwind and convert to a rejection; however, catching panics is not considered best practice. Since we do not require catching unwind to prevent unsound unwinding across FFI, it's better to have a course grained control where the `Deferred` rejects when it drops without being settled.

## Future Expansion

Node-API allows async work to be [cancelled](https://nodejs.org/api/n-api.html#n_api_napi_cancel_async_work) after it has been scheduled, but before it has executed. This is left as future improvement since we do not have a clear use case in mind and it can be added without breaking changes.

Two patterns for adding it without breaking change:
* Add `fn handle(&self) -> TaskHandle` to `TaskBuilder` to allow creating cancellation handles before scheduling
* Add methods that return a cancellation handle in addition to the thunk (e.g., `fn cancellable_promise(self, f: F) -> (TaskHandle, Handle<JsPromise>)`

## Unresolved questions
[unresolved]: #unresolved-questions

* ~~Are the thunk names `TaskBuilder::promise` and `TaskBuilder::complete` clear?~~ `TaskBuilder::and_then` is a better match for Rust naming conventions.
* ~~Is it helpful to mirror deferred constructors across all types instead of only having `cx.promise()`?~~ No, they will be removed to match `std::sync::mpsc::channel`.
* ~~Is `cx.promise()` a good name or would `cx.deferred()` be better?~~ `cx.promise()` is better because `JsPromise` is the primary thing wanted and `Deferred` is an implementation detail.
* ~~Is `Channel::send_and_settle` a good name? Is this API valuable or is the nested closure acceptable and clearer?~~ `Channel::settle_with` helps to describe the function by matching `Deferred::settle_with`.
