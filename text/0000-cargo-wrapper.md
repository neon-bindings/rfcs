- Feature Name: cargo_wrapper
- Start Date: 2017-11-01
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes replacing the `neon` command-line tool with a lightweight wrapper for the cargo tool called `vargo`. This tool can be thought of as the same as `cargo` but configured with defaults that make sense for Neon.

Wherever standard Neon settings differ from standard Rust settings, `vargo`'s defaults override the behavior of cargo's defaults. For example, `vargo new` creates a standard Neon project layout instead of a standard Rust layout, and `vargo build` builds a Neon project with all the right compiler flags. Otherwise, `vargo` delegates all commands and flags directly to cargo itself.

```
$ vargo new my-neon-lib
     Created library `my-neon-lib` project
$ cd my-neon-lib
$ vargo build
    Updating registry `https://github.com/rust-lang/crates.io-index`
 Downloading neon v0.1.20
 Downloading neon-runtime v0.1.20
 Downloading libc v0.2.33
 Downloading neon-build v0.1.20
   Compiling cslice v0.2.0
   Compiling utf8-ranges v1.0.0
   Compiling void v1.0.2
   Compiling neon-build v0.1.20
   Compiling gcc v0.3.54
   Compiling libc v0.2.33
   Compiling lazy_static v0.2.9
   Compiling regex-syntax v0.4.1
   Compiling unreachable v1.0.0
   Compiling neon v0.1.20
   Compiling my-neon-lib v0.1.0 (file:///home/dherman/Sources/my-neon-lib)
   Compiling thread_local v0.3.4
   Compiling memchr v1.0.2
   Compiling aho-corasick v0.6.3
   Compiling regex v0.2.2
   Compiling neon-runtime v0.1.20
    Finished release [optimized] target(s) in 37.4 secs
$ vargo run
hello neon!
```


# Motivation
[motivation]: #motivation

The purpose of the `vargo` CLI tool is to act like Cargo for Neon projects: a one-stop workflow tool for generating, building, and managing Neon projects. Although it overrides the Cargo defaults to abstract over some of the flags required to properly create and build Neon projects, the user's mental model should generally be the same as if they were using Cargo.

Ideally, the `vargo` tool would not exist as a separate command, but rather as customizations via a Cargo plugin. However, not all of the necessary extensibility hooks exist yet. Fortunately, the Rust community is working on these extension points, and eventually we should be able to eliminate `vargo` entirely. But in the interim, this tool allows us to create an easy-to-use workflow with the tools we have.

## Modified project layout

The current Neon project layout is set up so that the top-level directory is an npm package but not a Rust crate, and it _contains_ a Rust crate in the `native/` subdirectory. This means that different tools apply to different parts of the project (`npm` and `neon` to the project root; `cargo` to the `native/` subdirectory). This can also leave the user confused about which directory they should be in and which tool they should use at any given time.

```
my-neon-lib
├── lib
│   └── index.js
├── native
│   ├── build.rs
│   ├── Cargo.toml
│   ├── index.node
│   └── src
│       └── lib.rs
├── package.json
└── README.md
```

This RFC proposes collapsing this space of possibilities to **eliminate confusion and cognitive overhead** associated with knowing which directory a user must be in and which tool applies where. The new layout makes a Neon project **_both_ an npm package _and_ a Rust crate**. The `vargo` tool _replaces_ the `cargo` executable entirely in a Neon programmer's workflow; that is, `cargo` should never be used in a Neon project. Moreover, `vargo` should work regardless of what subdirectory of the project a user happens to be in (just as Cargo works today).

```
my-neon-lib
├── addon.node
├── build.rs
├── Cargo.toml
├── lib
│   └── index.js
├── package.json
├── README.md
└── src
    └── lib.rs
```

## Redesigned CLI: `vargo` is the new `neon`

The long-term goal of the `vargo` CLI should be to go away and be replaced by a pure `cargo` plugin, allowing developers to work entirely with a cargo-based workflow. When Cargo supports the necessary extension hooks to implement this functionality as a plugin, we can deprecate `vargo` and move to `cargo` proper.

### `vargo new`

Creates a Neon project, which is always a `--lib` crate. The `--bin` flag is an error.

### `vargo init`

Similar to `vargo new` but for initializing an existing directory as a Neon project.

### `vargo build`

Builds a Neon project with `cargo build`, passing in the necessary default flags, and produces `addon.node` from the resulting shared library. This command also uses some npm environment variables and the dirty state of `addon.node` to determine if the build is stale, in addition to all of the default Cargo rules for dirty checking.

### `vargo clean`

Clears out build artifacts just as with `cargo clean`, but also clears the additional build caching information created by `vargo build`.

### `vargo check`

Just like `cargo check`, this command checks a Neon project's Rust code and all its dependencies for errors without building. Like `neon build`, it passes in the necessary default flags.

### `vargo run`

As a convenience, the `vargo new` project generator should generate a simple binary at `src/bin/main.rs` that just launches `node -e 'require(".")'` from the project root directory. This allows Neon programmers to quickly test out projects with a single command. This is also useful for on-boarding documentation and talks, where people seeing Neon for the first time can see an end-to-end working "hello world" with one single command.

### `vargo test`

Initially this will not be supported, but a follow-up RFC should figure out how to make unit testing work. This will need to grapple with the question of debug vs release builds, as well as the question of whether unit tests can usefully use any of the Neon library without a running Node process.

### `vargo bench`

Initially this will not be supported, and `cargo bench` itself seems to be a fairly preliminary feature. However, it would be nice to see a follow-up RFC looking into ways we can simplify the process of benchmarking Neon projects.


### Directly delegated commands

The following commands delegate directly to `cargo` with no interesting modifications:

- `vargo doc`
- `vargo update`
- `vargo search`
- `vargo publish`
- `vargo install`
- `vargo uninstall`


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A Neon project is both an npm package _and_ a Rust crate, with JS sources in `lib/` and Rust sources in `src/` (following the two ecosystems' conventions). Conceptually, you can think of a Neon project as having a Node _interface_ and a Rust _implementation_. In particular, to the users of the package, they will interact with it as a regular JS library.

Neon provides a modified version of Rust's Cargo command-line tool for creating and working with Neon packages. You can install the Neon command-line interface via:

```
$ cargo install vargo
```

This installs a `vargo` executable on your machine, which, along with `npm` or `yarn`, will be your primary tool for working with your Neon projects. You should never need to use Cargo directly, since `vargo` provides the same commands as `cargo`.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The CLI documentation for the `vargo` CLI should mostly be taken from the `cargo` CLI documentation, with small modifications according to the changes listed above.

# Drawbacks
[drawbacks]: #drawbacks

Wrappers are a maintenance burden since they have to keep up with the tool they wrap. They can also lead to confusing or inconsistent error messages. We should pay close attention to error handling to try to make this manageable. This is also somewhat mitigated by explaining `vargo` as a thin wrapper, so the user's mental model is "Cargo with the right settings for Neon" instead of a distinct abstraction layer. And of course eventually we want to eliminate this, assuming Cargo will provide the right extension hooks. This seems to be a pretty high priority for the Rust tools subteam so I'm cautiously optimistic.

Some may feel the merged project layout will feel messy or confusing, as opposed to the current cordoning off of the Rust directory under `native/`. However, I think the benefit of decreased decisions and mental context outweighs the messiness, and the most important hygiene constraint--source files kept in separate subdirectories--is still met.

# Rationale and alternatives
[alternatives]: #alternatives

We could wait for Cargo to get all the extension hooks necessary, but in the meantime multiple users report mental model confusion about when to use `neon` and when to use `cargo`.

We could keep the existing project layout, but this adds mental overhead. Programmers have to move around directories to look at the project configuration, and possibly also can only use `neon` when sitting within the `native/` subdirectory. (We could make `neon` work from any subdirectory of the project root, but might be slightly more confusing.) In effect, the mental model is "this project is both an npm package and a Rust crate", which eliminates any doubt about whether Rust operations are applicable in any given context.

# Unresolved questions
[unresolved]: #unresolved-questions

- What about transpiled languages (e.g. TypeScript) that want to use `src/` for the un-transpiled source?
- Unit tests (see `vargo test` above)
- Benchmarks (see `vargo bench` above)
