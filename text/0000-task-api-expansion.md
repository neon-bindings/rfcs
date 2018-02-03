* Feature Name: task_api_expansion
* Start Date: 2018/01/31
* RFC PR: (leave this empty)
* Neon Issue: (leave this empty)

# Summary

[summary]: #summary

This RFC expands the Task API to support the needs of long-running tasks in a Rust thread. New traits allow for single-shot, observable, and bidirectional tasks. The previous `Task` trait remains as `NodeThreadPoolTask` for consumers who wish to continue running tasks on Node's `libuv` thread pool.

# Motivation

[motivation]: #motivation

The current `Task` trait has two limitations that render long-running async Rust work impractical:

* Tasks run in the Node thread pool using `uv_queue_work`. This means that long-running tasks can clog the thread pool, potentially blocking Node's filesystem and compression operations.
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

The `Task` trait defines a "single-shot" task: a short, asynchronous piece of work run in a Rust thread that terminates after providing a single completion value or error. It's analogous to a Node callback API (`runTask((err, value) => {})`). Implementing the trait involves defining the work in `perform` and defining how its results map to Javascript values in `error` and `complete`.

```rust
pub trait Task: Send + Sync + Sized + 'static {
    /// The type of the completion value returned from `perform`.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Complete: Send + Sync;

    /// The type of the error value returned from `perform`.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Error: Send + Sync + Error;

    /// The Javascript type of the value provided to the second parameter
    /// of the Javascript callback.
    type JsComplete: Value;

    /// Performs the task on a Rust thread, producing either a completion value or an error.
    fn perform(&self) -> Result<Self::Complete, Self::Error>;

    /// Transform the `Complete` value returned from `perform` into a Javascript value
    /// to be passed to the second parameter of the asynchronous callback.
    /// This method runs on the main thread event loop an indeterminate amount of time
    /// after finishing `perform`.
    fn complete<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        result: &Self::Complete,
    ) -> JsResult<Self::JsComplete>;

    /// Transform the `Error` value returned from `perform` into a Javascript value
    /// to be passed to the first parameter of the asynchronous callback.
    /// This method runs on the main thread event loop an indeterminate amount of time
    /// after finishing `perform`.
    fn error<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        error: &Self::Error,
    ) -> JsResult<JsError>;
}
```

### Example

```rust

```

## ObservableTask

The `ObservableTask` trait is a superset of the `Task` trait that supports emitting intermediate events. It's analogous to the `subscribe(onNext, onError, onComplete)` method of [common `Observable` types](https://github.com/tc39/proposal-observable#observable). Implementing the trait is the same as implementing `Task`, but with a few additions and a different `perform` method:

```rust
pub trait ObservableTask: Send + Sync + Sized + 'static {
    /// Methods and types shared with `Task` are omitted.

    /// Of note: in this trait, the `complete` and `error` methods provide values to
    /// `onComplete` and `onError` respectively, rather than the two parameters of
    /// a single Node-style callback.

    /// The type of an intermediate value emitted from `perform`.
    /// This type must be thread-safe, as the Rust thread passes it
    /// back to the main thread to convert into Javascript values.
    type Next: Send + Sync;

    /// The Javascript type of the value provided to the Javascript `onNext` callback.
    type JsNext: Value;

    /// Performs the task on a Rust thread. Uses the `emit` function to send errors,
    /// intermediate values, and completion signals to Javascript. Note that the
    /// task will hang indefinitely until calling `emit` with a `Message::Complete`.
    fn perform<N: Fn(Message<Self::Next, Self::Error, Self::Complete>)>(&self, emit: N);

    /// Transform the `Complete` value emitted from `perform` into a Javascript value
    /// to be passed to the `onNext` callback. This method runs on the main thread
    /// event loop an indeterminate amount of time  after finishing `perform`.
    fn next<'a, T: Scope<'a>>(
        &'a self,
        scope: &'a mut T,
        value: &Self::Next,
    ) -> JsResult<Self::JsNext>;
}
```

### Example

```rust

```

## WorkerTask

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
