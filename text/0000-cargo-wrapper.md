- Feature Name: cargo_wrapper
- Start Date: 2017-11-01
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes structuring the design of the `neon` command-line tool as a wrapper for `cargo`, but with its behavior modified to have defaults suitable for Neon projects.

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

The purpose of the `neon` CLI tool is to act like Cargo for Neon projects: a one-stop workflow tool for generating, building, and managing Neon projects. Although it overrides the Cargo defaults to abstract over some of the flags required to properly create and build Neon projects, the user's mental model should generally be the same as if they were using Cargo.

Ideally, the `neon` tool would not exist as a separate command, but rather as customizations via a Cargo plugin. However, not all of the necessary extensibility hooks exist yet. Fortunately, the Rust community is working on these extension points, and eventually we should be able to eliminate `neon` entirely. But in the interim, this tool allows us to create an easy-to-use workflow with the tools we have.

## Modified project layout

The current Neon project layout is set up so that the top-level directory is an npm package but not a Rust project, and it _contains_ a Rust crate in the `native/` subdirectory. This means that different tools apply to different parts of the project (`npm` and `neon` to the project root; `cargo` to the `native/` subdirectory). This can also leave the user confused about which directory they should be in and which tool they should use at any given time.

On the positive side, keeping the Rust and JavaScript directories separate avoids name collisions, in particular between the `src/` directory convention of Rust projects and the `src/` directory convention of JavaScript projects that use popular transpilers like Babel or TypeScript.

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

This RFC proposes keeping the directory hierarchy similar, but adds a `Cargo.toml` manifest to the root directory. The root manifest contains a `[workspace]` directive, which makes the top-level directory into a Rust [_virtual workspace_](http://doc.crates.io/manifest.html#virtual-manifest). This means that Rust tooling can be used in the project root directory, allowing programmers to avoid having to change directories to do different actions.

 The `neon` tool _replaces_ the `cargo` executable entirely in a Neon programmer's workflow; that is, `cargo` should never be used in a Neon project. The need for the wrapper tool is a result of `cargo` not having enough extensibility hooks for all of Neon's needs, but we will engage with the Cargo team to work towards a future where we can eliminate the `neon` wrapper and Neon programmers can directly use `cargo` with a Neon plugin. Importantly, the use of a virtual workspace means that not only can Neon programmers use the `neon` command-line tool from the project root directory today, but in the future they will be able to call `cargo` from the project root as well.

To clarify the role that the `native/` directory plays, this RFC proposes renaming it in the project structure to `crate/`. This comes closer to the Rust convention of having multiple sub-crates of a workspace live in a `crates/` directory, but the singular `crate/` nudges programmers to keep the Rust internals of a Neon package small (and push additional Rust logic out into other repositories with separate crates).

The mental model for Neon programmers becomes clear and easy to learn:

- A Neon project is, to the external world, an npm package.
- A Neon project is also internally a Rust workspace.
- A Neon project contains a Rust crate in its internal implementation, residing in the `crate/` directory.
- The workflow tool for developing and deploying the JS package is `npm` (or `yarn`), with the conventional `npm install` being the only step required for a complete build.
- The workflow tool for developing the Rust workspace is `neon`, which works just like `cargo` but for Neon packages. (Eventually this will be replaced with just `cargo`.)

The resulting project layout looks like this:

```
my-neon-lib
├── lib
│   └── index.js
├── crate
│   ├── build.rs
│   ├── Cargo.toml
│   ├── index.node
│   └── src
│       └── lib.rs
├── Cargo.toml
├── package.json
└── README.md
```

## Redesigned CLI

As described above, the long-term goal of the `neon` CLI is to go away and be replaced by a pure `cargo` plugin, allowing developers to work entirely with a cargo-based workflow. When Cargo supports the necessary extension hooks to implement this functionality as a plugin, we can deprecate `neon` and move to `cargo` proper. In the meantime, `neon` should be as similar in UI and behavior to `cargo` as possible, to ensure a smooth transition. (In particular, it's important to have a plausible path to the `cargo`-based workflows being not only functionally equivalent, but _equally or more ergonomic_ to `neon`. Otherwise, developers will not be willing to make the transition.)

### `neon new`

Creates a Neon project, which is always a `--lib` crate. The `--bin` flag is an error.

#### `neon new --neon-path PATH`

A flag allowing users to specify a filesystem path to a local `neon` project directory, for linking against custom builds of Neon.

### `neon init`

Similar to `neon new` but for initializing an existing directory as a Neon project.

### `neon build`

Builds a Neon project with `cargo build`, passing in the necessary default flags, and produces `addon.node` from the resulting shared library. This command also uses some npm environment variables and the dirty state of `addon.node` to determine if the build is stale, in addition to all of the default Cargo rules for dirty checking.

### `neon clean`

Clears out build artifacts just as with `cargo clean`, but also clears the additional build caching information created by `neon build`.

### `neon check`

Just like `cargo check`, this command checks a Neon project's Rust code and all its dependencies for errors without building. Like `neon build`, it passes in the necessary default flags.

### `neon run`

As a convenience, the `neon new` project generator should generate a simple binary at `src/bin/main.rs` that just launches `node -e 'require(".")'` from the project root directory. This allows Neon programmers to quickly test out projects with a single command. This is also useful for on-boarding documentation and talks, where people seeing Neon for the first time can see an end-to-end working "hello world" with one single command.

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

A Neon project is both an npm package _and_ a Rust crate, with JS sources in `lib/` and Rust sources in `src/` (following the two ecosystems' conventions). Conceptually, you can think of a Neon project as having a Node _interface_ and a Rust _implementation_. In particular, to the users of the package, they will interact with it as a regular JS library.

Neon provides a modified version of Rust's Cargo command-line tool for creating and working with Neon packages. You can install the Neon command-line interface via:

```
$ cargo install neon-cli
```

This installs a `neon` executable on your machine, which, along with `npm` or `yarn`, will be your primary tool for working with your Neon projects. You should never need to use Cargo directly, since `neon` provides the same commands as `cargo`.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The CLI documentation for the `neon` CLI should mostly be taken from the `cargo` CLI documentation, with small modifications according to the changes listed above.

# Drawbacks
[drawbacks]: #drawbacks

Wrappers are a maintenance burden since they have to keep up with the tool they wrap. They can also lead to confusing or inconsistent error messages. We should pay close attention to error handling to try to make this manageable. This is also somewhat mitigated by explaining `neon` as a thin wrapper, so the user's mental model is "Cargo with the right settings for Neon" instead of a distinct abstraction layer. And of course eventually we want to eliminate this, assuming Cargo will provide the right extension hooks. This seems to be a pretty high priority for the Rust tools subteam so I'm cautiously optimistic.

# Rationale and alternatives
[alternatives]: #alternatives

We could wait for Cargo to get all the extension hooks necessary, but in the meantime multiple users report mental model confusion about when to use `neon` and when to use `cargo`.

We could lift the Rust crate out into the project root directory in order to make Rust tooling accessible from the project root. But this leads to a messier project structure and creates a name conflict between different ecosystems' uses of the `src/` directory convention. Luckily, Rust's virtual workspaces were invented precisely for this purpose of using Rust tooling in a root directory that is not itself a Rust crate but contains 1 or more crates in subdirectories.

We could change the name of the CLI tool to something closer to `cargo`, such as `vargo`, to make it clearer to users that the mental model is a slight modification of `cargo`. However, the name `neon` is already so natural that it would actually likely make anything else more confusing, not to mention less aesthetically pleasing.

# Unresolved questions
[unresolved]: #unresolved-questions

Where should the source code for the convenience launcher for `neon run` live? I see three options:
- In the root workspace project (drawback: requires a source directory to the root project directory)?
- In the single crate (drawback: pollutes the crate with some boilerplate and a tiny bit of library bloat)?
- In an additional crate (drawback: requires a multiple-crate `crates/` directory)?
I'm pretty confident the first option is the worst: the whole point was to avoid mixing Rust and JS in the root directory. I am leaning towards preferring the middle option, since we can abstract out most of it into a helper crate, so the resulting extra lines in the manifest would just be:
```toml
[[bin]]
name = "run"
path = "src/run.rs"

[dependencies]
neon-run = "1.0"
```
and the resulting `src/run.rs` would just be:
```rust
extern crate neon_run;

use neon_run::run;

fn main() { run(); }
```
And this means we can continue to nudge Neon programmers away from large project repos with many Rust crates. On the other hand, I haven't yet completely convinced myself that Neon monorepos are so bad, so I might also be convinced that the third option is better.

- Unit tests (see `neon test` above)
- Benchmarks (see `neon bench` above)
