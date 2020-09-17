- Feature Name: native_cli
- Start Date: 2020-09-17
- RFC PR: (leave this empty)
- Neon Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes making it possible to use Neon with only standard CLI tools: `npm` for JavaScript workflows, and `cargo` for Rust workflows. This should lighten the cognitive burden of learning Neon, make it easier to learn, and generally feel like an even simpler end-to-end user experience.

# Motivation
[motivation]: #motivation

Historically, we created the `neon` command-line tool as a wrapper around the `npm` and `cargo` command-line tools to eliminate repetitive manual steps in the project creation and build steps. Excitingly, it appears there are enough extensibility points in these tools to be able to allow Neon users to use the native tooling for the usual workflows. This should have the following benefits:

- Learnability: New Neon users can use the `npm` commands they're already used to, without having to learn another CLI tool.
- Adoption: Lowering the barrier to entry should decrease a deterrence from adoption of Neon.
- Maintainability: Neon users no longer need to worry about maintaining the right version of `neon` installed globally, and can just rely on the `npm` and `cargo` tools they already have installed on their system.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Creating a new Neon project

Neon provides templates for conveniently creating various types of Neon projects, including:

- libraries;
- Node apps; and
- Electron apps.

### Libraries

To create a new Neon library, use the command:

```
npm init neon-lib my-lib
```

Use whatever name you want for your library in place of `my-lib`.

### Node apps

To create a new Node app, use the command:

```
npm init neon-app my-app
```

Use whatever name you want for your library in place of `my-app`.

### Electron apps

To create a new Electron app, use the command:

```
npm init neon-electron-app my-app
```

Use whatever name you want for your library in place of `my-app`.

## Building a Neon project

All of the standard Neon project templates are set up to allow you to do a complete build from source with the `install` script, meaning that all you have to do to build your project is the standard `npm install` (or `npm i` for short) command from the project root.

If you look at your `package.json`, you'll see that the install script delegates to Rust's `cargo` tool to build the native addon from Rust source. You can do this directly yourself by running `cargo build` (for a debug build) or `cargo build --release` (for a release build) from the project root.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Replacing `neon new`

The reason we can now move past `neon new` is that `npm` has added support for custom templates via extensibility of the `npm init` command. We have pre-reserved npm packages called:

- `create-neon-lib`
- `create-neon-app`
- `create-neon-electron-app`

The extensible design of [`npm init`](https://docs.npmjs.com/cli/init) means that these packages are automatically used as project generators for the commands:

- `npm init neon-lib <args...>`
- `npm init neon-app <args...>`
- `npm init neon-electron-app <args...>`

respectively.

## Replacing `neon build`

There are a couple of key developments that will allow us to move past `neon build`:

1. Once we move to the [N-API backend](https://github.com/neon-bindings/neon/issues/444), we no longer need to invalidate a Neon build when changing between versions of Node. This means we won't need to use the custom invalidation logic (which has proved to be fragile and a source of user frustration).
2. We discovered that we can skip the final step of copying the built DLL into a `.node` file by replacing the JS boilerplate that loads the addons with a call to the lower-level but official [`process.dlopen()`](https://nodejs.org/api/process.html#process_process_dlopen_module_filename_flags) API.

In order to make `cargo build` work from the root directory of a Neon project, we can put a `Cargo.toml` in the root directory that either defines a [Cargo workspace](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) or that shells out to running `cargo build` in the proper subdirectory. Alternatively, we might just try experimenting with flattening the directory structure so that the Cargo directory _is_ the root directory. We should experiment with prototypes to see what looks and feels best for the developer experience.

# Drawbacks
[drawbacks]: #drawbacks

Some users report liking the `neon` CLI. But we don't actually need to eliminate it, and should at least continue to support it for some time in order to avoid disrupting users. Over time, we can decide whether it's becoming less popular. But there's really no need to couple the decision of what to do with `neon` to the decision of whether to do this RFC. Deprecation and removal is a separate question.

There's some loss of simplicity in the source code from `require('addon')` to `process.dlopen(/* ..find the DLL.. */)`. We could perhaps abstract this code into a Neon library, but it may not even be necessary. It's still relatively straightforward and not really a part of the code that users touch anyway.

# Rationale and alternatives
[alternatives]: #alternatives

## Alternative: Only use `cargo`

We could build all the tooling through Cargo generators (e.g. with the [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) Cargo addon). This would double down on the idea that Neon is about using Rust in your Node projects.

However, on the outside, a Neon project _is_ a JS package, and you still need to use npm e.g. to install dependencies or to publish to the registry. So it makes sense to use `npm init` for creating a Neon project and `npm i` for building the project (including fetching and building any dependencies it may have). However, it's still useful for `cargo build` to work in the root directory for convenience, especially for cases where the user may add additional build logic to `npm install` but still want to be able to just build the Rust code by itself.

We could use an `@neon` namespace instead of `create-neon-lib` etc. However, this makes for a noisier and less convenient-to-type command-line experience:
```
npm init @neon/app my-app
```
(Note that as a _programming_ syntax, this might arguably be preferable for the visual distinction. But the command-line is meant to be quick and easy to type and remember.) Another option would be to support both as aliases for each other, but this might just overwhelm users with pointless choices.

# Unresolved questions
[unresolved]: #unresolved-questions

- What other build invalidation logic, if any, do we still need to add? Can this all be done within `build.rs`? We should review the `neon build` invalidation logic and check for anything this RFC has overlooked.
- What is the best way for `cargo build` to work in the root?
  * `cargo build --all`?
  * `cargo build` by shelling out to `cargo build` in the right subdirectory?
  * Flatten the directory structure so the root directory is both a Node package and a Rust crate? (This is an attractive option, as it simplifies the directory structure e.g. in the tree view of an IDE, and generally may make the cognitive load of a Neon project feel lighter.)
- On the topic of lightening the directory layout, should we also put the `index.js` into the root directory? This might look like:
```
my-app
├── Cargo.lock
├── Cargo.toml
├── index.js
├── node_modules
├── package-lock.json
├── package.json
└── src
    └── lib.rs
```
