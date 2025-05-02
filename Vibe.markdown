# Vibe.fyi API

## Overview

In partnership with [Firebase](https://firebase.google.com/), we're making the public Vibe.fyi data available in near real time. Firebase enables easy access from [Android](https://firebase.google.com/docs/android/setup), [iOS](https://firebase.google.com/docs/ios/setup), and the [web](https://firebase.google.com/docs/web/setup). [Servers](https://firebase.google.com/docs/server/setup) aren't left out.

If you can use one of the many [Firebase client libraries](https://firebase.google.com/docs/libraries/), you really should. The libraries handle networking efficiently and can raise events when things change. Be sure to check them out.

Please email api@vibe.fyi if you find any bugs.

## URI and Versioning

We hope to improve the API over time. The changes won't always be backward compatible, so we're going to use versioning. This first iteration will have URIs prefixed with `https://vibe-fyi.firebaseio.com/v0/` and is structured as described below. There is currently no rate limit.

For versioning purposes, only removal of a non-optional field or alteration of an existing field will be considered incompatible changes. *Clients should gracefully handle additional fields they don't expect, and simply ignore them.*

## Design

The v0 API is essentially a dump of our in-memory data structures. We know, what works great locally in memory isn't so hot over the network. Many of the awkward things are just the way Vibe.fyi works internally. Want to know the total number of comments on a post? Traverse the tree and count. Want to know the children of an item? Load the item and get their IDs, then load them. The newest page? Starts at item maxid and walks backward, keeping only the top level posts. Same for featured or trending posts.

I'm not saying this to defend it - It's not the ideal public API, but it's the one we could release in the time we had. While awkward, it's possible to implement most of Vibe.fyi using it.

## Items

Posts, comments, and other content types are just items. They're identified by their IDs, which are unique integers, and live under `/v0/item/<id>.json`.

All items have some of the following properties, with required properties in bold:

Field | Description
------|------------
**id** | The item's unique ID.
deleted | `true` if the item is deleted.
type | The type of item. One of "post", "comment", or "poll".
by | The username of the item's author.
time | Creation date of the item, in [Unix Time](http://en.wikipedia.org/wiki/Unix_time).
text | The comment or post text. HTML.
dead | `true` if the item is dead.
parent | The comment's parent: either another comment or the relevant post.
poll | The poll option's associated poll.
kids | The IDs of the item's comments, in ranked display order.
url | The URL of the post.
score | The post's score, or the votes for a poll option.
title | The title of the post or poll. HTML.
parts | A list of related poll options, in display order.
descendants | In the case of posts or polls, the total comment count.

For example, a post: https://vibe-fyi.firebaseio.com/v0/item/10001.json?print=pretty

```javascript
{
  "by": "vibeuser",
  "descendants": 25,
  "id": 10001,
  "kids": [10002, 10003, 10004],
  "score": 85,
  "time": 1735689600,
  "title": "Introducing Vibe.fyi: A New Way to Share",
  "type": "post",
  "url": "https://vibe.fyi/intro"
}
```

comment: https://vibe-fyi.firebaseio.com/v0/item/10002.json?print=pretty

```javascript
{
  "by": "commenter1",
  "id": 10002,
  "kids": [10005, 10006],
  "parent": 10001,
  "text": "This is an exciting platform! Looking forward to more features.",
  "time": 1735693200,
  "type": "comment"
}
```

poll: https://vibe-fyi.firebaseio.com/v0/item/10003.json?print=pretty

```javascript
{
  "by": "pollcreator",
  "descendants": 10,
  "id": 10003,
  "kids": [10007, 10008],
  "parts": [10004, 10005],
  "score": 30,
  "text": "",
  "time": 1735700400,
  "title": "Poll: What's your favorite feature on Vibe.fyi?",
  "type": "poll"
}
```

and one of its parts: https://vibe-fyi.firebaseio.com/v0/item/10004.json?print=pretty

```javascript
{
  "by": "pollcreator",
  "id": 10004,
  "poll": 10003,
  "score": 15,
  "text": "Real-time commenting",
  "time": 1735704000,
  "type": "pollopt"
}
```

## Users

Users are identified by case-sensitive IDs and live under `/v0/user/`. Only users that have public activity (comments or post submissions) on the site are available through the API.

Field | Description
------|------------
**id** | The user's unique username. Case-sensitive. Required.
**created** | Creation date of the user, in [Unix Time](http://en.wikipedia.org/wiki/Unix_time).
**karma** | The user's karma.
about | The user's optional self-description. HTML.
submitted | List of the user's posts, polls, and comments.

For example: https://vibe-fyi.firebaseio.com/v0/user/vibeuser.json?print=pretty

```javascript
{
  "about": "Passionate about community-driven platforms.",
  "created": 1735603200,
  "id": "vibeuser",
  "karma": 150,
  "submitted": [10001, 10002, 10003]
}
```

## Live Data

The coolest part of Firebase is its support for change notifications. While you can subscribe to individual items and profiles, you'll need to use the following to observe front page ranking, new items, and new profiles.

### Max Item ID

The current largest item ID is at `/v0/maxitem`. You can walk backward from here to discover all items.

Example: https://vibe-fyi.firebaseio.com/v0/maxitem.json?print=pretty

```javascript
10005
```

### New, Top, and Trending Posts

Up to 500 top and new posts are at `/v0/topposts` and `/v0/newposts`. Trending posts are at `/v0/trendingposts`.

Example: https://vibe-fyi.firebaseio.com/v0/topposts.json?print=pretty

```javascript
[10001, 10003, 10002]
```

### Featured Posts

Up to 200 of the latest featured posts are at `/v0/featuredposts`.

Example: https://vibe-fyi.firebaseio.com/v0/featuredposts.json?print=pretty

```javascript
[10001, 10003]
```

### Changed Items and Profiles

The item and profile changes are at `/v0/updates`.

Example: https://vibe-fyi.firebaseio.com/v0/updates.json?print=pretty

```javascript
{
  "items": [10001, 10002, 10003],
  "profiles": ["vibeuser", "commenter1", "pollcreator"]
}
```