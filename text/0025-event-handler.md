- Feature Name: EventHandler
- Start Date: 2018-11-30
- RFC PR: https://github.com/neon-bindings/rfcs/pull/25
- Neon Issue: https://github.com/neon-bindings/neon/pull/375

# Summary
[summary]: #summary

The goal of this RFC is to provide a way for bindings to call back into JavaScript from threads other than the Node.JS main thread.

# Motivation
[motivation]: #motivation

The main motivation of this is to provide a way to schedule the execution of JavaScript from any (Rust) thread. While `neon::Task` allows to perform a background task and call a JavaScript callback after it's completion, there is currently no way to propagate the progress of a task to JavaScript.   
Being able to call to JavaScript from any thread would be useful for binding authors trying to bind native libraries:
- with custom threading models
- with progress state
- which produce events
- with callbacks
- ...


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces a new struct `neon::event::EventHandler` that is clone-able and can be send across threads.    
The user provides a `function` (and an optional `this` context which defaults to the global object) to the struct constructors.
To actually invoke JavaScript code the `schedule` method is used. This method accepts a closure which will receive a `neon::Context` and the `this`/`function` pair provided during the creation of `neon::event::EventHandler`.

Example for providing the current progress of a background operation:

```rust
    let mut this = cx.this();
    let func = cx.argument::<JsFunction>(0)?;
    let cb = EventHandler::new(func);
    // or
    let cb = EventHandler::bind(this, func);
    thread::spawn(move || {
        for i in 0..100 {
            // do some work ....
            thread::sleep(Duration::from_millis(40));
            // schedule a call into javascript
            cb.schedule(move |cx| {
                // return the arguments of the function call
                vec![cx.number(i).upcast()]
            }
        }
    });
```

Here the `EventHandler` "captures" `this` and `func` and calls the closure from the JavaScript thread with the context (`cx`). The values returned by the closure are used as arguments of the function call.

*Note:* The closure is send to the main Javascript thread so every captured value will be moved.   

This approach should be familiar to every Rust programmer as it is the same as `std::thread::spawn` uses.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The struct is implemented as following:

```rust
struct EventHandlerInner(*mut c_void);

unsafe impl Send for EventHandlerInner {}
unsafe impl Sync for EventHandlerInner {}

impl Drop for EventHandlerInner {
    fn drop(&mut self) {
        // free resources
    }
}

#[derive(Clone)]
pub struct EventHandler(Arc<EventHandlerInner>);

impl EventHandler {
    // creates a new EventHandler with global as this
    pub fn new(function: Handle<JsFunction>) -> Self;
    // creates a new EventHandler with a custom this
    pub fn bind<T: Value>(this: Handle<T>, function: Handle<JsFunction>) -> Self;

    // schedule a js function call with the arguments returned from the closure
    pub fn schedule<T, F>(&self, closure: F)
        where T: Value
              F: FnOnce(&mut TaskContext, Handle<JsValue>, Handle<JsFunction>) -> Vec<Handle<T>>,
              F: Send + 'static;
}
```

The event handler struct is backed by an `std::sync::Arc`. This allows the event handler to be clone able and to be send across threads. The arc contains an opaque structure which is used to call the underlying C++ implementation.   
The C++ implementation is thread safe and handles the asynchronous call to the main thread and the v8 stack setup.   
Once the last clone of the `EventHandler` is out of scope the `Arc` will drop the `EventHandlerInner` which in turn will free any resources allocated by the C++ implementation.

*Coming back to the example above:*   

```rust
let cb = EventHandler::new(func);
// or
let cb = EventHandler::bind(this, func);
```
`EventHandler::new` allocates the C++ implementation, which stashes the provided `this`/`function` pair as well as the current `context` and initializes the async handle of `libuv` (`uv_async_init`). This has to occur on the main thread. After that the event handler can be cloned and send across threads. 
```rust
// schedule a call to the js function
cb.schedule(move |cx| {
    // return the arguments for the js function call
    vec![cx.number(i).upcast()]
}
```
  
When calling the `schedule` method, the given closure will be stashed in a thread safe queue and the main thread will be informed via `uv_async_send`. The main thread will call the function registered during the `uv_async_init` call.   
After setting up the correct v8 scope every closure in the queue will be called with the current context.   
The closure can now provide the arguments for the js function call, which will be performed with the provided `this` and `function` values.
Once `cb` goes out of scope the C++ implementation closes the async handle, which can only be done in the main thread, and delete itself.

The `Arc` on the Rust side guarantees that the `close` method of the C++ implementation will be only called once and that no further `schedule`s to the C++ implementation are possible.

# Drawbacks
[drawbacks]: #drawbacks

This will introduce some unsafe code and the implementation is tied directly to `libuv`
as it uses the `uv_async_t` API. Newer version of nodejs provide the [napi_threadsafe_function](https://nodejs.org/api/n-api.html#n_api_napi_threadsafe_function) API, which could be used once neon is ported to N-API.

# Rationale and alternatives
[alternatives]: #alternatives

There are no real alternatives, calling into JavaScript from any (Rust) thread is a useful feature for bindings author.

# Unresolved questions
[unresolved]: #unresolved-questions

