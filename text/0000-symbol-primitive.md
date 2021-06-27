- Feature Name: symbol_primitive
- Start Date: 2020-12-08
- RFC PR:
- Neon Issue: [#502](https://github.com/neon-bindings/neon/issues/502)

# Summary
[summary]: #summary

This RFC proposes an implementation for the `symbol` primitive: https://www.ecma-international.org/ecma-262/11.0/index.html#sec-ecmascript-language-types-symbol-type

# Motivation
[motivation]: #motivation

Currently, the `Symbol` function and primitive object wrapper API is available [using a workaround](https://github.com/neon-bindings/neon/issues/502#issuecomment-602568598) with `cx.global()`. However, typechecking a symbol passed in as an argument is not. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Constructing a `JsSymbol`:
```rust
// symbol with a JsString Handle description
let description = cx.string("foo");
let symbol = JsSymbol::with_description(&mut cx, description);

// symbol without a description
let symbol = JsSymbol::new(&mut cx);

// convenience construction method on Context for the common case with a string.
let symbol = cx.symbol("foo");
```

As with other primitives, we can type check it:
```rust
fn is_symbol(mut cx: FunctionContext) -> JsResult<JsBoolean> {
    let arg = cx.argument(0)?;
    let result = arg.is_a::<JsSymbol,_>(&mut cx);
    Ok(cx.boolean(result)) 
}
```

Unlike many other primitives, `JsSymbol` doesn't have a `value` method to convert to a Rust type. However, it has a `description` method that returns the underlying `Symbol.prototype.description` instance property:
```rust
let description_handle = cx.string("foo");
let symbol = JsSymbol::with_description(&mut cx, description_handle);
let description_string = symbol.description(&mut cx).unwrap().value(&mut cx);
assert_eq!(description_string, "foo".to_owned());

let symbol_without_description = JsSymbol::new(&mut cx);
let description = symbol_without_description.description(&mut cx);
assert_eq!(description, None);
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

#### JsSymbol
- `new` would be implemented using [napi_create_symbol](https://nodejs.org/docs/latest-v14.x/api/n-api.html#n_api_napi_create_symbol)
- `description` would use the [existing implementation](https://github.com/neon-bindings/neon/blob/5f6481a53203580c3a301be02567a502de53e871/crates/neon-runtime/src/napi/object.rs#L110-L114) for napi_get_property
- `is_typeof` would leverage the [existing implementation](https://github.com/neon-bindings/neon/blob/main/crates/neon-runtime/src/napi/tag.rs#L5-L9) for typechecking, with a `napi::ValueType::Symbol`
```rust
#[repr(C)]
#[derive(Copy, Clone)]
pub struct JsSymbol(raw::Local);

impl JsSymbol {
    pub fn new<'a, C: Context<'a>>(cx: &mut C) -> Handle<'a, JsSymbol>;
    pub fn with_description<'a, C: Context<'a>>(cx: &mut C, d: Handle<'a, JsString>) -> Handle<'a, JsSymbol>;
    pub fn description<'a, C: Context<'a>>(self, cx: &mut C) -> Option<Handle<'a, JsString>>;
}

impl Value for JsSymbol {}

impl Managed for JsSymbol {
    fn to_raw(self) -> raw::Local { self.0 }

    fn from_raw(_: Env, h: raw::Local) -> Self { JsSymbol(h) }
}

impl ValueInternal for JsSymbol {
    fn name() -> String { "symbol".to_string() }

    fn is_typeof<Other: Value>(env: Env, other: Other) -> bool {
        unsafe { neon_runtime::tag::is_symbol(env.to_raw(), other.to_raw())}
    }
}
```

`Context` would be modified to add a convenience method for constructing a `JsSymbol` with a description: 
```rust
trait Context<'a>: ContextInternal<'a> {
    fn symbol<S: AsRef<str>>(&mut self, s: S) -> Handle<'a, JsSymbol> {
        let desc = self.string(s);
        JsSymbol::with_description(self, desc)
    }
}
```

# Unresolved questions
[unresolved]: #unresolved-questions

This RFC does not currently propose anything for the `Symbol` object wrapper API, which includes Well-Known Symbols and the Symbol global registry (`Symbol.for()`/`Symbol.keyFor()`). 
These are available using the workaround linked above.
