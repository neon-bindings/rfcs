- Feature Name: class_construct
- Start Date: 2018-03-21
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

Two shorthand APIs for abstracting the boilerplate of extracting a class's constructor and invoking it with its `::construct()` method:
  - `JsClass::construct()`: given a `JsClass` handle, extracts the constructor and invokes it; and
  - `Class::construct()`: given a static `Class` implementation, extracts the `JsClass` handle and invokes `JsClass::construct()`.

# Motivation
[motivation]: #motivation

Convenience methods for class instantiation eliminate a lot of obvious boilerplate. As a simple example, a `JsUser` class:

```rust
class JsUser for User { /* ... */ }
```

could be instantiated with:

```rust
let obj = JsUser::construct(scope, vec![JsString::new_or_throw("dherman")?])?;
```

instead of:

```rust
let cls = JsUser::class(scope)?;
let ctor = cls.constructor(scope)?;
let obj = ctor::construct(scope, vec![JsString::new_or_throw("dherman")?])?;
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Each of these methods can be straightforwardly described as a shorthand/convenience method in the API docs.

The class tutorial should use the `Class::construct()` method to make introductory material more lightweight and understandable. Actually working with the class as a first-class handle is a pretty reflective concept and shouldn't have to be part of the initial onboarding process, and extracting the constructor function as a first-class value is relatively rare compared to creating instances of the class.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `JsClass::construct`

The `JsClass::construct` inherent method has the following signature:

```rust
impl<T: Class> JsClass<T> {
    /* ... */

    fn construct<'a, 'b, S: Scope<'a>, A, AS>(&self, _: &mut S, args: AS) -> JsResult<T>
        where A: Value + 'b,
              AS: IntoIterator<Item = Handle<'b, A>>
    {
        /* ... */
    }
}
```

Its implementation does the straightforward thing: extract the constructor function and call it with the arguments.

## `Class::construct`

The `Class::construct` trait method has the following signature:

```rust
pub trait Class: Managed + Any {
    /* ... */

    fn construct<'a, 'b, S: Scope<'a>, A, AS(_: &mut S, args: AS) -> JsResult<Self>
        where A: Value + 'b,
              AS: IntoIterator<Item = Handle<'b, A>>
    {
        /* ... */
    }
}
```

Its implementation does the straightforward thing: extract the constructor function and call it with the arguments.

# Critique
[critique]: #critique

We could try to keep the API minimal but I don't see much value; this is nicely layered so it doesn't add new core semantics to the library, just straightforward conveniences.

We could instead design the API to construct an instance that takes just the raw Rust data and bypasses the constructor. I think this is an orthogonal desire and needs to consider the ES6 `super()` protocol for subclassing. I don't see any problem with such an "allocation" method coexisting with these constructor invocation methods.

# Unresolved questions
[unresolved]: #unresolved-questions

The biggest open question is whether there can/should be a way to instantiate Neon classes with the underlying Rust data and bypassing the constructor function.
