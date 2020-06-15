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
* The new API returns a single `Output` generic rather than separate `Output` and `Error` types. This simplifies infallible tasks and is still expressive enough to allow users to use a `Result` type to represent fallible tasks.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The `Context` trait provides a `.task()` method for performing asynchronous tasks on a background thread and calling back to javascript on completion.

The callback follows standard node.js callback style with an error as the first argument and the resolved value as the second argument.

The following example introduces a method named `add_one` which accepts a `Number` and a callback, performing the operation asynchronously.

```rust
pub fn add_one(mut cx: FunctionContext) -> JsResult<JsValue> {
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

The `TaskBuilder::schedule` method submits the task to the queue and also returns an `undefined` to match the idiom of most async Javascript methods.

The returned `JsUndefined` is upcast to `JsValue` and `Ok` wrapped for maximum conveinance, allowing a user to terminate a Neon methon with a call to `.schedule`.

```rust
impl TaskBuilder {
    fn schedule(self, callback: Handle<JsFunction>) -> JsResult<JsValue> {}
}
```

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

### `fn schedule`

A a private type-safe wrapper for introduced for `neon_runtime::task::schedule`. The `schedule` function expects an `FnOnce` from `TaskBuilder`; boxing, leaking, and passing pointers to `neon_runtime::task::schedule`.

```rust
fn schedule<Perform, Complete, Output>(
    perform: Perform,
    callback: Handle<JsFunction>,
)
where
    Perform: FnOnce() -> Complete + Send + 'static,
    Complete: FnOnce(TaskContext) -> JsResult<Output> + Send + 'static,
    Output: Value,
{
    let data = Box::into_raw(Box::new(perform));

    unsafe {
        neon_runtime::task::schedule(
            data as *mut _,
            perform_task::<Perform, Complete>,
            complete_task::<Complete, Output>,
            callback.to_raw(),
        );
    }
}
```

The existing unsafe `perform_task` and `complete_task` passed as static pointers to `neon_runtime::task::schedule` are modified to expect boxed closures `Perform` and `Complete` respectively.

```rust
unsafe extern "C" fn perform_task<Perform, Output>(
    perform: *mut c_void,
) -> *mut c_void
where
    Perform: FnOnce() -> Output,
{
    let perform = Box::from_raw(perform as *mut Perform);
    let result = perform();

    Box::into_raw(Box::new(result)) as *mut _
}

unsafe extern "C" fn complete_task<Complete, Output>(
    complete: *mut c_void,
    out: &mut raw::Local,
)
where
    Complete: FnOnce(TaskContext) -> JsResult<Output>,
    Output: Value,
{
    let complete = *Box::from_raw(complete as *mut Complete);

    TaskContext::with(|cx| {
        if let Ok(result) = complete(cx) {
            *out = result.to_raw();
        }
    })
}
```

### `Task` trait

The default `schedule` implementation on the `Task` trait is replaced with a delegate to the new `fn schedule` function.

```rust
    fn schedule(self, callback: Handle<JsFunction>) {
        schedule(move || {
            let result = self.perform();

            move |cx| self.complete(cx, result)
        }, callback);
    }
```

Since the existing `Task` trait is both less convenient and less powerful than `TaskBuilder`, the `Task` trait will be _deprecated_.

* The `Task` trait requires more boilerplate to use since it requires defining a `struct` and implementing a trait
* The `Task` trait is less powerful because it only receives a shared `&self` reference in the `peform` method instead of full ownership.

The `Task` trait will *not* be removed in the near future. Instead, documentation will guide users towards the new API while existing users have time to migrate.

### `TaskBuilder`

The `TaskBuilder` struct contains two pieces of key data:

* A mutable reference to `Context`
    - Allows the conveinance of returning `undefined` from `.schedule`
    - The `N-API` implementation will require access to `Env`
* The `task` closure to be performed on another thread

The `TaskBuilder` struct is *public*.

```rust
pub struct TaskBuilder<'c, C, Perform> {
    context: &'c mut C,
    perform: Perform,
}
```

Even though the `TaskBuilder` struct is public, it may not be directly created outside of the `neon` crate.

```rust
impl<'c, C, Perform> TaskBuilder<'c, C, Perform> {
    pub(crate) fn new(context: &'c mut C, perform: Perform) -> Self;
}
```

Instead users, are expected to create instances of `TaskBuilder` from a method on the `Context` trait.

```rust
pub trait Context<'a> {
    fn task<Perform, Complete, Output>(
        &mut self,
        perform: Perform,
    ) -> TaskBuilder<Self, Perform>
    where
        Perform: FnOnce() -> Complete + Send + 'static,
        Complete: FnOnce(TaskContext) -> JsResult<Output> + Send + 'static,
        Output: Value,
    {
        TaskBuilder::new(self, perform)
    }
}
```

#### Limitation

Ideally the `cx.task(...)` method would not contain any additional bounds beyond what is required from `TaskBuilder`.

```rust
pub trait Context<'a> {
    fn task<Perform>(perform: Perform) -> TaskBuilder<Self, Perform>;
}
```

However, due to a limitation in Rust lifetime inference and grammar related to closures (https://github.com/rust-lang/rust/issues/41078), without an additional hint on the types contained by `TaskBuilder`, it is impossible to use `TaskBuilder::schedule`.

```
error[E0271]: type mismatch resolving `for<'r> <[closure@test/dynamic/native/src/js/tasks.rs:32:13: 32:61 result:_] as std::ops::FnOnce<(neon::context::TaskContext<'r>,)>>::Output == std::result::Result<neon::handle::Handle<'r, _>, neon::result::Throw>`
  --> test/dynamic/native/src/js/tasks.rs:34:10
   |
34 |         .schedule(cb)
   |          ^^^^^^^^ expected bound lifetime parameter, found concrete lifetime
```

#### Work-around

The full set of bounds required by `TaskBuilder::schedule` are also provided on `Context::task`. However, since `TaskBuilder` cannot be created directly, it leaves the API open for future expansion if this issue is resolved.

#### Methods

`TaskBuilder` provides two methods.

* `.schedule(cb)`. This is a higher level method that schedules the task and also returns an `undefined` as `JsResult<JsValue>` for easy returning from another neon function.
* `.schedule_task(cb)`. This is a lower level task that submits the task, but does not return a value.

```rust
impl<'a, 'c, C, Perform, Complete, Output> TaskBuilder<'c, C, Perform>
where
    C: Context<'a>,
    Perform: FnOnce() -> Complete + Send + 'static,
    Complete: FnOnce(TaskContext) -> JsResult<Output> + Send + 'static,
    Output: Value,
{
    pub fn schedule(self, callback: Handle<JsFunction>) -> JsResult<'a, JsValue>;
    pub fn schedule_task(self, callback: Handle<JsFunction>);
}
```

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

* What should we name `schedule_task`? Using the word `task` in a method on a struct named `TaskBuilder` is redundant.
* ~Is returning a closure for `complete` difficult to understand? Should there also be a `.complete(complete)` method available?~ Becuase of the lifetime inference limitation, it's not possible to support both APIs.
