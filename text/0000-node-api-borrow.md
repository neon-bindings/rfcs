- Feature Name: Node-API Binary Borrowing
- Start Date: 2021-03-31
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary

Neon should provide APIs for _safely_ and _ergonomically_ borrowing binary data. Borrowing should leverage the type system for statically checking safety as much as possible without sacrificing flexibility.

This document proposes an overhaul to the borrow APIs in Neon.

# Motivation

Ahead of the v1.0 release is the best time to make significant changes to the Neon borrowing APIs. Currently, Neon provides these in the `neon::borrow` module. However, with the introduction of [`JsBox`](https://github.com/neon-bindings/rfcs/blob/main/text/0029-jsbox.md), borrowing _only_ applies to buffers. The focused scope allows for more idiomatic wrappers and structuring of the module.

Additionally, there are several safety and correctness related issues with the current `Lock` API that will be addressed. See concerns outlined in the [`JsBox` RFC](https://github.com/neon-bindings/rfcs/blob/main/text/0029-jsbox.md#leverage-existing-borrow-api).

# Reference-level explanation

## Simplified Borrowing

Ideally, Neon would be able to provide statically checked, zero-cost borrowing of JavaScript buffers and views. Fortunately, Neon already provides a type (`Context`) that applies Rust's borrowing rules to the state of the VM.

As long as a reference is held to `Context`, JavaScript cannot execute to invalidate or change a buffer and other native code cannot access the contents.

### Pre-requisites

There are a few methods on the `Context` trait that do not take `&mut self` and can execute JavaScript or access data. These will need a **breaking** change to take an exclusive reference to `self`:

* `Context::lock`
* `Context::borrow`
* `Context::borrow_mut`
* `Context::execute_scoped`
* `Context::compute_scoped`

### API

The borrowing design consists of a statically checked, ergonomic API and a more complex, but powerful runtime checked API.

#### Simple API

Users are able to borrow the contents of a JavaScript buffer easily and infallibly:

```rust
impl JsArrayBuffer {
    pub fn as_slice<'a: 'b, 'b, C>(&'b self, cx: &'b C) -> &'b [u8]
        where
            C: Context<'a>,
    {
        todo!()
    }

    pub fn as_mut_slice<'a: 'b, 'b, C>(&'b mut self, cx: &'b mut C) -> &'b mut [u8]
        where
            C: Context<'a>,
    {
        todo!()
    }
}
```

The lifetime of the returned slice is bound by the lifetime of context borrow. This lifetime bound ensures:

* Neon APIs may not be called that could alter the contents of the slice
* The slice may not outlive the current execution scope 

Multiple immutable buffers may be borrowed at the same time. However, only a *single* buffer may be borrowed mutably because JavaScript handles may alias each other. This is enforced by holding a *mutable* reference to context.

#### Runtime Checked API

When borrowing a buffer mutably with the simple API, the returned slice is bound by an exclusive (`mut`) borrow of context. This limitation prevents a user from borrowing multiple buffers simultaneously if one is borrowed mutably. For example, a user could not directly copy the contents of a buffer to another buffer.

The runtime checked API relaxes these constraints at the cost of runtime bookkeeping.

##### Lock

Users may "lock" the VM by creating a `Lock` struct. Neon APIs which may execute JavaScript are statically prevented since the `Lock` struct holds a mutable reference to `Context`.

```rust
trait Context {
    fn lock(&mut self) -> Lock<Self>;
}

pub struct Lock<'cx, C> {
    cx: &'cx mut C,
    /* ... */
}
```

Additionally, the `Lock` maintains a ledger of currently active borrows.

##### Borrowing

Buffers which may be borrowed dynamically using `Lock` implement the `Borrow` trait.

```rust
pub trait Borrow<'env, C, T>
where
    C: Context<'env>,
{
    fn try_borrow<'b>(
        &self,
        lock: &'b Lock<'b, C>,
    ) -> Result<Ref<'b, C, T>, BorrowError>;

    fn try_borrow_mut<'b, 'cx>(
        &mut self,
        lock: &'b Lock<'b, C>,
    ) -> Result<RefMut<'b, C, T>, BorrowMutError>;
}
```

Attempting to borrow a buffer may *fail* and follows the same rules as [`std::cell::RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html).

On success, a smart pointer is returned which can be dereferenced into a slice. When dropping a smart pointer, the borrow is removed from the ledger.

```rust
pub struct Ref<'a, C, T> {
    ledger: &'a Ledger<'a, C>,
    data: &'a [T],
}

pub struct RefMut<'a, C, T> {
    ledger: &'a Ledger<'a, C>,
    data: &'a mut [T],
}

pub struct BorrowError {
    _private: (),
}

pub struct BorrowMutError {
    _private: (),
}

impl Error for BorrowError {}
impl Error for BorrowMutError {}
```

The implementation of `Borrow` on `JsArrayBuffer` would use `T: u8`:

```rust
impl<C> Borrow<C, u8> for JsArrayBuffer
where
    for<'env> C: Context<'env>,
{
    /* ... */
}
```

### `Lock` extension for `Borrow`

While the `Borrow` trait provides the required functionality, it requires that the trait be in scope. In order to make this more ergonomic for users, the methods are mirrored on `Lock`. These methods do *not* require the trait in scope.

```rust
impl<'cx, 'env, C: Context<'env>> Ledger<'cx, C> {
    fn try_borrow<'b, T>(
        &'b self,
        item: &impl Borrow<'env, C, T>,
    ) -> Result<Ref<'b, C, T>, BorrowError> {
        todo!()
    }

    fn try_borrow_mut<'b, T>(
        &self,
        item: &mut impl Borrow<'env, C, T>,
    ) -> Result<RefMut<'b, C, T>, BorrowMutError> {
        todo!()
    }
}
```

#### Typed Arrays

`TypedArray` was introduced in ECMAScript 2015 and defines a [typed _view_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray) into an `ArrayBuffer`.

While `ArrayBuffer` is always a chunk of `u8`, views may have different numeric parameters (`Int8Array`, `Uint8Array`, `Uint8ClampedArray`, `Int16Array`, `Uint16Array`, `Int32Array`, `Uint32Array`, `Float32Array`, `Float64Array`, `BigInt64Array`, `BigUint64Array`)

The specific type of a typed array is indicated by a type parameter on `JsTypeArray`.

```rust
pub struct JsTypedArray<T> {
    local: raw::Local,
    _type: PhantomData<T>,
}

impl JsTypedArray<i8> {}
impl JsTypedArray<u8> {}
impl JsTypedArray<i16> {}
impl JsTypedArray<u16> {}
impl JsTypedArray<i32> {}
impl JsTypedArray<u32> {}
impl JsTypedArray<f32> {}
impl JsTypedArray<f64> {}
impl JsTypedArray<i64> {}
impl JsTypedArray<u64> {}
```

Both `Uint8Array` and `Uint8ClampedArray` are represented with a type parameter of `u8`. "Clamped" is not represented because we cannot ergonomically enforce the writing constraint.

##### Borrow

It is possible to implement `Borrow` generically for all` JsTypedArray`.

```rust
impl<'env, C, T> Borrow<'env, C, T> for JsTypedArray<T>
where
    C: Context<'env>,
{
    /* ... */
}
```

However, since `JsTypedArray` may overlap with each other, it is important that `Lock` properly track the _region_ of memory and not only the starting pointer. `Lock` will maintain a set of pointer `Range` to track which buffers are currently borrowed.

### Module Layout

TODO

# Examples

## Summing the contents of two `JsArrayBuffer`

Multiple buffers may be borrowed at the same time.

```rust
fn sum_buffer(mut cx: FunctionContext) -> JsResult<JsNumber> {
    let a = cx.argument::<JsArrayBuffer>(0)?;
    let b = cx.argument::<JsArrayBuffer>(1)?;

    let n_a = a.as_slice(&cx).iter().fold(0.0, |y, x| y + x as f64);
    let n_b = b.as_slice(&cx).iter().fold(0.0, |y, x| y + x as f64);
    let n = n_a + n_b;

    Ok(cx.number(n))
}
```

## Copying Rust data into a JavaScript buffer

A buffer may be borrowed mutably to write from Rust data.

```rust
fn init_buf_count(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let result = cx.undefined();
    let buf = cx.argument::<JsTypedArray<u32>>(0)?;
    let slice = buf.as_mut_slice();

    for (i, n) in slice.iter_mut().enumerate() {
        *n = i as u32;
    }

    Ok(result)
}
```

## Copying one buffer into another buffer

Borrowing multiple buffers while one is mutably borrowed requires runtime checks.

```rust
fn copy_buffer(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let result = cx.undefined();
    let source = cx.argument::<JsArrayBuffer>(0)?;
    let dest = cx.argument::<JsArrayBuffer>(1)?;
    let lock = cx.lock();

    let source_buf = source.try_borrow(&lock).unwrap();
    let dest_buf = dest.try_borrow_mut(&lock).unwrap();

    (&dest[0..source_buf.len()]).copy_from_slice(&source[0..dest_buf.len()]);

    Ok(result)
}
```

# Drawbacks

* Adds many types to the public API of Neon
* Requires runtime checks for some usages
* Does not represent "clamped" in the type signature
* Cannot borrow a buffer as a different numeric type
* Rust *must* know the specific type of buffer; it can't accept any buffer as a bag of bytes

# Alternatives

### Existing

Neon already has a [Borrow API](https://docs.rs/neon/0.8.3/neon/borrow/index.html). Continuing to use this API would ease the migration path. However, it has several issues which make it awkward. Switching to the Node-API backend provides an opportunity for breaking changes.

* Soundness holes
* Only applies to binary data and not `JsBox`
* Always runtime checked

### Associated type parameter

This would afford additional flexibility like representing "clamped" in the type signature and borrowing as different numeric types. However, it would increase the cognitive load for users significantly. It's unlikely that most users will need to know if an array is clamped and this can be added in later as a runtime check. Additionally, converting types is better handled with other numeric crates.

### Only borrowing `u8`

This is the most attractive option because it simplifies the type system significantly and allows users to pass in arbitrary buffer types and handle it consistently. 

However, it fails to fully represent the richness of the view types in JavaScript and the safety of Rust slices.

# Future Expansion

## Persistent buffer handles

Buffers can be accessed from other threads as long as Rust holds a persistent reference to the buffer. The backing data of a buffer is guaranteed not to move. Neon could provide a convenient API for passing that data.

However, we would most likely mark it as `unsafe` since we could not prevent race conditions (JavaScript could modify the data).

## Creating Typed Arrays

Creating typed arrays is left to a future expansion. Technically it can be performed by grabbing the constructors from the global object, but at a performance cost. Additionally, it usually only needs to happen in two places:

1. Reading arguments
2. Returning a buffer

In both of those locations, it can be accomplished with JavaScript wrappers.

When adding typed array creation, we should also allow creating any typed array _from_ any typed array since JavaScript allows it.

## Clamped

If there is a demand for detecting if a typed array is clamped, a method for runtime checking can be added without any breaking changes.

```rust
impl JsTypedArray<u8> {
    pub fn is_clamped(&self) -> bool { 
        todo!()
    }
}
```

# Unresolved questions

* Should we also have `borrow` and `borrow_mut` that panic?
* Should we add `borrow` and `borrow_mut` aliases to the types themselves?
* Should we add a new `ResultExt` trait so you can `or_throw` the errors?
* Should creating typed views be part of this change?
