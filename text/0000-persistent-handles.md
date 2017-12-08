- Feature Name: persistent-handles
- Start Date: 2017-12-08
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Create a handle type for Node.js values that is unrestricted by scopes.

# Motivation
[motivation]: #motivation

Currently, Rust code can only store and interact with Node.js references in the context of a `Scope`. This means that all references can only exist in the main call stack and cannot be moved to other contexts. This makes it difficult to design an API that might hold a Node.js reference in native Rust code for later use.

For example, imagine a Rust libary that invokes an event handler when a trigger fires. This library may want to take in a callback at initialization time, and invoke it later on when the trigger fires. Without a perisistent handle type, Rust data structures cannot hold references to Node.js objects outside the lifetime of the current call stack. The library would have no way to asynchronously invoke the callback at a later date.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC proposes a `PersistentHandle` type. This `PersistentHandle` will have a very simple interface. It can be constructed from a normal `Handle`:

```rust
let foo = JsString::new(scope, "foo").unwrap();
let foo_handle = PersistentHandle::new(foo);
```

Note that the requirement of a handle implies that we are in a `Scope` context.

Once a `PersistentHandle` is constructed, it cannot be directly interacted with. `PersistentHandle` does not implement `Deref` and it does not hold type information. In order to interact with the referenced data, the `PersistentHandle` must be deconstructed like so:

```rust
let foo: Handle<JsString> = foo_handle.into_handle(scope)
    .check()
    .unwrap();
```

Note that this also requires a scope reference. Because of the interface requirements of `PersistentHandle`, the underlying data can only be accessed using a `Scope` reference, and therefore only on the event-loop thread. This ensures the thread-safety of the underlying data. `PersistentHandle` values can be cloned and moved between threads, but they can only be constructed, deconstructed, read, and modified on the main thread.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The `PersistentHandle` type will use a raw pointer to an instance of the `Nan::Persistent<v8::Value>` class. The class will be heap allocated, to allow the pointer to be safely passed between threads. Construction and deconstruction of `PersistentHandle` will use native C++ functions to interact with `Nan` types. Additional native calls will be made to clone and destroy `PersistentHandle` values using the `Clone` and `Drop` traits, respectively.

# Drawbacks
[drawbacks]: #drawbacks

An incorrect implementation of this feature could lead to silent memory leaks.

# Rationale and alternatives
[alternatives]: #alternatives

There might be alternative designs for a `PersistentHandle` which perserve type information. However, a design using type erasure is much simpler.

# Unresolved questions
[unresolved]: #unresolved-questions

Is the `Reset(value)` method of the `Nan::Persistent` class thread-safe? Can it be called from a non-main thread and ouside of a handle scope?
