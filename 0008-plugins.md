- Feature Name: plugins
- Start Date: 2024-12-02
- RFC PR: [LemmyNet/rfcs#0008](https://github.com/LemmyNet/rfcs/pull/0008)
- Lemmy Issue: [LemmyNet/lemmy#3562](https://github.com/LemmyNet/lemmy/issues/3562)

# Summary

Implement a plugin system for Lemmy based on Webassembly.

# Motivation

Plugins will make Lemmy more flexible, and allow implementing features which are too niche for merging into Lemmy core. It will allow Lemmy to focus on core features, while allowing outside developers to contribute new features without using Rust.

# Guide-level explanation

## Extism

[Extism](https://extism.org/) is an open source framework to develop plugins for Rust projects. It supports numerous different languages by compiling them to webassembly which gets called from Rust.

# Reference-level explanation

## File Structure

Each plugin consists of two files, a [manifest](https://extism.org/docs/concepts/manifest) and the actual wasm binary. The manifest specifies how to load the binary, and provides various configuration options. Plugins are loaded at startup from `./plugins/` or `LEMMY_PLUGIN_PATH`. Each plugin must have a `metadata` hook returning data in the format below, which is listed in a new field `active_plugins` under `/api/v4/site`.

```json
{
    "name": "My Plugin",
    "url": "http://example.com/plugin-info",
    "description": "Plugin which does the thing"
}
```

## Plugin Hooks

Lemmy will have hooks for specific actions which can then be used by plugins. In general there are two types of hooks: before an action is written to the database, so it can be rejected or altered. And after it is written to the database, mainly for different types of notifications.

Whenever a plugin hook is called, Lemmy blocks the API call until the plugin returns. This means that slow plugins can result in slow API actions for users, or delays for incoming federation. To avoid such problems, plugin developers should optimize their code to return as fast as possible, and perform expensive operations or network calls in a background thread.

Data is passed to plugins using the existing structs linked below, serialized to JSON. Additionally plugins can retrieve [config values](https://github.com/extism/go-pdk?tab=readme-ov-file#configs) `lemmy_version` (e.g. `0.19.8`) and `lemmy_url` (e.g. `http://localhost:8536`), in order to retrieve data or perform further actions via Lemmy API.

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
- Private Message
  - [create_local_private_message](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/private_message.rs#L42)
  - [update_local_post](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/private_message.rs#L60)
  - [receive_federated_private_message](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/private_message.rs#L42)
### After writing to Database

These are mainly useful to generate notifications.

- Post
  - [new_post](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/post.rs#L19)
  - [new_post_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/post.rs#L157)
- Comment
  - [new_comment](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L27)
  - [new_comment_vote](https://github.com/LemmyNet/lemmy/blob/main/crates/db_schema/src/source/comment.rs#L105)
- Private Message
  - [new_private_message](https://github.com/LemmyNet/lemmy/blob/0.19.7/crates/db_schema/src/source/private_message.rs#L25)

## Example

Below is a simple Go plugin which uses the `create_local_post` hook to replace `Rust` in post body with `Go`, and reject posts which mention `Java`.

Also checkout the documentation for Extism's [Go Plugin Development Kit](https://github.com/extism/go-pdk).

```golang
package main

import (
	"github.com/extism/go-pdk"
	"errors"
	"strings"
)

type Metadata struct {
	Name string `json:"name"`
	Url string `json:"url"`
	Description string `json:"description"`
}

// Returns info about the plugin which gets included in /api/v4/site
//go:wasmexport metadata
func metadata() int32 {
	metadata := Metadata {
		Name: "Test Plugin",
		Url: "https://example.com",
		Description: "Plugin to test Lemmy feature",
	}
	err := pdk.OutputJSON(metadata)
	if err != nil {
		pdk.SetError(err)
		return 1
	}
	return 0
}

// This hook gets called when a local user creates a new post
//go:wasmexport create_local_post
func create_local_post() int32 {
	// Load user parameters into a map, to make sure we return all the same fields later
	// and dont drop anything
	params := make(map[string]interface{})
	err := pdk.InputJSON(&params)
	if err != nil {
		pdk.SetError(err)
		return 1
	}

	// Dont allow any posts mentioning Java in title
	// (these will throw an API error and not be written to the database)
	name := params["name"].(string)
	if strings.Contains(name, "Java") {
		// Throw error to reject post
		pdk.SetError(errors.New("We dont talk about Java"))
		return 1
	}

	// Replace all occurences of "Rust" in post title with "Go"
	params["name"] = strings.Replace(name, "Rust", "Go", -1);

	err = pdk.OutputJSON(params)
	if err != nil {
		pdk.SetError(err)
		return 1
	}
	return 0
}
```

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

## Licensing

Plugins can use any [OSI-approved open source licenses](https://opensource.org/licenses). Unlike Lemmy's AGPL license, most of them don't require sharing the source code with people who use it over a network, so the source code only needs to be made available to server admins. This also allows for for paid plugins.

# Drawbacks

Plugins may have a performance impact as described above. It is up to plugin developers and instance admins to ensure that plugins run adequately. In any case plugins are only called for POST actions but not for GET. This means slow plugins might result in longer waiting time to submit posts, but passive browsing will be unaffected.

# Rationale and alternatives

The alternative would be to implement all possible functionality directly in the Lemmy backend. This is not always desirable because features may only be desired by a small fraction of users. Additionally Lemmy code can only be written in Rust, while plugins can use many different languages.

# Prior art

Plugins for other Fediverse platforms:

- https://docs.joinpeertube.org/contribute/plugins
- https://github.com/friendica/friendica-addons
- https://codeberg.org/hubzilla/hubzilla-addons
- https://nextcloud.github.io/news/features/plugins/
- https://developer.wordpress.org/plugins/

# Unresolved questions

# Future possibilities

In the future we can add hooks for other user actions, such as voting or uploading images. We could also support implementing new post ranking algorithms in plugins.

Additionally it should be possible for plugins to define new API routes for additional functionality.

We may also consider to move some existing features into plugins. This applies to features which are mostly separate from core functionality, and which are only used by few instances. For example:

- Slur filter
- Custom emojis
- Taglines

In the opposite way, popular plugins may be shipped as part of Lemmy releases, or reimplemented as part of Lemmy core.
