# cable

Version: 1.0 (draft)

Author: Kira Oakley

Contributors: Alexander Cobleigh, Noelle Leigh, Henry (cryptix)

## Abstract

This document describes the cable wire protocol. That is, the specific bytes
sent "over the wire" between peers wishing to speak the protocol to each other.

## 0. Background
Cabal is a distributed peer-to-peer computer program for private group chat. It
operates in a fashion differently from the typical server-client model, where
no machine is either an official nor de-facto authority over others in the
network. Instead, peers collaborate with each other to share documents to build
an eventually-consistent view of the shared data.

## 1. Introduction
The purpose of the cable wire protocol is the facilitation of members of a
Cabal group chat to exchange crytographically signed documents with each other,
such as chat messages, spread across various user-defined topics.

Cable is designed to be:
* fairly simple to implement in any language with only a single dependency (libsodium)
* able to bridge across different network transports
* useful, even if written as a partial implementation
* efficient in its use of network resources, by syncing relevant subsets of the full dataset
* compact over the wire
* independent of whatever kind of database an implementation may use

## 1.1 Scope
This protocol does NOT include encryption or authentication of the connection.
It may be provided by other layers in userland(e.g. Tor, I2P) or expanded upon
in future iterations of this document. In particular, a future handshake
protocol is planned, which will handle authenticating that connected peers are
members of the same Cabal as the host machine.

As such, it is assumed that peers speaking this protocol have proven to be
legitimate members of the cabal. See Security Considerations (below) for an
analysis of the types of anticipated attacks that even proper cabal members may
carry out and their mitigations.


## 2. Definitions
### Cabal
A private group chat that a number of users can participate in, comprised of **users** and zero or more **channel**s.

### Channel
A conceptual object with its own unique name, a set of member **user**s, and a set of chat **post**s written to it.

### User
An ED25519 pair of keys identifying a person: a **public key**, and a **private key**, for use within a single cabal. *User* and *member* may be used interchangeably.

### Public Key
An ED25519 key, which constitutes a user's public-facing identity within a cabal.

### Private Key
An ED25519 key, used for signing authored **post**s. Kept private and secret to all but the user who owns it.

### Post
An authored binary payload, signed by the private key of its creator user.

### Client
An running instance of an implementation of cable.

### Message
An informative binary payload sent by and received from other cable peers. Each
message is either a **request** or a **response**.

### Peer
A machine that a client is connected to over some network protocol who is also speaking the cable protocol.

### Request
A message originating from a particular **peer**, identified by a unique request ID.

### Response
A message, traversing the network to the peer who originally made a request with the same request ID.

### Hash
A 32-byte BLAKE2b digest of a particular sequence of bytes.

### Link
A **hash**, which acts as a reference to the post which hashes to said hash.

### UNIX Epoch
Midnight on January 1st, 1970.

### UNIX Time
A point in time, represented by the number of seconds since the UNIX Epoch. Here, this value is assumed to be non-negative, meaning dates before the UNIX Epoch can not be represented.

## 3. Software Dependencies

### 3.1 Cryptography

Implementing cable requires access to implementations of the following:

- [BLAKE2b](https://www.rfc-editor.org/rfc/rfc7693.txt) - Hash function described by RFC 7693. To be set to output 32-byte digests.
- [ED25519](https://ed25519.cr.yp.to/) - A public-key signature system. Used to generate, sign posts, and verify post signatures.

This cryptographic functionality can be provided by [libsodium](https://libsodium.org) 1.0.18-stable, if bindings exist for one's implementation language of choice. In particular, these functions may be utilized, with their default settings:

* `crypto_generichash()` - to hash messages with BLAKE2b
* `crypto_sign_keypair()` - to generate public and secret ED25519 keys
* `crypto_sign()` - to calculate the signature of a post (in combined mode)
* `crypto_sign_open()` - to verify the signature of a post (in combined mode)

### 3.1.1 BLAKE2b Parameters
The following are the general parameters to be used with BLAKE2b. If you are using 1.0.18-stable of libsodium, these are already set by default.

- Digest byte length: 32
- Key byte length (not used): 0
- Salt (hexadecimal): `5b6b 41ed 9b34 3fe0`
- Personalization (hexadecimal): `5126 fb2a 3740 0d2a`


## 4. data model & format (wip)

### high-level data model
- users
  - which values win & how
  - name string restrictions
  - key string restrictions
  - value string restrictions
- channels (fields/properties, verbs upon them, how chat msgs in channels are sorted)
  - which values win & how
  - name string restrictions
  - topic string restrictions
  - chat msg string restrictions

### mid-level model: posts, sync, ingestion, materialized views
- posts, their fields, their handling, ingestion

### low-level model: requests & responses, sync
- request/response model and lifetimes
- sync (of channels, of chat msgs)

## 5. Wire Format: Messages

### 5.1 Field tables
This section makes heavy use of tables to convey the expected ordering of bytes for various message types, such as the following:

field      | type     | desc
-----------|----------|-------------------------------------------------------------
`foo`      | `u8`     | description of the field `foo`
`bar`      | `u8[4]`  | description of the field `bar`

This example describes a binary payload that is 5 bytes long, where the one byte of field `foo` is followed immediately by the 4 bytes describing `bar`.

If `foo=17` and `bar=[3,6,8,64]`, the following binary payload would be expected:

```
0x11 0x03 0x06 0x08 0x40

 ^     ^^^^^^^^^^^^^^^
 |            |------------- bar = [3, 6, 8, 64]
 |
 |-------------------------- foo = 17
```

The following data types are used:
- `u8`: a single unsigned byte
- `u8[N]`: a sequence of exactly `N` unsigned bytes
- `varint`: a variable-length unsigned integer. cable uses Protocol Buffer-style [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints). (For an example implementation of varint encoding/decoding, see the [NodeJS varint package](https://www.npmjs.com/package/varint).)

### 5.2 Message Header

All messages begin with a `msg_len` and a `msg_type` varint, and a reserved 4-byte `circuit_id` field:

field         | type     | desc
--------------|----------|-------------------------------------------------------------
`msg_len`     | `varint` | number of bytes in rest of message, i.e. not including the `msg_len` field
`msg_type`    | `varint` | see fields below
`circuit_id`  | `u8[4]`  | id of a circuit for an established path, or `[0,0,0,0]` for no circuit

Message-specific fields follow after the `circuit_id`.

Each request and response type has a unique `msg_type` (see below).

Clients may experiment with custom message types beyond the ids used by this specification (where `msg_type >= 64`).

Clients encountering an unknown `msg_type` should ignore and discard it.

The `circuit_id` field is not currently specified, and should not be used. It is reserved for future use.

### 5.3 Requests

Every request begins with the following header:

field      | type       | desc
-----------|------------|-----------------------------------
`msg_len`  | `varint`   | number of bytes in this message
`msg_type` | `varint`   |
`req_id`   | `u8[4]`    | unique id of this request (random)
`ttl`      | `varint`   | number of hops remaining

More fields follow for different request types below.

The request ID, `req_id`, is to be a sequence of 4 bytes, generated randomly by
the requester, used to uniquely identify the request during its lifetime across
the swarm of peers who may handle it.

When forwarding a request, do not change the `req_id`, so that routing loops
can be more easily detected by peers.

#### 5.3.1 Request by Hash (`msg_type=2`)

Request data for a set of hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint`            |
`req_id`     | `u8[4]`             | unique id of this request (random)
`ttl`        | `varint`            | number of hops remaining
`hash_count` | `varint`            | number of hashes to request
`hashes`     | `u8[32*hash_count]` | BLAKE2b hashes concatenated together

Results are provided by a data response (`msg_type=1`).

This request expects one or more `data response` responses.

The expected behaviour is to return immediately with what data is locally
available, rather than holding on to the request in anticipation of perhaps
seeing the requested hashes in the future.

Responders are free to return the data for any subset of the requested hashes
(including none).

#### 5.3.2 Cancel Request (`msg_type=3`)

Indicate a desire to stop receiving responses for any request.

Some requests stay open and wait for data to arrive. You can close these
long-running subscriptions using a cancel request.

Receiving this request indicates that any further responses sent back using the
given `req_id` may be discarded.

field        | type                | desc
-------------|---------------------|-----------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint`            |
`req_id`     | `u8[4]`             | stop receiving results for this request id
`ttl`        | `varint`            | ignored

This request should be passed along to any peers to which this peer has forwarded the original request.

No response to this message is expected.

#### 5.3.3 Request Channel Time Range (`msg_type=4`)

Request text posts and text post deletions written to a channel between a start and end time.

field          | type               | desc
---------------|--------------------|----------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`time_start`   | `varint`           | seconds since UNIX Epoch
`time_end`     | `varint`           | seconds since UNIX Epoch
`limit`        | `varint`           | maximum number of hashes to return

This request returns 0 or more `hash response` responses.

Channel names are expected to be UTF-8 strings.

Expected are the hashes of all `post/text` and `post/delete` posts made to a
channel by members between `time_start` and `time_end`. `time_start` is the
post with the *oldest* timestamp one is interested in, `time_end` is the
newest.

If `time_end` is 0, request all messages since `time_start` and respond with
more posts as they arrive, up to `limit` number of posts.

A `limit` of 0 indicates a desire to receive an unlimited number of hashes.

#### 5.3.4 Request Channel State (`msg_type=5`)

Request posts that describe the current state of a channel and its users, and
optionally subscribe to future state changes.

field          | type               | desc
---------------|--------------------|-----------------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`channel_len`  | `varint`           | length of the channel's name, in bytes (UTF-8)
`channel`      | `u8[channel_len] ` | channel name as a string of text
`historic`     | `varint`           | set to `1` to receive peer's view of current channel state; `0` to not
`updates`      | `varint`           | maximum number of live / future hashes to return

This request expects 0 or more `hash response`s in response, that pertain to
posts that describe the current state of the channel. See "Channel State" under
"Definitions" for details on what posts comprise a channel's current state.

If `historic` is set to `1`, this request expects the hashes of *all* historic
posts that make up the channel state to be returned, followed by up to
`updates` number of live post hashes that further alter this channel's state.

Set `updates` to 0 to not receive any live / future state changes.

Responses from a peer will keep coming until
- this request is cancelled, or
- `updates` live post hashes are returned (if `updates >= 1`), and
- all known historic hashes are returned (if `historic == 1`)

#### 5.3.5 Request Channel List (`msg_type=6`)

Request a list of known channels from peers.

field          | type               | desc
---------------|--------------------|-----------------------------------
`msg_len`      | `varint`           | number of bytes in this message
`msg_type`     | `varint`           |
`req_id`       | `u8[4]`            | unique id of this request (random)
`ttl`          | `varint`           | number of hops remaining
`offset`       | `varint`           | number of channel names to skip (`0` to skip none)
`limit`        | `varint`           | maximum number of channel names to return

This request returns zero or more `channel list response`s.

A `limit` of 0 indicates a desire to receive the full set of known channels
from a peer. Unlike some other requests, `limit=0` does not mean to subscribe
to future updates; the request is concluded after a single response.

The `offset` field can be combined with the `limit` field to allow clients to
paginate through the list of all channel names known by a peer.

### 5.4 Responses

There are 3 types of responses:

* hash response - a list of hashes (most queries return this)
* data response - a list of data chunks (in response to a hash query)
* channel list response - a list of the names of known channels

Multiple responses may be generated for a single request and results trickle in from peers.

Every response begins with the following header:

field      | type       | desc
-----------|------------|-----------------------------------
`msg_len`  | `varint`   | number of bytes in this message
`msg_type` | `varint`   |
`req_id`   | `u8[4]`    | unique id of the request this is in response to

More fields follow for different response types below.

#### 5.4.1 Hash Response (`msg_type=0`)

Respond with a list of hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`msg_len`    | `varint`            | number of bytes in this message
`msg_type`   | `varint (=0)`       |
`req_id`     | `u8[4]`             | id this is in response to
`hash_count` | `varint`            | number of hashes in the response
`hashes`     | `u8[hash_count*32]` | BLAKE2b hashes concatenated together

#### 5.4.2 Data Response (`msg_type=1`)

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

Clients can hash an entire data payload to check whether it is data that it was
expecting (i.e. had sent out a `request by hash` for).

#### 5.4.3 Channel List Response (`msg_type=7`)

Respond with a list of names of known channels.

field          | type                | desc
---------------|---------------------|-------------------------------------
`msg_len`      | `varint`            | number of bytes in this message
`msg_type`     | `varint (=0)`       |
`req_id`       | `u8[4]`             | id this is in response to
`channel0_len` | `varint`            | length in bytes of the first channel name
`channel0`     | `u8[channel_len]`   | the first channel name (UTF-8)
`...`          |                     |
`channelN_len` | `varint`            | length in bytes of the Nth channel name
`channelN`     | `u8[channel_len]`   | the Nth channel name (UTF-8)

A recipient reads the zero or more (`channel_len`,`channel`) pairs until
`channel_len` is 0.

In order for pagination to work properly, clients are expected to use a *stable
sort order* for channel names.

## 6. Wire Format: Posts

### 6.1 Post Header
Every post begins with the following 6-field header:

field        | type              | desc
-------------|-------------------|-------------------------------------------------------
`public_key` | `u8[32]`          | ED25519 key that authored this post
`signature`  | `u8[64]`          | ED25519 signature of the fields that follow
`num_links`  | `varint`          | how many BLAKE2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]`| BLAKE2b hashes of the latest messages in this channel/context
`post_type`  | `varint`          | see custom post type sections below
`timestamp`  | `varint`          | seconds since UNIX Epoch

More fields follow for different post types below.

The post type sections below document the fields that follow these initial 5 fields depending on the
`post_type`. Most post types will link to the most recent posts in a channel from any user (from
their perspective) but self-actions such as setting a nickname link to the most recent
self-action. Refer to each `post/*` type for what to expect.

If using libsodium, one can use `crypto_sign()` in combined mode to generate
the signature field for the fields that come after `signature`, as
`crypto_sign()` will prepend the signature into the output, so the fields will
be in the correct order.

The hash of an incoming post is produced by hashing the entire post, including *all* fields.

The `post_type` is a varint, so if the post types below are inadequate, you can create your own
using unused numbers (`>64`).

Specify `num_links=0` if there is nothing to link to.

Clients should ignore posts with a `post_type` that they don't understand or support.

### 6.2 `post/text` (`post_type=0`)

Post a message in a channel.

field          | type               | desc
---------------|--------------------|---------------------------------
`public_key`   | `u8[32]`           | ED25519 key that authored this post
`signature`    | `u8[64]`           | ED25519 signature of the fields that follow
`num_links`    | `varint`           | how many BLAKE2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | BLAKE2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`timestamp`    | `varint`           | seconds since UNIX Epoch
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`text_len`     | `varint`           | length of the text field
`text`         | `u8[text_len] `    | message content (UTF-8)

The `text` body of a chat message is expected to be a UTF-8 string.

### 6.3 `post/delete` (`post_type=1`)

Request that peers encountering this post delete the referenced posts by their
hashes from their local storage, and not store the referenced posts in the
future.

field           | type                   | desc
----------------|------------------------|-------------------------
`public_key`    | `u8[32]`               | ED25519 key that authored this post
`signature`     | `u8[64]`               | ED25519 signature of the fields that follow
`num_links`     | `varint`               | how many BLAKE2b hashes this post links back to (0+)
`links`         | `u8[32*num_links]`     | BLAKE2b hashes of the latest messages in this channel/context
`post_type`     | `varint`               | see custom post type sections below
`timestamp`     | `varint`               | seconds since UNIX Epoch
`num_deletions` | `varint`               | how many hashes of posts there are to be deleted
`hash`          | `u8[32*num_deletions]` | BLAKE2b hashes of posts to be deleted

The expected behaviour of a client interpreting this post is to only perform
local deletion of the referenced posts if the author (`post.public_key`)
matches the author of the post to be deleted (i.e. only the user who authored a
post may delete it).

### 6.4 `post/info` (`post_type=2`)

Set public information about yourself.

field        | type               | desc
-------------|--------------------|-------------------------
`public_key` | `u8[32]`           | ED25519 key that authored this post
`signature`  | `u8[64]`           | ED25519 signature of the fields that follow
`num_links`  | `varint`           | how many BLAKE2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]` | BLAKE2b hashes of the latest messages in this channel/context
`post_type`  | `varint`           | see custom post type sections below
`timestamp`  | `varint`           | seconds since UNIX Epoch
`key1_len`   | `varint`           | length of the first key to set
`key1`       | `u8[key_len]`      | name of the first key to set (UTF-8)
`value1_len` | `varint`           | length of the first value to set, belonging to `key1`
`value1`     | `u8[value_len]`    | value of the first key:value pair
...          |                    |
`keyN_len`   | `varint`           | length of the Nth key to set
`keyN`       | `u8[key_len]`      | name of the Nth key to set (UTF-8)
`valueN_len` | `varint`           | length of the Nth value to set, belonging to `keyN`
`valueN`     | `u8[value_len]`    | value of the Nth key:value pair

Several key:value pairs can be set at once. A post indicates it is done setting
pairs by setting a final `keyN_len` of zero.

Keys are expected to be UTF-8 strings.

Recommended keys for clients to support:

key       | desc
----------|------------------------------------------------
`name`    | handle this user wishes to use as a pseudonym

To save space, client may discard from disk older versions of these messages
from a particular user.

### 6.5 `post/topic` (`post_type=3`)

Set a topic for a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ED25519 key that authored this post
`signature`    | `u8[64]`           | ED25519 signature of the fields that follow
`num_links`    | `varint`           | how many BLAKE2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | BLAKE2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`timestamp`    | `varint`           | seconds since UNIX Epoch
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`topic_len`    | `varint`           | length of the topic field
`topic`        | `u8[topic_len] `   | topic content

### 6.6 `post/join` (`post_type=4`)

Join a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ED25519 key that authored this post
`signature`    | `u8[64]`           | ED25519 signature of the fields that follow
`num_links`    | `varint`           | how many BLAKE2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | BLAKE2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`timestamp`    | `varint`           | seconds since UNIX Epoch
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)

Peers can obtain a link to anchor their join message by requesting a list of channels.

### 6.7 `post/leave` (`post_type=5`)

Leave (part) a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`public_key`   | `u8[32]`           | ED25519 key that authored this post
`signature`    | `u8[64]`           | ED25519 signature of the fields that follow
`num_links`    | `varint`           | how many BLAKE2b hashes this post links back to (0+)
`links`        | `u8[32*num_links]` | BLAKE2b hashes of the latest messages in this channel/context
`post_type`    | `varint`           | see custom post type sections below
`timestamp`    | `varint`           | seconds since UNIX Epoch
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)


## 7. Security Considerations

### 7.1 Out of scope Threats
1. Attacks on the transport layer by non-members of the cabal. This would cover the *confidentiality* of a connection between peers, and prevent *eavesdropping* and *man-in-the-middle attacks*, as well as message reading/deleting/modifying.

2. Attacks that attempt to gain illicit entry to a cabal, by posing as a member. This would cover *peer entity authentication*.

3. Actors capable of deriving an ED25519 private key from the public key via brute force, which would break data integrity and permit data insertion/deletion/modification of the compromised user's posts.

4. Attacks that stem from how cable data ends up being stored locally. For example, an attacker with access to the user's machine being able to access their stored private key or chat history on disk.

### 7.2 In-scope Threats

Documented here are attacks that can come from *within* a cabal -- by those who are technically legitimate members and can peer freely with other members. It is currently assumed (until something like a version of Cabal's subjective moderation system is designed & implemented) that those who are proper members of a cabal are trusted to not cause problems for other users, but even a future moderation design would benefit from a clearly laid-out of the attack surface.

### 7.2.1 Susceptibilities
#### 7.2.1.1 Inappropriate Use
1. An attacker could issue `post/topic` posts to edit channel topics to garbage text, offensive content, or malicious content (e.g. phishing). Since most chat programs have channel topics controlled by "moderators" or "admins", this could cause confusion if users do not realize that anyone can set these strings.
2. The list of channels in the `Channel List Response` message could be falsified to include channels that do not exist (i.e. no users have posted to them) or to omit the names channels that do exist.
    1. A possible future mitigation to this might be inclusion of an explicit `post/channel` post type, to denote channel creation, which `Channel List Response` responders would need to cite the hashes of. This doesn't seem very important though, since an attacker could trivially produce 1000s of legitimate noise-creating channels anyways.

#### 7.2.1.2 Spoofing
1. An attacker could issue a `post/info` to alter their display name to be the same as another user, causing confusion as to which user is authoring certain chat messages.
    1. Client-side mitigation options exist, such as colourizing names by the public key of the user, or displaying a short hash digest of their key next to their name for important operations.

#### 7.2.1.3 Denial of Service
1. Authoring very large posts (gigabytes or larger) and/or a large number of smaller posts, and sharing them with others to download.
2. Making a large quantity of expensive requests (e.g. a time range request on a channel with a long chat history that covers its entire lifetime, repeatedly).
    1. Clients could implement per-connection rate limiting on requests, to prevent a degradation of service from network participants.
3. Creating a excessively large number of new channels (by writing at least one `post/text` post to each). Since channels can only be created and not removed, this has the potential to make a cabal somewhat unusable by legitimate users, if there are so many garbage channels they cannot locate real ones.
    1. New channel creation could be rate-limited, although even at a limit of 1 channel/day, it still would not take long to produce high levels of noise.
4. Providing a `Data Response` with large amounts of bogus data. Ultimately the content hashes from the requested hash and the data will not match, but the machine may expend a great deal of time and computational power determining each data block's legitimacy.

#### 7.2.1.4 Confidentiality
1. An attacker who appears legitimate (e.g. via a stolen machine belonging to a legitimate member) could connect to other members of the cabal and make ongoing and historic requests for cabal data, effectively spying on all members' posts, undetected, indefinitely.

#### 7.2.1.5 Repudiation
1. While all posts are cryptographically signed, a user can still claim that their private signing key was stolen, making reliable non-repudiation infeasible.

#### 7.2.1.6 Message Omission
1. While a machine can not issue a `post/delete` to erase another user's posts, they could easily choose to omit post hashes from responses to requests made to them by others. This attack is only viable if the machine is a client's only means of accessing certain data (e.g. the client was unable to directly connect to any non-attacker machines). Once that client connects to other, non-malicious machines, they will be able to "fill the gaps" of missing data within the time window & channels in which they are interested.

#### 7.2.1.7 Attacker Expulsion
1. An attacker causing problems by means of spoofing, denial of service, passive listening, or inappropriate use cannot, as per the current protocol design, be expelled from the cabal group chat. Legitimate users have no means of recourse beyond starting a new cabal and not inviting the attacker.

### 7.2.2 Protections
#### 7.2.2.1 Replay Attacks
1. Posts that have already been ingested will be deduplicated (i.e. not re-ingested) by their content hash, so resending hashes or data does no harm in itself.

#### 7.2.2.2 Data Integrity
1. All posts are crytographically signed, and cannot be altered or forged unless a user's private key has been compromised.

2. Certain posts have implicit authorization (e.g. a `post/info` post can only alter its author's display name, and NOT be used to change another user's name), which is carried out by clients following the specification re: post ingestion logic.

#### 7.2.2.3 Privilege Escalation
Cabal has no privilege levels beyond that of a) member and b) non-member. Non-members have zero privileges (not even able to participate at the wire protocol level), and all members hold the same privileges.

### 7.3 Future Work
Future work is planned around the outer layers of cable security:

1. **Against non-member active & passive attackers**: having transport security (to prevent non-members of cabals from reading, modifying, or otherwise interacting with data sent between members) via a mechanism with end-to-end encryption
2. **Against unauthorized access**: having a handshake protocol, to prevent non-members from gaining illicit access
3. **Against inappropriate use by members**: having a system for moderation and write-access controls internal to a cabal, so that users can mitigate and expel attacks from those who have already gained legitimate membership.

---

# unsorted notes

## Channel Sync Model
`request channel state` and `request channel time range` are sufficient to
track a channel that a user is interested in. The former tracks state (who is
in the channel, its topic, information about users in the channel), and the
latter tracks chat message history.

A suggested way to use `request channel time range` to track a channel is to
maintain a "rolling window". For example, a user that wishes to stay up-to-date
with the last week's worth of chat history would, on client start-up, issue a
`request channel time range` request with `time_start=now()-25200` (25200
seconds in a week) for a given channel of interest. `hash response`s with
already-known hashes can be safely ignored, while new ones can induce `data
request`s for their content.

The purpose of keeping a rolling time window, instead of just asking for
`time_start=last_bootup_time`, is to capture messages that were missed because
either you or other peers were, at the time, offline or part of another
network. This allows a client to make posts while offline, and still have them
appear to others when they do come back online (within the client's rolling
window's duration).

## Chat Message Sorting
Chat messages are sorted based on two inputs: a post's `timestamp` value, and
*causal ordering*.

For example, using a timestamp alone, an ascending sort comparison function
might look like this:

```js
function cmp (a, b) {
  return b.timestamp - a.timestamp
}
```

However, if a user posted a message after another user, but their clock was
skewed backwards by an hour, their newer post would appear in the past instead
of being sorted ahead.

By allowing posts to *link* to other posts by their hash, a comparator function
can be written that takes causal links into account, and uses the timestamp
only as a fallback in case no link chain can be found:

```js
// Assumes an internal representation of chat messages of the form
// {
//   timestamp: Number,
//   text: String,
//   links: Array<String>
// }
function cmp (a, b) {
  if (containedInLinkChain(a, hash(b))) return -1
  if (containedInLinkChain(b, hash(a))) return +1
  return b.timestamp - a.timestamp
}

// Recursively follows the causal link chain to determine if the hash 'h' is
// linked to from 'msg'.
function containedInLinkChain (msg, h) {
  if (!msg) return false
  if (msg.links.indexOf(h) !== -1) return true
  for (let link of msg.links) {
    if (containedInLinkChain(globalStoreOfAllKnownPosts[link], h)) return true
  }
  return false
}
```

Given then this set of chat messages,
```js
const m1 = {
  timestamp: 17,
  text: 'hi',
  links: []
}
const m2 = {
  timestamp: 170,
  text: 'hi from not-the-future; it is actually clock skew',
  links: []
}
const m3 = {
  timestamp: 18,
  text: 'hi from the real future (i can prove it)',
  links: ['85b8d0f0a48e34064b50a17df4f8eec3644bb4f1ec69aeed05fe245df7d1392b'] // m1's hash
}
const m4 = {
  timestamp: 10,
  text: 'hi from the seeming past, but actually future',
  links: ['5297f8422fccdbd2a29b3f6b02a23a0f0d196e0d437e0303273d9aa505add4ed'] // m3's hash
}
```

The new sort comparator using links would give
```js
[
  'hi',
  'hi from not-the-future; it is actually clock skew',
  'hi from the real future (i can prove it)',
  'hi from the seeming past, but actually future',
]
```

whereas the timestamp-only comparator would give

```js
[
  'hi from the seeming past, but actually future',
  'hi',
  'hi from the real future (i can prove it)',
  'hi from not-the-future; it is actually clock skew',
]
```

# data model notes (wip)

## Users
Users of Cabal are identified by their ED25519 public key, and use shared
knowledge of their key combined with their private key to prove themselves as
verifiable authors of data shared with other peers.

## Channels

### Channel Membership
A user is considered a member of a channel at a particular point in time if,
from the client's perspective, that user has issued a `post/join` to that
channel and has not issued a matching `post/leave` since.

### Channel State
The state of the channel at any given moment from the perspective of a client
is fully described by all of the following:
- The latest of each user's `post/join` or `post/leave` post to the channel.
- The latest `post/info` post of each user who is *or was* a member of the
  channel.
- The latest `post/topic` post to channel, made by any user, regardless of
  current or past membership.

Here "latest" means the relevant post with greatest causal ordering, and, if
that's not possible, the greatest timestamp. See "Chat Message Sorting" for
more details on links and causal ordering.

## Request/Response Model
Currently, all request types can generate potentially *many* responses. A
request sent to a peer may result in several blocks of hashes or data being
sent back by them as they scan their local database, and, if that peers
forwards your request to its peers as well, they too may trickle back many
responses over time.

(upcoming: info about request lifetimes)

### Time to Live (`ttl`)
This field, set on requests, controls *how many more times* a request may be
forwarded to other peers. A client wishing a request not be forward beyond its
initial destination peer would set `ttl = 0` to signal this.

When an incoming request has a `ttl > 0`, a peer can choose to forward a
request along to other peers, and then forward the responses back to the peer
that made the request. Each peer performing this action should decrement the
`ttl` by one. A request with `ttl == 0` should not be forwarded.

The TTL mechanism exists to allow clients with limited connectivity to peers
(e.g. behind a strong NAT) to use the peers they can reach as a relay to
find and retrieve data they are interested in more easily.

### Limits (`limit`)
If a request has a `limit` field specifying an upper bound on how many hashes
it expects to get in response, and also sets a `ttl > 0`, a peer handling this
request should try to ensure that that `limit` is honoured. This can be done by
counting how many hashes a client sends back to the requester, **including**
hashes received through other peers that the client has forwarded the request
to.

For example, assume `A` sends a request to `B` with `limit=50` and `ttl=1`, and
`B` forwards the request to `C` and `D`. `B` may send back 15 hashes to `A` at
first, which means there are now a maximum of `50-15=35` hashes left for `C`
and `D` combined for `B` to potentially send back. `B` can choose to track
hashes and perform deduplication, so that if `C` and `D` were to both send back
a hash `f88954b3e6adc067af61cca2aea7e3baecfea4238cb1594e705ecd3c92a67cb1`, `B`
could ensure it was only passed back to `A` one time, thus reducing the
remaining `limit` by 1 instead of 2.

## Posts
All durable data exchanged over the protocol are called Posts, and are always
cryptographically signed by their author's private key. Any post may be
referred to by its 32-byte BLAKE2b hash. This protocol primary aim is to
facilitate the exchange of these posts between peers.

local disk storage. A post always has an author (via a required `public_key`
field), and always provides a signature (via the required `signature` field) to
prove they authored it.

When you "make a post", you are only writing to some local storage indexed by the hash of the
content. Posts only get sent to other peers in response to queries for content matching certain
criteria.

### Hash
All **post**s in Cable is referenced by the resulting output of putting a post
through a hash function. Specifically, 32-byte BLAKE2b (optionally using
libsodium's `crypto_generichash()` function).

To produce a correct hash, run `crypto_generichash()` on the entire post,
including the post header.

### Links
Part of the header every post is the `links` field. Every post is able to link
to 0 or more other posts by means of referencing those posts by their hash (the
output of running them through a specific hash function).

Referencing a post by its hash provides a *causal proof*: it demonstrates that
your post must have occurred after all of the posts you are referencing.

This can be useful for ordering chat messages in particular when a client's
hardware clock is skewed, and using post timestamps alone would provide
confusing ordering.

## ?. References
BLAKE2: https://www.blake2.net/blake2.pdf

