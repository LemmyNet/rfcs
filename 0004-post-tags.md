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

For example a lot of communities offer mixed content such as Memes, News and Discussions. Some people would like to only see News or Discussions. In such a case they could blacklist (filter out) the "Meme" tag, going forward they would not be presented with any Memes in that community.

Another use for tags is better handling of how posts are displayed. For example some people would like the content of a post to stay hidden for all users. This could be the case in a community focused on release discussions of Books, Movies, TV Shows or communities with content some users might find offensive but which is not neccesarily NSFW (for example jokes about certain topics).

Essentially the tags serve as an easier way to message what is *inside* the post. Combining multiple tags allows for even finer control. Made a post about news in poland? Add the "News" and "Poland" tags to it and people interested exclusively in polish news can find it easily or people not interested can avoid it just as easily.

# Reference-level explanation

### Protocol:
According to https://www.w3.org/TR/activitystreams-vocabulary/#dfn-tags a general tag object exists in the ActivityPub protocol. 
Entirely unmoderated tags are not an option for lemmy as the moderation workload would be too much. Additionally users being able to type out tags themselves introduces splintering in the tag contents due to typos. A better solution is a curated list of tags users can attach to their posts. The list of tags can be maintained by both admins and moderators allowing for each community to tailor tags to their specific needs.
Content for both sets of tags would be located in their respective root:

- Instance Tags: `https://example.org/t/tag`
- Community Tags: `https://example.org/c/instance/t/tag`

The tag URL can then be utilized as a unique identifier for the tag, populating the `id` field of the tag object. Using the object ID optional tag federation can also be achieved, allowing for communities across multiple instances to share content via tags (example: News tag shared across instances). This would also solve the issue of splintered communities across instances while not forcing it on the communities in question. For now tags will only be applicable to posts, however the general design allows for them to be attached to any kind of object later on, be that instance, community or user. The limited initial scope allows for easier modifications should any rough edges or missing features be discovered.

To fill all needed use cases 3 tag types will be needed:

- NSFW
- Content Warning
- Generic Tag

Theoretically NSFW could be implemented using a preset "Content Warning" tag but seperating out this tag allows instances to better filter it out for moderation purposes (for example if no admin/moderator is willing to moderate NSFW content).
Both "NSFW" and "Content Warning" tags should blur the post body by default. Additionally the `sensitive` post flag to `true` should either of these types be present in the post tags to ensure correct handling on other fediverse platforms.

Example Tag Objects:

**Generic Tag:**
```json
{
    "id": "https://example.org/t/tag",
    "name": "Generic Tag"
}
```

**NSFW Tag:**
```json
{
    "type": "nsfw",
    "id": "https://example.org/t/tag",
    "name": "NSFW"
}
```

**Spoiler Tag:**
```json
{
    "type": "cw",
    "id": "https://example.org/t/tag",
    "name": "Spoiler"
}
```

**Remote Instance Spoiler Tag:**
```json
{
    "type": "cw",
    "id": "https://example.org/t/tag@example_remote.org",
    "name": "Spoiler"
}
```

Below is an example of a News post in the news community. The post has a communtiy tag for the newspaper company that published it, as well as a content warning for Blood/Gore. In order to share this news post across instances it also is tagged with the news tag of a remote instance.

Example Post json:

``` json
{
  "content": ". . .",
  "sensitive": true,
  "tag": [
    {
      "id": "https://example.org/c/news/t/newspaper_company_1",
      "name": "Newspaper Company Tag"
    },
    {
      "type": "cw",
      "id": "https://example.org/t/blood",
      "name": "Blood/Gore"
    },
    {
      "id": "https://example.org/t/news@example_remote.org",
      "name": "News"
    }
  ]
}
```

### Backend:
I am not quite familiar with the Lemmy Backend so any corrections or additions for this section are appreciated.

Two new tables for "tags" would be required. One for instance wide tags and one for community tags. These new tables would consist of the columns `url`, `name`, `type`, `deleted`, `ìd` and `community_id` (only on the community tag table). Additional columns for tag display style (for example background & text color) could be added later by extending this table. Splitting the instance and community tags allows the `community_id` field to be non-nullable, making orphaned community tags impossible (Thanks to [0WN463](https://github.com/0WN463) for the [idea](https://github.com/Neshura87/Lemmy-RFC/issues/2)).

Additionally a table for listing the tags on posts would be needed as some Databases (at least Postgres) do not support Foreign Keys in Arrays. Since the tags are split into two tables there would need to be two of these meta tables as well. The tables would consist of two foreign key columns: `tag_id` and `post_id` with `tag_id` being linked to the id column in the respective instance or community tag table and `post_id` being linked to the id column of the post table.

Instance Admins should have the option to delete/ban tags.
Instances with nsfw disables/blocked could/should auto reject posts with an nsfw type tag.

The following API routes will need to be added:

**GET /tag/list:**
Parameters:

- community_id (optional) 
- auth (optional)

Returns:
List of all tags, if a community id is specified only tags of that communtiy are returned.
Deleted tags should not be returned in the list unless an admin/moderator.

```json
{
    "tags": [
        {
            "url": "https://example.org/t/tag",
            "name": "Generic Tag",
            "id": 1,
            "community_id": 1
        },
        {...}
    ]
}
```

**GET /tag:**
Parameters:

- tag_id
- auth (optional)

Returns:
Returns the information for a single tag.
Deleted tags should not be returned unless an admin/moderator.

```json
{
    "tag": 
    {
        "url": "https://example.org/t/tag",
        "name": "Generic Tag",
        "id": 1,
        "community_id": 1
    }
}
```

**POST /tag:**
Parameters:

- name
- type
- community_id (optional)
- auth

Tag URL will be generated using the name and communtiy id.

Returns:
Object of freshly created tag

**PUT /tag:**
Parameters:

- id
- name
- type
- auth

Replaces tag name and type with the provided info. Community Id will not be consumed as a parameter as tags should not be community transferable.
The new Tag URL will be generated using the name and communtiy id.

Returns:
Object of freshly created tag

**POST /tag/delete:**
Parameters:

- id
- deleted
- auth

Deletes the tag associated with the id. Updates `deleted` field in table. Allows for un-deleting a tag.

Returns:
Object of freshly created tag


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

Moderators need a UI Section in the community/instance settings for adding, edditing and deleting tags. This could also implement a way to "import" a tag from another instance.
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
- almost completely removes the ability of users to blacklist content tags (pronography would slip past a "pornography" blacklist).

# Unresolved questions

- How would tags federate/display on other Fediverse services such as Mastodon?
- Should instance wide and community tags be implemented separately or together. If separately which one should be first.
- There is a discussion about creation permissions for tags however that changes nothing about the fundamental functionality and can be discussed after initial experience is gathered.
- Should there be a limit on how many tags can be applied? If so, should this limit be hard coded or instance or community configurable?

# Future possibilities

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

