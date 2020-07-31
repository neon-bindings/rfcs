- Feature Name: `JsCell`
- Start Date: 2020-07-28
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

`JsCell` is a smart pointer with dynamically checked borrowing rules to data created in Rust and managed by the V8 garbage collector. `JsCell` are a less feature rich, but more ergonomic form of neon [classes](https://docs.rs/neon/0.4.0/neon/macro.declare_types.html).

# Motivation
[motivation]: #motivation

Classes require a substantial amount of boilerplate and place a large amount of code inside of a macro. While feature rich, classes can be unergonomic for general use. In many cases, users only need a way to safely pass native references across the FFI boundary.

The idea was first suggested as [`JsBox`in 2017](https://github.com/neon-bindings/rfcs/issues/6#issuecomment-353403121). There is also evidence that multiple users of Neon only use classes as a thin wrapper, instead relying on glue code in javascript to create classes by composing methods.

Example of boilerplate required to use a Neon Class as a thin wrapper:

```rust
pub struct Person {
	name: String,
}

impl Person {
	fn new(name: impl ToString) -> Self {
		Person {
			name: name.to_string(),
		}
	}

	fn greet(&self) -> String {
		format!("Hello, {}!", self.name)
	}
}

fn personGreet(mut cx: FunctionContext) -> JsResult<JsString> {
	let person = cx.this().downcast::<JsPerson>().or_throw(&mut cx)?;
	let greeting = {
		let lock = cx.lock();
		let person = person.borrow(&lock);

		person.greet()
	};

	Ok(cx.string(greeting))
}

declare_types! {
	pub class JsPerson for Person {
		init(mut cx) {
			let name = cx.argument::<JsString>(0)?.value();

			Ok(Person::new(name))
		}
	}
}
```

```js
const { Person, ...addon } = require('./native');

class Person {
    constructor(...args) {
        this.person = new Person(...args);
    }

    greet() {
        return addon.personGreet.apply(this.person, arguments);
    }
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When developing a Neon module, it may be necessary to create a Rust struct and reference it later. For example, a database connection pool that lives as long as the application. It is inefficient to create a new connection on every method call, instead a pool of connections is created and future calls request to borrow a connection.

Consider the following Rust code using a pool.

```rust
let pool = Pool::new();

for _ in 0..4 {
    let pool = pool.clone();

    std::thread::spawn(move || {
        let username = pool.get_by_id(id);

        println!("{}", username);
    });
}
```

Now consider the following Neon code that would like to use a pool.

```rust
fn create_pool(mut cx: FunctionContext) -> JsResult<JsValue> {
    let pool = Pool::new();

    // How do we return a `Pool`?
}

fn get_user(mut cx: FunctionContext) -> JsResult<JsString> {
    let id = cx.argument::<JsNumber>(0)?.value();
    let pool = { /* How do we get a pool? */ };

    let username = pool.get_by_id(id);

    Ok(cx.string(username))
}
```

The instance of `Pool` can be returned to Javascript using a `JsCell`. The `Pool` won't be dropped until the `JsCell` is garbage collected.

```rust
fn create_pool(mut cx: FunctionContext) -> JsResult<JsCell<Pool>> {
    let pool = Pool::new();

    cx.cell(pool)
}
```

The `Pool` instance can later be downcast and borrowed.

```rust
fn get_user(mut cx: FunctionContext) -> JsResult<JsString> {
    let pool_box = cx.argument::<JsCell<Pool>>(0)?;
    let id = cx.argument::<JsNumber>(1)?.value();

    let pool = pool_box.borrow(&mut cx)?;

    let username = pool.get_by_id(id)

    Ok(cx.string(username))
}
```

Data may also be borrowed mutably.

```rust
fn set_pool_size(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let size = cx.argument::<JsNumber>(1)?.value() as u32;
    let mut pool = cx.argument::<JsPool<Pool>>(0)?
        .borrow_mut(&mut cx);

    pool.set_size(size);

    Ok(cx.undefined())
}
```

Borrows are dynamically with a [`std::cell::RefCell`](https://doc.rust-lang.org/std/cell/struct.RefCell.html). The borrow may panic if the value is already borrowed. Instead, borrow failures may be handled explicitly:

```rust
fn set_pool_size(mut cx: FunctionContext) -> JsResult<JsNumber> {
    let size = cx.argument::<JsNumber>(1)?.value() as u32;
    let pool = cx.argument::<JsCell<Pool>>(0)?.borrow(&mut cx);

    cx.argument::<JsCell<Pool>>(0)?
        // Borrow fails because `Pool` is already borrowed immutably
        .try_borrow_mut(&mut cx)
        .or_throw(&mut cx)?
        .set_size(size);

    Ok(cx.number(pool))
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `neon::types::JsCell`

```rust
use std::cell::{BorrowError, BorrowMutError, Ref, RefCell, RefMut};

impl JsValue for JsCell {}

impl Managed for JsCell {}

impl This for JsCell {}

impl ValueInternal for JsCell {}

impl<T> JsCell where T: Send + 'static {
    /// This should only fail if the VM is in a throwing state or is
    /// in the process of stopping.
    pub fn new<'a, C: Context<'a>>(
        cx: &mut C,
        v: T,
    ) -> NeonResult<JsCell>;

    /// May panic if already mutably borrowed
    pub fn borrow<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    ) -> Ref<'a, T>;

    pub fn try_borrow<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    ) -> Result<Ref<'a, T>, BorrowError>;

    /// May panic if already borrowed
    pub fn borrow_mut<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    ) -> RefMut<'a, T>;

    pub fn try_borrow_mut<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    ) -> Result<RefMut<'a, T>, BorrowMutError>;

    /// The most common `RefCell` methods are provided on `JsCell` as a
    /// convenience. Other methods may can be accessed by calling `as_cell`
    pub fn as_cell<'a, C: Context<'a>>(
        &self,
        cx: &mut C,
    ) -> &RefCell<T>;
}
```

`JsCell` can be passed back and forth between Javascript and Rust like any other Js value.

#### `Send + 'static`

`JsCell` are passed across threads and therefore must be `Send`. It is *not* necessary for `JsCell` to be `Sync` because they may only be borrowed on the main thread.

### `trait Context`

A convenience method is provided on the `Context` trait for creating an external reference in a `JsCell`.

```rust
trait Context<'a> {
    pub fn cell<U: Send + 'static>(
        &mut self,
        v: U,
    ) -> Handle<'a, JsCell<U>> {
        JsCell::new(self, v)
    }
}
```

#### Implementation Notes

`JsCell` combines the properties of several N-API and Rust features to produce a powerful API for safely passing Rust structs across the FFI boundary.

##### N-API

* `napi_create_external()`. This API allocates a JavaScript value with external data attached to it.
* `napi_get_value_external()`. This API retrieves the external data pointer that was previously passed `to napi_create_external()`.
* `napi_typeof()`. Returns `napi_external` if the type is an external type.

*Note*: Passing an external created by another library, potentially even another neon library, is _undefined behavior_.

Progress is being made to add a [tagging feature](https://github.com/nodejs/node/pull/28237) to more safely unwrap externals. It should be incorporated into this design if it lands prior to acceptance.

##### Rust

```rust
// Outer box heap allocates the `RefCell` for a single width pointer
// `RefCell` provides dynamic borrow checking rules
// Inner Box converts to `Any` trait object
// `dyn Any` trait object for safe downcasting

// Outer box provides a single width pointer from the trait object
// Inner box provides the `dyn Any` trait object
// `dyn Any + 'static` trait object allows safe downcasting
// `RefCell` provides dynamic borrow checking rules
Box<Box<RefCell<T>>> as Box<Box<dyn Any + 'static>>
```

# Drawbacks
[drawbacks]: #drawbacks

* Overlaps with class API. However, the goal is to build the existing class API with this primitive and eventually deprecate it.
* Provides limited functionality and increases surface area.

# Rationale and alternatives
[alternatives]: #alternatives

### Statically checked borrowing

As an alternative to dynamically checked borrowing (`RefCell`), the lifetime of the reference could be constrained by the lifetime of a borrow to `Context`.

```rust
impl<T> JsCell where T: Send + 'static {
    pub fn borrow<'a: 'b, 'b, C: Context<'a>>(
        self,
        cx: &'b C,
    ) -> &'b T;

    pub fn borrow_mut<'a: 'b, 'b, C: Context<'a>>(
        self,
        cx: &mut 'b C,
    ) -> &'b mut T;
}
```

Borrowing rules on `Context` would enforce that only a single `JsCell` could be borrowed mutably at once. However, this approach has two significant drawbacks that impair ergonomics:

* Many `neon` methods require `&mut Context`. While a `JsCell` is borrowed, most `neon` methods could not be called. We could potentially mitigate this by performing a sweeping audit and relaxing constraints where a shared reference is acceptable.
* Multiple `JsCell`, even if they are unrelated, could not be borrowed at the same time because there is no way to statically prove they do not point to the same underlying data.

### Leverage existing borrow API

An existing [`neon::borrow`](https://docs.rs/neon/0.4.0/neon/borrow/index.html) implementation exists. However, the current implementation has multiple soundness holes:

* `Buffer` can be mutably borrowed multiple times
* `Lock` does not prevent overlapping memory regions
* Multiple [`Lock`](https://docs.rs/neon/0.4.0/neon/context/struct.Lock.html) can be created allowing aliased mutable borrows

Additionally, the existing `Ref` and `RefMut` structs are incompatible with the `RefCell` approach outlined in this RFC.

# Unresolved questions
[unresolved]: #unresolved-questions

- `JsBox`, `JsExternal`, JsRef`, `JsRefCell` and `JsExternalRef` have also been considered for final naming.
- Should we implement this for the legacy backend? We likely do not since this is a brand new API.
- Is it acceptable to let `JsCell::new` panic since it only fails when throwing or shutting down?
