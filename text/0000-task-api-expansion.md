* Feature Name: task_api_expansion
* Start Date: 2018/01/31
* RFC PR: https://github.com/neon-bindings/neon/pull/303
* Neon Issue: https://github.com/neon-bindings/neon/issues/228

# Summary

[summary]: #summary

This RFC expands the Task API and adds a Worker API to support the needs of long-running async work in a Rust thread. The expanded Task trait maintains support for running work on the `libuv` thread pool.

# Motivation

[motivation]: #motivation

The current `Task` trait has two limitations that render long-running async Rust work impractical:

* Tasks run in the Node thread pool using `uv_queue_work`. This means that long-running tasks can clog the thread pool, potentially blocking Node's filesystem and compression operations. See the Node docs [here](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/#don-t-block-the-worker-pool) for common cases that block the worker pool.
* Tasks can only present a single completion value to the calling Javascript. `Task` doesn't support intermediate events or receiving events from Javascript.

Adding new APIs to remove these limitations allows Neon to support new use cases:

* Reporting progress of asynchronous Rust tasks to Node
* Streaming data from Rust to Node (audio, video, websockets, etc.)
* Emitting hardware sensor data from Rust/FFI to Node (VR head tracking, traffic sensors, accelerometers, etc.)
* Long-running Rust daemons that can communicate bidirectionally with Node without the overhead of IPC

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Neon includes two Traits for running asynchronous work: `Task` and `Worker`.

`Task` is a convenience implementation of `Worker` that makes single-shot tasks easy to run and define. You can run a `Task` in either a Rust thread or the `libuv` thread pool. Tasks terminate after providing a either a completion value or error.

`Worker` gives you full control of asynchronous work–you can emit intermediate events and send messages to a `Worker` from Javascript without blocking the event loop. Within a `Worker`'s `perform` method, you're free to stream data or even create your own event loop that performs work in response to Javascript messages.

## Task

To implement the trait, define your work in `perform` and map its results to Javascript values in `complete`.

```rust
pub trait Task: Send + Sized + 'static {
    /// The type of the completion value `perform` returns.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Complete: Send;
    /// The type of the error value `perform` returns.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Error: Send + Error;
    /// The Javascript type of the value provided to the second parameter
    /// of the Javascript callback.
    type JsComplete: Value;
    /// Performs work without blocking the event loop, producing either a
    /// completion value or an error.
    fn perform(&self) -> Result<Self::Complete, Self::Error>;
    /// Transform the `Complete` value returned from `perform` into a
    /// Javascript value to pass to the provided callback. This method runs
    /// on the  event loop thread an indeterminate amount of time after
    /// finishing `perform`.
    fn complete<'a, S: Scope<'a>>(
        &'a self,
        scope: &'a mut S,
        result: Result<&Self::Complete, &Self::Error>,
    ) -> JsResult<Self::JsComplete>;
}
```

The `Task` trait provides two methods with default implementations, `run` and `run_uv`:

```rust
fn run<'a, T: Scope<'a>>(
    self,
    scope: &'a mut T,
    callback: Handle<JsFunction>,
) -> JsResult<'a, JsFunction>;

fn run_uv(self, callback: Handle<JsFunction>);
```

`run` spawns the task on a Rust thread, while `run_uv` runs it on the `libuv` thread pool. Long-running work in `run_uv` may block Node's thread pool; if in doubt, use `run`. Both take a callback with signature `function(err, result)`. Tasks terminate on error.

### Example

```rust
extern crate neon;

use std::error::Error;
use std::fmt;

use neon::concurrent::Task;
use neon::js::{JsFunction, JsNumber, JsUndefined};
use neon::scope::Scope;
use neon::vm::{Call, JsResult};

#[derive(Debug)]
struct TaskError;

impl Error for TaskError {
    fn description(&self) -> &str {
        "Oops"
    }
}

impl fmt::Display for TaskError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Oops")
    }
}

#[derive(Debug)]
struct SuccessTask;

impl Task for SuccessTask {
    type Complete = i32;
    type Error = TaskError;
    type JsComplete = JsNumber;

    fn perform(&self) -> Result<Self::Complete, Self::Error> {
        Ok(10 + 7)
    }

    fn complete<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        result: Result<&Self::Complete, &Self::Error>,
    ) -> JsResult<Self::JsComplete> {
        Ok(JsNumber::new(scope, *result.unwrap() as f64))
    }
}

pub fn perform_async_task(call: Call) -> JsResult<JsUndefined> {
    let callback = call.arguments
        .require(call.scope, 0)?
        .check::<JsFunction>()?;
    let _ = SuccessTask.run(call.scope, callback);
    Ok(JsUndefined::new())
}
```

## Worker

Implementing `Worker` is similar to implementing `Task`–define your work and transform its Rust results into Javascript values–but requires extra methods to handle incoming and outgoing events.

```rust
pub trait Worker: Send + Sized + 'static {
    /// The type of the completion value that `perform` emits in its callback.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Complete: Send;
    /// The type of the error value `perform` emits in its callback.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Error: Send + Error;
    /// The type of all event values `perform` emits in its callback.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Event: Send;
    /// The type of all event values Javascript passes to the worker.
    /// This is the _Rust_ type, transformed from the `JsEvent` type
    /// in the `on_incoming_event` method.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type IncomingEvent: Send;
    /// The type of all event values Javascript passes to the worker.
    /// This is the _Rust_ type that represents a failure  to transform
    /// the `JsEvent` type into an `IncomingEvent` type in the
    /// `on_incoming_event` method.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type IncomingEventError: Error + Sized;
    /// The Javascript type of the completion value. The worker calls
    /// its callback with this value as the second parameter.
    type JsComplete: Value;
    /// The Javascript type of the event value. The worker calls its
    /// callback with this value as the third parameter.
    type JsEvent: Value;
    /// Transform the `Complete` value emitted from `perform` into a
    /// Javascript value to pass to the provided callback. This method runs
    /// on the  event loop thread an indeterminate amount of time after
    /// finishing `perform`.
    fn complete<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        result: Result<&Self::Complete, &Self::Error>,
    ) -> JsResult<Self::JsComplete>;
    /// Transform the `Event` value emitted from `perform` into a
    /// Javascript value to pass to the provided callback. This method runs
    /// on the  event loop thread an indeterminate amount of time after
    /// finishing `perform`.
    fn event<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        value: &Self::Event,
    ) -> JsResult<Self::JsEvent>;
    /// Transform an incoming Javascript value into a Rust value.
    fn on_incoming_event<'a>(call: Call<'a>) -> Result<Self::IncomingEvent, Self::IncomingEventError>;
}
```

### Example

```rust
extern crate neon;

use std::error::Error;
use std::fmt;

use neon::concurrent::{Worker, Message};
use neon::js::{JsFunction, JsNumber, JsUndefined};
use neon::scope::Scope;
use neon::vm::{Call, JsResult};

#[derive(Debug)]
struct TaskError;

impl Error for TaskError {
    fn description(&self) -> &str {
        "Oops"
    }
}

impl fmt::Display for TaskError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Oops")
    }
}

#[derive(Debug)]
struct SuccessWorker;

impl Worker for SuccessWorker {
    type Complete = String;
    type Error = TaskError;
    type Event = String;
    type IncomingEvent = String;
    type IncomingEventError = TaskError;

    type JsComplete = JsString;
    type JsEvent = JsString;

    fn perform<N: FnMut(Message<Self::Event, Self::Error, Self::Complete>)>(
        &self,
        mut emit: N,
        receiver: Receiver<Self::IncomingEvent>,
    ) {
        let incoming = receiver.recv().unwrap();
        let hello = String::from("Hello");
        let world = String::from("World");

        emit(Message::Event(hello));
        emit(Message::Event(world));
        emit(Message::Complete(incoming))
    }

    fn on_incoming_event<'a>(
        call: Call<'a>,
    ) -> Result<Self::IncomingEvent, Self::IncomingEventError> {
        call.arguments
            .require(call.scope, 0)
            .and_then(|arg| arg.check::<JsString>())
            .and_then(|string| Ok(string.value()))
            .or_else(|_| Err(TaskError {}))
    }

    fn event<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        value: &Self::Event,
    ) -> JsResult<Self::JsEvent> {
        JsString::new_or_throw(scope, value)
    }

    fn complete<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        result: Result<&Self::Complete, &Self::Error>,
    ) -> JsResult<Self::JsComplete> {
        match result {
            Err(e) => JsError::throw(Kind::Error, e.description()),
            Ok(value) => JsString::new_or_throw(scope, value),
        }
    }
}

pub fn create_success_worker(call: Call) -> JsResult<JsFunction> {
    let callback = call.arguments
        .require(call.scope, 0)?
        .check::<JsFunction>()?;
    SuccessWorker.spawn(call.scope, callback)
}
```

In your Javascript code:

```js
const send = addon.create_success_worker((err, complete, event) => {
  // Workers don't terminate on errors–we want to be able to emit
  // multiple non-fatal errors, especially for long-running daemons.
  if (err) { console.error(err); }
  if (event) { 
    console.log(event);
  }
  if (complete) {
    console.log(complete)
  }
});

// You can receive this value in `perform`'s `receiver` channel!
send("Goodbye");
```

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

To run asynchronous code in a Rust thread without blocking the event loop thread, we use `libuv`'s async handles (`uv_async_t` in conjunction with `uv_async_send`). To start using `libuv` in an ergonomic way, we need bindings!

## `libuv` bindings

`bindgen` to the rescue! To generate minimal bindings for this implementation, run the following (assuming `nvm`):

```bash
bindgen ~/.nvm/versions/node/v8.11.1/include/node/uv.h \
    -o src/uv/pre-node-10.rs \
    --with-derive-default \
    --whitelist-function uv_async_init \
    --whitelist-function uv_async_send \
    --whitelist-type uv_async_t \
    --whitelist-function uv_close \
    --whitelist-function uv_default_loop
```

Note the `pre-node-10` naming scheme above. Node 10's `libuv` version isn't ABI compatible with the `libuv` versions in Node 6-8:

| Node (latest) | libuv     | ABI backwards compatible? |
| ---------- | ------ | -------------------- |
| 6               | 1.16.1  | Yes                            |
| 8               | 1.19.1  | Yes                            |
| 10              | 1.20.1  | **NO**                       |

(See the [`abi-laboratory`](https://abi-laboratory.pro/tracker/timeline/libuv/) report for full ABI history.)

To support the latest and all LTS versions of Node, we generate two sets of bindings: one for Node 6-8, and one for Node 10. For Node 6-8, run `bindgen` on either the Node 6 or 8 headers (they're ABI compatible). For Node 10, run `bindgen` on the Node 10 headers (output file `node_10.rs`).

To compile with the correct bindings, we add a step to the build script that snags the current Node version from `npm configure` and enable either the new `pre_node_10` (default) or `node_10` Cargo feature.

After creating the bindings, we add an `AsyncHandle` struct to correctly allocate and drop resources. `AsyncHandle` also contains a method, `wake_event_loop`, that calls a Rust closure on the main thread using `uv_async_send` under the hood.

## Implementing `Worker::spawn`

Since we can define `Task` as an implementation of `Worker`, our first task (heh) is to add a default implementation, `Worker::spawn`, that interacts with `libuv` to run code on the Rust thread, wake up the event loop, and transform results into Javascript values.

In `Worker::spawn`, we create the following resources:

* A persistent handle for the provided callback. We use a small wrapper in `neon-runtime`, `AsyncCallback`, to handle construction, destruction, and execution of this callback.
* An `AsyncHandle` that lets us wake the event loop up after performing work in a Rust thread.
* A concurrent queue for managing `libuv`'s coalesced `uv_async_send` calls (see the [`libuv` docs](http://docs.libuv.org/en/v1.x/async.html)).
* A `std::sync::mpsc` channel to notify the Rust thread that the main thread sent a worker completion event to Javascript. This keeps borrowed resources alive on the Rust thread while waiting for the main thread to stop using them.
* A `std::sync::mpsc` channel to send and receive events between the worker and Javascript.

We then spawn a Rust thread and run `Worker::perform` in it, passing it an emitter callback and the event channel receiver. Immediately after, we return a `JsFunction` with a boxed Rust closure that sends messages to `Worker::perform`'s event channel receiver.

The emitter callback we pass to `Worker::perform` is where the magic happens. Each time the user's `Worker::perform` calls the callback with an event, completion value, or error, the callback:

* Pushes the value onto the queue.
* Wakes up the event loop.
* For each value in the queue:
    * Create a new `RootScope`.
    * If the value is an event:
        * Convert the event into a Javascript value by calling `Worker::event`.
        * Call the persistent callback with the event as the third parameter.
    * If the value is an error:
        * Convert the error into a Javascript value by calling `Worker::error`.
        * Call the persistent callback with the error as the first parameter.
    * If the value is a completion value:
        * Convert the value into a Javascript value by calling `Worker::complete`.
        * Call the persistent callback with the value as the second parameter.
        * Destroy the persistent callback.
        * Send a completion message to the completion receiver in the Rust thread.

Once the completion receiver in the Rust thread receives a message, the thread dies and the worker is terminated.

## Implementing `Task::run`

`Task` is a convenience implementation of `Worker`, so it needs no new logic. `Task` uses the unit type or an equivalent empty type for associated types it doesn't care about (`Event`, `IncomingEvent`, `IncomingEventError`, and `JsEvent`) and returns simple wrapper `Ok`s around these types for methods it doesn't care about (`event`, `incoming_event`).

## Implementing `Task::run_uv`

`Task::run_uv` is a simple migration from the previous `Task` API using the Node thread pool.

# Drawbacks

[drawbacks]: #drawbacks

* The implementation is complex and depends on knowledge of `libuv` arcana and quirks.
* Introduces a dependency on `crossbeam` that otherwise has no place in the project.
* Adds implementation-specific Cargo features that depend on a new build step.
* Adds a significant amount of unsafe code that expands the task of making Neon exception-safe.

# Rationale and alternatives

[alternatives]: #alternatives

* Higher level than manually handling `Persistent`s and `uv_async_send`.
* Provides a simple API for the common case (`Task`) and a powerful API for complex cases (`Worker`).
* `Worker` provides primitives instead of opinions to give the user flexibility.
* Not adding `Task` means long-running work the existing API can block the Node thread pool.
* Not adding `Worker` means users with streaming data and long-running tasks may abandon Neon for C++ addons.

# Unresolved questions

[unresolved]: #unresolved-questions

* Can we avoid transmutes between `libuv` handle type structs? e.g. casting `uv_async_t` to `uv_handle_t`.
* Out of scope: is it possible to implement a Tokio `Executor` for the `libuv` event loop? If so, we can implement `Future` for `Task` and `Stream` for `Worker` to better fit into Rust's async ecosystem.
