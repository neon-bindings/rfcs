- Feature Name: Closure Functions
- Start Date: 2021-10-22
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Currently, Neon only allows creating `JsFunction` from static `fn`. However, Rust has a rich and expressive feature set for _closures_. Neon should allow creating functions from closures as well as static functions.

# Motivation
[motivation]: #motivation

Creating functions from closures has been a frequently requested feature:

* https://github.com/neon-bindings/neon/issues/264
* https://github.com/neon-bindings/neon/issues/809

However, even though this was always possible in Node-API, it was [previously impossible](https://github.com/neon-bindings/rfcs/pull/12#issuecomment-535526184) to do without leaking the closure.

Node-API 5 added the `napi_add_finalizer` function for adding arbitrary hooks when an object is garbage collected, allowing Neon to safely drop the closure.

In addition to being syntactically shorter, closures may also capture context from the environment allowing for many useful patterns. For example, creating an instance of async runtime and passing it to all functions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Creating a `JsFunction` from a closure in Neon uses the same syntax as creating a `JsFunction` from a static `fn`. The syntax is very familiar to Rust programmers.

```rust
fn static_function(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    Ok(cx.undefined())
}

cx.export_function("static", static_function);
cx.export_function("closure", |mut cx| Ok(cx.undefined()));
```

In addition to exporting functions, closures may be used when returning a function directly.

```rust
fn adder(mut cx: FunctionContext) -> JsResult<JsFunction> {
    let x = cx.argument::<JsNumber>(0)?.value(&mut cx)?;

    JsFunction::new(move |mut cx| {
        let y = cx.argument::<JsNumber>(0)?.value(&mut cx)?;

        Ok(cx.number(x + y))
    })
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

On Node-API < 5, the signatures of `ModuleContext::export_function` and `JsFunction::new` will not change. This is to prevent users on older Node versions from accidentally leaking.

On Node-API >= 5, the signatures will be updated to accept `Fn` closures.

```rust
impl<'cx> ModuleContext<'cx> {
    pub fn export_function<F, V>(&mut self, key: &str, f: F) -> NeonResult<()>
    where
        F: Fn(FunctionContext) -> JsResult<V> + 'static,
        V: Value,
    {
        todo!()
    }
}

impl JsFunction {
    pub fn new<'a, C, F, V>(cx: &mut C, f: F) -> JsResult<'a, JsFunction>
    where
        C: Context<'a>,
        F: Fn(FunctionContext) -> JsResult<V> + 'static,
        V: Value,
    {
        todo!()
    }
}
```

It would be *really* great if Neon could accept an `FnMut` closure. However, this would *not* be sound because `JsFunction` may be reentrant and the Rust compiler would not be able to enforce borrowing rules.

```rust
let mut calls = 0;

// If Neon allowed `FnMut`, this could be written:
let f = JsFunction::new(move |mut cx| {
    let current = calls;
    let f = cx.argument::<JsFunction>(0)?;
    let this = cx.undefined();

    // If `f` is a reference to this `JsFunction`, it will be called
    // recursively; each time incrementing `calls`. However, this is
    // unsound because the currently running code should have exclusive
    // ownership.
    f.call(&mut cx, this, [f])?

    calls = current + 1;

    Ok(cx.number(calls))
});
```

Unlike closures used in threading contexts (e.g., `Channel`, `TaskBuilder`), closures wrapped in a `JsFunction` do not need to implement `Send`. This is because they are only ever called from the event loop executing on a single thread.

Since the closures do not need to be `Send`, `Rc<RefCell<T>>` is a useful pattern for sharing state between methods.

```rust
#[derive(Clone, Default)]
struct Stats {
    calls: Rc<RefCell<usize>>,
}

impl Stats {
    fn inc(&self) {
        *self.calls.borrow_mut() += 1;
    }

    fn calls(&self) -> usize {
        self.calls.borrow()
    }
}

#[neon::main]
fn main(mut cx: ModuleContext) -> NeonResult<()> {
    let stats = Stats::default();

    cx.export_function("get_a", {
        let stats = stats.clone();

        move |mut cx| {
            stats.inc();
            Ok(cx.string("A"))
        }
    });

    cx.export_function("get_b", {
        let stats = stats.clone();

        move |mut cx| {
            stats.inc();
            Ok(cx.string("B"))
        }
    });

    cx.export_function("calls", {
        let stats = stats.clone();

        move |mut cx| Ok(cx.number(stats.calls()))
    });
}
```

# Drawbacks
[drawbacks]: #drawbacks

Calling functions could be slower by a very small amount from needing to perform dynamic dispatch.

# Rationale and alternatives
[alternatives]: #alternatives

The primary alternative considered was to allow closures on *all* Node-API versions. However, it is very subtle that this would leak. Especially since the leak is determined by Neon feature flags and not based on the current runtime.

Instead, we will only provide this feature for Node-API 5+. This costs a small amount of code complexity; however, we may be able to remove this later when all versions of Node on our support matrix provide Node-API 5.

# Unresolved questions
[unresolved]: #unresolved-questions

* ~~Do closures need to be `Send`?~~ `V8` heavily uses thread local storage for the vent loop. It would be an extremely large change to be able to switch the thread running the event loop. It would also require changes to `libuv`. Even thought it's not _guaranteed_ this won't change, it seems unlikely enough that the small risk is worth the improved API.
