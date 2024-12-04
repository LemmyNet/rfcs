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

[Extism](https://extism.org/) is an open source framework to develop plugins for Rust projects. It supports numerous different languages by compiling them to webassembly which gets called from Rust.

# Reference-level explanation

## Plugin Hooks

Lemmy will have to add hooks for specific actions, which can then be used by plugins. In general there are two types of hooks: before an action is written to the database, so it can be rejected or altered. And after it is written to the database, mainly for different types of notifications.

Data is passed to plugins using the existing structs linked below, serialized to JSON.

For the initial implementation, the following hooks will be available:

### Before writing to Database

These hooks can be used to reject or alter user actions based on different criteria.

- Post
    - [create_local_post](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/post.rs#L67)
    - [update_local_post](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/post.rs#L98)
    - [receive_federated_post](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/post.rs#L67)
    - [post_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/post.rs#L171)
- Comment
    - [create_local_comment](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L59)
    - [update_local_comment](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L84)
    - [receive_federated_comment](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L59)
    - [post_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L119)

### After writing to Database

These are mainly useful to generate notifications.

- Post
	- [new_post](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/post.rs#L19)
	- [new_post_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/post.rs#L157)
- Comment
	- [new_comment](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L27)
	- [new_comment_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L105)

## Example

Below is a simple Go plugin which uses the `create_local_post` hook to replace `Rust` in post body with `Go`, and reject posts which mention `Java`. 

Also checkout the documentation for Extism's [Go Plugin Development Kit](https://github.com/extism/go-pdk).

```golang
package main

import (
	"github.com/extism/go-pdk"
	"strings"
)

type CreatePost struct {
	Name string `json:"name"`
	Body *string `json:"body,omitempty"`
	...
  }

//export create_local_post
func my_plugin() int32 {
	params := CreatePost{}
	// Read post data
	err := pdk.InputJSON(&params)
	if err != nil {
		pdk.SetError(err)
		return 1
	}
	if strings.Contains(params.Body, "Java") {
		// Throw error to reject post
		pdk.SetError("We dont talk about Java)
		return 1
	}
	params.Body = strings.Replace(params.Body, "Rust", "Go", -1);
	// Write back post data
	err = pdk.OutputJSON(params)
	if err != nil {
		pdk.SetError(err)
		return 1
	}
	return 0
}

func main() {}
```

## Host Functions

Plugins will need access to the Lemmy database for various reasons, such as determining which instance a new post is coming from, or storing custom data. The easiest way to do this would be direct database access for plugins, but this has many problems:
- Database schema changes across Lemmy versions, which would make it very difficult to make plugins compatible with new Lemmy versions, and prevent instances from upgrading
- Plugins could accidentally delete data or break the database

As such we need to use a different approach, by exposting specific database calls from Lemmy via [host functions](https://extism.org/docs/concepts/host-functions). For the beginning the following functions will be exposed:

- Instance::read_or_create 

More functions may be added if they are needed for the initial plugins.

## Initial Plugins

These can mainly serve as proof of concept, and as examples which can be modified by plugin developers. Include integration tests to ensure that functionality works with new Lemmy changes. These plugins should be written in a variety of languages, and live in a separate repo `lemmy-plugin-examples`.

- Block only downvotes from a specific instance
- Replace words in post or comment text (eg "cloud to butt")
- Replace Youtube URLs in post links with [Invidious](https://invidious.io/)
- Push notifications (send data of new posts to localhost url, [#2631](https://github.com/LemmyNet/lemmy/issues/2631))
- Language detection (automatically apply language tag to posts/comments, [#2870](https://github.com/LemmyNet/lemmy/issues/2870))
- Archive previous versions of posts when they are edited
- Reject comments from specific user IDs (like [!santabot@slrpnk.net](https://slrpnk.net/post/11069853))
- Additional suggestions welcome

# Drawbacks

Plugins may have a negative impact on Lemmy's performance. It is up to plugin developers and instance admins to ensure that plugins run adequately. In any case plugins are only called for POST actions but not for GET. This means slow plugins might result in longer waiting time to submit posts, but passive browsing will be unaffected.

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
