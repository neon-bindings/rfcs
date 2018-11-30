- Feature Name: ThreadSafeCallback
- Start Date: 2018-11-30
- RFC PR:
- Neon Issue:

# Summary
[summary]: #summary

The goal of this RFC is to provide a way for bindings to call back into JavaScript from threads other than the Node.JS main thread.

# Motivation
[motivation]: #motivation

The main motivation of this is to provide a way to schedule the execution of JavaScript from any (Rust) thread. While `neon::Task` allows to perform a background task and call a JavaScript callback after it's completion, there is currently no way to propagate the progress of a task to JavaScript.   
Being able to call to JavaScript from any thread would be useful for binding authors trying to bind libraries:
- with custom threading models
- with progress state
- which produce events
- ...


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces a new struct `neon::ThreadSafeCallback` that is clone-able and can be send across threads.    
The user provides a `this`/`function` pair to the struct constructor.
To actually invoke JavaScript code the `call` method is used. This method accepts a closure which will receive a `neon::Context` and the `this`/`function` pair provided during the creation of `neon::ThreadSafeCallback`.

Example for providing the current progress of a background operation:

```rust
    let mut this = cx.this();
    let func = cx.argument::<JsFunction>(0)?;
    let cb = ThreadSafeCallback::new(this, func);
    thread::spawn(move || {
        for i in 0..100 {
            // do some work ....
            thread::sleep(Duration::from_millis(40));
            // call into javascript
            cb.call(move |context, this, func| {
                // call the js function
                func.call(context, this, vec![cx.number(i).upcast()]);
            }
        }
    });
```

Here the `ThreadSafeCallback` "captures" `this` and `func` and calls the closure from the JavaScript thread with a `context`, the `this` and `func` value.

*Note:* The closure is send to the main Javascript thread so every captured value will be moved.   

This approach should be familiar to every Rust programmer as it is the same as `std::thread::spawn` uses.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The struct is implemented as following:

```rust
struct ThreadSafeCallbackInner(*mut c_void);

unsafe impl Send for ThreadSafeCallbackInner {}
unsafe impl Sync for ThreadSafeCallbackInner {}

impl Drop for ThreadSafeCallbackInner {
    fn drop(&mut self) {
        // free resources
    }
}

#[derive(Clone)]
pub struct ThreadSafeCallback(Arc<ThreadSafeCallbackInner>);

impl ThreadSafeCallback {
    pub fn new<T: Value>(this: Handle<T>, function: Handle<JsFunction>) -> Self;
    pub fn call<F>(&self, closure: F)
        where F: FnOnce(&mut TaskContext, Handle<JsValue>, Handle<JsFunction>),
              F: Send + 'static;
}
```

The callback struct is backed by an `std::sync::Arc`. This allows the callback to be clone able and do be send across threads. The arc contains an opaque structure which is used to call the underlying C++ implementation.   
The C++ implementation is thread safe and handles the asynchronous call to the main thread and the v8 stack setup.   
Once the last clone of the `ThreadSafeCallback` is out of scope the `Arc` will drop the `ThreadSafeCallbackInner` which in turn will free any resources allocated by the C++ implementation.

*Coming back to the example above:*   

```rust
let cb = ThreadSafeCallback::new(this, func);
```
`ThreadSafeCallback::new` allocates the C++ implementation, which stashes the provided `this`/`function` pair as well as the current `context` and initializes the async handle of `libuv` (`uv_async_init`). This has to occur on the main thread. After that the callback can be cloned and send across threads. 
```rust
// call into javascript
cb.call(move |context, this, func| {
    // call the js function
    func.call(context, this, vec![cx.number(i).upcast()]);
}
```
  
When calling the `call` method, the given closure will be stashed in a thread safe queue and the main thread will be informed via `uv_async_send`. The main thread will call the function registered during the `uv_async_init` call.   
After setting up the correct v8 scope every closure in the queue will be called with the current context and the stashed `this`/`function` pair.   
The closure can now perform the Javascript call with the provided `this` and `func` values.
Once `cb` goes out of scope the C++ implementation closes the async handle, which can only be done in the main thread, and delete itself.

The `Arc` on the Rust side guarantees that the `close` method of the C++ implementation will be only called once and that no further `call`s to the C++ implementation are possible.

# Drawbacks
[drawbacks]: #drawbacks

This will introduce some unsafe code and the implementation is tied directly to `libuv`
as it uses the `uv_async_t` API.

# Rationale and alternatives
[alternatives]: #alternatives

There are no real alternatives, calling into JavaScript from any (Rust) thread is a useful feature for bindings author.

# Unresolved questions
[unresolved]: #unresolved-questions

- Would it make sense to generalize the API e.g. take a `Vec<JsValue>` instead of a `this`/`function` pair?
- Is `ThreadSafeCallback` a good name for this? Maybe `ValueCapture`?
