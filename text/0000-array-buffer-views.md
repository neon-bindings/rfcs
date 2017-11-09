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

The Rust type for JavaScript [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)s is [`JsArrayBuffer`](https://api.neon-bindings.com/neon/js/binary/struct.jsarraybuffer):

```rust
let buffer: Handle<JsArrayBuffer> = x.check::<JsArrayBuffer>()?;
```

## Reading buffer data

A `JsArrayBuffer` provides access to its internal buffer via the [`Lock::grab`](https://api.neon-bindings.com/neon/vm/trait.lock#method.grab) method. By calling the `grab` method with a callback, your code is given access to an `ArrayBufferView` struct. You can use this struct to get views over the buffer with different typed formats. For example, as a `u32` slice:

```rust
let first: u32 = buffer.grab(|contents| {
    contents.as_u32_slice()[0]
});
```

or an `f64` slice:

```rust
let first: f64 = buffer.grab(|contents| {
    contents.as_f64_slice()[0]
});
```

You can view the same buffer with multiple different view types:

```rust
let (first: u32, second: u32, third: f64) = buffer.grab(|contents| {
    let ints = contents.as_u32_slice();
    let floats = contents.as_f64_slice();
    (ints[0], ints[1], floats[1])
});
```

Notice how in the last example, the first two `u32` values are indexed at 32-bit offsets, and the final `f64` value is indexed at a 64-bit offset. That is, if the buffer data looks like:

```
+-----+-----+-----------+
| u32 | u32 |    f64    |
+-----+-----+-----------+
0     4     8           16
```

then the first two cells are located at byte offsets 0 and 4 respectively and `u32` offsets 0 and 1 respectively, and the third cell is located at byte offset 8 and `f64` offset 1.

## Writing to buffer data

The `as_mut_XXX_slice()` methods provide mutable access to a buffer's cata.

```rust
buffer.grab(|mut contents| {
    let mut ints = contents.as_mut_u32_slice();
    ints[0] = 17;
    ints[1] = 42;
});
```

Notice how the callback's `contents` parameter is annotated as `mut` in order to allow mutable access to the data.

Once again, here is an example of an `f64` slice:

```rust
buffer.grab(|mut contents| {
    let mut floats = contents.as_mut_f64_slice();
    floats[0] = 1.23;
    floats[1] = 3.14;
});
```

You can also extract differently-typed mutable slices, but as always with mutable references, they cannot coexist at the same time. So you have to create them in separate scopes:

```rust
buffer.grab(|mut contents| {
    {
        let mut ints = contents.as_mut_u32_slice();
        ints[0] = 17;
        ints[1] = 42;
    }
    {
        let mut floats = contents.as_mut_f64_slice();
        floats[1] = 3.14;
    }
});
```

## Generic version

If you prefer, you can instead use the generic `as_slice` and `as_mut_slice` versions of the API. These leads to more concise code, which can be nice when the type is clear from context:

```rust
let first: u32 = buffer.grab(|contents| {
    contents.as_slice()[0]
});
```

Here, because of the type annotation on `first`, the `as_slice` method is inferred to produce a `CSlice<u32>`.

### Custom view types

The `as_slice` and `as_mut_slice` methods produce slices of any type that implements the `ViewType` trait. By default, all of the primitive types corresponding to JavaScript typed array view types implement this trait, in addition to `u64` and `i64`. (JavaScript doesn't provide typed arrays for 64-bit integers since they aren't expressible as JavaScript primitive values.)

This also makes it possible to create custom `ViewType` implementations for custom view types, but these must be implemented with `unsafe` code.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `ArrayBufferView` type

The primary change to `JsArrayBuffer` is in its implementation of the `Lock` trait. Instead of defining the associated type `Internals` directly as `CMutSlice<u8>`, we change it to a newly-defined `neon::js::binary::ArrayBufferView` struct type:

```rust
struct ArrayBufferView;

impl ArrayBufferView {
    fn as_slice<T: ViewType>(&self) -> CSlice<T>;
    fn as_mut_slice<T: ViewType>(&mut self) -> CMutSlice<T>;
    fn len(&self) -> usize;
}
```

The `as_slice` method produces a read-only slice of the buffer data.

The `as_mut_slice` method produces a mutable slice of the buffer data.

The `len` method produces the byte length of the buffer.

## The `ViewType` trait

The `ViewType` trait is defined as `unsafe` since its alignment and size must be computed correctly to stay within the bounds of the buffer data.

```rust
unsafe trait ViewType;
```

### Built-in `ViewType` implementations

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

### Custom `ViewType` implementations

While it requires `unsafe` code, this design allows users to define their own `ViewTypes` for compound types such as tuples or structs.

## Convenience methods

Some contexts of use of `as_slice()` may not provide enough information to Rust's type inference algorithm to determine the `ViewType`, leading to potentially confusing errors. Especially for teaching material and for making this more accessible to new Rust programmers, this proposal also includes convenience methods that are fixed to a specific type.

```rust
impl ArrayBufferView {
    fn as_u8_slice(&self) -> CSlice<u8>;
    fn as_mut_u8_slice(&mut self) -> CMutSlice<u8>;

    fn as_i8_slice(&self) -> CSlice<i8>;
    fn as_mut_i8_slice(&mut self) -> CMutSlice<i8>;

    fn as_u16_slice(&self) -> CSlice<u16>;
    fn as_mut_u16_slice(&mut self) -> CMutSlice<u16>;

    fn as_i16_slice(&self) -> CSlice<i16>;
    fn as_mut_i16_slice(&mut self) -> CMutSlice<i16>;

    fn as_u32_slice(&self) -> CSlice<u32>;
    fn as_mut_u32_slice(&mut self) -> CMutSlice<u32>;

    fn as_i32_slice(&self) -> CSlice<i32>;
    fn as_mut_i32_slice(&mut self) -> CMutSlice<i32>;

    fn as_u64_slice(&self) -> CSlice<u64>;
    fn as_mut_u64_slice(&mut self) -> CMutSlice<u64>;

    fn as_i64_slice(&self) -> CSlice<i64>;
    fn as_mut_i64_slice(&mut self) -> CMutSlice<i64>;

    fn as_f32_slice(&self) -> CSlice<f32>;
    fn as_mut_f32_slice(&mut self) -> CMutSlice<f32>;

    fn as_f64_slice(&self) -> CSlice<f64>;
    fn as_mut_f64_slice(&mut self) -> CMutSlice<f64>;
}
```

# Drawbacks
[drawbacks]: #drawbacks

It's annoying that multiple mutable views have to be in separate scopes. This might be something that would be relaxed by non-lexical lifetimes, which is coming pretty soon to Rust. And generally I don't know if there's any way to do better; it would violate the core semantics of Rust references to allow aliasing.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative approach would be to use JavaScript typed array views to determine the type of the Rust slice. But this would require allocating different JS objects and passing values back and forth between Rust and JS. All Rust really needs is the backing store; by contrast, the typed array view objects only exist because they are a dynamic language's approach to aliasing views over a backing store. So this design prefers to focus on the underlying `ArrayBuffer` and let Rust operate on its data through different Rust view types.

# Unresolved questions
[unresolved]: #unresolved-questions

Should we include methods for splitting an `ArrayBufferView` at a particular index, allowing distinct sub-slices to be aliased to different slice types at the same time? This seems potentially useful but probably could be separated into its own proposal.

Is there value in also defining an API for working with typed arrays? It seems less crucial but maybe avoids unnecessary allocations. At any rate it seems like it could be presented orthogonally.

Should we ensure that multibyte data is [always little-endian](http://calculist.org/blog/2012/04/25/the-little-endian-web/), regardless of the architecture? I think the answer is probably yes for maximum portability. Maybe we could offer a less convenient variant that is platform-specific, like `struct u32_sys_endian(u32)` for the uncommon case of non-portable system-endianness?

Are there any security or portability issues around encodings of NaN or subnormals?

What are the detailed requirements of the `ViewType` trait?

We should also be able to consume and produce typed array objects, even though this API focuses on the `ArrayBuffer` so far. That should be a lighter-weight layer on top of the one described here. This should be added to this RFC.
