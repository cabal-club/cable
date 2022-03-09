### request channel time range (`msg_type=4`)

Request the hashes of all posts in a channel between a time start and end.

field         | type              | desc
--------------|-------------------|----------------------------
msg\_len      | varint            | number of bytes in this message
msg\_type     | varint            |
req\_id       | u8[4]             | unique id of this request (random)
circuit\_id   | u8[4]             | circuit for an established path or `[0,0,0,0]` for no circuit
ttl           | varint            | number of hops remaining
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
time\_start   | varint            | seconds since the epoch
time\_end     | varint            | seconds since the epoch
limit         | varint            | maximum number of records to return

If `time_end` is 0, request all messages since `time_start` and respond with more results as they
arrive, up to `limit` number of results.

### request channel state (`msg_type=5`)

Request the state of a channel and subscribe to updates.

field         | type              | desc
--------------|-------------------|-----------------------------------
msg\_len      | varint            | number of bytes in this message
msg\_type     | varint            |
req\_id       | u8[4]             | unique id of this request (random)
circuit\_id   | u8[4]             | circuit for an established path or `[0,0,0,0]` for no circuit
ttl           | varint            | number of hops remaining
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
limit         | varint            | maximum number of records to return
updates       | varint            | maximum number of live updates to return

The response is a list of hashes that pertain to posts that encompass the current state of the
channel: parts, joins, topic changes, deletes in this channel, info updates for users in this
channel.

Set `limit` to 0 to not receive any historical updates and set `updates` to 0 to not receive any
live updates.

The responses will keep coming until this request is cancelled or until the `updates` limit is
reached.

### request channel list (`msg_type=6`)

Request a list of known channels from peers.

field        | type    | desc
-------------|---------|-----------------------------------
msg\_len     | varint  | number of bytes in this message
msg\_type    | varint  |
req\_id      | u8[4]   | unique id of this request (random)
circuit\_id  | u8[4]   | circuit for an established path or `[0,0,0,0]` for no circuit
ttl          | varint  | number of hops remaining
limit        | varint  | maximum number of records to return

The results for this query are provided by a data response.

# post

Each post type begins with the same 4 fields: 

field       | type   | desc
------------|--------|-------------------------------------------------------
public\_key | u8[32] | ed25519 key that authored this post
signature   | u8[64] | ed25519 signature of the fields that follow
link        | u8[32] | blake2b hash of latest message in this channel/context
post\_type  | varint | see custom post type sections below

The post type sections below document the fields that follow these initial 4 fields depending on the
`post_type`. Most post types will link to the most recent post in a channel from any user (from
their perspective) but self-actions such as naming or moderation will link to the most recent
self-action.

You can use `crypto_sign()` in combined mode to generate the signature field for the fields that
come after `signature`, as `crypto_sign()` will prepend the signature into the output, so the fields
will be in the correct order.

Use `crypto_generichash()` with the default settings (`crypto_generichash_BYTES=32`) to hash
incoming messages in order to resolve links. Hash the entire message with all fields. 

When you "make a post", you are only writing to some local storage indexed by the hash of the
content. Posts only get sent to other peers in response to queries for content matching certain
criteria.

The `post_type` is a varint, so if the post types below are inadequate, you can create your own
using unused numbers (`>64`).

Use a string of 32 zeros if there is nothing to link to.

Clients should ignore posts with a `post_type` that they don't understand or support.

## post/text (`post_type=0`)

Post a message in a channel.

field         | type              | desc
--------------|-------------------|---------------------------------
public\_key   | u8[32]            | ed25519 key that authored this post
signature     | u8[64]            | ed25519 signature of the fields that follow
link          | u8[32]            | blake2b hash of latest message in this channel/context
post\_type    | varint            | see custom post type sections below
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
timestamp     | varint            | seconds since unix epoch
text\_size    | varint            | length of the text field
text          | u8[text\_size]    | message content

## post/delete (`post_type=1`)

Request that peers delete a post by its hash.

field        | type             | desc
-------------|------------------|-------------------------
public\_key  | u8[32]           | ed25519 key that authored this post
signature    | u8[64]           | ed25519 signature of the fields that follow
link         | u8[32]           | blake2b hash of latest message in this channel/context
post\_type   | varint           | see custom post type sections below
timestamp    | varint           | seconds since unix epoch
hash         | u8[32]           | blake2b hash of post

Clients may choose to interpret this message based on their moderation perspective.

Clients that store data should subscribe to delete requests.

## post/info (`post_type=2`)

Set public information about yourself.

field       | type            | desc
------------|-----------------|-------------------------
public\_key | u8[32]          | ed25519 key that authored this post
signature   | u8[64]          | ed25519 signature of the fields that follow
link        | u8[32]          | blake2b hash of latest message in this channel/context
post\_type  | varint          | see custom post type sections below
timestamp   | varint          | seconds since unix epoch
key\_size   | varint          | length of the name field
key         | u8[key\_size]   | name string
value\_size | varint          | length of the value field
value       | u8[value\_size] | value

Recommended fields for clients to support:

key      | desc
---------|------------------------------------------------
name     | handle to use as a pseudonym
max\_age | string maximum number of seconds to store posts
blocks   | json object mapping hex keys to flag objects
hides    | json object mapping hex keys to flag objects

Set `max_age=0` to instruct peers to delete all data about you.

To save space, discard old versions of these messages on disk.

Flag objects have these properties (all optional):

* reason (string)
* timestamp (seconds since epoch, number)

NOTE: use of json here means clients need a json implementation.
might want to swap with custom binary

## post/topic (`post_type=3`)

Set a topic for a channel.

field         | type              | desc
--------------|-------------------|-------------------------------------------------------
public\_key   | u8[32]            | ed25519 key that authored this post
signature     | u8[64]            | ed25519 signature of the fields that follow
link          | u8[32]            | blake2b hash of latest message in this channel/context
post\_type    | varint            | see custom post type sections below
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
timestamp     | varint            | seconds since unix epoch
topic\_size   | varint            | length of the topic field
topic         | u8[topic\_size]   | topic content

Depending on moderation settings, other peers may choose to accept or reject your choice of topic.

## post/join (`post_type=4`)

Join a channel.

field         | type              | desc
--------------|-------------------|-------------------------------------------------------
public\_key   | u8[32]            | ed25519 key that authored this post
signature     | u8[64]            | ed25519 signature of the fields that follow
link          | u8[32]            | blake2b hash of latest message in this channel/context
post\_type    | varint            | see custom post type sections below
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
timestamp     | varint            | seconds since unix epoch

Peers can obtain a link to anchor their join message by requesting a list of channels.

## post/leave (`post_type=5`)

Leave (part) a channel.

field         | type              | desc
--------------|-------------------|-------------------------------------------------------
public\_key   | u8[32]            | ed25519 key that authored this post
signature     | u8[64]            | ed25519 signature of the fields that follow
link          | u8[32]            | blake2b hash of latest message in this channel/context
post\_type    | varint            | see custom post type sections below
channel\_size | varint            | length of the channel in bytes
channel       | u8[channel\_size] | channel name as a string of text
timestamp     | varint            | seconds since unix epoch

