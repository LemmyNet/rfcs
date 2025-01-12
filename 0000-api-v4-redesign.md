- Feature Name: API v4 Redesign
- Start Date: 2024-1-12
- RFC PR: [LemmyNet/rfcs#0000](https://github.com/LemmyNet/rfcs/pull/0000)
- Lemmy Issue: [LemmyNet/lemmy#4428](https://github.com/LemmyNet/lemmy/issues/4428)

# Summary

Lemmy's HTTP API (v3) is currently written mostly in a RPC style: endpoints are mostly verbs the correspond to actions that can be run on the server. In addition to this, responses from the API are often "view" objects that are usually used for the database; this returns more than clients need much of the time, resulting in data that is often either redundant or not used by the client being sent over the network. This RFC proposes endpoints for an API that is closer to (but not fully) RESTful, while pruning responses from the server.

# Motivation

With many of the features needed for Lemmy v1.0.0 getting completed, the first major release is nearish on the horizon. Given that major releases in semantic versioning denote breaking changes — and this will be the first non-zero prefixed release — if there are any breaking changes maintainers have in mind for Lemmy that can improve the experience of users, admins, moderators, alternative frontend maintainers, and the maintainers of other cool stuff that uses its API, now is the best time to make them. I believe the API can benefit from several improvements.

One improvement would be pruning unnecessary properties from responses. Besides the bandwidth concerns mentioned in the summary, there is also a code maintenance aspect: loosening the coupling between data transfer objects and database models can help use avoid potential future breaking changes on the API if the database is changed significantly.

Regarding making the API closer to REST than RPC: I believe the domain of Lemmy (and social media generally) lends itself to being conceptualized by focusing on resources more than it does actions/functions/procedures. I will elaborate on this in a later section.

# Guide-level explanation

Since this is a change to the API, any explanation should be targeted at potential new contributors/maintainers and developers of third-party software (like frontends and bots). To this end, an OpenAPI specification will be provided. A first draft of this specification is included with this RFC and will repeatedly be referenced in the more detailed explanation section. Note that the included spec only includes endpoints with methods, operation IDs, and summaries and will need to have things like expected requests and responses, status codes, and auth handled in the future.

You can view the spec by copy-pasting it into [Swagger Editor](https://editor.swagger.io/); it's much easier to examine like that than it is reading the raw YAML.

# Reference-level explanation

## Resource Archetypes

An understanding of REST resource archetypes will help in understanding the design decisions made in this RFC. There are 4 archetypes of resources: document, collection, store, and controller. The following 4 subsections are lifted from [_REST API Design Rulebook_ by Mark Masse](https://github.com/Dhinendran/studymaterial/blob/master/Book%20-%20O%27Reilly%20REST%20API%20Design%20Rulebook.pdf).

### Document

A document resource is a singular concept that is akin to an object instance or database record. A document’s state representation typically includes both _fields_ with values and _links_ to other related resources. With its fundamental field and link-based structure, the document type is the conceptual _base archetype_ of the other resource archetypes. In other words, the three other resource archetypes can be viewed as specializations of the document archetype.

Each URI below identifies a document resource:

```
http://api.soccer.restapi.org/leagues/seattle
http://api.soccer.restapi.org/leagues/seattle/teams/trebuchet
http://api.soccer.restapi.org/leagues/seattle/teams/trebuchet/players/mike
```

A document may have child resources that represent its specific subordinate concepts. With its ability to bring many different resource types together under a single parent, a document is a logical candidate for a REST API’s root resource, which is also known as the _docroot_. The example URI below identifies the docroot, which is the Soccer REST API’s advertised entry point:

```
http://api.soccer.restapi.org
```

### Collection

A collection resource is a server-managed _directory_ of resources. Clients may propose new resources to be added to a collection. However, it is up to the collection to choose to create a new resource, or not. A collection resource chooses what it wants to contain and also decides the URIs of each contained resource.

Each URI below identifies a collection resource:

```
http://api.soccer.restapi.org/leagues
http://api.soccer.restapi.org/leagues/seattle/teams
http://api.soccer.restapi.org/leagues/seattle/teams/trebuchet/players
```

### Store

A store is a client-managed resource repository. A store resource lets an API client put resources in, get them back out, and decide when to delete them. On their own, stores do not create new resources; therefore a store never generates new URIs. Instead, each stored resource has a URI that was chosen by a client when it was initially put into the store.

The example interaction below shows a user (with ID _1234_) of a client program using a fictional Soccer REST API to insert a document resource named _alonso_ in his or her store of _favorites_:

```
PUT /users/1234/favorites/alonso
```

### Controller

A controller resource models a procedural concept. Controller resources are like executable functions, with parameters and return values; inputs and outputs.

Like a traditional web application’s use of HTML forms, a REST API relies on controller resources to perform application-specific actions that cannot be logically mapped to one of the standard methods (create, retrieve, update, and delete, also known as
CRUD).

Controller names typically appear as the last segment in a URI path, with no _child_ resources to follow them in the hierarchy. The example below shows a controller re- source that allows a client to resend an alert to a user:

```
POST /alerts/245743/resend
```

## Exploring the Proposed Endpoint Paths

As stated in the guide-level explanation, an OpenAPI spec is included in this RFC in YAML form. It is much easier to explore using a OpenAPI editor like [Swagger Editor](https://editor.swagger.io/).

### Plural vs Singular

In API v3, most nouns are singular (e.g. `/post`, `/comment`, `/community`). The proposed API has many more plural nouns, almost always corresponding to collection resources (e.g. `/posts`, `/comments`, `/communities`). When a consumer of the API hits a pluralized endpoint with a GET request, they can be sure they will be getting back multiple results (assuming the specific endpoint supports GET). There are still some singular resources, such as `/me` and `/site`. Note that these have singular wording because they refer to unique resources, in this case the current user (as determined by the Authorization header) and site; both of those endpoints are document resources.

### Path Parameters

When referring to an individual member of a collection, the ID of the resource is used as part of the URL path, e.g. `/posts/1234`. This is a way of referring to individual members of collections; it goes well with the plural nouns used in paths.

### More Substantial Path Name Changes

Some resources have had name changes that go beyond being made plural. One major change is that things that used to be done under `/account` now instead use `/me`. This is to make sure it's thought of as a resource; it refers to the currently logged in user. While `/account` is also a singular noun that could work as a resource name, I think the new path name makes it more obvious that the account being referred to is not just any account, but the account of the person making the request.

One of the resources under `/me` of note is `/me/sessions`. This is used to handle logging in, logging out, and viewing all of the sessions of the currently logged in user.

Some resources already more or less had names that are nouns also got names changes. One such change it calling private messages private messages `/direct-messages`. I replaced "private" with "direct" to better communicate that while these messages are sent directly to other users, they aren't private al a Matrix DMs. Another change is that what used to be referred to as "images" in the API is now "media". This change is partly to make it consistent with the list media endpoints from v3, and partly because it is technically also possible to upload non-images (like videos) with the old images endpoint. One more change is using the term "suspended" instead "remove" when referring to posts, comments, and communities that get taken down by moderators. This is because I wanted to make it clear that content that gets removed isn't necessarily removed permanently since mods can reinstate it. I'm not super happy with "suspended" and am open to ideas on alternatives.

There were some other name changes to this effect, but the ones listed above are likely the most jarring ones.

### PUT is Sometimes Used for Creation as well as Updates

Some endpoints may seem odd at first glance because of this. Isn't PUT supposed to be used for updates? It is, but it can also be used for creating resources under certain circumstances. Recall the store resource archetype mentioned earlier: it says that URIs are chosen by the client instead of the server. What does this mean? An example will help make this clear.

Take the path `/posts/locked/{postId}`. This uses PUT to lock a post, and DELETE to unlock a post. Note that the last part of the path is `postId`. `postId` is the ID of a post that already exists. In order to make the call, the client must already have the ID of a post in mind to lock. With this in mind, the PUT and DELETE operations on this endpoint can be though of more as a box that clients can put posts into and remove posts from.

A major benefit of this approach is that it communicates that the operation is idempotent. A client could call `/posts/locked/1234` multiple times, and the effect would be the same as if they called it only a single time: lock a post if it's not already locked. Compare this to the v3 API, where locking a post requires POSTing to `/post/lock`. Even though the actual operation is idempotent, the POST method in HTTP is classified as not being idempotent; this results in some miscommunication about the API. PUT, on the other hand, is classified as being idempotent.

### Rough Edges

While I've done my best to make the API route design only use nouns (and perhaps adjectives for store archetypes), some actions the API supports required breaking this convention. The main offenders are moderation actions that include a reason both when performing and undoing something. For example, to ban a user, the a client must make a PUT request to `/site/users/banned/{personId}`; unbanning the user requires a POST request to `/site/users/banned/{personId}/unban`. I'm not particularly happy with the approach I picked for this, since it is inconsistent with how most of the rest of the API is designed. However, I considered an alternative that seems at least as bad.

Bans and content suspensions could be treated as stores on their respective collections, and e.g. users could be added and removed from the ban list with `PUT /users/banned/{personId}` and `DELETE /users/banned/{personId}`, with the DELETE taking a request body with the unban reason. The problem with this approach is that DELETE requests with bodies can explained with [an excerpt from RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html#DELETE).

> Although request message framing is independent of the method used, content received in a DELETE request has no generally defined semantics, cannot alter the meaning or target of the request, and might lead some implementations to reject the request and close the connection because of its potential as a request smuggling attack (Section 11.2 of [HTTP/1.1]). A client SHOULD NOT generate content in a DELETE request unless it is made directly to an origin server that has previously indicated, in or out of band, that such a request has a purpose and will be adequately supported. An origin server SHOULD NOT rely on private agreements to receive content, since participants in HTTP communication are often unaware of intermediaries along the request chain.

Besides this, there are a few parts of the API that uses the controller archetype (like purging content and appointing community administrators).

# Drawbacks

This would be a very breaking change, potentially causing a lot of difficulty for consumers of the API. If the first stable version wasn't such an appropriate place to make breaking changes, I would not have made this proposal in the first place.

# Rationale and alternatives

## Important Note

Before the meat of this section, it must be stated that even though I am framing this as the existing (v3) API being RPC and the proposed (v4) API being REST, **the current API has some restful elements and the proposed API still has some RPC elements**. For the purposes of this RFC, REST and RPC are on the opposite ends of a spectrum, and this proposal moves the API to be closer to REST than it is to RPC.

## What's an RPC API?

Remote procedure call (RPC) APIs are APIs for running functions on a remote server. The various functions may use some HTTP semantics (e.g. GET for reads and POST for things that aren't reads, hierarchical URLs, query params, etc.), but most of the semantic heavy lifting is done by the application; that is to say, semantics are more specific to the individual application with functions for things like banning users, running CI stages, starting a scan — things that don't generalize to network applications more generally.

## What's REST?

REST APIs are web APIs that conform to REST architectural principles, which are ([copying from a redhat article](https://www.redhat.com/en/topics/api/what-is-a-rest-api)):

- A client-server architecture made up of clients, servers, and resources, with requests managed through HTTP.
- Stateless client-server communication, meaning no client information is stored between get requests and each request is separate and unconnected.
- Cacheable data that streamlines client-server interactions.
- A uniform interface between components so that information is transferred in a standard form. This requires that:
  - resources requested are identifiable and separate from the representations sent to the client.
  - resources can be manipulated by the client via the representation they receive because the representation contains enough information to do so.
  - self-descriptive messages returned to the client have enough information to describe how the client should process it.
  - hypertext/hypermedia is available, meaning that after accessing a resource the client should be able to use hyperlinks to find all other currently available actions they can take.
- A layered system that organizes each type of server (those responsible for security, load-balancing, etc.) involved the retrieval of requested information into hierarchies, invisible to the client.
- Code-on-demand (optional): the ability to send executable code from the server to the client when requested, extending client functionality.

Some of these principles are more relevant to our use-case than others: for instance, it's already given that we're using a client-server architecture, and code-on-demand doesn't fit into our use-case at all.

## How is Lemmy's API RPC-like?

In both the [v3 API](https://github.com/LemmyNet/lemmy/blob/c0342292951c237ec5f575f2165758e4f0712e6f/src/api_routes_v3.rs) and current state of the [v4 API](https://github.com/LemmyNet/lemmy/blob/c0342292951c237ec5f575f2165758e4f0712e6f/src/api_routes_v4.rs) (as of January 12, 2025) operations that aren't creating, reading, and updating certain types of content all have URLs that correspond to actions instead of resources, e.g. locking a post, creating a report, and logging in. Further, unique resources are rarely identified by URL, typically being retrieved by passing a query parameter with a specific ID (e.g. GET /post?id=123).

## Why RESTful?

One major reason to make the API more RESTful is that Lemmy's domain lends itself well to being modeled as resources. Some resources in the domain are:

- posts
- comments
- users
- communities
- federated instances
- direct messages

All of these resources act as nouns. By having the routes for endpoints focused on nouns (and, in some cases, adjectives), we can lean on HTTP verbs to show which action is being taken on a resource. We can also better describe errors by being more mindful of which HTTP status codes we return.

# Prior art

In order to see how this compares to other social media platforms, I looked at the APIs of 3 different fediverse applications: Mastodon, Pleroma, and PeerTube.

## Mastodon

[Mastodon's REST API docs can be found here.](https://docs.joinmastodon.org/api/) You will need to scroll down far enough so that the "REST API" and "API METHODS" sections of the nav on the left show up to explore. It uses plural nouns similar to what this proposal does (e.g. `GET /statuses` to list statuses, `POST /statuses` to create a new status, `GET /status/{id}` to retrieve a specific status, `PUT /status/{id}` to update a specific status, and `DELETE /status/{id}` to delete a specific status). Mastodon also handles new user creation with `POST /accounts` as opposed to a controller archetype endpoint named something like "register. Finally, images and videos are more broadly called "media" like in this proposal.

Unlike this proposal, Mastodon doesn't use the store archetype. For example, to favorite and unfavorite a status, instead of using something like `PUT /statuses/favorites/{statusId}` and `DELETE /statuses/favorites/{statusId}`, it uses `POST /statuses/{statusId}/favorite` and `POST /statuses/{statusId}/unfavorite`. They also don't have any single resource referring to the logged in user. To get the current user's profile, query `/profile`; to get the current user's settings, query `/preferences`; to block a user, post to `/accounts/{id}/block`. Compare these to `GET /me`, `GET /me/settings`, and `PUT /me/blocked/users/{personId}` from this proposal.

Curiously, status creation can included an `Idempotency-Key` header to prevent accidental duplicate posts. While not in the scope of this RFC, something similar could prove useful for Lemmy's API.

## Pleroma

[Pleroma's API docs can be found here.](https://api.pleroma.social/) Pleroma is similar to Mastodon in its use of plural nouns for collections, using ids to single out individual resources in collections, calling image attachments "media", and using many controller archetype resources but no store archetype resources.

## PeerTube

[PeerTube has an OpenAPI spec that can be viewed here.](https://github.com/Chocobozzz/PeerTube/blob/develop/support/doc/api/openapi.yaml) Similar to Mastodon and Pleroma, PeerTube uses plural nouns for collections of resources and ids as part of the path to identify specific resources. Unlike those 2 (but like this proposal) resources related to the current user are grouped under a resource specific to it (in this case `/uses/me`). It leans harder on controller archetype resources than the other 2 APIs being referenced. It does not use the term "media" for media content, however this is probably due to the fact that it specializes in video sharing.

Strangely, there is both a `POST /users/register` endpoint and a `POST /users` endpoint for creating new users. It is not clear to me why that is the case.

# Unresolved questions

The OpenAPI spec provided with this only lists paths, methods, path parameters. There are still several aspects of the API that need to be hashed out:

- Request schemas
- Response schemas
- Response codes
- Query parameters (where appropriate)

The second is especially important since the v3 (and current as of January 12, 2025 v4) API returns very large responses often with more data than what the client needs. I limited the proposal spec to just routes because the changes I propose are already quite large.

One other potential change worth mentioning is adopting [hypertext as the engine of application state (HATEOAS)](https://restfulapi.net/hateoas/) for the API. I don't feel strongly about it myself, but since I framed this proposal as making the API more RESTful and HATEOAS is a component for designing RESTful APIs, it would be remiss of me not to mention it.

Besides all of the above, the design of the routes presented in this proposal's spec should be discussed.

# Future possibilities

One thing that would be very good to have (but not in the scope of this RFC) would be a way to generate an OpenAPI spec from Lemmy's source code. I've found 2 good libraries that can be used for generating an OpenAPI 3.0 spec: [`apistos`](https://crates.io/crates/apistos) and [`oasgen`](https://crates.io/crates/oasgen).

Besides new things, I think taking a closer look at our API and the data it accepts and returns could help with future refactors if any are done. In particular, I think the current codebase lacks a separation of concerns between data used in the database and data returned by the API. The worst offenders in this regard are endpoints that return entire view objects, like [`get_post`](https://github.com/LemmyNet/lemmy/blob/11e05135923b7060df29e4525183314f653b6353/crates/api_crud/src/post/read.rs#L110-L115) which returns both a post view and a community view. A lack of separation of concerns is also visible in the route handler functions, which all handle not only parsing requests and sending responses, but much of the business logic and even some of the database handling logic (the last insofar as `&mut context.pool()` needs to be passed around everywhere).
