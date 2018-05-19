- Feature Name: pre_builds
- Start Date: 2018-05-17
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes a strategy for making it practical and feasible for developers of Node libraries implemented with Neon to deploy pre-builds so that downstream consumers can use the library as a dependency _without_ requiring any special software---in particular, the Rust toolchain---installed on their system.

# Motivation
[motivation]: #motivation

Building a native library with Neon currently means either requiring all downstream consumers (including apps and libraries that transitively depend on Neon without necessarily knowing what it is) to have Rust installed on their system, or signing up to deploy pre-built binaries for every combination of OS and Node version that downstream consumers might ever use. Either option puts the barrier too high for using Neon for general-purpose Node libraries.

Making the `neon-runtime` crate into a dynamic library allows us to move all Node-version-specific details and system library dependencies into the runtime and take on the burden of responsibility for building the full (OS x Node) support matrix. By providing a stable runtime ABI and having the build and bootup process for a Neon library dynamically link to the right version of the runtime, **Neon projects would then only need to pre-build one binary per OS** they want to support, instead of one binary per OS and Node version.

We would also want to provide some good conventions, automation, and guides to make it easy to build and release Neon pre-builds as part of a project's CI.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There should be a guide for pre-building libraries, with as few manual steps as possible. There will probably need to be some small number of manual steps for setting up authentication, but we should try to absolutely minimize the number of hurdles users have to clear to get _something_ working.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Strategy overview:

- Build `neon-runtime` as a dylib.
- Work our way towards a stable ABI of the dylib.
- Place all non-stable or non-portable APIs, and any code that requires finicky build machine setup (e.g. bindgen, or LLVM) inside the dylib
- We prebuild a version of `neon-runtime` for every pair of supported Node version x OS.
- Neon libraries have an install hook that fetches every prebuilt `neon-runtime` for this OS.
- Neon libraries dynamically load the right DLL by querying the Node version.

# Critique
[critique]: #critique

This RFC is non-trivial, but I think it should be feasible. Probably the biggest drawback is having to keep up with Node releases and create pre-builds for all major OSes every time a new Node version comes out. But I'd rather we pay this cost once for the project instead of making every customer pay it for every Neon project they build. And hopefully we can build some automation to make it easy to generate and release new prebuilds of the runtime.

Otherwise, I can think of a few alternatives:

- **Do nothing.** Not having a good portability story for libraries is a real limit to Neon adoption.
- **Build a wasm fallback.** WebAssembly is tempting, but there are other people working on the Rust/WebAssembly story, and it feels like a fundamentally different goal from Neon. Neon is for native plugins, and provides full access to the native OS, whereas WebAssembly is sandboxed. I think for now it's best to define WebAssembly as out of scope for Neon.
- **Build a Neon build service.** This would be... hard.

# Unresolved questions
[unresolved]: #unresolved-questions

- What hosting services (GitHub releases, S3, BinTray, ...?) should we document in the guides?
- How zero-config can we get the instructions for the pre-build guides?
- What automation do we need for the runtime pre-builds?
