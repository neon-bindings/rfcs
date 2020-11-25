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

The `npm init neon` command makes it easy to start a new Neon project. You can use `npm init neon my-app` to create a new `my-app` Neon directory, or `npm init neon` to initialize an existing directory as a Neon project.

## Building a Neon project

Once you've created a standard Neon project, the repo is set up to allow you to do a complete build from source with the `install` script, meaning that all you have to do to build your project is the standard `npm install` (or `npm i` for short) command from the project root.

If you look at your `package.json`, you'll see that the install script delegates to Rust's `cargo` tool to build your module from source. You can do this directly yourself from the project root with standard Rust commands: `cargo build` for a debug build, or `cargo build --release` for a release build.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Replacing `neon new`

The reason we can now move past `neon new` is that `npm` has added support for custom templates via extensibility of the [`npm init`](https://docs.npmjs.com/cli/init) command. This RFC proposes just one single command, `npm init neon`, which would be implemented with a `create-neon` npm package.

## Replacing `neon build`

There are a couple of key developments that will allow us to move past `neon build`:

1. Once we move to the [N-API backend](https://github.com/neon-bindings/neon/issues/444), we no longer need to invalidate a Neon build when changing between versions of Node. This means we won't need to use the custom invalidation logic (which has proved to be fragile and a source of user frustration).
2. We discovered that we can skip the final step of copying the built DLL into a `.node` file by setting a `rustc` flag via Cargo [build flag instructions](https://doc.rust-lang.org/cargo/reference/build-scripts.html#rustc-cdylib-link-arg) from within `build.rs`:

```rust
let manifest_dir = env::var_os("CARGO_MANIFEST_DIR").unwrap();
let output = Path::new(&manifest_dir).join("index.node");

println!("cargo:rustc-cdylib-link-arg=-o");
println!("cargo:rustc-cdylib-link-arg={}", output.display());
```
3. Because of the ABI and API stability of N-API, none of the `electron-build-env` settings should be needed for setting custom targets or header files for Electron builds. And with [delayed loading](https://github.com/neon-bindings/neon/issues/584), we should also be able to avoid needing custom `.lib` files for Windows Electron builds as well.

## Enabling `cargo build`

We should make it a goal to make `cargo build` work as a standard way of building a Neon project from source (of course, `npm install` should work as always, too). We have a couple of options for what this might look like.

One approach is to put a `Cargo.toml` in the root directory that defines a [Cargo workspace](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html).

Another option is to try flattening the directory structure so that the Cargo directory _is_ the root directory.

Experimenting with implementations will help us get hands-on impressions of the developer experience of the various options.

## Flattening Neon project layout

Historically, `neon new` generated a project layout that assumed that the addon should always be a submodule of a pure-JS wrapper module. This incurs some cognitive overhead for understanding the directory layout, and it's not clear the wrapper structure is actually providing value in the common case.

Breaking down the design space, there are several dimensions that determine the context in which a user wants a Neon source tree:

- Is the public deliverable a Node app, an Electron app, or a library?
- Does the addon define a published package (i.e., with its own `package.json`) or a submodule (i.e., without its own `package.json`) of a containing package?
  - If it's a published package, is it nested within a monorepo or in its own separate repo?

We can simplify the combinatorics with a few observations:

1. In general, the addon directory should have the following structure:

```
├── Cargo.toml
├── README.md
├── build.rs
├── index.node
├── package.json
└── src
    └── lib.rs
```

2. Node apps and Electron apps shouldn't need to be any different with the N-API backend.

3. Nesting an addon within an existing project is likely a less common case, or at least a case where people can reasonably expected to pass additional information to `npm init neon` in a command-line flag.

```
npm init neon [name] [--workspace [root] | --submodule [root]]

  --workspace: Specify a root directory to serve as the workspace
               root of this package.
               Default: Nearest ancestor directory containing a
                        package.json file.
  --submodule: Specify a root directory to serve as the package root
               of this module. The module will not contain its own
               package.json but instead only be accessible to consumers
               as a submodule of the parent package.
               Default: Nearest ancestor directory containing a
                        package.json file.
```



# Drawbacks
[drawbacks]: #drawbacks

Some users report liking the `neon` CLI. But we don't actually need to eliminate it, and should at least continue to support it for some time in order to avoid disrupting users. Over time, we can decide whether it's becoming less popular. But there's really no need to couple the decision of what to do with `neon` to the decision of whether to do this RFC. Deprecation and removal is a separate question.

# Rationale and alternatives
[alternatives]: #alternatives

## Alternative: Only use `cargo`

We could build all the tooling through Cargo generators (e.g. with the [cargo-generate](https://github.com/ashleygwilliams/cargo-generate) Cargo addon). This would double down on the idea that Neon is about using Rust in your Node projects.

However, to consumers of a Neon project, it _is_ a JS package, and you still need to use npm e.g. to install dependencies or to publish to the registry. So it makes sense to use `npm init` for creating a Neon project and `npm i` for building the project (including fetching and building any dependencies it may have). However, it's still useful for `cargo build` to work in the root directory for convenience, especially for cases where the user may add additional build logic to `npm install` but still want to be able to just build the Rust code by itself.

We could use an `@neon` namespace instead of `create-neon-lib` etc. However, this makes for a noisier and less convenient-to-type command-line experience:
```
npm init @neon/app my-app
```
(Note that as a _programming_ syntax, this might arguably be preferable for the visual distinction. But the command-line is meant to be quick and easy to type and remember.) Another option would be to support both as aliases for each other, but this might just overwhelm users with pointless choices.

# Unresolved questions
[unresolved]: #unresolved-questions

- What other build invalidation logic, if any, do we still need to add? Can this all be done within `build.rs`? We should review the `neon build` invalidation logic and check for anything this RFC has overlooked.
- ~~What is the best way for `cargo build` to work in the root?~~
  * ~~`cargo build --all`?~~
  * ~~`cargo build` by shelling out to `cargo build` in the right subdirectory?~~
  * ~~Flatten the directory structure so the root directory is both a Node package and a Rust crate? (This is an attractive option, as it simplifies the directory structure e.g. in the tree view of an IDE, and generally may make the cognitive load of a Neon project feel lighter.)~~
- ~~On the topic of lightening the directory layout, should we also put the `index.js` into the root directory?~~
- For nested directory structures like `--workspace` and `--submodule`, will Rust-Analyzer get confused in IDEs when the `Cargo.toml` isn't in the project root? Should the default behavior also create a Cargo workspace in the repo root in order to make that work? If not default, should we add an opt-in parameter to configure a Cargo workspace for that?
