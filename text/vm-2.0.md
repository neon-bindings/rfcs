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
- the `Lock` trait

and collapsing them into a single `vm::Context` trait that represents a contextual view of the JavaScript virtual machine.

Like today’s `Scope` trait, the `Context` trait encapsulates access to the memory management scope, so methods that interact with the GC require an argument with mutable access to a `Context`. For example, the current API requires passing an `&mut impl Scope<'_>` to a VM operation such as:

```rust
let foo = obj.get(call.scope, "foo")?;
```

With this RFC, a function call takes an owned implementation of `Context` and passes mutable references to it to the APIs that previously required a scope:

```rust
let foo = obj.get(&mut cx, "foo")?;
```

This effectively raises the level of abstraction in the mental model of the Neon user: instead of thinking about `Scope` as a mechanism that cleverly encodes memory safety with Rust’s mutability rules, the user can simply think of it as **a representation of mutable access to the JS virtual machine**, where mutable access represents **modifying the state of the virtual machine**.

Here's an example using the new API:
```rust
fn zero_first_byte(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let mut array_buffer = cx.argument::<JsArrayBuffer>(0)?;
    {
        let guard = cx.lock();
        let mut contents = array_buffer.borrow_mut(&guard);
        let mut buf = contents.as_u8_slice();
        buf[0] = 0;
    }
    Ok(cx.undefined())
}
```

# Motivation
[motivation]: #motivation

This RFC addresses several issues with the usability and understandability of the current Neon API design:

**The `Scope` abstraction is confusing.** The invention of `Scope` grew incrementally from V8’s `HandleScope` abstraction in C++. But it has proved unintuitive to users: the name doesn’t provide a strong intuition, and the use of the mutability rules of Rust’s type system are overly clever while not particularly “saying what the programmer means.”

**The subscoping operations are especially confusing.** The “chain” and “nest” abstractions are important for making it possible to limit the lifetime of allocations and prevent leaks, but they are particularly confusing. Like `Scope`, the names are very generic without really expressing the core of what they do (execute a sub-computation).

**Abstracting over `FunctionCall` is fragile.** Some things happen to just work with the `call.scope` field, but when refactoring them into helper functions they often require wrestling with the vagaries of the Rust type system. For example, `call.scope` is already an `&mut` reference, but in order to pass a *reference* to `call` to a helper function, it’s necessary to annotate the `call` parameter as `mut` and pass an `&mut` pointer to the call (since passing an immutable reference would prevent the helper function from being able to use the `&mut` `scope` field).

**The VM lock API is a bit confusing and leads to rightward drift.** The current approach to “locking” the JavaScript VM relies on a subtle invariant: that none of the APIs that touch the VM implement the `Send` trait, so the `Lock::grab` API can run its callback with the guarantee that the engine won’t be touched. Encoding this invariant with frozen `&` references to a VM abstraction is a more idiomatic. Moreover, Rust’s immutable borrows allow freezing in the remainder of a block without needing rightward drift.

The key insight behind this RFC is that we can provide an abstraction that intuitively represents a *context-sensitive* view into the JavaScript VM. By parameterizing the type over a lifetime (representing the duration of the particular VM context), the abstraction can serve the same purpose as the current `Scope` abstraction: namely, to track the lifetimes of GC handles. But this allows programmers to worry less about memory management and simply think of the `Context` parameter as something they have to thread through calls to the Neon API as simple plumbing.

What’s more, the use of `mut` now corresponds to whether the JavaScript VM’s state can be modified, which is a more natural use of mutability annotations. This also makes it easier to explain *why* the `cx` argument should be annotated as `mut` for passing mutable references to other functions.

Finally, this design sets the stage for [a more complete design of the macro syntax](https://gist.github.com/dherman/d1d6d62e4e44ffbee1e1eebd51b53f06#file-classes_two_point_oh-rs-L48-L63), which should be fleshed out in a subsequent "Classes 2.0" RFC. In particular, the new syntax should allow an optional identifier (conventionally, this would normally be named `cx`), and an optional `mut` annotation, before the formal parameters list. Taking a representation of the state of the JavaScript VM, which is required for passing to many Neon APIs, again provides a higher level of abstraction than a representation of a function activation in the form of a `FunctionCall` struct. This makes for a gentler slope from the highest-level convenience syntaxes to the more expressive forms.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The `Context` trait

Most Neon APIs take an extra parameter in the form of a mutable reference to an implementation of the `Context` trait. This parameter roughly represents access to the JavaScript virtual machine. For example, the `Object::get` method for getting a property of a JavaScript object requires an extra `Context` parameter:

```rust
let foo = obj.get(&mut cx, "foo");
```

A good way to think about a VM context is that it represents **the power to manipulate the JavaScript virtual machine** in some execution context. In particular, mutable access to the `Context` makes it possible to modify objects, execute JavaScript code, and perform garbage collection.

A powerful implication of this is that an API that does *not* take a mutable reference to a `Context` **can never modify the JavaScript virtual machine**! This allows Neon to provide the ability to safely *lock* the virtual machine temporarily, allowing Rust code to inspect and modify the internals of native objects or typed arrays.

## Accessing the VM

When implementing a native Neon function or method, the implementation takes an owned context parameter. You can give it whatever variable name you like, but the name `cx` is customary:

```rust
fn zero_first_byte(cx: FunctionContext) -> JsResult<JsUndefined> {
    // ...
}
```

This `cx` parameter has a type that implements the `Context` trait. You will typically want to declare the `cx` parameter to be mutable:

```rust
fn zero_first_byte(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    // ...
}
```

This allows you to use APIs that require a mutable reference to the VM context.

## Locking the VM

Some type of JavaScript values—such as typed arrays, or instances of Neon classes—implement a `Borrow` trait, which allows you to _borrow_ their internals in Rust code. In order to get access to these internals, you first need to *take out a lock* on the JavaScript VM. The `Context::lock` method “freezes” the VM and produces a temporary `VmGuard` object, which you can then use to get temporary access to their internals.

For example, a simple Neon function that takes a single `ArrayBuffer` object and sets its first byte to 0 would be implemented by calling `cx.lock()` and then `array_buffer.borrow_mut(&guard)` with the resulting guard object:

```rust
fn zero_first_byte(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let array_buffer = cx.argument::<JsArrayBuffer>(0)?;
    {
        let guard = cx.lock();
        let mut contents = array_buffer.borrow_mut(&guard);
        let mut buf = contents.as_u8_slice();
        buf[0] = 0;
    }
    Ok(cx.undefined())
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `Context` trait

The `Context` trait has the following signature:

```rust
trait Context<'a> {
    fn lock(&self) -> &VmGuard;

    fn execute_scoped<'b, T, V, F>(&self, f: F) -> T
        where F: FnOnce(ExecuteContext<'b>) -> T;
    fn compute_scoped<'b, V, F>(&self, f: F) -> JsResult<'a, V>
        where V: Value,
              F: FnOnce(ComputeContext<'b>) -> JsResult<'b, V>;

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

**cx.lock()**
Freezes the VM by taking out a lock.

**cx.execute_scoped(|mut cx| { … })**
Execute a callback with a new `Context` instance. Any locally allocated handles created during the body of the callback are only kept alive for the duration of the callback. This method is useful for executing code that may generate temporary data and allowing that temporary data to get freed by the JavaScript garbage collector, especially inside of a loop that may result in large amounts of temporary garbage.

**cx.compute_scoped(|mut cx| { … })**
Execute a callback with a new `Context` instance and return a JavaScript value. Any locally allocated handles created during the body of the callback are only kept alive for the duration of the callback, except for the result value, which is kept alive for the duration of the outer `Context`. This method is useful for executing code that may generate temporary data and allowing that temporary data to get freed by the JavaScript garbage collector, especially inside of a loop that may result in large amounts of temporary garbage.

**cx.number(f)**
**cx.boolean(b)**
**cx.string(s)**
**cx.null()**
**cx.undefined()**
Convenience constructors for primitive value types.

**cx.empty_object()**
**cx.empty_array()**
Convenience constructors for vanilla objects and array objects.

**cx.array_buffer(byte_len)**
Convenience constructor for creating a new `ArrayBuffer` object.

## Implementations of `Context`

The `CallContext`, `ModuleContext`, `TaskContext`, `ExecuteContext`, and `EvaluateContext` types are the five implementations of `Context`. Each type represents a view of the JavaScript virtual machine in a different kind of execution context.

### `CallContext`

A `CallContext` is the execution context of a call into a native Neon function.

```rust
struct CallContext<'a, T: This> { /* ... */ };

impl<'a, T: This> CallContext<'a, T> {
    fn len(&self) -> i32;
    fn argument_opt(&mut self, i: i32) -> Option<Handle<'a, JsValue>>;
    fn argument<V: Value>(&mut self, i: i32) -> JsResult<'a, V>;
    fn this(&mut self) -> JsResult<'a, T>;
    fn callee(&mut self) -> JsResult<'a, JsFunction>;
}

impl<'a, T: This> Context<'a> for CallContext<'a, T> { /* ... */ };
```

There are two type shorthands:

```rust
type FunctionContext<'a> = CallContext<'a, JsObject>;
type MethodContext<'a, T> = CallContext<'a, T>;
```

### `ModuleContext`

A `ModuleContext` is the execution context of the initialization of a native Neon module.

```rust
struct ModuleContext<'a> { /* ... */ }

impl<'a> ModuleContext<'a> {
    pub fn export_function<T: Value>(&mut self, key: &str, f: fn(CallContext<JsObject>) -> JsResult<T>) -> VmResult<()>;
    pub fn export_class<T: Class>(&mut self, key: &str) -> VmResult<()>;
    pub fn export_value<T: Value>(&mut self, key: &str, val: Handle<T>) -> VmResult<()>;
    pub fn exports_object(&mut self) -> JsResult<'a, JsObject>;
}
```

### `TaskContext`

A `TaskContext` is the execution context of the completion of a Neon task.

```rust
struct TaskContext<'a> { /* ... */ };

impl<'a> Context<'a> for TaskContext<'a> { /* ... */ };
```

### `ExecuteContext`

An `ExecuteContext` is the execution context of a call to `cx.execute_scoped()`.

```rust
struct ExecuteContext<'a> { /* ... */ };

impl<'a> Context<'a> for ExecuteContext<'a> { /* ... */ };
```

### `ComputeContext`

A `ComputeContext` is the execution context of a call to `cx.compute_scoped()`.

```rust
struct ComputeContext<'a, 'outer> { /* ... */ };

impl<'a, 'outer> Context<'a> for ComputeContext<'a, 'outer> { /* ... */ };
```

## The `VmGuard` type

The `VmGuard` type is produced by the `cx.lock()` method described above, and can be used to borrow the internals of JavaScript values. These values must implement the `Borrow` trait:

```rust
unsafe trait Borrow {
    type Internals;

    fn borrow(&self, guard: &VmGuard) -> Ref<Self::Internals>;
    fn try_borrow(&self, guard: &VmGuard) -> Option<Ref<Self::Internals>>;
}
```

Mutable borrows require the `BorrowMut` trait:

```rust
unsafe trait BorrowMut: Borrow {
    fn borrow_mut(&mut self, guard: &VmGuard) -> RefMut<Self::Internals>;
    fn try_borrow_mut(&mut self, guard: &VmGuard) -> Option<RefMut<Self::Internals>>;
}
```

The implementation of `Borrow` and `BorrowMut` will typically use an unsafe constructor for the `Ref` and `RefMut` types, which dynamically borrow the memory region associated with the JS object. This maintains a set of pointers internally in the state of the `VmGuard` object to ensure that pointers are never multiply-borrowed, respecting the rules of Rust borrowing. The `Ref` and `RefMut` types also implement `Drop`, so the pointer being borrowed is removed from the set when a reference is dropped.

```rust
impl Ref<T> {
    unsafe new(lock: &VmGuard, address: *const c_void, size: usize) -> Option<Ref<T>>;
}

impl RefMut<T> {
    unsafe new(lock: &VmGuard, address: *const c_void, size: usize) -> Option<RefMut<T>>;
}
```

# Critique
[critique]: #critique

This approach is less respecting of the “single-responsibility” principle of type design. However, the distinct abstractions in the current design aren’t particularly powerful (e.g., it’s rare to need to pass around an `Arguments` struct), and incur a lot of syntactic overhead (e.g., `call.arguments.require(call.scope, 0)` as opposed to `vm.argument(0)`). And fattening the `Context` interface also allows many operations to have a simple, single-dispatch OO variant without the extra `&mut cx` argument—at least in the form of convenient shorthands.

I have come up with a few alternatives, but none of them worked out quite as well as this one:

- **Call it "VM" instead of "context" (`vm` instead of `cx`)**: This was my original idea, but it made the methods for the specific implementations like `CallContext` read awkwardly, such as `vm.argument(0)`. And conceptually "a `Context` is a context-sensitive view of the VM" is bit more honest and easy to explain than "a `VM` is a context-sensitive view of the VM."
- **A `cx.vm()` method**: We could separate out a context from a VM -- this is less of a divergence from the original Neon API, but it's still an improvement over the "scope" concept. So you could use the context to call context-specific methods, like `CallContext::arguments` and `ModuleContext::export_function`. Since this method would take an `&mut` and return an `&mut`, I _think_ this would mean the borrow checker would prevent using the `cx` while using the `vm`. In practice, this would mean you would have to extract all of the arguments from a `CallContext` up front before doing anything useful with them, including type conversions. So simple idioms like this would be disallowed:
```rust
let foo = cx.argument::<JsValue>(0)?.to_js_string(&mut cx)?;
let bar = cx.argument::<JsValue>(1)?.to_js_string(&mut cx)?;
```
- **A `cx.into_vm()` method**: Similarly, we could just be up front about the fact that extracting the VM is the last thing you can do with a context by consuming the context. Again, idioms like the previous example would be disallowed.

There are also a couple of pieces of the API where we have to consider an RAII approach versus a higher-order function (HOF) approach. So that leads to a couple more alternatives:

- **RAII for `compute_scoped` and `execute_scoped`**: These operations need to be able to guarantee that the underlying `HandleScope` destructor runs at the end, and RAII destructors aren't guaranteed to run. Whenever you need a guarantee that a piece of code runs after a computation, you have to use a HOF. (This was the soundness hole discovered in the experimental RAII `thread::scoped` Rust API, and which led to the [scoped_threadpool](https://crates.io/crates/scoped_threadpool) crate, which uses a HOF instead.)
- **Stick with HOF for locking**: Unlocking the VM isn't a safety concern, so having an RAII API seems nice and flexible. But it might still be good to offer a convenient shorthand API that borrows a single value with a callback.

A different question, relating to consistency of API return types:

- **Coercion in `CallContext::argument_opt()`**: This method could be generic and abstract away the type coercion just like `CallContext::argument()` does. But the point is to discover if the argument is present, and it would 

Finally, a convenience I abandoned:

- **Can `CallContext` implement `std::ops::Index`?** As far as I can tell, there's not a straightforward way to make this work, since `Index::index()` returns a `&Self::Output` reference.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should there be any convenience APIs for common cases of `CallContext` argument extraction, like using `JsValue` as the return type, and converting to `JsString` via `to_js_string()`?
- Is the type of `CallContext::callee` wrong? In particular, should it have static knowledge of the `this`-type?
