# cable (cabal protocol)

This document describes the bytes over the wire to speak the cable protocol.

You will need [libsodium](https://doc.libsodium.org/) bindings or their equivalents for:

* `crypto_generichash()` - to hash messages with blake2
* `crypto_sign_keypair()` - to generate public and secret ed25519 keys
* `crypto_sign()` - to calculate the signature of a post (in combined mode)
* `crypto_sign_open()` - to verify the signature of a post (in combined mode)

This does not include encryption or authentication of the connection, which are provided by other
layers. For example, if you are using i2p, the network already provides you with an authentication
mechanism as part of the network routing and messages are already encrypted. Otherwise you might
want to use [noise][] yourself ([which is what i2p uses][ntcp2]).

[noise]: http://noiseprotocol.org/
[ntcp2]: https://geti2p.net/spec/ntcp2

# goals

* simple to implement in any language with only libsodium dependency
* bridge across different network transports
* partial implementations are still useful
* sparse access
* compact over the wire
* use any kind of database

# Definitions
## "client"
An instance of a running implementation of cable.

## User
A pair of ED25519 keys (public and private) is all that is needed to constitute
a "user".

## Channel Membership
A user is considered a member of a channel at a particular point in time if,
from the client's perspective, that user has issued a `post/join` to that
channel and has not issued a matching `post/leave` since.

## Channel State
The state of the channel at any given moment from the perspective of a client
is fully described by all of the following:
- The latest of each user's `post/join` or `post/leave` post to the channel.
- The latest `post/info` post of each user who is *or was* a member of the
  channel.
- The latest `post/topic` post to channel, made by any user, regardless of
  current or past membership.

## Binary format tables
The binary format of messages and posts are described in this document using tables like the following:

field      | type     | desc
-----------|----------|-------------------------------------------------------------
`foo`      | `u8`     | description of the field `foo`
`bar`      | `u8[4]`  | description of the field `bar`

This example describes a binary payload that is 5 bytes long, where the one byte of field `foo` is followed immediately by the 4 bytes describing `bar`.

If `foo=17` and `bar=[3,6,8,64]`, the following binary payload would be expected:

```
0x12 0x03 0x06 0x08 0x40

 ^     ^^^^^^^^^^^^^^^
 |            |------------- bar = [3, 6, 8, 64]
 |
 |-------------------------- foo = 17
```

The following data types are used:
- `u8`: a single unsigned byte
- `u8[N]`: a sequence of exactly `N` unsigned bytes
- `varint`: a variable-length unsigned integer. cable uses protobuf-style [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints). (For an example implementation of varint encoding/decoding, see the [nodejs varint package](https://www.npmjs.com/package/varint).)


# messages

All messages begin with a `msg_len` and a `msg_type` varint:

field      | type     | desc
-----------|----------|-------------------------------------------------------------
`msg_len`  | `varint` | number of bytes in rest of message, i.e. not including the `msg_len` field
`msg_type` | `varint` | see fields below

More fields follow after the `msg_type`.

Requests and responses all have a unique `msg_type`.

Clients may experiment with custom message types beyond the ids used by this specification
(where `msg_type>=64`).

## responses

There are 2 types of responses:

* hash response - a list of hashes (most queries return this)
* data response - a list of data chunks (in response to a hash query)

Multiple responses may be generated for a single request and results trickle in from peers.

### hash response (`msg_type=0`)

Respond with a list of hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint (=0)`       |
`req_id`     | `u8[4]`             | id this is in response to
`hash_count` | `varint`            | number of hashes in the response
`hashes`     | `u8[hash_count*32]` | blake2b hashes concatenated together

### data response (`msg_type=1`)

Respond with a list of results for data lookups by hash.

field        | type                | desc
-------------|---------------------|--------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint` (=1)       |
`req_id`     | `u8[4]`             | id that this is in response to
`data0_len`  | `varint`            | length of first data payload
`data0`      | `u8[data_len]`      | first data payload
`data1_len`  | `varint`            | length of second data payload
`data1`      | `u8[data_len]`      | second data payload
`...`        |                     |
`dataN_len`  | `varint`            | length of Nth data payload
`dataN`      | `u8[data_len]`      | Nth data payload

A recipient reads zero or more (`data_len`,`data`) pairs until `data_len` is 0.

## requests

Each request begins with these two fields (after the `msg_type` applicable to all messages):

field      | type       | desc
-----------|------------|-----------------------------------
`msg_len`  | `varint`   | number of bytes in this message
`msg_type` | `varint`   |
`req_id`   | `u8[4]`    | unique id of this request (random)
`ttl`      | `varint`   | number of hops remaining

More fields follow for different request types below.

When `ttl > 0`, a peer can choose to forward a request along to other peers and forward the response
back to the peer that made the request. Each peer performing this action should decrement the `ttl`
by one.

When forwarding a request, do not change the `req_id` so that routing loops can be more easily
detected by peers.

### request by hash (`msg_type=2`)

Request data for a set of hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint`            |
`req_id`     | `u8[4]`             | unique id of this request (random)
`ttl`        | `varint`            | number of hops remaining
`hash_count` | `varint`            | number of hashes to request
`hashes`     | `u8[32*hash_count]` | blake2b hashes concatenated together

Results are provided by a data response (`msg_type=1`).

### cancel request (`msg_type=3`)

Indicate a desire to stop receiving responses for any request.

Some requests stay open and wait for data to arrive. You can close these long-running subscriptions
using a cancel request.

field        | type                | desc
-------------|---------------------|-----------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint`            |
`req_id`     | `u8[4]`             | stop receiving results for this id
`ttl`        | `varint`            | ignored

This request should be passed along to any peers to which this peer has forwarded the original request.

### request channel time range (`msg_type=4`)

Request text posts and text post deletions written to a channel between a start and end time.

field          | type               | desc
---------------|--------------------|----------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`time_start`   | `varint`           | seconds since the epoch
`time_end`     | `varint`           | seconds since the epoch
`limit`        | `varint`           | maximum number of records to return

This request returns 0 or more `hash response` responses.

This request expects the hashes of all `post/text` and `post/delete` posts made
to a channel by members between `time_start` and `time_end`.

If `time_end` is 0, request all messages since `time_start` and respond with
more posts as they arrive, up to `limit` number of posts.

### request channel state (`msg_type=5`)

Request posts that describe the state of a channel and its users, and
optionally subscribe to future state changes.

field          | type               | desc
---------------|--------------------|-----------------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`historic`     | `varint`           | set to `1` to receive peer's view of current channel state; `0` to not
`updates`      | `varint`           | maximum number of live / future posts to return

This request expects 0 or more `hash response`s in response, that pertain to
posts that describe the current state of the channel. See "Channel State" under
"Definitions" for details on what posts comprise a channel's current state.

If `historic` is set to `1`, this request expects *all* historic posts that
make up the channel state to be returned, followed by up to `updates` number of
live posts that further alter this channel's state.

Set `updates` to 0 to not receive any live / future state changes.

Responses from a peer will keep coming until
- this request is cancelled, or
- `updates` live post hashes are returned (if `updates >= 1`), and
- all known historic hashes are returned (if `historic == 1`)

### request channel list (`msg_type=6`)

Request a list of known channels from peers.

field          | type               | desc
---------------|--------------------|-----------------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`limit`        | `varint`           | maximum number of records to return

# post

Each post type begins with the same 5 fields:

field        | type              | desc
-------------|-------------------|-------------------------------------------------------
`public_key` | `u8[32]`          | ed25519 key that authored this post
`signature`  | `u8[64]`          | ed25519 signature of the fields that follow
`num_links`  | `varint`          | how many blake2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]`| blake2b hashes of the latest messages in this channel/context
`post_type`  | `varint`          | see custom post type sections below

The post type sections below document the fields that follow these initial 5 fields depending on the
`post_type`. Most post types will link to the most recent posts in a channel from any user (from
their perspective) but self-actions such as setting a nickname link to the most recent
self-action. Refer to each `post/*` type for what to expect.

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

Specify `num_links=0` if there is nothing to link to.

Clients should ignore posts with a `post_type` that they don't understand or support.

## post/text (`post_type=0`)

Post a message in a channel.

field          | type               | desc
---------------|--------------------|---------------------------------
`public_key`   | `u8[32]`           | ed25519 key that authored this post
`signature`    | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`    | `varint`           | how many blake2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`timestamp`    | `varint`           | seconds since unix epoch
`text_size`    | `varint`           | length of the text field
`text`         | `u8[text_size]`    | message content

## post/delete (`post_type=1`)

Request that peers delete a post by its hash.

field        | type               | desc
-------------|--------------------|-------------------------
`public_key` | `u8[32]`           | ed25519 key that authored this post
`signature`  | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`  | `varint`           | how many blake2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`  | `varint`           | see custom post type sections below
`timestamp`  | `varint`           | seconds since unix epoch
`hash`       | `u8[32]`           | blake2b hash of post to be deleted

## post/info (`post_type=2`)

Set public information about yourself.

field        | type               | desc
-------------|--------------------|-------------------------
`public_key` | `u8[32]`           | ed25519 key that authored this post
`signature`  | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`  | `varint`           | how many blake2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`  | `varint`           | see custom post type sections below
`timestamp`  | `varint`           | seconds since unix epoch
`key_size`   | `varint`           | length of the name field
`key`        | `u8[key_size]`     | name string
`value_size` | `varint`           | length of the value field
`value`      | `u8[value_size]`   | value

Recommended fields for clients to support:

key       | desc
----------|------------------------------------------------
`name`    | handle to use as a pseudonym
`max_age` | string maximum number of seconds to store posts

Set `max_age=0` to instruct peers to delete all data about you.

To save space, discard old versions of these messages on disk.

Flag objects have these properties (all optional):

* reason (string)
* timestamp (seconds since epoch, number)

NOTE: use of json here means clients need a json implementation.
might want to swap with custom binary

## post/topic (`post_type=3`)

Set a topic for a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ed25519 key that authored this post
`signature`    | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`    | `varint`           | how many blake2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`timestamp`    | `varint`           | seconds since unix epoch
`topic_size`   | `varint`           | length of the topic field
`topic`        | `u8[topic_size]`   | topic content

## post/join (`post_type=4`)

Join a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ed25519 key that authored this post
`signature`    | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`    | `varint`           | how many blake2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`timestamp`    | `varint`           | seconds since unix epoch

Peers can obtain a link to anchor their join message by requesting a list of channels.

## post/leave (`post_type=5`)

Leave (part) a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ed25519 key that authored this post
`signature`    | `u8[64]`           | ed25519 signature of the fields that follow
`num_links`    | `varint`           | how many blake2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | blake2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`channel_size` | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_size]` | channel name as a string of text
`timestamp`    | `varint`           | seconds since unix epoch

