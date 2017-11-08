- Feature Name: array_buffer_views
- Start Date: 2017-11-07
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes changing `JsArrayBuffer`'s implementation of [vm::Lock](https://api.neon-bindings.com/neon/vm/trait.lock) to expose a zero-cost `ArrayBufferView` type, which gives access to the underlying buffer data with all the view types of JavaScript's [typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays).

# Motivation
[motivation]: #motivation

The `JsArrayBuffer` API currently gives direct access to a `CMutSlice<u8>`, but if you want to operate on the data at different primitive types, you have to use unsafe code to transmute the slice. Any time Neon users are tempted to use unsafe code we should make sure there are easier safe APIs available for them to use instead.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Rust type for JavaScript [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)s is [`JsArrayBuffer`](https://api.neon-bindings.com/neon/js/binary/struct.jsarraybuffer).

A `JsArrayBuffer` provides access to its internal buffer via the [`Lock::grab`](https://api.neon-bindings.com/neon/vm/trait.lock#method.grab) method. By calling the `grab` method with a callback, your code is given access to an `ArrayBufferView` struct. You can use this struct to get views over the buffer with different typed formats. For example, as a `u32` slice:

```rust
let ab = x.check::<JsArrayBuffer>()?;
ab.grab(|mut contents| {
    let mut ints = contents.as_u32_slice();
    ints[0] = 17;
    ints[1] = 42;
})
```

Or an `f64` slice:

```rust
let ab = x.check::<JsArrayBuffer>()?;
ab.grab(|mut contents| {
    let mut floats = contents.as_f64_slice();
    floats[0] = 1.23;
    floats[1] = 3.14;
})
```

You can also extract differently-typed slices in separate scopes:

```rust
let ab = x.check::<JsArrayBuffer>()?;
ab.grab(|mut contents| {
    {
        let mut ints = contents.as_u32_slice();
        ints[0] = 17;
        ints[1] = 42;
    }
    {
        let mut floats = contents.as_f64_slice();
        floats[1] = 3.14;
    }
});
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `ArrayBufferView` type

The primary change to `JsArrayBuffer` is in its implementation of the `Lock` trait. Instead of defining the associated type `Internals` directly as `CMutSlice<u8>`, we change it to a newly-defined `neon::js::binary::ArrayBufferView` struct type:

```rust
struct ArrayBufferView;

impl ArrayBufferView {
    fn as_slice<T: ViewType>(&mut self) -> CMutSlice<T>;
    fn len(&self) -> usize;
}
```

## The `ViewType` trait

The `ViewType` trait is defined as `unsafe` since its alignment and size must be computed correctly to stay within the bounds of the buffer data.

```rust
unsafe trait ViewType;
```

The types that implement `ViewType` by default are the same as JavaScript typed array element types. This proposal also adds support for 64-bit integers since, unlike in JavaScript, they provide no particular challenge for Rust.

- `u8`
- `i8`
- `u16`
- `i16`
- `u32`
- `i32`
- `u64`
- `i64`
- `f32`
- `f64`

## Convenience methods

Some contexts of use of `as_slice()` may not provide enough information to Rust's type inference algorithm to determine the `ViewType`, leading to potentially confusing errors. Especially for teaching material and for making this more accessible to new Rust programmers, this proposal also includes convenience methods that are fixed to a specific type.

```rust
impl ArrayBufferView {
    fn as_u8_slice(&mut self) -> CMutSlice<u8>;
    fn as_i8_slice(&mut self) -> CMutSlice<i8>;
    fn as_u16_slice(&mut self) -> CMutSlice<u16>;
    fn as_i16_slice(&mut self) -> CMutSlice<i16>;
    fn as_u32_slice(&mut self) -> CMutSlice<u32>;
    fn as_i32_slice(&mut self) -> CMutSlice<i32>;
    fn as_u64_slice(&mut self) -> CMutSlice<u64>;
    fn as_i64_slice(&mut self) -> CMutSlice<i64>;
    fn as_f32_slice(&mut self) -> CMutSlice<f32>;
    fn as_f64_slice(&mut self) -> CMutSlice<f64>;
}
```

# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[alternatives]: #alternatives


# Unresolved questions
[unresolved]: #unresolved-questions

Should we include methods for splitting an `ArrayBufferView` at a particular index, allowing distinct sub-slices to be aliased to different slice types at the same time? This seems potentially useful but probably could be separated into its own proposal.

Is there value in also defining an API for working with typed arrays? It seems less crucial but maybe avoids unnecessary allocations. At any rate it seems like it could be presented orthogonally.
