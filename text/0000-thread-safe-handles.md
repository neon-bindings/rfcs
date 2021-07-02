- Feature Name: Threadsafe Handles
- Start Date: 2020-07-21
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Two new `Send` and `Sync` features are introduced: `EventQueue: Send + Sync` and `Root: Send`. These features work together to allow multi-threaded modules to interact with JavaScript.

* `EventQueue`: Provides a `send` method for queueing a Rust closure to be executed by the JavaScript thread that created the `EventQueue`.
* `Root<_>`: An external reference to JavaScript object that can be sent across threads. It may only be dereferenced by the JavaScript thread that created it.

# Motivation
[motivation]: #motivation

High level event handlers have gone through [several](https://github.com/neon-bindings/rfcs/pull/25) [iterations](https://github.com/neon-bindings/rfcs/pull/28) without quite providing [ideal](https://github.com/neon-bindings/rfcs/issues/31) ergonomics or [safety](https://github.com/neon-bindings/neon/issues/551).

The Threadsafe Handlers feature attempts to decompose the high level `EventHandler` API into lower level, more flexible primitives.

These primitives can be used to build a higher level and _safe_ `EventHandler` API allowing experimentation with patterns outside of `neon`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Neon provides a smart pointer type `Handle<'a, T: Value>` for referencing values on a JavaScript heap. These references are bound by the lifetime of `Context<'a>` to ensure they can only be used while the VM is locked in a synchronous neon function. These are neither `Send` or `Sync`.

### `neon::handle::Root<T>`

As a developer, I may want to retain a reference to a JavaScript object while returning control back to the VM.

`Handle<'_, _>` may not outlive the context that created them.

```rust
// Does not compile because `cb` cannot be sent across threads
fn thread_log(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?;

    std::thread::spawn(move || {
        println!("{:?}", cb);
    });

    Ok(cx.undefined())
}
```

A `Root<_>` may be created from a `Handle<'_, _>` which can be sent across threads.

```rust
// Compiles
fn thread_log(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);

    std::thread::spawn(move || {
        println!("{:?}", cb);
    });

    Ok(cx.undefined())
}
```

While `Root<_>` may be shared across threads, the inner contents can only be accessed by the JavaScript thread that owns it.

```rust
fn log(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb_root = cx.argument::<JsFunction>(0)?.root(&mut cx);
    let cb = cb_root.into_inner(&cx);

    println!("{:?}", cb);

    Ok(cx.undefined())
}
```

Our earlier example of `fn thread_log` compiled, but would `panic` when executed. A `Root<_>` may only be dropped on the JavaScript thread that created it. `Root<_>` references that are not unwrapped must be manually dropped.

```rust
fn log(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);

    // The callback is no longer needed
    cb.drop(&cx);

    Ok(cx.undefined())
}
```

### `neon::event::EventQueue`

Once a value is wrapped in a `Root<_>`, it must be sent back to the JavaScript that created it to unwrap. `EventQueue` provides a mechanism for requesting work be performed on a JavaScript thread.

To schedule work, send a Rust closure to the event queue:

```rust
fn thread_callback(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);
    let queue = cx.event_queue(&mut cx);

    std::thread::spawn(move || {
        queue.send(move |mut cx| {
            let this = cx.undefined();
            let msg = cx.string("Hello, World!");
            let cb = cb.into_inner(&cx);

            let _ = cb.call(&mut cx, this, vec![msg])?;

            Ok(())
        });
    });

    Ok(cx.undefined())
}
```

`Root<_>` can be cloned from the JavaScript thread that created them. `EventQueue` cannot be cloned, but are `Sync` and can be wrapped in an `Arc` without addition synchronization.

```rust
fn thread_callback(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);
    let queue = Arc::new(cx.event_queue(&mut cx));

    for i in (1..=10) {
        let queue = queue.clone();
        let cb = cb.clone(&mut cx);

        std::thread::spawn(move || {
            queue.send(move |mut cx| {
                let this = cx.undefined();
                let msg = cx.string(format!("Count: {}", i));
                let cb = cb.into_inner(&cx);

                let _ = cb.call(&mut cx, this, vec![msg])?;

                Ok(())
            });
        });
    }

    // Each thread has a clone; drop the original.
    cb.drop(&cx);

    Ok(cx.undefined())
}
```

Instances of `EventQueue` will keep the event loop running and prevent the process from exiting. The `EventQueue::unref` method is provided to change this behavior and allow the process to exit while an instance of `EventQueue` still exists.

However, calls to `EventQueue::send` _might_ not execute before the process exits.

This is identical to the API provided by [timers](https://nodejs.org/api/timers.html) in Node.

```rust
fn thread_callback(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);
    let mut queue = cx.event_queue(&mut cx);

    queue.unref();

    std::thread::spawn(move || {
        std::thread::sleep(std::time::Duration::from_secs(1));

        // If the event queue is empty, the process may exit before this executes
        queue.send(move |mut cx| {
            let this = cx.undefined();
            let msg = cx.string("Hello, World!");
            let cb = cb.into_inner(&cx);

            let _ = cb.call(&mut cx, this, vec![msg]);

            Ok(())
        });
    });

    Ok(cx.undefined())
}
```

## Usage with `JsBox`

A user may want to model a stream by storing a callback in a `Root<_>` that can be called many times.

* The `Root<_>` will likely be contained in a `JsBox<_>`
* `JsBox` requires a `Finalize` trait
* The `Finalize` trait provides a `finalize` method which will be executed on the JavaScript thread that created the `JsBox`

`Finalize::finalize` can provide an ergonomic way to drop a `Root<_>` when it is contained in a `JsBox<_>` that is no longer needed

```rust
type MyCounterBox = JsBox<RefCell<MyCounter>>;

struct MyCounter {
    count: usize,
    callback: Root<JsFunction>,
}

impl MyCounter {
    fn new(callback: Root<JsFunction>) -> Self {
        Self {
            count: 0,
            callback,
        }
    }

    fn incr<'a, C: Context<'a>>(&mut self, cx: &mut C) -> JsResult<JsUndefined> {
        *self.count += 1;

        let this = cx.undefined();
        let msg = cx.number(self.count as f64);
        let cb = self.callback.clone(&mut cx).into_inner(&cx);

        let _ = cb.call(&mut cx, this, vec![msg])?;

        Ok(cx.undefined())
    }
}

impl Finalize for MyCounter {
    fn finalize<'a, C: Context<'a>>(self, cx: &mut C) {
        self.callback.drop(&cx);
    }
}

fn my_counter_new(cx: FunctionContext) -> JsResult<MyCounterBox> {
    let cb = cx.argument::<JsFunction>(0)?.root(&mut cx);
    let counter = RefCell::new(MyCounter::new(cb));

    Ok(counter)
}

fn my_counter_incr(cx: FunctionContext) -> JsResult<JsUndefined> {
    let counter = cx.argument::<MyCounterBox>(0)?;

    counter.borrow_mut().incr(&mut cx)
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `neon::handle::Root<T: Object>`

`Root<_>` are `'static` references to objects on the v8 heap. If objects are moved, the internal pointer will be updated.

_`Root` references may only be used for `Object` and `Function` types. This is a limitation of `n-api`._

```rust
/// `Send` Root handle to JavaScript objects
impl Root<T: Object> {
    pub fn new<'a, C: Context<'a>>(
        cx: &mut C,
        v: &T,
    ) -> Self;

    pub fn into_inner<'a, C: Context<'a>>(
        self,
        cx: &C,
    ) -> T;

    pub fn clone<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    );

    pub fn drop<'a, C: Context<'a>>(
        self,
        cx: &C,
    );
}
```

#### `trait Object`

The idiomatic way to create a `Root<_>` is from an `Object`.

```rust
trait Object: Value {
    fn root<'a, C: Context<'a>>(&self, cx: &mut C) -> Root<Self>;
}
```

#### `trait Finalize`

```rust
impl<T> Finalize for Root<T> {
    fn finalize<'a, Context<'a>>(self, cx: &mut C) {
        self.drop(cx);
    }
}
```

`Root<_>` implement `Finalize`to allow transparent dropping within containers; e.g., `Arc`. The following boilerplate can be used to properly dispose of an `Arc<Root<_>>`.

```rust
struct MyCallback {
    queue: EventQueue,
    callback: Arc<Root<JsFunction>>,
}

impl Finalize for MyStruct {
    fn finalize<'a, Context<'a>>(self, cx: &mut C) {
        self.callback.finalize(cx);
    }
}

impl MyCallback {
    fn call<'a, C: Context<'a>>(&self) -> JsResult<JsUndefined> {
        let queue = self.queue.clone();
        let callback = Arc::clone(self.callback);

        std::thread::spawn(move || queue.send(move |cx| {
            let callback = Arc::try_unwrap(callback)
                .or_else(|cb| cb.clone(&mut cx))
                .into_inner(&mut cx);

            let this = cx.undefined();
            let args = vec![cx.string("Hello, World!")];
            let _ = cb.call(&mut cx, this, args)?;

            Ok(cx.undefined())
        }));
    }
}
```

#### Design Notes

A `Root<_>` can only be dropped on the JavaScript thread that created it because they include a non-atomic reference count. There are two ways to safely drop a `Root`: `Root::into_inner` and `Root::drop`.

The `Drop` implementation of `Root` will `panic` to alert users of the leak and guide them towards correct usage.

```rust
impl Drop for Root {
    fn drop(&mut self) {
        if !std::thread::panicking() {
            panic!("neon::handle::Root leaked. Must call into_inner or drop.");
        }
    }
}
```

In the future, neon may include a higher level construct that abstracts away this error prone usage.

### `neon::unbounded::EventQueue`

```rust
impl EventQueue {
    /// Creates an unbounded queue
    pub fn new<'a, C: Context<'a>>(cx: &mut C) -> Self;

    /// Allow the Node event loop to exit while this `EventQueue` exists.
    /// _Idempotent_
    pub fn unref<'a, C: Context<'a>>(&mut self, cx: &mut C);

    /// Prevent the Node event loop from existing while this `EventQueue` exists. (Default)
    /// _Idempotent_
    pub fn reference<'a, C: Context<'a>>(&mut self, cx: &mut C);

    /// Returns a boolean indicating if this `EventQueue` will prevent the Node event
    /// queue from exiting.
    pub fn has_ref(&self) -> bool;

    /// Schedules a closure to execute on the JavaScript thread that created this EventQueue
    /// Will panic if there is an error scheduling a callback from libuv
    pub fn send(
        &self,
        f: impl FnOnce(TaskContext) -> NeonResult<()>,
    );

    /// Schedules a closure to execute on the JavaScript thread that created this EventQueue
    pub fn try_send<F>(&self, f: F) -> Result<(), EventQueueError>
        where F: FnOnce(TaskContext) -> NeonResult<()>;
}
```

The native `napi_call_threadsafe_function` can fail in several ways:

* `napi_queue_full` if the queue is full.
* `napi_closing` if the threadsafe function has been closed.
* `napi_invalid_arg` if the thread count is zero.
* `napi_generic_failure` if the call to `uv_async_send` fails.

For an unbounded queue, the only error that applies is `napi_generic_failure`

* `napi_queue_full` cannot happen on an unbounded queue
* `napi_closing` is only returned when `napi_tsfn_abort` is used. Neon will not use this.
* `napi_invalid_arg` is only returned when the thread count is zero. The thread count is decremented in a `Drop` hook and protects against calls after it reaches zero.

```rust
struct EventQueueError;

impl Error for EventQueueError {}
```

#### `Send + Sync`

`EventQueue` are both `Send` and `Sync` since N-API threadsafe functions include reference counting and synchronization.

#### `trait Context`

The idiomatic way to create an `EventQueue` is from a `Context.

```rust
trait Context<'a> {
    fn queue(&mut self) -> EventQueue {
        EventQueue::new(self)
    }
}
```

```rust
let queue = cx.event_queue();
```

#### Design Notes

The backing `napi_threadsafe_function` is already `Send` and `Sync`, allowing the use of `send` with a shared reference.

Both `ref` and `unref` mutate the behavior of the underlying threadsafe function and require exclusive references. These methods are infallible because Rust is capable of statically enforcing the invariants. We may want to optimize with `assert_debug!` on the `napi_status`.

##### Safety

Neon must never use `napi_tsfn_abort`. Calling `napi_tsfn_abort` causes unsafety and may trigger undefined behavior.

A call to `napi_release_threadsafe_function` with `napi_tsfn_abort` immediately decrements the threadcount to zero and begins the process of destroying the threadsafe function. Future calls may cause a use after free.

However, it is unlikely that `napi_tsfn_abort` would of use in Neon. Since `EventQueue` cannot be cloned, the thread count will never be higher than `1`.

#### Future Expansion

N-API `napi_threadsafe_function` include an internal queue that can be unbounded or have a maximum capacity. In order to keep this RFC and its implementation smaller, bounded queues are left for future expansion. They are compatible with this RFC and can added later.

```rust
// mod neon::handle::bounded;

impl EventQueue {
    /// Creates a bounded queue
    pub fn new<'a, C: Context<'a>>(cx: &mut C, size: usize) -> Self;

    /// Allow the Node event loop to exit while this `EventQueue` exists.
    /// _Idempotent_
    pub fn unref<'a, C: Context<'a>>(&mut self, cx: &mut C);

    /// Prevent the Node event loop from existing while this `EventQueue` exists. (Default)
    /// _Idempotent_
    pub fn reference<'a, C: Context<'a>>(&mut self, cx: &mut C);

    /// Returns a boolean indicating if this `EventQueue` will prevent the Node event
    /// queue from exiting.
    pub fn has_ref(&self) -> bool;

    /// Schedules a closure to execute on the JavaScript thread that created this EventQueue
    /// Blocks if the event queue is full
    pub fn send(
        &self,
        f: impl FnOnce(TaskContext) -> NeonResult<()>,
    ) -> Result<(), EventQueueError>;

    /// Schedules a closure to execute on the JavaScript thread that created this EventQueue
    /// Non-blocking
    pub fn try_send<F>(&self, f: F) -> Result<(), EventQueueError<F>>
        where F: FnOnce(TaskContext) -> NeonResult<()>;
}

enum EventQueueError<F> {
    /// The bounded queue was full. The closure is returned in the error.
    QueueFull(F),
    /// A generic failure. Most likely on `uv_async_send(&async)`.
    Unknown,
}

impl Error for EventQueueError {}

// mod neon::handle::unbounded

use super::unbounded;

impl EventQueue {
    /// Creates a bounded queue
    pub fn with_capacity<'a, C: Context<'a>>(cx: &mut C, size: usize) -> unbounded::EventQueue;
}
```

# Drawbacks
[drawbacks]: #drawbacks

![standards](https://imgs.xkcd.com/comics/standards.png)

Neon already has `Task`, `EventHandler`, a [proposed](https://github.com/neon-bindings/rfcs/pull/30) `TaskBuilder` and an accepted, but currently unimplemented, [update](https://github.com/neon-bindings/rfcs/pull/28) to `EventHandler`. This is a large amount of API surface area without clear indication of what a user should use.

This can be mitigated with documentation and soft deprecation of existing methods as we get a clearer picture of what a high-level, ergonomic API would look like.

It's also important to note that `Task` and `TaskBuilder` give access to the Node.js `libuv` thread pool which is orthogonal to sharing references across threads. There is room for both APIs to exist.

# Rationale and alternatives
[alternatives]: #alternatives

These are fairly thin wrappers around N-API primitives. The most compelling alternatives are continuing to improve the existing high level APIs.

## `EventQueue: Clone`

Since N-API threadsafe functions include a reference count, it's possible to safely implement `Clone`. However, this has a couple of disadvantages.

Primarily, it makes expressing `unref` and `reference` awkward. These methods mutate the state of the threadsafe function using the JavaScript `Env` to enforce synchronoization. Since there is no obvious wrapper providing interior mutability, it may be unclear that mutating an `EventQueue` will also impact its clones.

```rust
let queue = cx.event_queue();
let mut queue_copy = queue.clone();

// `queue` is neither mutable or in any obvious way related to `queue_copy` but, it is also `unref`
queue_copy.unref();
```

Alternatively, `unref` can produce a new `EventQueue`. However, this makes it impossible to switch an `EventQueue` back and forth. This could be desirable to allow only keeping the event loop alive while an event is queued. For example:

```rust
let mut queue = cx.event_queue();

queue.unref();

// We're about to queue an event
queue.reference();
queue.send(move |cx| {
    // Done. We can let the process exit.
    queue.unref();

    Ok(())
});
```

# Unresolved questions
[unresolved]: #unresolved-questions

- ~Should this be implemented for the legacy runtime or only n-api?~ Only for `n-api`.
- ~Should `EventQueue` be `Arc` wrapped internally to encourage users to share instances across threads?~ No. External `Arc` wrapping makes shared mutable state more obvious and allows users to optimize reference counting.
- ~A global `EventQueue` is necessary for dropping `Root`. Should this be exposed?~ No. This is no longer required for the design.
- ~The global `EventQueue` requires instance data. Are we okay using an [experimental](https://nodejs.org/api/n-api.html#n_api_environment_life_cycle_apis) API?~ This is no longer required for the design.
- ~Should `EventQueue::send` accept a closure that returns `()` instead of `NeonResult<()>`?~ In most cases, the user will want to allow `Throw` to become an `uncaughtException` instead of a `panic` and `NeonResult<()>` provides a small ergonomics improvement.
- ~`persistent(&mut cx)` is a little difficult to type. Should it have a less complicated name?~ Changed the name to `Root` which is a common term when describing garbage collection.
