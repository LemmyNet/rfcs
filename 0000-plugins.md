- Feature Name: plugins
- Start Date: 2024-12-02
- RFC PR: [LemmyNet/rfcs#0000](https://github.com/LemmyNet/rfcs/pull/0008)
- Lemmy Issue: [LemmyNet/lemmy#0000](https://github.com/LemmyNet/lemmy/issues/3562)

# Summary

Plugins will make Lemmy more flexible, and allow implementing features which are too niche for merging into Lemmy core. It will allow Lemmy to focus on core features, while allowing outside developers to contribute new features without using Rust.

# Motivation

TODO

# Guide-level explanation

## Extism

An open source framework to develop plugins for Rust projects. It supports numerous different languages by compiling them to webassembly which gets called from Rust.

https://extism.org/

# Reference-level explanation

## Plugin Hooks

Lemmy will have to add hooks for specific actions, which can then be used by plugins. In general there are two types of hooks: before an action is written to the database, so it can be rejected or altered. And after it is written to the database, mainly for different types of notifications.

For the initial implementation, the following hooks will be available:

### Before writing to Database

These hooks can be used to reject or alter user actions based on different criteria.

- Post
    - Create local post
    - Update local post
    - Receive federated post
- Comment
    - Create local comment
    - Update local comment
    - Receive federated comment
- New vote
    
### After writing to Database

These are mainly useful to generate notifications.

- Post created or edited
- Comment created or edited
- New vote

## Host Functions

Lemmy database functions can be exposed by plugins via [host functions](https://extism.org/docs/concepts/host-functions). For the beginning the following functions will be exposed:

- Instance::read_or_create 

More functions may be added if they are needed for the initial plugins.

## Initial Plugins

These can mainly serve as proof of concept, and as examples which can be modified by plugin developers. Include integration tests to ensure that functionality works with new Lemmy changes. These plugins should be written in a variety of languages. It could make sense to put them into a separate repo.

- Block only downvotes from a specific instance
- Replace words in post or comment text (eg "cloud to butt")
- Replace Youtube URLs in post links with [Invidious](https://invidious.io/)
- Push notifications (send data of new posts to localhost url, [#2631](https://github.com/LemmyNet/lemmy/issues/2631))
- Language detection (automatically apply language tag to posts/comments, [#2870](https://github.com/LemmyNet/lemmy/issues/2870))
- Additional suggestions welcome

# Drawbacks

Plugins will likely have worse performance than native Rust.

# Rationale and alternatives

TODO

# Prior art

TODO

# Unresolved questions

What are the licensing requirements for plugins, if any? Are proprietary plugins allowed or do they need to be under AGPL like Lemmy? May have to ask Extism maintainers about legal conditions.

# Future possibilities

In the future we can add hooks for other user actions, as well as additional host functions.

We could also support implementing new post ranking algorithms in plugins.

Additionally it should be possible for plugins to define new API routes for additional functionality.

We may also consider to move some existing features into plugins. This applies to features which are mostly separate from core functionality, and which are only used by few instances. For example:
- Slur filter
- Custom emojis
- Taglines
