- Feature Name: Async Builder
- Start Date: 2020-05-31
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

The `Task` API was introduced in Neon `v0.4.0` and has proven to be incredibly valuable to developers. It empowers users to execute long running tasks on the libuv thread pool without blocking the main javascript thread.

The `TaskBuilder` API expands upon this concept with an improved ownership model and ergonomics.

# Motivation
[motivation]: #motivation

The current `Task` trait passes `self` as a shared reference (`&self`) to `fn perform`. Multiple users (https://github.com/neon-bindings/neon/pull/283) (https://github.com/neon-bindings/neon/pull/520) have found this ownership limiting. Many usages require a mutable or owned `self`. Work-arounds require error prone runtime checks (e.g., `Cell` and `Option`).

While unlikely to cause issues, changing `&self` to `&mut self` is a breaking change. Users could have manually called `fn perform` in a context that relied on shared references.

Instead of introducing a breaking change, a new API is introduced with an _owned_ `self`. The following addtional changes are made:

* The new API relies on closures instead of a `trait`; thanks to many improvements in the borrow checker and closure ergonomics
* The new API returns a single `Output` generic rather than separate `Output` and `Error` types. This simplifies infallible tasks.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Context` trait provides a `.task()` method for performing asynchronous tasks on a background thread and calling back to javascript on completion.

The callback follows standard node.js callback style with an error as the first argument and the resolved value as the second argument.

The following example introduces a method named `add_one` which accepts a `Number` and a callback, performing the operation asynchronously.

```rust
pub fn add_one(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let n = cx.argument::<JsNumber>(0)?.value();
    let cb = cx.argument::<JsFunction>(1)?;

    cx.task(move || {
            let result = n + 1.0;

            move |mut cx| Ok(cx.number(result))
        })
        .schedule(cb)
}

register_module!(mut cx, {
    cx.export_function("add_one", addOne)
});
```

The `add_one` method can be called from Javascript:

```js
const { addOne } from './native';

addOne(41, (err, val) => {
    if (err) {
        console.error(err);
    } else {
        console.log(`Result: ${val}`);
    }
});
```

The `cx.task(...)` method creates a new `TaskBuilder`. It expects a single argument. This argument is a `closure` that will be executed on the `libuv` threadpool.

Any setup that requires the V8 VM (e.g., getting an `f64` from a `Number`) must be performed outside of this closure.

The closure is expected to return _another closure_. This higher-order method will be familiar to many Javascript developers. The returned closure is the `complete` method and will be performed back on the main Javascript thread. In the `complete` closure, the user may convert Rust types back to Javascript types.

Finally, a `schedule` method is provided to submit the task to the queue. For conveinance, it returns `JsUndefined`, matching Javascript semantics for most asynchronous methods.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Changes to low-level Task

The existing low-level `Task` implementation in C++ is modified to match a new ownership model. The `perform` model will _consume_ self. The `complete` method must be changed to no longer pass `self` to prevent use-after-free.

```rust
pub fn Neon_Task_Schedule(task: *mut c_void,
                          perform: unsafe extern fn(*mut c_void) -> *mut c_void,
                          complete: unsafe extern fn(*mut c_void, &mut Local),
                          callback: Local);
```

### Private `BaseTask` trait

A new (private) `BaseTask` trait is introduced. The `BaseTask` trait is very similar to the existing `Task` trait except `fn perform` is changed to take ownership of `self` and `fn complete` is changed to an associated method (does not use self).

A single `Output` parameter is provided and it is up to the user to return a `Result` if desired.

```rust
trait BaseTask: Send + Sized + 'static {
    type Output: Send + 'static;
    type JsEvent: Value;

    fn perform(self) -> Self::Output;
    fn complete<'a>(cx: TaskContext<'a>, result: Self::Output) -> JsResult<Self::JsEvent>;
    fn schedule(self, callback: Handle<JsFunction>);
}
```

### `Task` trait

An implementation of `BaseTask` is added for `T: Task` to provide backwards compatibility to existing implementations of `Task`. Since `complete` is an associated method, `self` is threaded through the `Output` parameter.

The existing `Task` trait will be _deprecated_.

```rust
impl<T: Task> BaseTask for T {
    // Output contains a `Result` to match the `Task` trait
    type Output = (Result<T::Output, T::Error>, T);
    type JsEvent = T::JsEvent;

    fn perform(self) -> Self::Output {
        // `self` is threaded through `Output` as part of a tuple
        (Task::perform(&self), self)
    }

    fn complete<'a>(cx: TaskContext<'a>, output: Self::Output) -> JsResult<Self::JsEvent> {
        // `task` is unpacked from the tuple to use the `complete` method
        let (result, task) = output;

        task.complete(cx, result)
    }
}
```

### `TaskBuilder`

The `TaskBuilder` struct contains two pieces of key data:

* A mutable reference to `Context`
    - Allows the conveinance of returning `JsUndefined` from `.schedule`
    - The `N-API` implementation will require access to `Env`
* The `task` closure to be performed on another thread

#### Limitation

Ideally we would have a more simple struct type and only call out the specific trait bounds on `fn schedule`.

```rust
pub struct TaskBuilder<'a, C, T> {
    context: &'a mut C,
    task: T,
}
```

However, due to a limitation in Rust type inference (https://github.com/rust-lang/rust/issues/41078), rust is not able to find the implementation of `schedule` without type annotating closure. This is an unacceptable hit to ergonomics.

#### Work-around

The full set of trait bounds is provided on `TaskBuilder`. This requires a `PhantomData` to track the lifetime of `ContextInternal`.

```rust
pub struct TaskBuilder<'a, 'b, C, Perform, Output>
where
    C: Context<'b>,
    Perform: FnOnce() -> Output + Send + 'static,
{
    _phantom: PhantomData<&'b C>,
    context: &'a mut C,
    task: PerformTask<Perform>,
}
```

#### Methods

`TaskBuilder` provides two methods.

* `.schedule(cb)`. This is a higher level method that schedules the task and also returns a `JsResult<JsUndefined>` for easy returning from another neon function.
* .schedule_task(cb)`. This is a lower level task that submits the task, but does not return a value.

### `PerformTask`

The `PerformTask` struct is a simple wrapper around a closure task used to implement the `BaseTask` trait.

# Drawbacks
[drawbacks]: #drawbacks

* Bloats the Neon API
* Closures can be unwieldy to work with combined with lifetimes
* Compile error messages may be difficult for new Rust users

# Rationale and alternatives
[alternatives]: #alternatives

* Changing the existing `Task` trait `perform` method to take `&mut self` or `self`
* Passing `complete` as another builder method instead of returning from `perform` (allows `complete` to be optional, but forces more complicated threading of `self`)
* Leaving the existing API and documenting proper use of `Cell<Option<T>>`

# Unresolved questions
[unresolved]: #unresolved-questions

* What should we name `schedule_task`?
* Is returning a closure for `complete` difficult to understand? Should there also be a `.complete(complete)` method available?
- Should `PerformTask` be eliminated and `BaseTask` be implemented directly on the closure?
- Do we want to allow user implementations of `BaseTask`?
