- Feature Name: module_reorg
- Start Date: 2018-05-31
- RFC PR: https://github.com/neon-bindings/rfcs/pull/20
- Neon Issue: https://github.com/neon-bindings/neon/pull/324

# Summary
[summary]: #summary

A re-organization of the modules in `neon::*`, designed to optimize:
- The learnability and understandability of the API docs;
- The convenience of importing from Neon;
- Names that reflect abstractions of the JavaScript language (e.g., values and execution contexts), instead of the JavaScript engine (e.g., memory management and VMs).

# Motivation
[motivation]: #motivation

The current organization of Neon's modules makes it both difficult to know where to find APIs and inconvenient to import them.

This RFC proposes two major changes:

1. Create a `neon::prelude` module that re-exports the most commonly-needed APIs, so that most projects can get away with just using:
```rust
use neon::prelude::*;
```
2. Reorganize the APIs into smaller but non-nested modules, each oriented around a tightly-defined concept that matches a user's intuition.

The "prelude pattern" is very popular in the Rust ecosystem. It's especially convenient for newcomers just getting started with an API, for tutorials and presentations, and for rapid prototyping. But it's perfectly appropriate for production use as well.

Organizing the APIs into smaller modules makes it easier to guess and to remember where a particular API is defined. Avoiding hierarchical nesting also makes it easier to remember, since it doesn't require remembering the entire nested path from the Neon crate root.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

For guides, this radically simplifies introductions to the Neon API, since today's large sequences of `use` declarations can be replaced with a single `use neon::prelude::*`.

Each public module can be explained like so:

- `neon::context`: Node _execution contexts_, which manage access to the JavaScript engine at various points in the Node.js runtime lifecycle
- `neon::result`: Types and traits for working with JavaScript exceptions
- `neon::borrow`: Types and traits for obtaining temporary access to the internals of JavaScript values
- `neon::handle`: Safe _handles_ to managed JavaScript memory
- `neon::types`: Representations of JavaScript's core builtin types
- `neon::object`: Traits for working with JavaScript objects
- `neon::task`: Asynchronous background _tasks_ that run in the Node thread pool
- `neon::meta`: Utilities exposing metadata about the Neon version and build
- `neon::prelude`: A convenience module that re-exports the most commonly-used Neon APIs

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section describes the organization of each public module in detail.

## `neon::context`

- `Context`, `CallContext`, `FunctionContext`, `MethodContext`, `ComputeContext`, `ExecuteContext`, `ModuleContext`, `TaskContext`
- `CallKind`
- `Lock` (renamed from `VmGuard`)

## `neon::result`

- `NeonResult` (renamed from `VmResult`), `JsResult`, `JsResultExt`, `Throw`

## `neon::borrow`

- `Borrow`, `BorrowMut`, `Ref`, `RefMut`, `LoanError`

## `neon::handle`

- `Handle`, `Managed`
- `DowncastError`, `DowncastResult`

## `neon::types`

- `JsArray`, `JsArrayBuffer`, `JsBoolean`, `JsBuffer`, `JsError`, `JsFunction`, `JsNull`, `JsNumber`, `JsObject`, `JsString`, `JsUndefined`, `JsValue`
- `JsResult`
- `Value`
- `BinaryData`, `BinaryDataViewType`
- `StringOverflow`, `StringResult`

## `neon::object`

- `Class`, `ClassDescriptor`
- `Object`, `PropertyKey`
- `This`

## `neon::task`

- `Task`

## `neon::meta`

No changes.

## `neon::prelude`

```rust
// excludes: Lock
pub use neon::context::{Context, CallContext, FunctionContext, MethodContext, ComputeContext, ExecuteContext, ModuleContext, TaskContext, CallKind};

// excludes: Throw
pub use neon::result::{NeonResult, JsResult, JsResultExt};

// excludes: Managed, DowncastError, DowncastResult
pub use neon::handle::Handle;

// excludes: Ref, RefMut, LoanError
pub use neon::borrow::{Borrow, BorrowMut};

// excludes: BinaryDataViewType, StringOverflow, StringResult
pub use neon::types::{JsArray, JsArrayBuffer, JsBoolean, JsBuffer, JsError, JsFunction, JsNull, JsNumber, JsObject, JsString, JsUndefined, JsValue, JsResult, Value, BinaryData};

// excludes: PropertyKey, This, ClassDescriptor
pub use neon::object::{Class, Object};

pub use neon::task::Task;

// excludes: neon::meta::*;
```

# Critique
[critique]: #critique

I'm not a huge fan of the hoity-toity name "prelude," but it's a Rust community convention so it's best to stick with convention.

We could do nothing, but experience shows that it's pretty hard to keep track of what goes where. And we really need to get rid of named concepts like `scope` and `vm`, which are low-level, daunting, and confusing for people who've never dug deep into programming language or engine implementation concepts.

I experimented with [a couple of alternative organization schemes](https://gist.github.com/dherman/add90b760549f15cf90b3e249a06f504) as well:
- Almost completely flat: Everything except for `thread` and `meta` goes into the Neon top-level. Unfortunately this makes the API docs really overwhelming and hard to navigate.
- Moderately layered: This attempts to divide the world into a "VM" abstraction layer and a "JS" abstraction layer. I found it very hard to explain these layers or to figure out how to sort the various definitions into one or the other layer.

At the end of the day, I think it's best explained as: Neon is offering a single abstraction layer, which roughly corresponds to the JavaScript language semantics, but with a couple of extra concepts (namely: handles and contexts) for making interaction with it safe. Within that single layer, there are a number of (interrelated) concepts, and it's helpful to group the API by those concepts.

# Unresolved questions
[unresolved]: #unresolved-questions

None.
