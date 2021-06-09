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

The current declare_types macro has a two-part initialization with the second part being optional.

- (Required). Constructor. This creates the Rust struct that gets wrapped into the JS class.
- (Optional). Initialization. This has access to the created class and can perform actions like assigning properties. This can be thought of as using this in a JS constructor.

Possible ways to implement it:

- Another macro attribute like #[neon::class_init] for that phase
- Need to call a function and return a class instead of the data type.
```rust
create_class(Person { name, age })
```

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

From kjvalencik
- What is the name of the type? Do we generate complete new types or do we have something like `JsClass<Person>`?
- Do we implicitly create a `RefCell` or do we only allow `&self`? I think we should start simple and class methods can only take `&self`. If they take a different form of `&self` we fail to compile and if they don't take `self` at all we create a static method.
- Are there any other critical features? In my opinion we should start simple and not try to add everything.
- What happens if the class is called as a function instead of a constructor?
