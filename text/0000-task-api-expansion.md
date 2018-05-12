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

Neon includes three traits for defining and running asynchronous tasks in a Rust thread. Here's what each trait can do and how to use them, in order of complexity:

## Task

The `Task` trait defines a "single-shot" task: a short, asynchronous piece of work run in either a Rust thread or the `libuv` thread pool that terminates after providing a single completion value or error. It's analogous to a Node callback API (`runTask((err, result) => {})`). Implementing the trait involves defining the work in `perform` and defining how its results map to Javascript results or errors in `complete`.

```rust
pub trait Task: Send + Sized + 'static {
    /// The type of the completion value returned from `perform`.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Complete: Send;
    /// The type of the error value returned from `perform`.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Error: Send + Error;
    /// The Javascript type of the value provided to the second parameter
    /// of the Javascript callback.
    type JsComplete: Value;
    /// Performs work without blocking the event loop, producing either a
    /// completion value or an error.
    fn perform(&self) -> Result<Self::Complete, Self::Error>;
    /// Transform the `Complete` value returned from `perform` into a Javascript value
    /// to pass to the provided callback. This method runs on the  event loop thread an
    /// indeterminate amount of time after finishing `perform`.
    fn complete<'a, S: Scope<'a>>(
        &'a self,
        scope: &'a mut S,
        result: Result<&Self::Complete, &Self::Error>,
    ) -> JsResult<Self::JsComplete>;
}
```

The Task trait provides two methods with default implementations, `run` and `run_uv`:

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

## WorkerTask

*TODO*
The `WorkerTask` trait is a superset of the `ObservableTask` trait that allows Javascript to send messages to a running task. `WorkerTask`s enable bidirectional communication between Node and long-running Rust daemons. Its `perform` method returns a Javascript function that the user can call to send values to the task. Implementing the trait is the same as implementing `ObservableTask`, but with a few additions and an augmented `perform` method:

```rust
pub trait ObservableTask: Send + Sync + Sized + 'static {
    /// Methods and types shared with `ObservableTask` are omitted.

    /// The type of a Rust value to send to the task.
    /// This type must be thread-safe, as the Rust thread passes it
    /// from the main thread to the task thread.
    type Incoming: Send + Sync;

    /// An `Error` type representing an argument conversion failure in `on_next`.
    /// The Javascript sender function will throw a `JsError` with its `description()'.
    type IncomingError: Error;

    /// On calling the sender function from Javascript, convert its arguments
    /// into an `Incoming` value that the Rust thread can read and interact with.
    fn on_next<'a>(call: Call<'a>) -> Result<Self::Incoming, Self::IncomingError>;

    /// Performs the task on a Rust thread. Keeps the `emit` argument from
    /// `ObservableTask`, with an additional `receiver` argument that ingests
    /// messages from Javascript via an `std::sync::mpsc::Receiver`.
    /// You can do anything with this receiver, e.g wait to start until
    /// receiving a message or wait for messages in a loop and pattern
    /// match on their types.
    fn perform<N: Fn(Message<Self::Next, Self::Error, Self::Complete>)>(
        &self,
        emit: N,
        receiver: Receiver<Self::Incoming>,
    );
}
```

### Example

```rust

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
