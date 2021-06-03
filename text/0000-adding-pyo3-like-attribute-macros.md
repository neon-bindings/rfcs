- Feature Name: adding_pyo3_like_attribute_macros_for_classes_and_methods
- Start Date: 2021-06-02
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes adding [PyO3-like](https://pyo3.rs/v0.13.2/class.html) attribute-macros to define JavaScript classes as Rust structs. 

# Motivation
[motivation]: #motivation

At the moment of migration to `napi`, there is no idiomatic way of writing JavaScript classes in pure Rust. This proposal adds the attribute macros to help users to write classes and methods easily. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

I will describe some potential use cases here:
```rust
#[neon::class]
struct Person {
    name: string,
    age: u8,
}
#[neon::methods]
impl Person {
    fn greet(&self, cx: FunctionContext) -> JsResult<JsString> {
        Ok(cx.string(format!("Hello, my name is {}", self.name)))
    }

    #[neon::constructor]
    fn new(mut cx: FunctionContext) -> Self {
        let name = cx.argument::<JsString>(0)?.value(&mut cx);
        let age = cx.argument::<JsNumber>(1)?.value(&mut cx);
        Person { name, age }
    }
}
```

Then we can use `Person` class as the following
```JavaScript
let p = Person("Peter", 25);
p.greet(); // Hello, my name is Peter
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

This section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
