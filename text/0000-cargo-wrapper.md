- Feature Name: cargo_wrapper
- Start Date: 2017-11-01
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes changing the `neon` command-line tool to be a complete wrapper for the cargo tool. Effectively, the `neon` executable should be thought of as the same as `cargo` but configured with defaults that make sense for Neon.

Wherever standard Neon settings differ from standard Rust settings, `neon`'s defaults override the behavior of cargo's defaults. For example, `neon new` creates a standard Neon project layout instead of a standard Rust layout, and `neon build` builds a Neon project with all the right compiler flags. Otherwise, `neon` delegates all commands and flags directly to cargo itself.

```
$ neon new my-neon-lib
     Created library `my-neon-lib` project
$ cd my-neon-lib
$ neon build
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
$ neon run
hello neon!
```


# Motivation
[motivation]: #motivation

The purpose of the `neon` CLI tool is to act like Cargo for Neon projects: a one-stop workflow tool for generating, building, and managing Neon projects. While abstract over some of the flags required to properly build a Neon project.

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

This RFC proposes collapsing this space of possibilities to **eliminate confusion and cognitive overhead** associated with knowing which directory a user must be in and which tool applies where. The new layout makes a Neon project **_both_ an npm package _and_ a Rust crate**. The `neon` tool _replaces_ the `cargo` executable entirely in a Neon programmer's workflow; that is, `cargo` should never be used in a Neon project. Moreover, the `neon` tool should work regardless of what subdirectory of the project a user happens to be in (just as cargo works today).

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

## Redesigned `neon` CLI

The long-term goal of the `neon` CLI should be to go away and be replaced by a pure `cargo` plugin, allowing developers to work entirely with a cargo-based workflow. However, this relies on cargo extension hooks that do not yet exist.

### `neon new`

Creates a Neon project, which is always a `--lib` crate. The `--exe` flag is an error.

### `neon init`

Similar to `neon new` but for initializing an existing directory as a Neon project.

### `neon build`

Builds a Neon project with `cargo build`, passing in the necessary default flags, and produces `addon.node` from the resulting shared library. This command also uses some npm environment variables and the dirty state of `addon.node` to determine if the build is stale, in addition to all of the default Cargo rules for dirty checking.

### `neon clean`

Clears out build artifacts just as with `cargo clean`, but also clears the additional build caching information created by `neon build`.

### `neon check`

Just like `cargo check`, this command checks a Neon project's Rust code and all its dependencies for errors without building. Like `neon build`, it passes in the necessary default flags.

### `neon run`

As a convenience, the `neon new` should generate a simple binary at `src/bin/main.rs` that just launches `node -e 'require(".")'` from the project root directory. This allows Neon programmers to quickly test out projects with a single command. This is also useful for on-boarding documentation and talks, where people seeing Neon for the first time can see an end-to-end working "hello world" with one single command.

### `neon test`

Initially this will not be supported, but a follow-up RFC should figure out how to make unit testing work. This will need to grapple with the question of debug vs release builds, as well as the question of whether unit tests can usefully use any of the Neon library without a running Node process.

### `neon bench`

Initially this will not be supported, and `cargo bench` itself seems to be a fairly preliminary feature. However, it would be nice to see a follow-up RFC looking into ways we can simplify the process of benchmarking Neon projects.


### Directly delegated commands

The following commands delegate directly to `cargo` with no interesting modifications:

- `neon doc`
- `neon update`
- `neon search`
- `neon publish`
- `neon install`
- `neon uninstall`


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


# Drawbacks
[drawbacks]: #drawbacks


# Rationale and alternatives
[alternatives]: #alternatives


# Unresolved questions
[unresolved]: #unresolved-questions

