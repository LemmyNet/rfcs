- Feature Name: Private Communities
- Start Date: 2024-02-15
- RFC PR: [LemmyNet/rfcs#0005](https://github.com/LemmyNet/rfcs/pull/5)
- Lemmy Issue: [LemmyNet/lemmy#187](https://github.com/LemmyNet/lemmy/issues/187)

# Summary

Lemmy is currently limited to public communities with public content. This RFC proposes a way to implement private, federated communities where only approved users can read and write.

# Motivation

Private spaces are a fundamental aspect of human communication. They allow discussing sensitive topics without interruption from outsiders. Supporting private communities in Lemmy gives a major new use case for talking among friends or within organizations. This use case is currently not covered by major Fediverse projects.

# Detailed design

The functionality requires only minor changes to the existing community implementation. The community profile with sidebar, icon etc remains public, so that anyone can follow the community. When a follow request is received, it needs to be manually approved by a moderator, using an interface similar to the existing registration applications. Posts and comments inside a private community are only visible to approved followers.

This change is in some ways similar to [Local-only communities](https://github.com/LemmyNet/lemmy/pull/4350). That PR can be used as a reference for parts of code which need to be changed.

## Database

The `CommunityVisibility` enum used in `community.visibility` gets a new variant `Private`. Queries in `post_view.rs` and `comment_view.rs` need to be adjusted so that content in communities set to private is only shown to followers. This also needs to be covered by unit tests.

## API

When a new follow request for a local private community is made, it needs to set `pending = true`. Moderators can review and approve/reject pending follows with the following endpoints:
- `GET /api/v3/community/pending_follows/count`
- `GET /api/v3/community/pending_follows/list`
- `POST /api/v3/community/pending_follows/approve`

The follow_request list for each item should contain a boolean value `is_new_instance`. This value should be true if the community has no followers from the user's instance yet, and allows showing a warning when content will be federated to a new server.

Once the approve endpoint is called, the follow request can be deleted. If the request was approved, write the follow relation to `community_follower`. If it was rejected, we could notify the user about this (optional). 

Users who don't follow the private community need to be prevented from posting or making other actions in it. This can be done by adding a check in [check_community_user_action()](https://github.com/LemmyNet/lemmy/blob/main/crates/api_common/src/utils.rs#L171).

When [processing mentions](https://github.com/LemmyNet/lemmy/blob/main/crates/utils/src/utils/mention.rs#L24), the code needs to check if it is inside a private community, and in that case ignore mentions of users which don't follow it.

The new endpoints should be covered by [Typescript API tests](https://github.com/LemmyNet/lemmy/tree/main/api_tests).

## Federation

When a `Follow` activity is received, Lemmy currently automatically responds with an `Accept`. This needs to be changed so that private communities only store the pending follow in the database, without sending an `Accept`. Once a moderator approves the request, send an `Accept` to the user. Note that only moderators registered on the same instance as the community will see follow requests.

The `Group` actor for private communities remains public and can be fetched without restrictions, so that remote users can fetch it and follow it. It needs a new attribute to indicate that the group is not public. [ActivityStreams](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-group) doesn't seem to provide anything suitable, so we can simply add a custom attribute `private: true`.

With Activitypub federation there are two ways instances can communicate with each other: Fetch data via HTTP GET, or POST an activity to the inbox. Activities are only sent to approved followers, so we don't need to worry about leaking any private data. However the fetching needs adjustments. For this we can use authorized fetch, which is already [implemented behind a setting](https://github.com/LemmyNet/lemmy/blob/main/src/lib.rs#L182). The simplest solution here is to remove the setting and sign all all outgoing requests, so we don't have to track which specific objects require auth or not.

On the server side we need to verify the signature when an object gets fetched in a private community. The code for this is [here](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/http/post.rs), [here](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/http/comment.rs) and [here](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/http/community.rs#L76). The logic for signature verification is [implemented in the activitypub-federation crate](https://github.com/LemmyNet/activitypub-federation-rust/blob/main/src/http_signatures.rs#L146), but not yet part of the public API.

Like in the API code we need to ensure that only followers can perform actions in private communities. Here a check needs to be added to [verify_person_in_community()](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/activities/mod.rs#L77).

[Objects](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/objects/post.rs#L127) and [activities](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/activities/create_or_update/post.rs#L53) are currently marked as public, we need to get rid of this in private communities. Additionally, the method [verify_is_public()](https://github.com/LemmyNet/lemmy/blob/main/crates/apub/src/activities/mod.rs#L127) needs to take community visibility into account. If the community is public, reject any non-public content (just like now). if it is private, reject any public content.

## RSS

The community RSS feed needs to be adjusted to avoid leaking private content. The easiest solution is to disable it for private communities, but it would also be possible to serve private community feeds with authentication, similar to the existing subscribed feed.

Other feed types need to exclude any content from private communities. This should already be handled by changes to `post_view.rs`

## Frontends and Apps

Community moderators need to get access to the new `community.visibility` setting in the sidebar. Community visibility should also be indicated to users.

Frontends need to show an interface for moderators to approve new followers. This can likely reuse much of the code from registration applications, and use the endpoints described under "API". When `is_new_instance` is true on a given application, show a warning similar to the following before approving:

> Warning: This is the first follower from example.com. After approval, the admin of example.com is technically able to access all past and future content in this community. It is also possible that the instance at example.com makes community posts publicly available. If the community has sensitive content, make sure to only approve followers from trusted instances.

# Alternatives

The only alternative would be not to implement this feature in Lemmy, and rely on different platforms for private communities instead. Considering that Lemmy already has a large number of users and many important features, it is better to provide this feature directly.

# Unresolved questions

Images in private communities are publicly available for anyone who knows the url. This is probably fine as image URLs are long and randomized, so they are impossible to guess.

When setting an existing community to private, this setting will generally also federate to other instances. However it is possible that some instances don't refetch the community, and old content remains public. And of course old content can be stored in external archives. This problem could be avoided by preventing existing, public communities to be marked as private.

In a public community, old content missing from your instance can be fetched by visiting the community on its home instance and copying the URL of individual posts and comments into your own instance's search field to fetch them. This is not easily possible with a private community, as it cannot be viewed publicly. However this is a general problem with Lemmy communities and can be handled separately.
