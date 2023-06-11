This document presents a database schema for the first version of the project.
The fields were gathered by browsing through competitive UIs and narrowing them down to the most important bits.
Not all database tables are listed here.
For example, the schema for chat is in a separate document.

For large images we store both `url` and [`blurhash`][blurhash] strings.
[Blurhash][blurhash] generates 20-30 character strings that are used to display a blurred version of the image while the actual image is loading.

# `users`

- `primary_email` (text, index)
- `salt` (char 16)
- `pwd`
  - bcrypt(`salt` + raw pwd)
- `display_name` (text)
  - can be changed as long as it doesn't change the `slug`
- `slug` (text, index)
  - unique
  - generated from `display_name` hence no need for unique constraint there
  - used in URLs
- `bio` (text, optional)
- `avatar_url` (text, optional)

# `user_kvs`

Key-value store for optional user data.
Data which won't need filtering or that have an unclear schema are stored here.
Helps with keeping the `users` table clean and row size small.
We keep an index on (`user_id`, `key`).

- `user_id` (fk)
- `k` (text)
- `v` (text)

User keys:

- `profile_banner`
  - value is a JSON in format `{"url": "...", "blurhash": "..."}`
- `twitter_url`
- `ig_url`
- `yt_url`
- `discord_url`

Streamer-only keys:

- `about`
  - value used in streamer profile page
  - value is a JSON in format `{"col1":"html to render", "col2": "...", "col3": "..."}`
- `thumbnail`
  - the streamer can set a thumbnail for their stream
  - value is a JSON in format `{"url": "...", "blurhash": "..."}`
- `preview`
  - if the thumbnail is not set, we default to a preview
  - the preview image can also be shown when user hovers over the streamer
  - value is a JSON in format `{"url": "...", "blurhash": "..."}`

# `streamers`

- `user_id` (fk, index)
- `title` (text, optional)
  - streamer can change this at any time, and hence UI must update without refresh
- `go_live_notification` (text, optional)
  - next time the streamer presses the _Go Live_ button, this is shown to followers
- `category_id` (fk, optional, index)
  - index because we want to display a list of streamers for a category
  - streamer can change this at any time, and hence UI must update without refresh
- `live_since` (timestamp, optional)
  - if null, then streamer is offline
  - once the streamer goes offline, their broadcast is saved in `past_broadcasts`
- `active_viewers`
  - TBD on how this is calculated and updated

# `streamers_view`

Postgres view.
Additionally from `streamers` table data also joins data from other tables to contain:

- `followers` (int)
- `subscribers` (int)

# `streamer_tags`

- `streamer_id` (fk, index)
- `tag_id` (fk, index)
  - index because we want to display list of streamers for a given tag

# `category_tags`

These tags are assumed for a streamer when they select a category.

- `category_id` (fk, index)
- `tag_id` (fk)

# `tags`

- `slug` (text, index)
  - unique
- `name` (text)

# `categories`

- `slug` (text, index)
- `name` (text)
- `description` (text)
- `thumbnail_url` (text)

# `past_broadcasts`

- `streamer_id` (fk, index)
- `started_at` (timestamp)
- `ended_at` (timestamp)
- `rec_url` (text, optional)
  - we keep the recording only for X days in our S3 bucket
  - when a recording is deleted, this field is set to null
- `viewers_timeline` (text)
  - a timeseries of snapshosts of active_viewers
  - a JSON array of 2-member integer tuples where the first member represents seconds since `started_at` and the second member represents the number of active viewers at that time
  - e.g. `[[0, 0], [10, 3], [20, 15], ...]`

# `follows`

- `streamer_id` (fk, index)
- `user_id` (fk, index)
- `created_at` (timestamp)

# `subscriptions`

- `streamer_id` (fk, index)
- `user_id` (fk, index)
- `streak` (int)
  - number of consecutive months the user has subscribed
- `cumulative` (int)
  - number of months the user has subscribed in total
  - ie. how many times have we charged the user
- `created_at` (timestamp)
  - when the user subscribed for the first time

<!-- List of References -->

[blurhash]: https://github.com/woltapp/blurhash
