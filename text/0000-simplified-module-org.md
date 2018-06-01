- Feature Name: simplified_module_org
- Start Date: 2018-05-31
- RFC PR:
- Neon Issue:

# Summary
[summary]: #summary

A simpler organization of the modules in `neon::*`, to make it easier to know how to import from Neon.

# Motivation
[motivation]: #motivation

For a user of Neon, a granular organization of the Neon library makes it hard to remember where to import things from. (For the maintainers of Neon, it's nice to keep the library's internal module sizes small, but there's no reason we need to impose all the details of internal layout of the project on its public API.)

This RFC proposes two major changes:

1. Group most of the public Neon APIs into a smaller number of coarse-grained top-level modules.
2. Create a `neon::prelude` module that re-exports the most commonly needed APIs, so that most projects can get away with just using:
```rust
use neon::prelude::*;
```

This "prelude pattern" is very popular in the Rust ecosystem. It's especially convenient for newcomers just getting started with an API, for tutorials and presentations, and for rapid prototyping.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

For guides, this radically simplifies introductions to the Neon API, since today's large sequences of `use` declarations can be replaced with a single `use neon::prelude::*`.

Each public module can be explained like so:

- `neon::vm`: types and traits that control the JavaScript VM
- `neon::js`: types and traits providing access to JavaScript values
- `neon::thread`: types and traits for implementing multithreading computation
- `neon::meta`: metadata about the Neon library
- `neon::prelude`: the Neon "prelude," a re-exported collection of the most commonly-used Neon APIs

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section describes the organization of each public module in detail.

## `neon::vm`

Collapse the current `neon::vm`, `neon::mem`, and `neon::scope` modules into one single `neon::vm` module.

## `neon::js`

Collapse the current `neon::js`, `neon::js::binary`, `neon::js::error`, and `neon::js::class` modules into one single `neon::js` module.

Rename the `neon::js::error::Kind` type to `neon::js::ErrorKind`.

## `neon::thread`

Rename `neon::task` to `neon::thread`.

## `neon::meta`

No changes.

## `neon::prelude`

This convenience module re-exports everything except:

- the contents of `neon::meta`
- the contents of `neon::thread`
- any of the tagging traits:
  * `vm::{Managed, This}`
  * `js::{Key, BinaryDataViewType}`
- any of the types that implement `Scope` (or `Context` in [VM 2.0](https://github.com/neon-bindings/rfcs/pull/14))
- `js::ClassDescriptor`

# Critique
[critique]: #critique

I'm not a huge fan of requiring everyone to use the hoity-toity term "prelude," but it's a Rust community convention so it's best to stick with convention.

We could do nothing, but experience shows that it's pretty hard to keep track of what goes where.

We could collapse everything down to one completely flat namespace. But I think a tiny bit of coarse-grained structure helps make the overall API not feel so huge.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
