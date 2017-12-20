- Feature Name: VM 2.0
- Start Date: 2017-12-19
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes eliminating several of the distinct abstractions in the current API:

- the `Scope` trait
- the `FunctionCall` type
- the `Arguments` type
- the `vm` module

and collapsing them into a single `Vm` trait that represents a contextual view of the JavaScript virtual machine.

Like today’s `Scope` trait, the `Vm` trait encapsulates access to the memory management scope, so methods that interact with the GC require an argument with mutable access to the `Vm`. For example, the current API requires passing an `&mut impl Scope` to a VM operation such as:

```rust
let foo = obj.get(call.scope, "foo")?;
```

With this RFC, a function call takes an owned `impl Vm` and passes a mutable reference to that:

```rust
let foo = obj.get(&mut vm, "foo")?;
```

This effectively raises the level of abstraction in the mental model of the Neon user: instead of thinking about `Scope` as a mechanism that cleverly encodes memory safety with Rust’s mutability rules, the user can simply think of it as **a representation of the state of the JS virtual machine**, and mutable access to it naturally represents **modifying the state of the virtual machine**.

# Motivation
[motivation]: #motivation

This RFC addresses several issues with the usability and understandability of the current Neon API design:

**The** `**Scope**` **abstraction is confusing.** The invention of `Scope` grew incrementally from V8’s `HandleScope` abstraction in C++. But it has proved unintuitive to users: the name doesn’t provide a strong intuition, and the use of the mutability rules of Rust’s type system are overly clever while not particularly “saying what the programmer means.”

**The subscoping operations are especially confusing.** The “chain” and “nest” abstractions are important for making it possible to limit the lifetime of allocations and prevent leaks, but they are particularly confusing. Like `Scope`, the names are very generic without really expressing the core of what they do (execute a sub-computation).

**Abstracting over** `**FunctionCall**` **is fragile.** Some things happen to just work with the `call.scope` field, but when refactoring them into helper functions they often require wrestling with the vagaries of the Rust type system. For example, `call.scope` is already an `&mut` reference, but in order to pass a *reference* to `call` to a helper function, it’s necessary to annotate the `call` parameter as `mut` and pass an `&mut` pointer to the call (since passing an immutable reference would prevent the helper function from being able to use the `&mut` `scope` field).

**The VM lock API is a bit confusing and leads to rightward drift.** The current approach to “locking” the JavaScript VM relies on a subtle invariant: that none of the APIs that touch the VM implement the `Send` trait, so the `Lock::grab` API can run its callback with the guarantee that the engine won’t be touched. Encoding this invariant with frozen `&` references to a VM abstraction is a much more idiomatic representation. Moreover, Rust’s immutable borrows allow freezing in the remainder of a block without needing rightward drift.

The key insight behind this RFC is that we can provide an abstraction that intuitively represents a *context-sensitive* view into the JavaScript VM. By parameterizing the type over a lifetime (representing the duration of the particular VM context), the abstraction can serve the same purpose as the current `Scope` abstraction: namely, to track the lifetimes of GC handles. But this allows programmers to worry less about memory management and simply think of the `Vm` parameter as something they have to thread through calls to the Neon API as simple plumbing.

What’s more, the use of `mut` now corresponds to whether the JavaScript VM’s state can be modified, which is a much more natural use of mutability annotations. This also makes it easier to explain *why* the `vm` argument should be annotated as `mut` for passing mutable references to other functions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The `Vm` trait

Most Neon functions take an extra parameter in the form of a mutable reference to an implementation of the `Vm` trait. This parameter roughly represents access to the JavaScript virtual machine. For example, the `Object::get` method for getting a property of a JavaScript object requires an extra `Vm` parameter:

```rust
let foo = obj.get(&mut vm, "foo");
```

A good way to think about the `Vm` object is that it represents **the power to manipulate the JavaScript virtual machine**. In particular, mutable access to the `Vm` makes it possible to modify objects, execute JavaScript code, and perform garbage collection.

A powerful implication of this is that an API that does *not* take a mutable reference to a `Vm` **can never modify the JavaScript virtual machine**! This allows Neon to provide the ability to safely *lock* the virtual machine temporarily, allowing Rust code to inspect and modify the internals of native objects or typed arrays.

## Locking the VM

Some type of JavaScript values—such as typed arrays, or instances of Neon classes—implement an `Inspect` trait, which allows Rust to access their internals. In order to get access to these internals, you first need to *take out a lock* on the JavaScript VM. The `Vm::lock` method “freezes” the VM and produces a temporary `Lock` object, which you can then use to inspect the object.

For example, a simple Neon function that takes a single `ArrayBuffer` object and sets its first byte to 0 would be implemented by calling `vm.lock()` and then `lock.inspect()` with the resulting lock object:

```rust
fn zero_first_byte(mut vm: impl Vm) -> JsResult<Handle<JsNumber>> {
    let array_buffer = vm.argument(0).unwrap().check::<JsArrayBuffer>()?;
    let lock = vm.lock();
    let mut contents = lock.inspect(array_buffer);
    let mut buf = contents.as_u8_slice();
    buf[0] = 0;
    Ok(())
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `Vm` trait

The `Vm` trait has the following signature:

```rust
trait Vm<'a, T: This> {
    fn lock(&self) -> &Lock;

    fn argument(&mut self, i: i32) -> Option<Handle<'a, JsValue>>;
    fn this(&mut self) -> JsResult<'a, T>;
    fn callee(&mut self) -> JsResult<'a, JsFunction>;

    fn execute<'b, U, V, F>(&self, f: F) -> U
        where F: FnOnce(ExecuteContext<'b, T>) -> U;
    fn evaluate<'b, U, F>(&self, f: F) -> JsResult<'a, U>
        where U: Value,
                F: FnOnce(EvaluateContext<'b, T>) -> JsResult<'b, U>;

    fn number(&mut self, f: f64) -> JsResult<'a, JsNumber>;
    fn boolean(&mut self, b: bool) -> JsResult<'a, JsBoolean>;
    fn string(&mut self, s: &str) -> JsResult<'a, JsString>;
    fn null(&mut self) -> JsResult<'a, JsNull>;
    fn undefined(&mut self) -> JsResult<'a, JsUndefined>;

    fn empty_object(&mut self) -> JsResult<'a, JsObject>;
    fn empty_array(&mut self) -> JsResult<'a, JsArray>;

    fn array_buffer(byte_len: u32) -> JsResult<'a, JsArrayBuffer>;

    // future: 
    // fn function(&mut self, /* ??? */) -> JsResult<'a, JsFunction>;
    // fn array<I>(&mut self, elts: I) -> JsResult<'a, JsArray>
    //     where I: IntoIterator,
    //           I::Item: ToJs;
    // fn object<I>(&mut self, props: I) -> JsResult<'a, JsObject>
    //     where I: IntoIterator,
    //           I::Item: (Key, ToJs);
}
```

**vm.lock()**
Freezes the VM by taking out a lock.

**vm.argument(i)**
Return the *i*'th argument of the current function call.

**vm.this()**
Return the `this` binding of the current function call.

**vm.callee()**
Return the callee (i.e., the called function) of the current function call.

**vm.execute(|mut vm| { … })**
Execute a callback with a new `Vm` instance. Any locally allocated handles created during the body of the callback are only kept alive for the duration of the callback. This method is useful for executing code that may generate temporary data and allowing that temporary data to get freed by the JavaScript garbage collector, especially inside of a loop that may result in large amounts of temporary garbage.

**vm.evaluate(|mut vm| { … })**
Execute a callback with a new `Vm` instance and return a JavaScript value. Any locally allocated handles created during the body of the callback are only kept alive for the duration of the callback, except for the result value, which is kept alive for the duration of the outer `Vm`. This method is useful for executing code that may generate temporary data and allowing that temporary data to get freed by the JavaScript garbage collector, especially inside of a loop that may result in large amounts of temporary garbage.

**vm.number(f)**
**vm.boolean(b)**
**vm.string(s)**
**vm.null()**
**vm.undefined()**
Convenience constructors for primitive value types.

**vm.empty_object()**
**vm.empty_array()**
Convenience constructors for vanilla objects and array objects.

**vm.array_buffer(byte_len)**
Convenience constructor for creating a new `ArrayBuffer` object.

## Implementations of `Vm`

The `CallContext`, `ExecuteContext`, and `EvaluateContext` types are the three implementations of `Vm`:

```rust
struct CallContext<'a, T: This> { /* ... */ };

impl<'a, T: This> Vm<'a, T> for CallContext<'a, T> { /* ... */ };
```

```rust
struct ExecuteContext<'a, T: This> { /* ... */ };

impl<'a, T: This> Vm<'a, T> for ExecuteContext<'a, T> { /* ... */ };
```

```rust
struct EvaluateContext<'a, T: This> { /* ... */ };

impl<'a, T: This> Vm<'a, T> for EvaluateContext<'a, T> { /* ... */ };
```

These types correspond to different context-sensitive views of the JavaScript virtual machine: the context of a call into a native Rust implementation of a JavaScript function, the context of a call to the `vm.execute()` method, or the context of a call to the `vm.evaluate()` method, respectively. From the user’s perspective, though, the differences between these types are unimportant, and they can simply all be thought of as instances of `Vm`.

## The `Lock` type

The `Lock` type is produced by the `vm.lock()` method described above, and can be used to inspect the internals of JavaScript values:

```rust
impl Lock {
    fn inspect<T: Inspect>(&self, value: Handle<T>) -> &mut T::Internals;
}
```

## Indexing into a `Vm`

We should experiment with making instances of the `Vm` trait implement `Index` and `IndexMut`, to make it more convenient to access arguments:

```rust
impl<'a, T: This> Index<i32> for Vm<'a, T> {
    type Output = JsValue;
    fn index(&self, i: i32) -> &JsValue {
        &self.argument(i).unwrap()
    }
}

impl<'a, T: This> IndexMut<i32> for Vm<'a, T> {
    type Output = JsValue;
    fn index_mut(&mut self, i: i32) -> &mut JsValue {
        &mut self.argument(i).unwrap()
    }
}
```

It’s an open question whether this works and needs experimentation.

# Drawbacks
[drawbacks]: #drawbacks

This approach is less respecting of the “single-responsibility” principle of type design. However, the distinct abstractions in the current design aren’t particularly powerful (e.g., it’s rare to need to pass around an `Arguments` struct), and incur a lot of syntactic overhead (e.g., `call.arguments.require(call.scope, 0)` as opposed to `vm.argument(0)`). And fattening the `Vm` interface also allows many operations to have a simple, single-dispatch OO variant without the extra `&mut vm` argument—at least in the form of convenient shorthands.

# Rationale and alternatives
[alternatives]: #alternatives

Quite frankly, I’ve struggled to come up with alternative solutions to the type design of Neon. I’m very much open to other designs, but this does seem like a promising direction if we don’t come up with viable alternatives.

# Unresolved questions
[unresolved]: #unresolved-questions

- Do I have the lifetimes right? This needs an experimental implementation to see what I may have missed.
- Can we make `Vm` indexable with `[…]` syntax via `IndexMut` or will there not be a simple, understandable set of idioms for using this? Also needs experimental implementation.
