- Feature Name: Post Tags
- Start Date: 2023-11-04
- RFC PR: [LemmyNet/rfcs#0004](https://github.com/LemmyNet/rfcs/pull/4)
- Lemmy Issue: [LemmyNet/lemmy#317](https://github.com/LemmyNet/lemmy/issues/317)

# Summary

Ability to add descriptive tags to Posts similar to Reddit's Flair system primarily on a per-community basis.

# Motivation

Post Tags would allow for vastly improved content filtering (and to a lesser extent searching) both for Users and Admins/Moderators. 

# Guide-level explanation

Tags grant you more fine grained control over the content you see on Lemmy. Certain users do not want to see certain content and tags allow you to do exactly that.

For example a lot of communities offer mixed content such as Memes, News and Discussions. Some people would like to only see News or Discussions. In such a case they could blocklist (filter out) the "Meme" tag, going forward they would not be presented with any Memes in that community.

Another use for tags is better handling of how posts are displayed. For example some people would like the content of a post to stay hidden for all users. This could be the case in a community focused on release discussions of Books, Movies, TV Shows or communities with content some users might find offensive but which is not neccesarily NSFW (for example jokes about certain topics).

In order to ensure this consistency while filtering tags can only be created by privileged users, initially that is intended to be Admins and Community moderators.

Essentially the tags serve as an easier way to message what is *inside* the post. Combining multiple tags allows for even finer control. Made a post about news in poland? Add the "News" and "Poland" tags to it and people interested exclusively in polish news can find it easily or people not interested can avoid it just as easily.

# Reference-level explanation

### Protocol:
The [ActivityStreams vocabulary](https://www.w3.org/TR/activitystreams-vocabulary/#dfn-tags) defines that any object can have a list of tags associated with it. Tags in AS can be of any type, so we define our own type `CommunityPostTag`:

```jsonc
{
  "type": "lemmy:CommunityPostTag",
  "id": "community-url/tag/tag-id", // e.g. https://example.org/c/example/tag/News
  "name": "Readable Name of Tag"
}
```


In JSON-LD terms, this object will live in the lemmy namespace as `https://join-lemmy.org/ns#CommunityPostTag`), which in objects lemmy sends out is always imported from https://join-lemmy.org/context.json as the prefix `lemmy:`, so we can refer to it as `lemmy:CommunityPostTag`. For example, a lemmy post object (Page in AP terms) would look like this:

```json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://join-lemmy.org/context.json"
  ],
  "id": "https://enterprise.lemmy.ml/post/55143",
  "type": "Page",
  "audience": "https://enterprise.lemmy.ml/c/tenforward",
  "name": "Post title",
  "content": "<p>This is a post in the /c/tenforward community</p>\n",
  "tag": [
    {
      "type": "lemmy:CommunityPostTag",
      "id": "https://enterprise.lemmy.ml/c/tenforward/tag/news",
      "name": "News"
    }
  ]
  // ... more required props
}
```


Entirely unmoderated tags are not an option for lemmy as the moderation workload would be too much. Additionally users being able to type out tags themselves introduces splintering in the tag contents due to typos. A better solution is a curated list of tags users can attach to their posts. The list of tags can be maintained by both admins and moderators allowing for each community to tailor tags to their specific needs.
Content for tags would be located in the respective root: `https://example.org/c/example/tag/tag`

The ID of a CommunityPostTag is a URL prefixed with `https://instance/c/community/tags/` and a user-chosen or generated (from the name) ID. The ID itself will rarely be user-visible, since the name is used there.

For now tags will only be applicable to posts, however the general design allows for them to be attached to any kind of object later on, be that instance, community or user. The limited initial scope allows for easier modifications should any rough edges or missing features be discovered.
Additionally per Tag Inheritance could be added by expanding on this structure, more details can be found in [Future Possibilities > Tag Inheritance](#tag-inheritance).

Example Tag Objects:

```json
{
    "type": "lemmy:CommunityPostTag",
    "url": "https://example.org/c/news/tag/newspaper-company-1",
    "name": "Related to Newspaper Company 1"
}
```

```json
{
    "type": "lemmy:CommunityPostTag",
    "url": "https://example.org/c/news/tag/blood-gore",
    "name": "Blood/Gore"
}
```

```json
{
    "type": "lemmy:CommunityPostTag",
    "url": "https://example.org/c/news/tag/news",
    "name": "News"
}
```

Below is an example of a News post in the news community. The post has a communtiy tag for the newspaper company that published it, as well as a content warning for Blood/Gore. In order to share this news post across instances it also is tagged with the news tag of a remote instance.

Example Post json:

``` json
{
  "@context": [
    "https://www.w3.org/ns/activitystreams",
    "https://join-lemmy.org/context.json"
  ],
  "to": [
    "https://example.org/c/news",
    "https://www.w3.org/ns/activitystreams#Public"
  ],
  "audience": "https://example.org/c/news",
  "content": ". . .",
  "sensitive": true,
  "tag": [
    {
      "type": "lemmy:CommunityPostTag",
      "url": "https://example.org/c/news/tag/newspaper-company-1",
      "name": "Related to Newspaper Company 1"
    },
    {
      "type": "lemmy:CommunityPostTag",
      "url": "https://example.org/c/news/tag/blood-gore",
      "name": "Blood/Gore"
    },
    {
      "type": "lemmy:CommunityPostTag",
      "url": "https://example.org/c/news/tag/news",
      "name": "News"
    }
  ]
}
```

### Backend:
I am not quite familiar with the Lemmy Backend so any corrections or additions for this section are appreciated.

One new table for "community_post_tag" would be required:

```sql
CREATE TABLE community_post_tag (
  id bigserial PRIMARY KEY,
  community_id bigint REFERENCES community.id,
  ap_id text UNIQUE NOT NULL,
  name text NOT NULL,
  published timestamptz NOT NULL,
  deleted timestamptz NULL,
);
```
  

Additional columns for tag display style (for example background & text color) and hierarchy could be added later by extending this table.

Additionally, a way to store the association between posts and tags is needed. This can either be done with a `tags: array[bigint]` column in the post table and a inverse index (`GIN` or `GIST`) or with a new table `post_tag (post_id, tag_id)`

Instance Admins should have the option to delete/ban tags.
Instances with nsfw disables/blocked could/should auto reject posts with an nsfw flavor tag.

The following API routes will need to be added:

**GET /api/v3/tag/list:**
Parameters:

- community_id (required)
- auth (optional)

Returns:
List of all tags in a community.
Deleted tags should not be returned in the list unless an admin/moderator.

```json
{
    "tags": [
        {
            "ap_id": "https://example.org/c/community/tag/foo", // activitypub id
            "name": "Foo Tag",
            "id": 1, // the lemmy-internal id
            "community_id": 1 // lemmy-internal community id
        },
        {...}
    ]
}
```


**POST /api/v3/tag/create:**
Parameters:

- id_slug // the generated url will be https://instance/c/community/tags/id_slug
- name
- community_id


Returns:
Object of freshly created tag

**PUT /api/v3/tag:**

Request to update tag object, see the post update API request.
**POST /api/v3/tag/delete:**

Request to delete tag object, see the post delete API request.

**ActivityPub Endpoint: GET /c/community/tag/foo (Content-Type: application/activity+json)**

This is the AP endpoint (not a lemmy api endpoint) and must contain the tag object response in JSON-LD:

```json
{
  "@context": [
    "https://join-lemmy.org/context.json",
    "https://www.w3.org/ns/activitystreams"
  ],
  "id": "https://example.org/c/community/tag/foo", // activitypub id
  "type": "lemmy:CommunityPostTag",
  "name": "Foo Tag"
}
```
**Additionally:**

Posts will need to be made editable by moderators, this is currently not possible but will be required to allow mods to fix missing or misused tags on posts.
Ideally the endpoint that will be opened for this should only allow the addition or removal of tags from a post and nothing else to prevent abuse.
As such I propose the use of:
`POST /post/tag` and `POST /post/tag/delete` for these roles. 
Nesting these endpoints under the `/post` endpoint seems appropriate.
Shared parameters would be authentication, `post_id` and `tag_id` with an additional `delete` parameter for the delete endpoint.

Groups that should be able to add/edit the tags:
- Instance Admins of the Community
- Moderators of the Community
- Post creator

### Frontend:

**General Changes:**

Tags can be listed next to the Post title in the feed and at the end of the post in the post view.
Adding tags to a post can be implemented via a separate tag box where they can be typed in with search support. 

![tags post view iamge](https://github.com/Neshura87/Lemmy-RFC/blob/main/tags_post_view.png?raw=true)

Tags in the feed should be displayed next to the post interaction elements.
![tags feed view iamge](https://github.com/Neshura87/Lemmy-RFC/blob/main/tags_feed_view.png?raw=true)



**Moderator Changes:**

Moderators need a UI Section in the community settings for adding, edditing and deleting tags. This could also implement a way to "import" a tag from another instance.
Moderators should also be able to add tags to user posts.

# Drawbacks

Adding the tag system would increase backend complexity, increased workload for alternative frontends and apps as well as add another barrier for full federation between lemmy and other services. On top of that the moderation workload would somewhat increase since posts would need to be checked for proper usage of tags as well.

# Rationale and alternatives

The proposed system aims to be a balance between the extremely limited flair feature of reddit and the often fragmented tag system of image hosting platforms, combining the best properties of both options into a theoretically ideal version of the system. Additionally the current system with a singular NSFW flag is far too inflexible to properly moderate the scope of content that falls under NSFW. Many instances have implemented a blanket NSFW ban because of this, greatly limiting what content can be posted where. Aside from that the inability to better sort content is a often discussed topic, especially story communities are desperately craving for a better option to both sort posts based on story association as well as hide/blur posts if they contain spoilers. Given the very high community demand not implementing a tag system of some sort will likely lead to a (soft-)fork of lemmy. The feature is extremely desired by many communities. An implementation of tags via UI rendering ([] brackets for tags in text would turn into a tag on a post) would be possible but ultimately far more limiting and would lead to an incosistent experience across lemmy apps/frontends as well as an inability to properly federate the tags.

# Prior art

Similar implementations can be found on Reddit as well as a great many image hosting platforms.

Reddit's Flair System is rather restrictive, a Post can only have a single custom Flair (such as "News") greatly limiting the usefullness of the feature.

Pros:
- low moderation effort due to predefined tags
- no/little content fragmentation since typos are not possible
- no flair spam due to only 1 flair being allowed at a time

Cons:
- extreme moderation effort for 1st time setup due to inability to mix flairs
- filter lists get extremely long because all variants of a flair need to be filtered (for example "News Europe", "News America", etc. instead of just "News")
- flairs need to be created before being usable

Tag/Flair Systems on Image hosting platform generally tend to favor a more unmoderated experience, often allowing Users to type in the tags themselves.

Pros:
- enhanced user freedom when applying tags
- no delay between arising demand for tag and creation of the tag

Cons:
- greater moderation demand since tags have to be checked for inappropriate content
- very high chance for content fragmentation (typos, differences in naming things, language barriers)
- almost completely removes the ability of users to blocklist content tags (pronography would slip past a "pornography" blocklist).

# Unresolved questions

- How would tags federate/display on other Fediverse services such as Mastodon?
- ~~Should instance wide and community tags be implemented separately or together. If separately which one should be first.~~
-> resolved, community tags will be implemented first with instance tags to follow some time later.
- There is a discussion about creation permissions for tags however that changes nothing about the fundamental functionality and can be discussed after initial experience is gathered.
- Should there be a limit on how many tags can be applied? If so, should this limit be hard coded or community configurable?
-> should be made configurable, still up for debate whether to be implemented with this RFC or later on.

# Future possibilities
### Global Tags

Many tags will probably be the same across different communities. As such, it makes sense to create global tags in the future. These tag objects would still be assigned to posts by users or moderators, but their type would be `GlobalPostTag` and their ID would be global (e.g. `https://join-lemmy.org/tags/xxx`). This way, a user could configure user-global allow/block lists of tags in their Lemmy UI / App.  For example Spoiler,Gore,NSFL,Trigger Warning, ...
### Filtering of user feeds based on tags

### Pre-Selected tags determined by Instance/Community/User settings:

Allows for easier handling for cases where a few tags are used very often (example: a news community could pre-select the "News" tag, thereby automatically adding it to every users post creation form)

Addition by [RocketDerp](https://github.com/RocketDerp) (Source: [GitHub Comment](https://github.com/Neshura87/Lemmy-RFC/commit/a11727bb992aa91e89354286876d7144b61b926f#commitcomment-125201367)):

Tagging of Instances, Communitites and Users using the same system.

>There are issues open on Lemmy project to have flares for users, flares for posts. I think community itself being tagged is another thing to consider and could be a means to implement a multi-community (multi-reddit) presentation system for blending communities on a post list. 
>[...]
>Perhaps a scope smallint added to the proposed database table... scope 0 = all, 1 = community itself tagged, 2 = post tagged, 3 = user tagged. And community_id specific ones would have to be 2 = post.

### Migration and replacement of current "NSFW" flag in favor of "NSFW" tag:

The current flag used to mark posts NSFW can be replaced by a Instance wide default "NSFW" flag once the feature has been field tested.

### Tag Amount Limit:

Unlimited Tag usage might lead to undesired situations and extra moderation effort, long term the addition of Instance or Community limits to number of tags on a Post might be useful.

### Instance wide or other tags:

The basic framework should allow this system to be used in many other contexts, however due to the complexity this should be done after an initial implementation.
Areas that could use this system include:
- Instance wide post tags (for example a single instance wide NSFW tag or some such)
- User flairs
- Tags on Communities (as opposed to on Posts)

### Tag Inheritance:

Initially Tags will have a flat structure, however an inheritance system could be introduced later on to easy usability for both users and moderators.
Such a system would add an attribute `inherits` to the tag object which would then reference the URL of the parent tag. If no or an empty `inherits` field is given the tag is a "root" tag and has no parent.
The inheritance trees could then be used to enhance search filtering, tag settings upon creation and/or cosmetic behaviour of tags.