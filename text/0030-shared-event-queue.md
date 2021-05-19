- Feature Name: Shared ThreadsafeFunction in EventQueue
- Start Date: 2021-05-19
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Make `EventQueue` use a shared `ThreadsafeFunction` to leverage the
[performance optimization][0] in recent Node.js versions.

# Motivation
[motivation]: #motivation

`EventQueue` provides a way to invoke Rust closures on a JavaScript thread from
another thread by using a `ThreadsafeFunction`. Recent Node.js versions have a
[performance optimization][0] for the case where a single `ThreadsafeFunction`
is invoked multiple times in a row. Thus using a single shared
`ThreadsafeFunction` in `EventQueue`s would lead to significant performance
improvements for Neon users.

These improvements are perhaps most noticeable when Neon addon is used from the
Electron, because scheduling new UV ticks in Electron are very costly.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Whenever `cx.queue()` is invoked - a new instance of `EventQueue` is created.
With ithe proposed changes the instance would point to a shared
`ThreadsafeFunction` internally. For the most scenarios there would be no
difference between current and proposed implementation. Calling `queue.send()`
will still invoke the closure on the JavaScript thread:
```rust
let queue = cx.queue();

std::thread::spawn(move || {
    queue.send(|| {
        Ok(())
    })
});
```

Calling `queue.send()` several times in a row using different `cx.queue()` would
be significantly faster:
```rust
loop {
    cx.queue().send(|| {
        Ok(())
    })
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

With the recent [node.js changes][0] `ThreadsafeFunction` will loop over pending
calls, invoking up to 1000 of them in the same tick instead of running each new
call on a separate UV loop tick. This is an implementation detail that is
intentionally not covered by `napi_create_threadsafe_function` documentation to
allow for potential performance optimizations in the future.

Neon doesn't currently leverage it without the proposed changes because a new
`ThreadsafeFunction` is created for each new `EventQueue`, and `cx.queue()`
creates a new `EventQueue` every time as requested. So instead of doing that it
is proposed that we use existing `lifetime::InstanceData` internal API layer to
hold a wrapper around `ThreadsafeFunction` tailored to execute Rust closures
specifically, and to use an `Arc` reference to a `RwLock` over this wrapper in
the `EventQueue` for a shared access.

# Drawbacks
[drawbacks]: #drawbacks

If some of the users of `cx.queue()` would post significantly more closures on
the queue than the others it might lead to a potential starvation:
```rust
let queue = cx.queue();
std::thread::spawn(move || {
    loop {
        thread::sleep(Duration::from_secs(1));

        queue.send(|| {
            println!("every second");
            Ok(())
        })
    }
});

let another_queue = cx.queue();
std::thread::spawn(move || {
    loop {
        another_queue.send(|| {
            println!("immediate");
            Ok(())
        })
    }
});
```

The example above wouldn't print "every second" once per second. The prints
will be delayed by the closures scheduled on `another_queue`.

While the example above reproduces the issue - it is quite pathological and
should be avoided even without the proposed changes.

# Rationale and alternatives
[alternatives]: #alternatives

As an alternative, we could expose `lifecycle::InstanceData` to let API users
store instances of `EventQueue` in it themselves, shifting the burden of
performance optimization from Neon's hands to users.

Another alternative would be to make `EventQueue::new()` use new
`ThreadsafeFunction` unlike `cx.queue()` that would still return the
`EventQueue` instance that encapsulates a shared `ThreadsafeFunction`.

# Unresolved questions
[unresolved]: #unresolved-questions

The only unresolved question as it seems is whether we should go with an
alternative approach vs going with the proposed changes.

[0]: https://github.com/nodejs/node/commit/7abc7e45b2e177307f67f33906663b5de02f2916
