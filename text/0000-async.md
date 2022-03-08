- Feature Name: Minimum Async
- Start Date: 2022-01-19
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Both JavaScript and Rust provide asynchronous, concurrent control flow. The goal of this RFC is to provide a minimal feature set to connect the two worlds.

It does **not** attempt to allow writing `async` Rust functions that translate to JavaScript `async` functions through proc macros or other means.

# Motivation
[motivation]: #motivation

It can be difficult and error-prone to move back and forth between the JavaScript main thread and Rust threads. Two frequent questions are posed in GitHub and Slack:

1. How do I use `tokio` (or Futures) with Neon?
2. How do I `await` a `Promise` in Rust?

Neon should provide a clear answer to these questions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

### Channel

`neon::event::Channel` provides the core primitive for scheduling work to be performed on the main JavaScript thread from another thread.

```rust
fn example(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let channel = cx.channel();

    std::thread::spawn(move || {
        // Code here does not block the JavaScript thread, but cannot execute JavaScript

        // Call back to the JavaScript main thread; JavaScript can be executed
        channel.send(|mut cx| {
            // Code in here *does* block the main JavaScript thread
            let version = cx.global()
                .get::<JsObject, _, _>(&mut cx, "process")?
                .get::<JsString, _, _>(&mut cx, "version")?
                .value(&mut cx);

            println!("Node version: {}", version);
        });
    });

    Ok(cx.undefined())
}
```

However, code may need the result of the JavaScript computation. `Channel::send` returns a [`JoinHandle`][neon-join-handle] that can be used to "join" on the result of the computation.

```rust
let version = channel
    .send(|mut cx| {
        cx.global()
            .get::<JsObject, _, _>(&mut cx, "process")?
            .get::<JsString, _, _>(&mut cx, "version")?
            .value(&mut cx)
    })
    .join()
    .unwrap();

println!("Node version: {}", version);
```

`neon::event::JoinHandle` is conceptually similar to [`std::thread::JoinHandle`][std-join-handle] and returns a `Result`. The Neon `JoinHandle` may fail to `join` if the `Channel` closure panicked or threw a JavaScript exception.

An issue with `join` in an `async` context is that it blocks the current thread. When using Neon with async Rust, a `JoinHandle` may be awated asynchronously without blocking:

```rust
fn example(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let channel = cx.channel();

    tokio::spawn(async move {
        let version = channel
            .send(|mut cx| {
                cx.global()
                    .get::<JsObject, _, _>(&mut cx, "process")?
                    .get::<JsString, _, _>(&mut cx, "version")?
                    .value(&mut cx)
            })
            .await
            .unwrap();

        println!("Node version: {}", version);
    });

    Ok(cx.undefined())
}
```

[neon-join-handle]: https://docs.rs/neon/0.10.0-alpha.3/neon/event/struct.JoinHandle.html
[std-join-handle]: https://doc.rust-lang.org/std/thread/struct.JoinHandle.html

### Promise

[Promises][promise-mdn] are JavaScript's idiomatic mechanism for representing the eventual completion or failure of an asynchronous operation. From JavaScript, it is typical to `await promise` to defer the continuation of the currently executing code until the promise resolves. Similarly, Rust has `.await` for Futures.

However, in Neon, `JsPromise` are simply objects and can not be directly awaited. The `to_future` adapter can be used to convert them to a Rust `Future`.

```rust
fn example(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let channel = cx.channel();
    let get_version_async = cx.argument::<JsFunction>(0)?;
    let promise = get_version_async
        .call_with(&cx)
        .apply::<JsPromise, _>(&mut cx)?;

    let future = promise.to_future(|mut cx, res| match res {
        Ok(value) => value.downcast_or_throw::<JsString, _>(&mut cx)?.value(&mut cx),
        Err(_) => todo!("Handle rejected promise"),
    });

    tokio::spawn(async move {
        let version = future.await.unwrap();

        println!("Node version: {}", version);
    });

    Ok(cx.undefined())
}
```

### `JsFuture`

`JsFuture` represents a `JsPromise` that has been adapted to a `Future`. It can fail for the same reasons as `channel.send(..).await`:

* The closure panicked
* The event loop stopped

Since it has the same failure conditions, it also shares an error type (`JoinError`).

In order for the future to be `Send + Static` and work with multi-threaded runtimes, the returned data must also be `Send + Static`. This means that JavaScript values must be converted to Rust types when returning from the `to_future` closure.

[promise-mdn]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Feature Flag

Async features should be left behind an `async` feature flag. Unlike many of the Neon feature flags that are for pre-release features, this feature flag would be *permanent*. The reason `async` will be opt-in is because it opts into behavior that users may not want:

* `JoinHandle` is a `Future` and `JoinHandle::join()` is removed
* Additional crate dependencies (e.g., `futures`)

### Channel

With `features = ["async"]`, `Channel::join` is removed. This is because it relies on a synchronous oneshot channel that will be replaced with an asynchronous oneshot channel to support `impl Future for JoinHandle`. Additionally, if a user is opting into using futures, they most likely do not want a blocking API and if they do want to block, they can use their runtime's `block_on` behavior.

```rust
impl Channel {
    pub fn try_send<T, F>(&self, f: F) -> Result<JoinHandle<T>, SendError>
        where
            T: Send + 'static,
            F: FnOnce(TaskContext) -> NeonResult<T> + Send + 'static,
    {
        let (tx, rx) = futures::channel::oneshot();

        /* ... */

        Ok(JoinHandle { rx })
    }
}

pub struct JoinHandle<T> {
    rx: futures::channel::oneshot::Receiver<NeonResult<T>>,
}

impl<T> Future for JoinHandle<T> {
    type Output = Result<T, JoinError>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut task::Context) -> Poll<Self::Output> {
        match Pin::new(&mut self.rx).poll(cx) {
            // Future is still pending
            Poll::Pending => Poll::Pending,

            // Failure polling means the channel was dropped without sending
            Poll::Ready(Err(_)) => Poll::Ready(Err(JoinErrorType::Panic)),

            // Exception thrown
            Poll::Ready(Ok(Err(Throw))) => Poll::Ready(Err(JoinErrorType::Throw)),

            // Success
            Poll::Ready(Ok(Ok(value))) => Poll::Ready(Ok(value)),
        }
    }
}
```

### `JsPromise` and `JsFuture` 

`JsPromise` provides `JsPromise::to_future` which adapts a `Promise` to a `Future`.

Conceptually, adapting a `JsPromise` uses the following steps:

* Create oneshot async channel to send the result
* Create a `JsFunction` from a closure that maps to success and sends on the channel 
* Create a `JsFunction` from a closure that maps to failure and sends on the channel
* Call `.then(..)` and `.catch(..)` on the `JsPromise` with the two functions 

Spec compliant `Promise` should only call either `then` or `catch` at most once. Rust cannot understand these guarantees requiring either some unsafe code or wrappers. This is hidden from the user and Neon may start with more expensive wrapping and optimize later if it is a bottleneck.

```rust
impl JsPromise {
    pub fn to_future<'a, O, C, F>(&self, cx: &mut C, f: F) -> NeonResult<JsFuture<O>>
        where
            O: Send + 'static,
            C: Context<'a>,
            F: FnOnce(
                FunctionContext,
                Result<Handle<JsValue>, Handle<JsValue>>,
            ) -> NeonResult<O>
            + Send
            + 'static,
    {
        let (tx, rx) = futures::channel::oneshot();

        // Cloned and passed to `.then(..)` and `.catch(..)`
        let state = Arc::new(Mutex::new(Some((f, tx))));
        
        /* ... */

        Ok(JsFuture { rx })
    }
}

impl Future for JsFuture<T> {
    type Output = Result<T, JoinError>;
  
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        todo!()
    }
}
```

In JavaScript, `await` operates on then-ables. Additionally, any value can be awaited. These are all valid and return `42`:

```js
await Promise.resolve(42);
await { then(cb) { cb(42); } }
await 42;
```

However, in Neon, the `to_future` adapter requires a native `Promise`. In order to adapt a value to a `JsFuture`, it first needs to be converted to a `JsPromise`. Two helper methods are added to ease the boilerplate:

```rust
impl JsPromise {
    pub fn resolve<'a, C, V>(cx: &mut cx, value: &V) -> Handle<Self>
    where
        C: Context<'a>,
        V: Value,
    {
        let (deferred, promise) = cx.promise();
        deferred.resolve(cx, value);
        promise
    }

    pub fn reject<'a, C, V>(cx: &mut cx, value: &V) -> Handle<Self>
    where
        C: Context<'a>,
        V: Value,
    {
        let (deferred, promise) = cx.promise();
        deferred.reject(cx, value);
        promise
    }
}
```

These methods are equivalent to `Promise.resolve` and `Promise.reject`.

```rust
let n = cx.number(42);
let fut = JsPromise::resolve(&mut cx, &n).to_future(|mut cx, res| {
    res.unwrap().value(&mut cx)
});
```

# Rationale and alternatives
[alternatives]: #alternatives

### External Implementation

Everything in this RFC could be implemented outside of Neon. We could leave it to that space. However, it's very compelling for Neon to have a good story here. We can also gain some efficiencies by having `JoinHandle` implementation change.

### Single threaded executor

A single threaded executor could do many interesting things with holding contexts and JavaScritp values. However, it would be very complicated for Neon to maintain multiple versions and in most cases users will want a multi-threaded executor.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should panics resume unwinding on the awaiting side or should they be error variants?
- In JavaScript, you can `await` anything, including native `Promise`, then-ables (3rd party promises), or even simple values. How should this be handled?
    * Force users to always use native `Promise`, wrapping in JavaScript if necessary
    * `JsValue::to_future` instead of `JsPromise::to_future` and automatically `Promise.resolve` wrap everything
    * Provide a `JsPromise::resolve` that can adapt any `JsValue` to a promise and then be converted.
- ~~Should we always require futures to be `Send + 'static`?~~ A single threaded executor might be able to keep this data local. I think since this would be very susceptible to deadlocks, we shouldn't consider it as part of this RFC.
