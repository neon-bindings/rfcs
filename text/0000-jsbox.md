- Feature Name: `JsBox`
- Start Date: 2020-07-28
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

`JsBox` is a smart pointer to data created in Rust and managed by the V8 garbage collector. `JsBox` is a basic building block for higher level APIs like neon [classes](https://docs.rs/neon/0.4.0/neon/macro.declare_types.html).

# Motivation
[motivation]: #motivation

Classes require a substantial amount of boilerplate and place a large amount of code inside of a macro. While feature rich, classes can be unergonomic for general use. In many cases, users only need a way to safely pass references across the FFI boundary.

`JsBox` was [first suggested in 2017](https://github.com/neon-bindings/rfcs/issues/6#issuecomment-353403121). There is also evidence that multiple users of Neon only use classes as a thin wrapper, instead relying on glue code in JavaScript to create classes by composing methods.

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

The instance of `Pool` can be returned to JavaScript using a `JsBox`. The `Pool` won't be dropped until the `JsBox` is garbage collected.

```rust
fn create_pool(mut cx: FunctionContext) -> JsResult<JsBox<Pool>> {
    let pool = Pool::new();

    Ok(cx.boxed(pool))
}
```

The `Pool` instance can later be borrowed:

_Note: Through `std::ops::Deref` and the magic of auto-deref, `JsBox<T>` can be treated as `&T`._

```rust
fn get_user(mut cx: FunctionContext) -> JsResult<JsString> {
    let pool = cx.argument::<JsBox<Pool>>(0)?;
    let id = cx.argument::<JsNumber>(1)?.value();

    let username = pool.get_by_id(id)

    Ok(cx.string(username))
}
```

Data may only be borrowed immutably. However, `RefCell` can be used to introduce interior mutability with dynamic borrow checking rules:

```rust
fn create_pool(mut cx: FunctionContext) -> JsResult<JsBox<RefCell<Pool>>> {
    let pool = RefCell::new(Pool::new());

    Ok(cx.boxed(pool))
}

fn set_pool_size(mut cx: FunctionContext) -> JsResult<JsUndefined> {
    let size = cx.argument::<JsNumber>(1)?.value() as u32;
    let mut pool = cx.argument::<JsBox<RefCell<Pool>>>(0)?;

    pool.borrow_mut().set_size(size);

    Ok(cx.undefined())
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### `neon::types::JsBox`

```rust
impl<T: Send + 'static> JsValue for JsBox<T> {}

impl<T: Send + 'static> Managed for JsBox<T> {}

impl<T: Send + 'static> ValueInternal for JsBox<> {}

impl<T> JsBox where T: Send + 'static {
    /// This should only fail if the VM is in a throwing state or is
    /// in the process of stopping.
    pub fn new<'a, C: Context<'a>>(
        cx: &mut C,
        v: T,
    ) -> Handle<'a, JsBox<T>>;
}
```

`JsBox` can be passed back and forth between JavaScript and Rust like any other Js value.


#### `std::ops::Deref`

`JsBox<T>` acts as a smart-pointer and can be coerced to a `&T` by auto-deref.

```rust
impl<'a, T: Send + 'static> Deref for JsBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target;
}
```

It is not possible to statically prove that two `JsBox<T>` do not alias the same pointer. For example, if a user passes the same value as two arguments to a function. Therefore, it is only safe for `JsBox<T>` to implement `Deref` and not `DerefMut`.

#### `Send + 'static`

Ownership of `JsBox` is moved to V8 and therefore the internal value must be `Send`. Requiring `Send` prevents passing values to other contexts that should remain local, for example an [`Rc`](https://doc.rust-lang.org/stable/std/rc/index.html). Most owned types are `Send` and this is a common restriction for multi-threaded applications.

It is *not* necessary for `JsBox` to be `Sync` because they may only be borrowed on the main thread. The JavaScript engine guarantees that `JsBox` can't be accessed from multiple threads concurrently.

### `trait Context`

A convenience method is provided on the `Context` trait for boxing types. The term `boxed` was chosen because `box` is a keyword.

```rust
trait Context<'a> {
    pub fn boxed<U: Send + 'static>(
        &mut self,
        v: U,
    ) -> Handle<'a, JsBox<U>> {
        JsBox::new(self, v)
    }
}
```

#### Implementation Notes

`JsBox` combines the properties of several N-API and Rust features to produce a powerful API for safely passing Rust structs across the FFI boundary.

##### N-API

* `napi_create_external()`. This API allocates a JavaScript value with external data attached to it.
* `napi_get_value_external()`. This API retrieves the external data pointer that was previously passed `to napi_create_external()`.
* `napi_typeof()`. Returns `napi_external` if the type is an external type.

##### Safety

The `JsBox` API is not entirely safe. It relies on the user only passing N-API externals created within that Neon library. Passing an external created by another library, potentially even another neon library, is _undefined behavior_.

Attempting to downcast a `JsValue` that is not an external will fail predicably. However, attempting to downcast an external created by another native library **is** undefined behavior. This is due to a limitation in n-api.

Progress is being made to add a [tagging feature](https://github.com/nodejs/node/pull/28237) to more safely unwrap externals. It would allow branding externals with a tag that uniquely identifies the module that created them. If available, it should be used in the implementation; however, it could be added later to close the safety hole.

##### Rust

Internally, `JsBox` is represented as a boxed [`Any`](https://doc.rust-lang.org/std/any/) trait object. The `Any` trait provides type tagging to enable dynamic typing.

```rust
// Outer box provides a single width pointer from the trait object
// Inner box provides the `dyn Any` trait object
// `dyn Any + 'static` trait object allows safe downcasting
Box<Box<T>> as Box<Box<dyn Any + 'static>>
```

# Drawbacks
[drawbacks]: #drawbacks

* Overlaps with class API. However, the goal is to build the existing class API with this primitive and eventually deprecate the macro.
* Provides limited functionality and increases surface area.

# Rationale and alternatives
[alternatives]: #alternatives

### Statically checked borrowing

As an alternative to only allowing shared references, the lifetime of the reference could be constrained by the lifetime of a borrow to `Context`.

```rust
impl<T> JsBox where T: Send + 'static {
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

Borrowing rules on `Context` would enforce that only a single `JsBox` could be borrowed mutably at once. However, this approach has two significant drawbacks that impair ergonomics:

* Many `neon` methods require `&mut Context`. While a `JsBox` is borrowed, most `neon` methods could not be called. We could potentially mitigate this by performing a sweeping audit and relaxing constraints where a shared reference is acceptable
* Multiple `JsBox`, even if they are unrelated, could not be borrowed at the same time because there is no way to statically prove they do not point to the same underlying data
* Impairs ergonomics on APIs that do not require mutable borrowing since `Deref` cannot be implemented

### `JsCell`

Neon could provide a `JsCell` type that is identical to `JsBox`, but also wraps the underlying `T` in a `std::cell::RefCell`. This allows the user to also mutably borrow the value.

```rust
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

This approach has several significant drawbacks:

* Increases the API substantial
* Adds an abstraction that not all users might need
* Prevents implementing `Deref`

Instead, Neon can lean on documentation to guide users towards `RefCell` if they need interior mutability.

#### Future Expansion

It may be beneficial for neon to export a `JsRefCell` type alias:

```rust
type JsRefCell<T> = JsBox<RefCell<T>>;
```

This type alias would provide a convenient place for Neon to document how to address interior mutability in a `JsBox`. It would be best to also include a `Context` method for creating it.

```
trait Context<'a> {
    fn ref_cell<T: Send>(&mut self, v: T) -> JsRefCell<T> {
        JsBox::new(RefCell::new(v))
    }
}
```

### Naming

Several other names were considered:

* `JsExternal`. Matches the name in N-API documentation (external reference); however, `cx.external(v)` might be confusing.
* `JsRef`. Also, matches N-API but, does not match Rust naming conventions.
* `JsCell`. Only appropriate for the `RefCell` approach.

Overall, `JsBox<T>` behaves very similar to a `std::boxed::Box<T>`.

### Leverage existing borrow API

An existing [`neon::borrow`](https://docs.rs/neon/0.4.0/neon/borrow/index.html) implementation exists. However, the current implementation has [multiple soundness holes](https://github.com/neon-bindings/neon/tree/kv/borrow-soundness):

* `Buffer` can be mutably borrowed multiple times
* `Lock` does not prevent overlapping memory regions
* Multiple [`Lock`](https://docs.rs/neon/0.4.0/neon/context/struct.Lock.html) can be created allowing aliased mutable borrows

# Open questions
[open]: #open-questions

- ~Should we implement this for the legacy backend? We likely will not since this is a brand new API.~
- ~Is it acceptable to let `JsBox::new` panic since it only fails when throwing or shutting down?~ Similar to other types `JsBox` will panic if the VM is is throwing.
- Implementing `std::ops::Deref` depends on type checking prior to calls to `ValueInternal::from_raw` because it cannot fail. Missing a check is not unsafe, but could result in a panic.

## `Lock` and `Borrow` APIs

Neon has an existing API (`Lock`) for "locking" the VM and allowing borrows to native data. The `Lock` API allows borrows to two types of data:

* `JsArrayBuffer`
* Rust data wrapped in a `JsClass`

When comparing the `Lock` API to `JsBox` there is an important distinction:

* Rust data (`JsBox`) can be protected by Rust's Cell invariants, so we don't need to worry about reborrows via VM re-entrancy
* C/C++ data (`JsArrayBuffer`) can't be protected by Rust's Cell invariants, so we do need to worry about reborrows via VM re-entrancy.

Since the plan is to implement the classes using `JsBox`, the `Borrow` API that requires `Lock` will only be used for `JsArrayBuffer`. We may want to consider moving it from `neon::borrow::Borrow` to `neon::binary::Borrow` to clarify the relationship. Additionally, `Lock` / `Borrow` requires the following improvements for soundness:

* Fix issue with the incorrect pointer being hashed for `JsArrayBuffer`
* Enhance the `Ledger` to support regions of memory since `JsArrayBuffer` may overlap
