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

#### `trait TypedArray`

```rust
pub trait TypedArray: private::Sealed {
    type Item;

    /// Statically checked immutable borrow of binary data.
    ///
    /// This may not be used if a mutable borrow is in scope. For the dynamically
    /// checked variant see [`TypedArray::try_borrow`].
    fn as_slice<'a: 'b, 'b, C>(&'b self, cx: &'b C) -> &'b [Self::Item]
        where
            C: Context<'a>;

    /// Statically checked mutable borrow of binary data.
    ///
    /// This may not be used if any other borrow is in scope. For the dynamically
    /// checked variant see [`TypedArray::try_borrow_mut`].
    fn as_mut_slice<'a: 'b, 'b, C>(&'b mut self, cx: &'b mut C) -> &'b mut [Self::Item]
        where
            C: Context<'a>;

    /// Dynamically checked immutable borrow of binary data, returning an error if the
    /// the borrow would overlap with a mutable borrow.
    ///
    /// The borrow lasts until [`Ref`] exits scope.
    ///
    /// This is the dynamically checked version of [`TypedArray::as_slice`].
    fn try_borrow<'a: 'b, 'b, C>(
        &self,
        lock: &'b Lock<'b, C>,
    ) -> Result<Ref<'b, Self::Item>, BorrowError>
        where
            C: Context<'a>;

    /// Dynamically checked mutable borrow of binary data, returning an error if the
    /// the borrow would overlap with an active borrow.
    ///
    /// The borrow lasts until [`RefMut`] exits scope.
    ///
    /// This is the dynamically checked version of [`TypedArray::as_mut_slice`].
    fn try_borrow_mut<'a: 'b, 'b, C>(
        &mut self,
        lock: &'b Lock<'b, C>,
    ) -> Result<RefMut<'b, Self::Item>, BorrowError>
        where
            C: Context<'a>;
}
```

All JavaScript values that may be borrowed as binary data will implement the `TypedArray` trait. The trait provides four methods which may be split into two groups:

* [Statically Checked](#statically-checked-api)
* [Dynamically Checked](#dynamically-checked-api)

_**Note**: The trait is `Sealed` to prevent external implementations which may be unsound. See the Rust API guidelines for more details on the [Sealed pattern](https://rust-lang.github.io/api-guidelines/future-proofing.html#sealed-traits-protect-against-downstream-implementations-c-sealed)._

#### Statically Checked API

Users are able to borrow the contents of a JavaScript buffer easily and infallibly with `TypedArray::as_slice` and `TypedArray::as_mut_slice`.

For example, a Neon function that copies bytes from one `Buffer` to another:

```rust
fn copy_buffer(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let source = cx.argument::<JsBuffer>(0)?;
    let mut dest = cx.argument::<JsBuffer>(1)?;

    let source = source.as_slice(&mut cx).to_vec();
    let dest = dest.as_mut_slice(&mut cx);

    dest.copy_from_slice(&source);

    Ok(cx.undefined())
}
```

The lifetime of the returned slice is bound by the lifetime of context borrow. This lifetime bound ensures:

* Neon APIs may not be called that could alter the contents of the slice
* The slice may not outlive the current execution scope 

Multiple immutable buffers may be borrowed at the same time. However, only a *single* buffer may be borrowed mutably because JavaScript handles may alias each other or their contents may overlap. This is enforced by holding a *mutable* reference to context.

The previous example cloned the data to a temporary `Vec<u8>`. This is necessary to avoid a compile error.

```rust
// error[E0499]: cannot borrow `cx` as mutable more than once at a time
fn copy_buffer(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let source = cx.argument::<JsBuffer>(0)?;
    let mut dest = cx.argument::<JsBuffer>(1)?;

    let source = source.as_slice(&mut cx);
    //                           ------- first mutable borrow occurs here
    let dest = dest.as_mut_slice(&mut cx);
    //                           ^^^^^^^ second mutable borrow occurs here     

    dest.copy_from_slice(&source);
    //                   ------- first borrow later used here

    Ok(cx.undefined())
}
```

#### Dynamically Checked API

When borrowing a buffer mutably with the [statically checked API](#statically-checked-api), the returned slice is bound by an exclusive (`mut`) borrow of context. This limitation prevents a user from borrowing multiple buffers simultaneously if one is borrowed mutably. For example, a user could not directly copy the contents of a buffer to another buffer.

The runtime checked API relaxes these constraints at the cost of runtime bookkeeping.

The [`TypedArray` trait](#trait-typedarray) provides dynamically checked APIs as `TypedArray::try_borrow` and `TypedArray::try_borrow_mut`. Each of these methods requires a reference to the VM [`Lock`](#lock).

##### Lock

Users may "lock" the VM by creating a `Lock` struct. Neon APIs which may execute JavaScript are statically prevented since the `Lock` struct holds a mutable reference to `Context`.

```rust
trait Context<'a> {
    fn lock<'b>(&'b mut self) -> Lock<Self>
        where
            'a: 'b;
}

pub struct Lock<'cx, C> {
    cx: &'cx mut C,
    ledger: RefCell<Ledger>,
}

struct Ledger {
    // Mutable borrows. Should never overlap with other borrows.
    owned: Vec<Range<*const u8>>,

    // Immutable borrows. May overlap or contain duplicates.
    shared: Vec<Range<*const u8>>,
}

impl<'a: 'cx, 'cx, C> Lock<'cx, C>
    where
        C: Context<'a>,
{
    /// Constructs a new [`Lock`] and locks the VM. See also [`Context::lock`].
    pub fn new(cx: &'cx mut C) -> Lock<'cx, C> {
        Lock {
            cx,
            ledger: Default::default(),
        }
    }
}
```

Additionally, the `Lock` maintains a ledger of currently active borrows.

_**Note**: This is a non-semver compatible reimplementation of the current [`neon::context::Lock`](https://docs.rs/neon/0.9.0/neon/context/struct.Lock.html)._

##### Borrowing

Attempting to borrow a buffer may *fail* and follows the same rules as [`std::cell::RefCell<T>`](https://doc.rust-lang.org/std/cell/struct.RefCell.html).

On success, a smart pointer is returned which can be dereferenced into a slice.

```rust
pub struct Ref<'a, T> {
    data: &'a [T],
    ledger: &'a RefCell<Ledger>,
}

pub struct RefMut<'a, T> {
    data: &'a mut [T],
    ledger: &'a RefCell<Ledger>,
}

pub struct BorrowError {
    _private: (),
}

impl Error for BorrowError {}
```

The `Ref` and `RefMut` borrow guards serve two purposes:

* Associate the lifetime of the borrow with the lifetime of the `Lock`. This ensures that references be held after the VM is unlocked.
* Proide an `impl Drop` that removes the borrow from the ledger

The implementation of `TypedArray` on `JsArrayBuffer` would use `type Item: u8`:

```rust
impl TypedArray for JsArrayBuffer {
    type Item = u8;

    /* ... */
}
```

The `BorrowError` type may also be converted to an exception using the *new* `neon::result::ResultExt` trait:

```rust
/// Extension trait for converting Rust [`Result`](std::result::Result) values
/// into [`NeonResult`](NeonResult) values by throwing JavaScript exceptions.
pub trait ResultExt<T> {
    fn or_throw<'a, C: Context<'a>>(self, cx: &mut C) -> NeonResult<T>;
}
```

The `ResultExt` trait is identical to `JsResultExt` except that it does not require the `Ok` branch to implement `Value`. We will most likely want to deprecate `JsResultExt` in the future since this trait is strictly more powerful.

#### Typed Arrays

`TypedArray` was introduced in ECMAScript 2015 and defines a [typed _view_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray) into an `ArrayBuffer`.

While `ArrayBuffer` is always a chunk of `u8`, views may have different numeric parameters (`Int8Array`, `Uint8Array`, `Uint8ClampedArray`, `Int16Array`, `Uint16Array`, `Int32Array`, `Uint32Array`, `Float32Array`, `Float64Array`, `BigInt64Array`, `BigUint64Array`)

The specific type of typed array is indicated by a type parameter on `JsTypeArray`.

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

Both `Uint8Array` and `Uint8ClampedArray` are represented with a type parameter of `u8`. "Clamped" is not represented in this design because it could not be ergonomically enforced.

##### `TypedArray`

It is possible to implement `TypedArray` generically for all` JsTypedArray`.

```rust
impl<T: Copy> TypedArray for JsTypedArray<T> {
    type Item = T;

    /* ... */
}
```

However, since `JsTypedArray` may overlap with each other, it is important that `Lock` properly track the _region_ of memory and not only the starting pointer. `Lock` will maintain a set of pointer `Range` to track which buffers are currently borrowed.

### Module Layout

```
neon
├── context
│   └── Lock
└── types
    ├── JsArrayBuffer
    ├── JsBuffer
    ├── JsTypedArray<T>
    └── buffer
        ├── TypedArray
        ├── BorrowError
        ├── Ref
        └── RefMut
```

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

### Generic type parameter in `TypedArray` trait

This would allow us to implement `TypedArray` multiple times for a single type. For example, borrowing `JsBuffer` as `u8` or `u16`. However, it comes at two significant ergonomic issues:

* Complicates the definition of the trait when used as a bound
* User *must* hint at the type they want to borrow, even when there's only a single implementation

Given these limitations, Neon will recommend using the `bytes` crate or similar to convert between representations.

### Only borrowing `u8`

This is the most attractive option because it simplifies the type system significantly and allows users to pass in arbitrary buffer types and handle it consistently. 

However, it fails to fully represent the richness of the view types in JavaScript and the safety of Rust slices.

### `TypedArrayMutError` unique type

This significantly complicates error handling where a user may have different error types (e.g., copying from immutable borrows to a mutable borrow) while not providing much value.

### `TypedArray` trait methods mirrored on `Lock`

This would avoid needing to have the trait in scope without putting the methods on every type, but it's confusing since the type is more general and doesn't result in a very easy to understand API.

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

We could also create a `JsTypedArray<Clamped>` since the trait uses an associated type.

## Removing `JsResultExt`

The new `ResultExt` provides a superset of functionality of `JsResultExt`. `JsResultExt` should be removed in favor of the more general trait.

# Unresolved questions

* ~~Should we also have `borrow` and `borrow_mut` that panic?~~ No, since this is much more difficult to reason about than on a `RefCell` where the borrows are mostly local.
* ~~Should we add `as_slice` and `as_slice_mut` aliases to the types themselves? This would mean the trait never needs to be in scope.~~ This is backwards compatible and will be left as a future extension. 
* ~~Should the `Borrow` trait have a different name? It will conflict with `std::borrow::Borrow`, so we may want to pick a name less likely to conflict.~~ Renamed to `TypedArray`
* ~~What about the module structure? Should parts be moved out of `types`?~~
