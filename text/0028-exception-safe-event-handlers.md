- Feature Name: exception_safe_event_handlers
- Start Date: 2020-01-31
- RFC PR: https://github.com/neon-bindings/rfcs/pull/28
- Neon Issue: https://github.com/neon-bindings/neon/issues/526

# Summary
[summary]: #summary

[RFC 25](https://github.com/neon-bindings/rfcs/blob/master/text/0025-event-handler.md) introduced a powerful new API for transmitting a JavaScript event handler to other Rust threads so that they can asynchronously signal events back to the main JavaScript thread. This RFC proposes a modification to the API to make the event handler's callback protocol both **simpler to understand** and **safe for handling JavaScript exceptions** that can occur while preparing the call the callback.

# Motivation
[motivation]: #motivation

[RFC 25](https://github.com/neon-bindings/rfcs/blob/master/text/0025-event-handler.md) introduced a powerful new API for transmitting a JavaScript event handler to other Rust threads so that they can asynchronously signal events back to the main JavaScript thread.

However, in that design, the Rust code that prepares the results to send to the event handler has no way to manage operations that can trigger JavaScript exceptions. This shows up even in the simple examples in that RFC, which are forced to use `.unwrap()` to deal with `NeonResult` values:

```rust
let handler = EventHandler::new(...);

handler.schedule(move, |cx, this, f| {
    let buf = cx.buffer(1024).unwrap(); // panics on exception!
    ...
});
```

For the high-level `schedule()` API, this RFC proposes changing the Rust closure to return a `JsResult` and using the standard, classic Node callback protocol of sending an error value as the first argument (or `null`) and a success value as the second argument (or `null`).

For the low-level `schedule_with()` API, this RFC proposes only the small change of a `NeonResult<()>` output type, to allow the Rust callback to propagate uncaught JavaScript exceptions to the Node top-level. However, in another RFC we could propose a `try_catch()` API to allow defensive code to handle exceptions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `neon::event::EventHandler` type is clone-able and can be sent across threads.

The user provides a function and a `this` context to the struct constructors.

An `EventHandler` contains methods for scheduling events to be fired on the main JavaScript thread. Each event scheduling method takes a Rust callback we refer to as the **_event launcher_**, which runs on the JavaScript thread in its own turn of the event loop, and whose job it is to pass the event information to the event handler.

## Scheduling Events

The `schedule` method is the usual way to schedule an event on the JavaScript thread. Scheduling is "_fire and forget_," meaning that it sends the event to the main thread and immediately returns `()`.

The event launcher takes a `neon::Context` and computes the result, which will automatically be passed to the event handler by Neon.

Following Node convention, a successful result will be passed as the second argument to the handler, with `null` as the first argument; conversely, if the event launcher throws a JavaScript exception, the exception is passed as the first argument to the handler with `null` as the second argument.

Example for providing the current progress of a background operation:

```rust
    let mut this = cx.this();
    let cb = cx.argument::<JsFunction>(0)?;
    let handler = EventHandler::new(cb);
    // or:      = EventHandler::bind(this, cb);
    thread::spawn(move || {
        for i in 0..100 {
            // do some work ....
            thread::sleep(Duration::from_millis(40));
            // schedule a call into javascript
            handler.schedule(move |cx| {
                // successful result to be passed to the event handler
                Ok(cx.number(i))
            }
        }
    });
```

Here the `EventHandler` "captures" `this` and `cb` and calls the closure from the JavaScript thread with the context (`cx`). The successful result value produced by the closure is then passed as the second argument of the event handler callback, with `null` passed as the first argument, indicating no error occurred.

*Note:* The closure is sent to the main JavaScript thread so every captured value will be moved.   

This approach should be familiar to Rust programmers as it is the same as `std::thread::spawn` uses.

## Low-level API

For cases where you need more control, Neon offers a lower-level primitive, the `schedule_with` method. This method also takes an event launcher, but Neon does not automatically call the event handler. Instead, the event launcher receives a `neon::Context`, the `this` context, and the event handler function object, and is given total control over whether and how to call the event handler.

If the event launcher throws a JavaScript exception, it behaves like an uncaught exception in the Node event loop.

Example for providing the current progress of a background operation:

```rust
    let mut this = cx.this();
    let cb = cx.argument::<JsFunction>(0)?;
    let handler = EventHandler::new(cb);
    // or:      = EventHandler::bind(this, cb);
    thread::spawn(move || {
        for i in 0..100 {
            // do some work ....
            thread::sleep(Duration::from_millis(40));
            // schedule a call into javascript
            handler.schedule_with(move |cx, this, cb| {
                // call the event handler callback
                let args = vec![
                    cx.null().upcast(),
                    cx.number(i).upcast()
                ];
                cb.call(args)?;
                Ok(())
            }
        }
    });
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

```rust
struct EventHandler {

    pub fn new<'a, C: Context<'a>, T: Value>(cx: &C, this: Handle<T>, callback: Handle<JsFunction>) -> Self;

    pub fn schedule<T, F>(&self, arg_cb: F)
        where T: Value,
              F: for<'a> FnOnce(&mut EventContext<'a>) -> JsResult<'a, T>,
              F: Send + 'static;

    pub fn schedule_with<F>(&self, arg_cb: F)
        where F: FnOnce(&mut EventContext, Handle<JsValue>, Handle<JsFunction>) -> NeonResult<()>,
              F: Send + 'static;

}
```

# Drawbacks
[drawbacks]: #drawbacks

The type signatures are perhaps a bit more complex. But we should lean on the API docs and examples as the way to explain the API, not the type signatures.

# Rationale and alternatives
[alternatives]: #alternatives

Currently the `neon::event` module is protected by a feature flag, and can't be made generally available until we resolve the problem that it has no way to handle JavaScript exceptions.

We should separately propose a `try_catch()` API for wrapping computations that might throw JavaScript exceptions in a closure and converting the result of the computation into a `Result`. We could leave the `neon::event` API as-is and just tell people to use that. But the high-level API would be less ergonomic, since you'd commonly have to wrap everything in a `try_catch`. Also it seems better to decouple the two APIs and release `neon::event` without having to design and implement a complete `try_catch` solution.

Another benefit of `try_catch()` would be to avoid C++ in the implementation of this API. But that's not a blocker, just a way to eventually streamline the implementation.

# Unresolved questions
[unresolved]: #unresolved-questions

None
