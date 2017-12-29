- Feature Name: array_buffer_views
- Start Date: 2017-11-07
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes changing `JsArrayBuffer`'s implementation of [vm::Lock](https://api.neon-bindings.com/neon/vm/trait.lock) to expose a zero-cost `ArrayBufferData` type, which gives access to the underlying buffer data with all the view types of JavaScript's [typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Typed_arrays).

# Motivation
[motivation]: #motivation

The `JsArrayBuffer` API currently gives direct access to a `CMutSlice<u8>`, but if you want to operate on the data at different primitive types, you have to use unsafe code to transmute the slice. Any time Neon users are tempted to use unsafe code we should make sure there are easier safe APIs available for them to use instead.

Moreover, the use of `CMutSlice` instead of Rust's built-in slice types turns out to have been unnecessary. Giving Rust programs direct access to native Rust slices means Neon programs can make use of any abstractions that operate on Rust slices.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Rust type for JavaScript [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)s is [`JsArrayBuffer`](https://api.neon-bindings.com/neon/js/binary/struct.jsarraybuffer):

```rust
let buffer: Handle<JsArrayBuffer> = x.check::<JsArrayBuffer>()?;
```

## Reading buffer data

A `JsArrayBuffer` provides access to its internal buffer via the [`Lock::grab`](https://api.neon-bindings.com/neon/vm/trait.lock#method.grab) method. By calling the `grab` method with a callback, your code is given access to an `ArrayBufferData` struct. You can use this struct to get views over the buffer with different typed formats. For example, as a `u32` slice:

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

Here, because of the type annotation on `first`, the `as_slice` method is inferred to produce an `&[u32]`.

### Custom view types

The `as_slice` and `as_mut_slice` methods produce slices of any type that implements the `ViewType` trait. By default, all of the primitive types corresponding to JavaScript typed array view types implement this trait, in addition to `u64` and `i64`. (JavaScript doesn't provide typed arrays for 64-bit integers since they aren't expressible as JavaScript primitive values.)

This also makes it possible to create custom `ViewType` implementations for custom view types, but these must be implemented with `unsafe` code.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The `ArrayBufferData` type

The primary change to `JsArrayBuffer` is in its implementation of the `Lock` trait. Instead of defining the associated type `Internals` directly as `CMutSlice<u8>`, we change it to a newly-defined `neon::js::binary::ArrayBufferData` struct type:

```rust
struct ArrayBufferData;

impl ArrayBufferData {
    fn as_slice<T: ViewType>(&self) -> &[T];
    fn as_mut_slice<T: ViewType>(&mut self) -> &mut [T];
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
impl ArrayBufferData {
    fn as_u8_slice(&self) -> &[u8];
    fn as_mut_u8_slice(&mut self) -> &mut [u8];

    fn as_i8_slice(&self) -> &[i8];
    fn as_mut_i8_slice(&mut self) -> &mut [i8];

    fn as_u16_slice(&self) -> &[u16];
    fn as_mut_u16_slice(&mut self) -> &mut [u16];

    fn as_i16_slice(&self) -> &[i16];
    fn as_mut_i16_slice(&mut self) -> &mut [i16];

    fn as_u32_slice(&self) -> &[u32];
    fn as_mut_u32_slice(&mut self) -> &mut [u32];

    fn as_i32_slice(&self) -> &[i32];
    fn as_mut_i32_slice(&mut self) -> &mut [i32];

    fn as_u64_slice(&self) -> &[u64];
    fn as_mut_u64_slice(&mut self) -> &mut [u64];

    fn as_i64_slice(&self) -> &[i64];
    fn as_mut_i64_slice(&mut self) -> &mut [i64];

    fn as_f32_slice(&self) -> &[f32];
    fn as_mut_f32_slice(&mut self) -> &mut [f32];

    fn as_f64_slice(&self) -> &[f64];
    fn as_mut_f64_slice(&mut self) -> &mut [f64];
}
```

# Drawbacks
[drawbacks]: #drawbacks

It's annoying that multiple mutable views have to be in separate scopes. This might be something that would be relaxed by non-lexical lifetimes, which is coming pretty soon to Rust. And generally I don't know if there's any way to do better; it would violate the core semantics of Rust references to allow aliasing.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative approach would be to use JavaScript typed array views to determine the type of the Rust slice. But this would require allocating different JS objects and passing values back and forth between Rust and JS. All Rust really needs is the backing store; by contrast, the typed array view objects only exist because they are a dynamic language's approach to aliasing views over a backing store. So this design prefers to focus on the underlying `ArrayBuffer` and let Rust operate on its data through different Rust view types.

By providing direct access to the buffer at various Rust slice types, we make the endianness of operations on typed arrays non-portable. An alternative approach would be to use a wrapper type that either guarantees little-endianness (translating with a slower path on big-endian systems), or requires programs to be explicit about which they are using. However, JavaScript has also made the same decision to use the native system endianness, and in practice, little-endianness seems to have taken over the world. So this should make Neon code no less portable than pure JavaScript code that operates on typed arrays, and the increasingly rare big-endian-only systems will simply suffer from occasional bugs. In short: [JavaScript has bet on the death of big-endian systems](http://calculist.org/blog/2012/04/25/the-little-endian-web/), and we are drafting off of their decision.

Similarly, we might also have chosen to put a protective abstraction in front of the slices to canonicalize NaN values. JavaScript engines have to do this when converting between data in the backing store and JavaScript values, but we don't have to be responsible for that. If signalling NaN values were a source of undefined behavior, we could have had a problem. Luckily, [signalling Nan is defined in Rust](https://twitter.com/gankro/status/931535748628729856) so we're safe.

We could have added more API conveniences, including splitting views and working with various typed array types. We can safely leave these considerations to future RFCs, since they don't affect the design of the core API.

# Unresolved questions
[unresolved]: #unresolved-questions

We need to determine the detailed requirements of the `ViewType` trait. This can be determined during the initial implementation work for this RFC.
