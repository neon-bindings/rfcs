# Neon RFCs

A Neon RFC ("Request For Comments") is a proposal for a significant change to the Neon design or architecture.

Many changes, such as bug fixes and documentation improvements, can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes are substantial, and we ask that these be put through a bit of design process in order to build consensus amongst the Neon community.

The RFC process is intended to provide a consistent, well-considered path for features and changes to make it into the project, so that all stakeholders can feel confident about the direction of the project.

# When you need an RFC

You need an RFC if you intend to make substantial changes to Neon, the Neon API, the Neon CLI, or the RFC process itself. What constitutes a "substantial" change is evolving based on community norms, but may including the following:

  - Any change to the public API (including modules, functions, types, traits, or macros).
  - Removing features.
  - Changes to the CLI command syntax or behavior.
  - Changes to the expected directory and file layout of Neon packages.

Some changes do not require an RFC:

  - Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not change meaning."
  - Additions that strictly improve objective, numerical criteria (e.g., performance improvements, removing compiler warnings from the builds, better platform coverage, more parallelism, trapping more errors, etc).
  - Additions only likely to be _noticed_ by other developers of Neon, invisible to users of Neon.

If you submit a pull request to implement a new feature without going through the RFC process, we may politely ask you to submit an RFC first and close the pull request.

# The process

In short, getting a major feature added to Neon requires an RFC to be merged into the RFC repository. At that point the RFC is "active" and may be implemented with the goal of eventual inclusion into Neon.

The steps are:

- Fork the [RFC repository](https://github.com/neon-bindings/rfcs).
- Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive; don't assign an RFC number yet).
- Fill in the RFC. Put care in the details: RFCs that do not present convincing motivation, demonstrate understanding of the impact of the design, or are disingenuous about the drawbacks or alternatives tend to be poorly received.
- Submit a pull request. As a pull request, the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
- Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the project maintainers to get help identifying stakeholders and obstacles.
- The project maintainers will discuss the RFC pull request, as much as possible in the comment thread. Offline discussion will be summarized on the pull request comment thread.
- At some point, the project maintainers will announce a proposed conclusion and tag the RFC with a `final comment period` label, giving the community a last opportunity to discuss the proposed conclusion.
- Once an RFC enters its final comment period, if the level of discussion is relatively quiet, the maintainers will finalize the decision and either merge or reject the RFC pull request.
- On the other hand, if there is a significant new amount of discussion, the maintainers will decide to remove the `final comment period` label and allow discussion to continue.

# License

This repository is licensed under either of

- Apache License, Version 2.0, ([LICENSE-APACHE](https://github.com/neon-bindings/rfcs/blob/master/LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](https://github.com/neon-bindings/rfcs/blob/master/LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

# Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
