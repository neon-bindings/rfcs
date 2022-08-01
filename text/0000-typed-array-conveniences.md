- Feature Name: typed_array_conveniences
- Start Date: 2022-07-23
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

Core support for reading and writing typed arrays was added in [RFC 39](https://github.com/neon-bindings/rfcs/blob/main/text/0039-node-api-borrow.md). This RFC adds APIs for conveniently **constructing and accessing metadata from** typed arrays.

# Motivation
[motivation]: #motivation

Constructing typed arrays can be done in JavaScript or by accessing the constructors from `cx.global()`, but this requires a lot of boilerplate and is slower than calling the direct Node-API functions for doing the same.

Reading metadata such as the byte length can be performed by borrowing a typed array's contents and doing arithmetic on the slice size. But this is error-prone and not particularly discoverable.

Having a richer API for working with typed arrays also makes for a better learning experience in the API docs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This example creates a typed array of unsigned 32-bit integers with a user-specified length:

```rust
fn create_int_array(mut cx: FunctionContext) -> JsResult<JsTypedArray<u32>> {
    let len = cx.argument::<JsNumber>(0)?.value(&mut cx) as usize;
    JsTypedArray::new(&mut cx, len)
}
```

This example creates a typed array as a view over a region of an existing buffer:

```rust
// Allocate a 16-byte ArrayBuffer and a uint32 array of length 2 (i.e., 8 bytes)
// starting at byte offset 4 of the buffer:
//
//       0       4       8       12      16
//      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
// buf: | | | | | | | | | | | | | | | | |
//      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
//               ^       ^
//               |       |
//              +-------+-------+
//         arr: |       |       |
//              +-------+-------+
//               0       1       2
let buf = cx.array_buffer(16);
let arr = JsUint32Array::from_region(&mut cx, buf.region(4, 2))?;
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Convenience types

The `neon::types` module contains the following convenient type aliases:

| Rust Type           | Convenience Type   | JavaScript Type                    |
| --------------------| ------------------ | ---------------------------------- |
| `JsTypedArray<u8>`  | `JsUint8Array`     | [`Uint8Array`][Uint8Array]         |
| `JsTypedArray<i8>`  | `JsInt8Array`      | [`Int8Array`][Int8Array]           |
| `JsTypedArray<u16>` | `JsUint16Array`    | [`Uint16Array`][Uint16Array]       |
| `JsTypedArray<i16>` | `JsInt16Array`     | [`Int16Array`][Int16Array]         |
| `JsTypedArray<u32>` | `JsUint32Array`    | [`Uint32Array`][Uint32Array]       |
| `JsTypedArray<i32>` | `JsInt32Array`     | [`Int32Array`][Int32Array]         |
| `JsTypedArray<u64>` | `JsBigUint64Array` | [`BigUint64Array`][BigUint64Array] |
| `JsTypedArray<i64>` | `JsBigInt64Array`  | [`BigInt64Array`][BigInt64Array]   |
| `JsTypedArray<f32>` | `JsFloat32Array`   | [`Float32Array`][Float32Array]     |
| `JsTypedArray<f64>` | `JsFloat64Array`   | [`Float64Array`][Float64Array]     |

These can be convenient to type, and also create a discoverable place for documentation and examples in the API docs.

## Constructing typed arrays

This RFC defines a number of APIs for constructing typed arrays.

### `JsTypedArray`

The `JsTypedArray` type gets three new constructor methods for creating a new typed array.

```rust
impl<T: Binary> JsTypedArray<T> {
    // Constructs a new typed array from a length, allocating a new backing buffer.
    pub fn new<'cx, C>(
        cx: &mut C,
        len: usize,
    ) -> JsResult<'cx, Self>
    where
        C: Context<'cx>;

    // Constructs a new typed array from an existing buffer.
    pub fn from_buffer<'cx, 'b: 'cx, C>(
        cx: &mut C,
        buffer: Handle<'b, JsArrayBuffer>,
    ) -> JsResult<'cx, Self>
    where
        C: Context<'cx>;

    // Constructs a new typed array from a region of an existing buffer.
    pub fn from_region<'c, 'r, C>(
        cx: &mut C,
        region: Region<'r, T>,
    ) -> JsResult<'c, Self>
    where
        C: Context<'c>;
}
```

### `JsArrayBuffer`

The `JsArrayBuffer` type gets a new method for constructing a `Region` struct, which describes a region of a buffer (see below).

```rust
impl Handle<'cx, JsArrayBuffer> {
    pub fn region<T: Binary>(&self, offset: usize, len: usize) -> Region<'cx, T>;
}
```

Since the above method only shows up in the documentation for `Handle`, the `JsArrayBuffer` type also gets a static method that behaves identically:

```rust
impl JsArrayBuffer {
    pub fn region<'cx, T: Binary>(buffer: &Handle<'cx, JsArrayBuffer>, offset: usize, len: usize) -> Region<'cx, T>;
}
```

### Helper types

#### `neon::types::buffer::Binary`

The constructor functions require passing a type tag to Node-API functions, so this RFC defines a sealed `Binary` trait to define on all the primitive element types of typed arrays (`i8`, `u8`, `i16`, `u16`, etc) in order to define the type tag as an associated constant. This is exposed for documentation and to allow userland generic helper functions that work on all typed array types.

```rust
trait Binary: Sealed + Copy + Debug {
    const TYPE_TAG: /* hidden */;
}
```

#### `neon::types::buffer::Region`

The `Region` struct defines a region of an underlying buffer. For performance, it's not actually checked to be a valid region until actually constructing a typed array object. (Otherwise, every method on the type would have to re-validate every time it's called, because of the possibility of the buffer getting detached.)

```rust
#[derive(Clone, Copy)]
struct Region<'cx, T: Binary> { /* hidden */ }

impl<'cx, T: Binary> Region<'cx, T> {
    // Constructs a new typed array for this region.
    pub fn to_typed_array<'c, C>(&self, cx: &mut C) -> JsResult<'c, JsTypedArray<T>>
    where
        C: Context<'c>;
}
```

## Convenient queries

This RFC also defines a number of methods that query typed arrays for metadata.

### `JsTypedArray`

```rust
impl<T: Binary> JsTypedArray<T> {
    // Returns the region of the backing buffer for this typed array.
    pub fn region<'cx, C>(
        &self,
        cx: &mut C,
    ) -> Region<'cx, T>
    where
        C: Context<'cx>;

    // Returns the backing buffer.
    pub fn buffer<'cx, C>(
        &self,
        cx: &mut C,
    ) -> Handle<'cx, JsArrayBuffer>
    where
        C: Context<'cx>;

    // Returns the byte offset into the backing buffer.
    pub fn offset<'cx, C>(
        &self,
        cx: &mut C,
    ) -> usize
    where
        C: Context<'cx>;

    // Returns the number of elements in the typed array.
    pub fn len<'cx, C>(
        &self,
        cx: &mut C,
    ) -> usize
    where
        C: Context<'cx>;
}
```

### `TypedArray`

This RFC adds a convenience method to the `TypedArray` trait:

```rust
pub trait TypedArray: Sealed {
    /* as before... */

    // Returns the size of this typed array in bytes.
    fn size<'cx, C>(&self, cx: &mut C) -> usize
    where
        C: Context<'cx>;
}
```

### `Region`

The `Region` struct also defines a number of convenience methods and combinators:

```rust
impl<'cx, T: Binary> Region<'cx, T> {
    // Returns the underlying ArrayBuffer for this region.
    pub fn buffer(self) -> Handle<'cx, JsArrayBuffer>;

    // Returns the byte offset of this region from the start of the ArrayBuffer.
    pub fn offset(self) -> usize;

    // Returns the number of elements of type T that fit in this region.
    pub fn len(self) -> usize;

    // Returns the size of this region in bytes.
    pub fn size(self) -> usize;
}
```


# Drawbacks
[drawbacks]: #drawbacks

There aren't any real drawbacks to exposing this functionality, since it's provided by Node-API. There are, however, a number of tradeoffs in the particulars of the design.

The main drawbacks of this RFC's proposed design are:

## Possible confusion between `len()` and `size()`

In JavaScript, these APIs are called `length` and `byteLength`, respectively, which helps disambiguate them and makes their units a bit more self-evident. Using `len` and `size` could lead to confusion between the two methods, and make their units less obvious. But `len()` and `size()` are idiomatic in Rust (see e.g. [`Vec::len()`][std::vec::Vec::len], [`str::len()`][std::primitive::str::len], [`Layout::size()`][std::alloc::Layout::size], [`Metadata::size()`][std::fs::Metadata::size]), so learning the distinction is useful for Rust developers anyway, and the API documentation should make the units explicit.

## Inconsistency with JavaScript `length`, `byteLength`, and `byteOffset`

Deviating from the JavaScript naming conventions of `length`, `byteLength`, and `byteOffset` could make these APIs less obvious or guessable for JavaScript developers. But it's better for Rust code to use Rust idioms, and the API docs should link to the corresponding JavaScript APIs to help make the connection more explicit.

## Unfortunate docs placement of `region()`

In order to make the lifetimes work out, `region()` is a method of `Handle<JsArrayBuffer>` instead of `JsArrayBuffer`. This has the unfortunate effect of placing the method in the API docs for [`neon::handle::Handle`][neon::handle::Handle] instead of [`neon::types::JsArrayBuffer`][neon::types::JsArrayBuffer], which makes them less discoverable. The other docs should make an effort to mention the method prominently, use it in examples, and link to its docs in order to mitigate this issue.

We could eliminate the underlying buffer from the `Region` data structure. Conversion to a typed array would require passing the buffer in as an additional parameter, which ends up more verbose and redundant:
```rust
buffer.region(4, 2).to_typed_array(&mut cx, buffer)
```

We could make `region()` a `&self` method of `JsArrayBuffer` and take a context reference in order to make the lifetime explicit, similar to [`JsFunction::call_with`][neon::types::JsFunction::call_with]. But not only would this be more verbose, it risks users stumbling into borrow errors due to the need to borrow the context (unlike `CallOptions`, which is generally just used as a temporary builder, a `Region` is likely to be passed around for larger sections of code).

We opted instead for the more generic and convenient API, with a static method on `JsArrayBuffer` just for the sake of documentation.

We should also reach out to the rustdoc maintainers to see if they might consider future functionality for allowing smart pointer types like `Handle` to move or copy their method docs onto the pages of their target types.

## Cost of multiple FFI calls

The `.buffer()`, `.offset()`, `.len()`, and `.size()` methods all make FFI calls to the same underlying Node-API function ([`napi_get_typedarray_info`][napi_get_typedarray_info]), which incurs unnecessary FFI overhead. Unfortunately, the semantics of JavaScript buffer detachment means that `.offset()`, `.len()`, and `.size()` can all change their value over time, which means it's not safe to cache the results of this FFI call.

The `.region()` method mitigates this issue by allowing performance-conscious Neon users to incur a single FFI call to retrieve all these values at once.

# Rationale and alternatives
[alternatives]: #alternatives

## Conversion via `From`/`Into`

Making use of the standard conversion APIs of Rust to convert between buffers and typed arrays, or between regions and typed arrays, isn't really an option since all conversions need a context.

## Range syntax

It's appealing to imagine using Rust's [range syntax][std::ops::Range] for regions, and even to use [index overloading][std::ops::Index] to enable syntax like:

```rust
JsUint32Array::from_range(&mut cx, buffer[4..12])
```

This runs into two problems:

1. **Impedance mismatch:** The JavaScript range APIs all take a byte offset and a typed length, whereas the Rust range syntax takes a start offset and end offset.
2. **`Index` incompatibility:** The signature of [`Index::index()`][std::ops::Index::index] returns a reference, so it's not allowed to allocate a custom struct. This makes it [effectively incompatible](https://users.rust-lang.org/t/how-to-implement-index-range-usize-for-a-custom-slice-type-with-additional-data/66201) with custom slice-like types such as `Region`.

## Untyped regions

It might seem nonobvious why `Region` is typed. An untyped alternative would lose track of the meaning of the `len()` value. Another alternative would be to use byte length instead, but this would be less consistent with the JavaScript typed array APIs.

It's also possible to leverage type inference to avoid having to actually state the type explicitly:
```rust
let arr = JsUint32Array::from_region(&mut cx, buf.region(4, 2))?;
```

## Flat constructor for `from_region`

This RFC's design makes `from_region` and `region` symmetric signatures with `Region` as an input type vs. return type, respectively. An alternative approach would be to make `from_region` take the `offset` and `len` as immediate arguments of `from_region`, and have `region` return a tuple. However, this would risk being more error-prone: users would have to remember what the two different `usize` elements of the tuple represent, and it would be easy to confuse the `len` integer for a byte length. It would also be untyped (see above).

## Builder pattern for `Region`

Instead of `buffer.region(offset, len)`, we could expose builder methods for constructing regions, like `buffer.offset(offset).len(len)`. This would have the benefit of making the semantics of the integers more unambiguous, but doesn't seem worth the extra verbosity.

## Alternative names for `Region`

One option would be to use "slice" terminology, since a `Region` is similar to a Rust slice. But a slice is a particular datatype in Rust, whereas this is a custom type. Also, a slice is a live view, and a region is more like a _description_ of a slice but it's not a live view until it's converted to a typed array.

Another option would be "range" terminology, but again, a [range][std::ops::Range] is a specific thing in Rust.

## Compositional regions

We could add a combinator to `Region` that produces sub-regions, but this would require validation to avoid allowing nonsensical cases like misaligned regions or region overruns. This validation conflicts with the design approach of this RFC, which defers validation to the FFI call of constructing a typed array from a region. So we stick with a flat type that serves only as an intermediate data structure for the conversion between buffers and typed arrays. This is also consistent with the architecture of the JavaScript typed array design, which is also flat (for example, it does not offer views-of-views).


# Unresolved questions
[unresolved]: #unresolved-questions

None.

[std::ops::Index]: https://doc.rust-lang.org/stable/std/ops/trait.Index.html
[std::ops::Range]: https://doc.rust-lang.org/stable/std/ops/struct.Range.html
[std::ops::Index::index]: https://doc.rust-lang.org/stable/std/ops/trait.Index.html#tymethod.index
[std::vec::Vec::len]: https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.len
[std::primitive::str::len]: https://doc.rust-lang.org/stable/std/primitive.str.html#method.len
[std::alloc::Layout::size]: https://doc.rust-lang.org/stable/std/alloc/struct.Layout.html#method.size
[std::fs::Metadata::size]: https://doc.rust-lang.org/stable/std/fs/struct.Metadata.html#method.size
[neon::handle::Handle]: https://docs.rs/neon/latest/neon/handle/struct.Handle.html
[neon::types::JsArrayBuffer]: https://docs.rs/neon/latest/neon/types/struct.JsArrayBuffer.html
[neon::types::JsFunction::call_with]: https://docs.rs/neon/latest/neon/types/struct.JsFunction.html#method.call_with
[napi_get_typedarray_info]: https://nodejs.org/api/n-api.html#napi_get_typedarray_info
[Uint8Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array
[Int8Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array
[Uint16Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint16Array
[Int16Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int16Array
[Uint32Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array
[Int32Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int32Array
[BigUint64Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigUint64Array
[BigInt64Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt64Array
[Float32Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array
[Float64Array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float64Array
