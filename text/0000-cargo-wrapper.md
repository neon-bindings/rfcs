- Feature Name: cargo_wrapper
- Start Date: 2017-11-01
- RFC PR: 
- Neon Issue: 

# Summary
[summary]: #summary

This RFC proposes changing the `neon` command-line tool to be a complete wrapper for `cargo` with defaults that make sense for Neon. Wherever standard Neon settings differ from standard Rust settings, `neon`'s defaults override the behavior of cargo's defaults. For example, `neon new` creates a standard Neon project layout instead of a standard Rust layout. Otherwise, `neon` delegates all commands and flags directly to cargo itself.

# Motivation
[motivation]: #motivation

The long-term goal of the `neon` CLI should be to go away and be replaced by a pure `cargo` plugin, allowing developers to work entirely with a cargo-based workflow. However, this relies on cargo extension hooks that do not yet exist.

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

