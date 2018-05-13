* Feature Name: task_api_expansion
* Start Date: 2018/01/31
* RFC PR: https://github.com/neon-bindings/neon/pull/303
* Neon Issue: https://github.com/neon-bindings/neon/issues/228

# Summary

[summary]: #summary

This RFC expands the Task API and adds a Worker API to support the needs of long-running work in a Rust thread. The expanded Task trait maintains support for running work on the `libuv` thread pool.

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

`Worker` gives you the power to define most asynchronous patterns–you can emit intermediate events and send messages to a `Worker` from Javascript without blocking the event loop. Within a `Worker`'s `perform` method, you're free to stream data or even create your own event loop that performs work in response to Javascript messages.

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
}
```

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

* Its interaction with other features is clear.
* It is reasonably clear how the feature would be implemented.
* Corner cases are dissected by example.

This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks

[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives

[alternatives]: #alternatives

* Why is this design the best in the space of possible designs?
* What other designs have been considered and what is the rationale for not choosing them?
* What is the impact of not doing this?

# Unresolved questions

[unresolved]: #unresolved-questions

* Does Node or `libuv` detect blocked threads in the pool and automatically spin up new ones? This would mitigate the `uv_queue_work` problem, although observable/bidirectional tasks still require `uv_async_send`.

* What parts of the design do you expect to resolve through the RFC process before this gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
