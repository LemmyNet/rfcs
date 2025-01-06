- Feature Name: API v4 Redesign
- Start Date: 2024-12-30
- RFC PR: [LemmyNet/rfcs#0000](https://github.com/LemmyNet/rfcs/pull/0000)
- Lemmy Issue: [LemmyNet/lemmy#0000](https://github.com/LemmyNet/lemmy/issues/0000)

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

## Rest vs RPC

### Important Note

Before the meat of this section, it must be stated that even though I am framing this as the existing (v3) API being RPC and the proposed (v4) API being REST, **the current API has some restful elements and the proposed API still has some RPC elements**. For the purposes of this RFC, REST and RPC are on the opposite ends of a spectrum, and this proposal moves the API to be closer to REST than it is to RPC.

### What's an RPC API?

Remote procedure call (RPC) APIs are APIs for running functions on a remote server. The various functions may use some HTTP semantics (e.g. GET for reads and POST for things that aren't reads, hierarchical URLs, query params, etc.), but most of the semantic heavy lifting is done by the application; that is to say, semantics are more specific to the individual application with functions for things like banning users, running CI stages, starting a scan — things that don't generalize to network applications more generally.

### What's REST?

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

### How is Lemmy's API RPC-like?

In both the [v3 API](https://github.com/LemmyNet/lemmy/blob/c0342292951c237ec5f575f2165758e4f0712e6f/src/api_routes_v3.rs) and current state of the [v4 API](https://github.com/LemmyNet/lemmy/blob/c0342292951c237ec5f575f2165758e4f0712e6f/src/api_routes_v4.rs) (as of January 2, 2025) operations that aren't creating, reading, and updating certain types of content all have URLs that correspond to actions instead of resources, e.g. locking a post, creating a report, and logging in. Further, unique resources are rarely identified by URL, typically being retrieved by passing a query parameter with a specific ID (e.g. GET /post?id=123).

### Why RESTful?

One major reason to make the API more RESTful is that Lemmy's domain lends itself well to being modeled as resources. Some resources in the domain are:

- posts
- comments
- users
- communities
- federated instances
- direct messages

All of these resources act as nouns. By having the routes for endpoints focused on nouns (and, in some cases, adjectives), we can lean on HTTP verbs to show which action is being taken on a resource. We can also better describe errors by being more mindful of which HTTP status codes we return.

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:


- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

In particular explain the following, if applicable:

- Which database changes are necessary?
- How does the HTTP API change?
- How does the new feature work over federation? Are there similar features in other Fediverse platforms which should be compatible? Also see the [federation docs](https://join-lemmy.org/docs/contributors/05-federation.html). -->

# Drawbacks

Why should we _not_ do this?

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
