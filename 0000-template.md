- Feature Name: (fill me in with a unique ident, `my_awesome_feature`)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [LemmyNet/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [LemmyNet/lemmy#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary

One paragraph explanation of the feature.

# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation

Explain the proposal as if it was already included in Lemmy and you were teaching it to another user. That generally means:

- Explaining the feature largely in terms of examples.
- Explaining how Lemmy users should *think* about the feature, and how it should impact the way they use Lemmy. It should explain the impact as concretely as possible.
- For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

In particular explain the following, if applicable:

- Which database changes are necessary?
- How does the HTTP API change?
- How does the new feature work over federation? Are there similar features in other Fediverse platforms which should be compatible? Also see the [federation docs](https://join-lemmy.org/docs/contributors/05-federation.html).

# Drawbacks

Why should we *not* do this?

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
- Could this change be implemented in an external project instead? How much complexity does it add?

# Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other social media platforms and what experience have their community had?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other social networks, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other social networks is some motivation, it does not on its own motivate an RFC.

# Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project in your proposal.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
