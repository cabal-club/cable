# cable

Version: 2023.03-draft1

Author: Kira Oakley

Contributors: Alexander Cobleigh, Noelle Leigh, Henry (cryptix)

## Abstract

This document describes the network protocol "cable", used to facilitate
peer-to-peer group chatrooms.

## Table of Contents
* [0. Background](#0-background)
* [1. Introduction](#1-introduction)
  + [1.1 Terminology](#11-terminology)
* [2. Scope](#2-scope)
* [3. Definitions](#3-definitions)
* [4. Software Dependencies](#4-software-dependencies)
  + [4.1 Cryptography](#41-cryptography)
  + [4.1.1 BLAKE2b Parameters](#411-blake2b-parameters)
* [5. Data Model](#5-data-model)
  + [5.1 Posts](#51-posts)
    - [5.1.2 Signing](#512-signing)
    - [5.1.3 Addressing](#513-addressing)
    - [5.1.4 Links](#514-links)
      * [5.1.4.1 Setting links](#5141-setting-links)
    - [5.1.5 Ordering](#515-ordering)
      * [5.1.5.1 Rationale](#5151-rationale)
  + [5.2 Users](#52-users)
    - [5.2.1 State](#521-state)
  + [5.3 Channels](#53-channels)
    - [5.3.1 Names](#531-names)
    - [5.3.2 Membership](#532-membership)
    - [5.3.3 State](#533-state)
    - [5.3.4 Synchronization](#534-synchronization)
  + [5.4 Requests & Responses](#54-requests--responses)
    - [5.4.1 Lifetime of a Request](#541-lifetime-of-a-request)
    - [5.4.2 Time To Live](#542-time-to-live)
    - [5.4.3 Limit Counting](#543-limit-counting)
* [6. Wire Formats](#6-wire-formats)
  + [6.1 Field tables](#61-field-tables)
  + [6.2 Messages](#62-messages)
    - [6.2.1 Message Header](#621-message-header)
    - [6.2.2 Requests](#622-requests)
      * [6.2.2.1 Header](#6221-header)
      * [6.2.2.2 Request Post](#6222-request-post)
      * [6.2.2.3 Cancel Request](#6223-cancel-request)
      * [6.2.2.4 Request Channel Time Range](#6224-request-channel-time-range)
      * [6.2.2.5 Request Channel State](#6225-request-channel-state)
      * [6.2.2.6 Request Channel List](#6226-request-channel-list)
    - [6.2.3 Responses](#623-responses)
      * [6.2.3.1 Hash Response](#6231-hash-response)
      * [6.2.3.2 Post Response](#6232-post-response)
      * [6.2.3.3 Channel List Response](#6233-channel-list-response)
  + [6.3 Posts](#63-posts)
    - [6.3.1 Header](#631-header)
    - [6.3.2 `post/text`](#632-posttext)
    - [6.3.3 `post/delete`](#633-postdelete)
    - [6.3.4 `post/info`](#634-postinfo)
    - [6.3.5 `post/topic`](#635-posttopic)
    - [6.3.6 `post/join`](#636-postjoin)
    - [6.3.7 `post/leave`](#637-postleave)
* [7. Security Considerations](#7-security-considerations)
  + [7.1 Out of scope Threats](#71-out-of-scope-threats)
  + [7.2 In-scope Threats](#72-in-scope-threats)
  + [7.2.1 Susceptibilities](#721-susceptibilities)
    - [7.2.1.1 Inappropriate Use](#7211-inappropriate-use)
    - [7.2.1.2 Spoofing](#7212-spoofing)
    - [7.2.1.3 Denial of Service](#7213-denial-of-service)
    - [7.2.1.4 Confidentiality](#7214-confidentiality)
    - [7.2.1.5 Repudiation](#7215-repudiation)
    - [7.2.1.6 Message Omission](#7216-message-omission)
    - [7.2.1.7 Attacker Expulsion](#7217-attacker-expulsion)
  + [7.2.2 Protections](#722-protections)
    - [7.2.2.1 Replay Attacks](#7221-replay-attacks)
    - [7.2.2.2 Data Integrity](#7222-data-integrity)
    - [7.2.2.3 Privilege Escalation](#7223-privilege-escalation)
  + [7.3 Future Work](#73-future-work)
* [8. Normative References](#8-normative-references)
* [9. Informative References](#9-informative-references)

## 0. Background
[Cabal][Cabal] is the pre-cursor iteration of this new protocol: an existing
distributed peer-to-peer computer program for private group chats. It operates
in a fashion different from the typical server-client model, where no machine
is either an official nor *de facto* authority over others in the network.
Instead, peers collaborate with each other to share documents to build an
eventually-consistent view of the shared data.

Cabal's original protocol was based off of [hypercore][hypercore], which was
found to have limitations and trade-offs that didn't suit Cabal's needs well.
This was the impetus for the formulation of this new protocol, designed with
peer-to-peer group chat specifically in mind.

## 1. Introduction
The purpose of the cable wire protocol is to facilitate the members of a
Cabal group chat to exchange crytographically signed documents with each other,
such as chat messages, spread across various user-defined topics.

cable is designed to be:

* fairly simple to implement in any language with only a single dependency (libsodium)
* able to bridge across different network transports
* useful, even if written as a partial implementation
* efficient in its use of network resources, by syncing only the relevant subsets of the full dataset
* compact over the wire
* independent of whatever kind of database an implementation may use

### 1.1 Terminology
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [BCP 14][BCP 14], [RFC 2119][RFC 2119], [RFC
8174][RFC 8174] when, and only when, they appear in all capitals, as shown
here.

## 2. Scope
This protocol focuses on the over-the-wire bytes that get sent between peers
and enable the exchange of chat messages alongside user and channel information.

This protocol does not include encryption or authentication of the connection,
nor a mechanism for the discovery of network peers. These may be provided by
other layers in userland (e.g. [Tor][Tor], [I2P][I2P]) or specified in future
iterations of this document. In particular, a future handshake protocol is
planned, which will handle authenticating peers to the Cabal.

As such, it is assumed that peers speaking this protocol are already authorized to access the
Cabal. Section [Security Considerations](#7-security-considerations) includes an analysis of
the types of anticipated attacks that legitimate cabal members may carry out.

## 3. Definitions
**Cabal**: A private group chat that a number of users can participate in, comprised of **users** and zero or more **channel**s.

**Channel**: A conceptual object with its own unique name, a set of member **user**s, and a set of chat **post**s written to it.

**User**: An Ed25519 pair of keys identifying a person: a **public key**, and a **private key**, for use within a single cabal. Sometimes also written as "member of a cabal".

**Public Key**: An Ed25519 key, which constitutes a user's public-facing identity within a cabal.

**Private Key**: An Ed25519 key, used for signing authored **post**s. Kept private and secret to all but the user who owns it.

**Post**: An authored binary payload, signed by the private key of its creator (a user).

**Client**: A running instance of a computer program that implements cable (this specification).

**Peer**: A machine that a client is connected to over some transport protocol, on top of which the cable protocol is being spoken.

**Message**: An informative binary payload sent by and received from other cable peers. Each message is either a **request** or a **response**.

**Request**: A message originating from a particular **peer**, identified by a unique request ID.

**Response**: A message, traversing the network to the peer who originally made a request with the same request ID.

**Requester**: A client authoring a request message, to be sent to other peers.

**Responder**: A peer authoring a response message, in reference to a request sent to them.

**Hash**: A 32-byte BLAKE2b digest of a particular sequence of bytes.

**Link**: A **hash**, which acts as a reference to the post which hashes to said hash.

**Chat message**: A post of type `post/text` authored within a particular channel.

**UNIX Epoch**: Midnight UTC on January 1st, 1970.

**UNIX Time**: A point in time represented by the number of seconds since the UNIX Epoch.

**Unicode**: The [Unicode 15.0.0 standard](https://www.unicode.org/versions/Unicode15.0.0/).

**UTF-8**: The "UTF-8" encoding scheme outlined in [Chapter 2 Section 5](https://www.unicode.org/versions/Unicode15.0.0/ch02.pdf#G11165) of the Unicode 15.0.0 specification.

## 4. Software Dependencies

### 4.1 Cryptography

Implementing cable requires access to implementations of the following:

- [BLAKE2b](https://www.rfc-editor.org/rfc/rfc7693.txt) - Hash function described by RFC 7693. To be set to output 32-byte digests.
- [Ed25519](https://ed25519.cr.yp.to/) - A public-key signature system. Used to generate, sign posts, and verify post signatures.

This cryptographic functionality can be provided by [libsodium](https://libsodium.org) 1.0.18-stable, if bindings exist for one's implementation language of choice. In particular, these functions may be utilized, with their default settings:

* `crypto_generichash()` - to hash messages with BLAKE2b
* `crypto_sign_keypair()` - to generate public and secret Ed25519 keys
* `crypto_sign()` - to calculate the signature of a post (in combined mode)
* `crypto_sign_open()` - to verify the signature of a post (in combined mode)

### 4.1.1 BLAKE2b Parameters
The following are the general parameters to be used with BLAKE2b. If one is using 1.0.18-stable of libsodium, these are already set by default.

- Digest byte length: 32
- Key byte length (not used): 0
- Salt (hexadecimal): `5b6b 41ed 9b34 3fe0`
- Personalization (hexadecimal): `5126 fb2a 3740 0d2a`

## 5. Data Model

### 5.1 Posts
All durable data exchanged over the protocol are called **posts**.

Any post can be referred to by its BLAKE2b hash.

A post always has an author (via the required `public_key` field), and always
provides a crytographic signature (via the required `signature` field) to prove
they in fact authored it.

When a user "makes a post", they are only writing to some local storage,
perhaps indexed by the hash of the content for easy querying. Posts are only
sent to other peers in response to queries about them (e.g. chat messages
within some time range).

#### 5.1.2 Signing
A signature for a post is produced by signing the entire post, starting
immediately after the `signature` field of the post header (defined below).

If using libsodium, one can use `crypto_sign()` in combined mode to generate
the signature field for the fields that come after `signature`, as
`crypto_sign()` will prepend the signature into the output, so the fields will
be in the correct order.

#### 5.1.3 Addressing
As stated above, any post in cable can be addressed or referenced by its hash.

Specifically, a hash for a post is produced by putting a post's verbatim binary
content, including the post header, through the BLAKE2b function.

Implementations may benefit from a storage design that allows for quick look-up
of a post's contents by its hash.

#### 5.1.4 Links
Every post has a field named `links` in its header. This enables any post to
*link* to 0 or more other posts by means of referencing those posts by their
hash.

Referencing a post by its hash provides a **causal proof**: it demonstrates
that a post must have occurred after all of the other posts referenced. This
can be useful for ordering chat messages, since a timestamp alone can cause
ordering problems if a client's hardware clock is skewed, or timestamp is
spoofed.

Implementations are RECOMMENDED to set and utilize links on chat messages
specifically (`post/text`).

##### 5.1.4.1 Setting links
This is done by tracking each channel's current **heads**. In this context, a
**head** is a post that no other known post links to.

To do this quickly, an implementation may maintain a reverse look-up table for
posts, so that, given a post's hash, the hashes of all posts linking to it may
be found. If no hashes are returned, the post is a head.

Over time, this creates an eventually consistent data structure where all chat
messages in a channel become roughly causally ordered. ("Roughly" because of
the possible participation of client implementations that do not set links.)

#### 5.1.5 Ordering
Only chat messages need to be sorted, and sorting only needs to happen at the
level of a client implementation — not something a wire protocol
implementation needs to worry about. Still, it is included here as a learning
aid until a separate guide for implementors is written.

The algorithm for ordering any pair of posts is to pick either the largest
timestamp OR higher **causal value**, with the latter having higher priority.

A post's causal value relative to another post is decided by whether there is a
known chain of links that leads from one post to the other. If Post B links to
Post A by mentioning Post A's hash, Post B has a higher causal value than Post
A (relative to each other). This would be true even if Post B linked to Post A
by means of intermediary Posts C and D.

The term **latest** is taken to refer to the post with the greatest sort value
using this ordering algorithm.

##### 5.1.5.1 Rationale
Using a timestamp alone, an ascending sort comparison function might look like
the following Javascript function:

```js
function cmp (a, b) {
  return b.timestamp - a.timestamp
}
```

However, if a user posted a message after another user, but their clock was
skewed backwards by an hour, their newer post would appear in the past instead
of being sorted ahead.

By allowing posts to link to other posts by their hash, a comparator function
can be written that takes causal links into account, and uses the timestamp
only as a fallback in case no link chain can be found at a given point in time:

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

Given the following set of chat messages,

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

The new sort comparator using `links` would yield:

```js
[
  /* m1 */ 'hi',
  /* m2 */ 'hi from not-the-future; it is actually clock skew',
  /* m3 */ 'hi from the real future (i can prove it)',
  /* m4 */ 'hi from the seeming past, but actually future',
]
```

whereas the timestamp-only comparator would yield:

```js
[
  '/* m4 */ hi from the seeming past, but actually future',
  '/* m1 */ hi',
  '/* m3 */ hi from the real future (i can prove it)',
  '/* m2 */ hi from not-the-future; it is actually clock skew',
]
```

Here, the timestamp-only comparator provides a result that is clearly (to a
human) out of order, while the comparator that takes causal links into account
produces an ordering that would more closely capture the real temporal ordering.

Note that the overall ordering of chat messages could change over time, as new
information is collected. For example, a post might be downloaded that proves a
chain of links leading from Post A to Post F, resulting in the causal chain
being used to sort rather than A and F's respective timestamps. Clients would
have to choose how to handle this based on, perhaps, what would be least
disorientating or most informative to users (e.g. moving around chat messages
in the display, or keeping the ordering stable).

### 5.2 Users
Users of Cabal are identified by their Ed25519 public key, and use it and
their private key to prove themselves as verifiable authors of data shared with
other peers. This is done by including their public key and a crytographic signature on any
post they author.

#### 5.2.1 State
A user is fully described at a given point in time by the following:

1. their public key, and
2. the key/value pairs of the latest known `post/info` post made by that user.

As of present, the only supported key is `name`, which defines a user's display
name. Older `post/info` posts made by a user are considered obsolete and MAY be
safely ignored or discarded.

### 5.3 Channels
A channel is a named collection consisting of the following:

1. chat messages (`post/text`), and
2. user joins and leaves (`post/join` or `post/leave`).

The act of a user issuing a post that writes a chat message to a channel or
joins that channel implies that that named channel has been created, and now
exists.

#### 5.3.1 Names
- A valid channel name MUST be a UTF-8 encoded string.
- A valid channel name MUST between 1 and 64 codepoints.

#### 5.3.2 Membership
A user is considered a member of a channel at a particular point in time if,
from a client's perspective, that user has issued a `post/join`, `post/text`,
or `post/topic` to that channel and has not issued a `post/leave` since.

Clients SHOULD issue a `post/join` post before issuing any other posts that
constitute participation in a channel. Namely, `post/text` and `post/topic`.

A user whose latest interaction with a channel is a `post/leave` is considered
an **ex-member**.

#### 5.3.3 State
A channel at any given moment, from the perspective of a client, is fully
described by the following:

1. The latest `post/info` post of each member and ex-member.
2. The latest of each member and ex-member's `post/join` or `post/leave` post to the channel.
3. The latest `post/topic` post made to the channel, made by any member or ex-member.
4. All known `post/text` posts made to channel.

Above, "to the channel" refers to the `channel` field set on a given post.

#### 5.3.4 Synchronization
The `Request Channel State` and `Request Channel Time Range` requests are
sufficient for a client to track the state of a channel that a user is
interested in. The former request allowing tracking general state (who is in
the channel, its topic, information about users in the channel), and the latter
tracks its chat message history.

The recommended way to use `Request Channel Time Range` to track a channel is
to maintain a "rolling window". For example, a user that wishes to stay
up-to-date with the last week's worth of chat history would, on client
start-up, issue a `Request Channel Time Range` request with
`time_start=now()-25200` (25200 seconds in a week) for a given channel of
interest. Known hashes provided by `Hash Response` messages can be safely ignored, while newly
discovered hashes can be made to induce `Post Request` messages for their content.

The purpose of keeping a rolling time window instead of asking for
`time_start=last_bootup_time`, is to capture messages that were missed because
either a client or other peers were, at the time, offline or part of another
network. This allows a client to make posts while offline, and still have them
appear to others when they do come back online (within a given client's rolling
window).

### 5.4 Requests & Responses
All request types can generate multiple responses. A request sent to a peer may
result in several blocks of hashes or data being sent back by them as they scan
their local database. Additionally, if that peer forwards your request to its
own set of peers, they too may trickle back several responses over time.

#### 5.4.1 Lifetime of a Request
In the lifetime of a given request, there are three exclusive roles an involved
client machine can have:

1. The **original requester**, who allocated the new request, and has a set of
   *outbound peers* they have sent the request to.

2. An **intermediary peer**. This is any client who received the request from
   one or more peers and has also forwarded it to others. An intermediary peer
   has both a set of *inbound peers* for a request as well as a set of *outbound peers*.

3. A **terminal peer**. This is a client who received the request from one or
   more peers and has NOT forwarded it to any others. A terminal peer has only
   a set of *inbound peers*.

A peer handling a request who has *outbound peers* (original requester,
intermediary peer) MUST satisfy any of the following, for each outbound peer:

1. Receives a "no more data" `Hash Response` (`hash_count=0`) from the peer.
2. Sends the peer a `Cancel Request`. This could be induced by an explicit
   client action or e.g. a local timeout a client set on the request.
3. The connection to the peer is lost.

A peer handling a request who has *inbound peers* (intermediary peer, terminal
peer) MUST satisfy any of the following, for each inbound peer:

1. Sends a "no more data" response back to the peer.
2. Receives a `Cancel Request`.
3. The connection to the peer is lost.

A request SHOULD be considered "concluded" and be safely deallocated (e.g. its
Request ID forgotten) once a given peer role (above) has satisfied all
conditions for all inbound and outbound peers.

#### 5.4.2 Time To Live
The `ttl` field, set on all requests' header, controls how many more times a
request MAY be forwarded to other peers. A client wishing a request not be
forwarded beyond its initial destination peer would set `ttl = 0` to signal this.

When an incoming request has a `ttl > 0`, a peer MAY choose to forward a
request along to other peers, and then SHOULD forward their responses back to
the peer who made the original request. Each peer performing this action SHOULD
decrement the `ttl` by one. A request with `ttl == 0` SHOULD NOT be forwarded.

The TTL mechanism exists to allow clients with limited connectivity to peers
(e.g. behind a strong NAT or a restricted mobile connection) to use the peers
they can reach as a relay to find and retrieve data they are interested in more
easily.

To prevent request loops in the network, an incoming request with a known
`req_id` (Request ID) SHOULD be discarded.

#### 5.4.3 Limit Counting
Some requests have a `limit` field specifying an upper bound on how many hashes
a client expects to receive in response. A peer responding to such a request
SHOULD honour this limit by counting how many hashes they send back to the
requester, **including** hashes received through other peers that the client
has forwarded the request to.

For example, assume `A` sends a request to `B` with `limit=50` and `ttl=1`, and
`B` forwards the request to `C` and `D`. `B` may send back 15 hashes to `A` at
first, which means there are now a maximum of `50-15=35` hashes left for `C`
and `D` combined for `B` to potentially send back. `B` can choose to track
hashes and perform deduplication, so that if `C` and `D` were to both send back
a hash `f88954b3e6adc067af61cca2aea7e3baecfea4238cb1594e705ecd3c92a67cb1`, `B`
could ensure it was only passed back to `A` one time, thus reducing the
remaining `limit` by 1 instead of 2.

## 6. Wire Formats

### 6.1 Field tables
This section makes heavy use of tables to convey the expected ordering of bytes
for various message types, such as the following:

field      | type     | desc
-----------|----------|-------------------------------------------------------------
`foo`      | `u8`     | description of the field `foo`
`bar`      | `u8[4]`  | description of the field `bar`

This example describes a binary payload that is 5 bytes long, where the one byte
of field `foo` is followed immediately by the 4 bytes describing `bar`.

If `foo=17` and `bar=[3,6,8,64]`, the following binary payload would be expected:

```
 foo   bar
 0x11  0x03 0x06 0x08 0x40
 ^^^^  ^^^^^^^^^^^^^^^^^^^
 |     ╰-------------------- bar = [3, 6, 8, 64]
 |
 ╰-------------------------- foo = 17
```

The following data types are used:
- `u8`: a single unsigned byte
- `u8[N]`: a sequence of exactly `N` unsigned bytes
- `varint`: a variable-length unsigned integer. cable uses Protocol Buffer-style [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints). (For an example implementation of varint encoding/decoding, see the [Node.js varint package](https://www.npmjs.com/package/varint).)


### 6.2 Messages

#### 6.2.1 Message Header

All messages MUST begin with the following header fields:

field         | type     | desc
--------------|----------|-------------------------------------------------------------
`msg_len`     | `varint` | number of bytes in rest of message, i.e. not including the `msg_len` field
`msg_type`    | `varint` | see fields below
`circuit_id`  | `u8[4]`  | id of a circuit for an established path, or `[0,0,0,0]` for no circuit
`req_id`      | `u8[4]`  | unique id of this request (random)

Message-specific fields follow after the `req_id`.

Each request and response type has a unique `msg_type` (see below).

Clients MAY experiment with custom message types beyond the ids used by this
specification (where `msg_type >= 256`).

Clients encountering an unknown `msg_type` SHOULD ignore and discard it.

The `circuit_id` field is not currently specified, and SHOULD be set to all
zeros. It is reserved for future use.

The request ID, `req_id`, is a sequence of 4 bytes, generated randomly by the
requester, used to uniquely identify the request during its lifetime across the
set of peers who may handle it.

When forwarding a request to further peers, the `req_id` SHOULD NOT be changed,
so that routing loops can be more easily detected by peers in the network.

#### 6.2.2 Requests

##### 6.2.2.1 Header

Every request MUST begin with the above message header, followed by the
following request header fields:

field      | type       | desc
-----------|------------|-----------------------------------
`ttl`      | `varint`   | number of hops remaining (described above)

More fields follow for different request types below.

##### 6.2.2.2 Request Post

Request a set of posts, given their hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`hash_count` | `varint`            | number of hashes to request
`hashes`     | `u8[32*hash_count]` | BLAKE2b hashes concatenated together

Its `msg_type` MUST be set to `2`.

Results are provided by one or more `Post Response` messages.

The responder SHOULD immediately return what data is locally available, rather
than holding on to the request in anticipation of perhaps seeing the requested
hashes in the future.

Responders MAY return the data for any subset of the requested hashes
(including none).

##### 6.2.2.3 Cancel Request

Indicates a desire to conclude a given `req_id` and stop receiving responses
for that request.

Some requests are long-lived, and stay open waiting for data to arrive.
Such requests can be terminated using this request.

field        | type                | desc
-------------|---------------------|-------------------------------------
`cancel_id`  | `varint`            | The Request ID (`req_id`) of the request to be cancelled

Receiving this request indicates that any further responses sent back with a
`req_id` matching the given `cancel_id` are likely to be discarded.

Its `msg_type` MUST be set to `3`.

Its `cancel_id` SHOULD be set to the Request ID (`req_id`) of the request to be cancelled.

No response to this message is expected.

Like any other request, this MUST have its own unique `req_id` in order to
function as intended. `cancel_id` is used to set the Request ID to cancel, not
its `req_id`.

A peer receiving a `Cancel Request` SHOULD forward it along the same route, to
the same peers, as the original request matching the given `req_id`, so that
all peers involved in the request are notified. This request's `ttl` SHOULD be
ignored in service of this end.

##### 6.2.2.4 Request Channel Time Range

Request chat messages and chat message deletions written to a channel between a
start and end time.

field          | type               | desc
---------------|--------------------|----------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`time_start`   | `varint`           | seconds since UNIX Epoch
`time_end`     | `varint`           | seconds since UNIX Epoch
`limit`        | `varint`           | maximum number of hashes to return

Its `msg_type` MUST be set to `4`.

This request returns 0 or more `Hash Response` messages.

Restrictions on channel names are defined above.

A responder SHOULD include the hashes of all known `post/text` and
`post/delete` posts made to a channel between `time_start` and `time_end`.
`time_start` is the post with the oldest timestamp one is interested in,
`time_end` is the newest.

If `time_end` is 0, request all chat messages since `time_start` and respond
with more posts as they arrive, up to `limit` number of posts.

A responding client is RECOMMENDED to respond with all known chat messages
within the requested time range, though they may desire not to in certain
circumstances, particularly if a channel has a very long history and the
responding client lacks sufficient resources at the time to return thousands or
hundreds of thousands of chat message hashes.

A `limit` of 0 indicates a desire to receive an unlimited number of hashes.

##### 6.2.2.5 Request Channel State

Request posts that describe the current state of a channel and its users, and
optionally subscribe to future state changes.

field          | type               | desc
---------------|--------------------|-----------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes (UTF-8)
`channel`      | `u8[channel_len] ` | channel name as a string of text
`historic`     | `varint`           | set to `1` to recv current state, or `0` to not
`updates`      | `varint`           | maximum number of live / future hashes to return

Its `msg_type` MUST be set to `5`.

This request expects 0 or more `Hash Response` in response, that pertain to
posts that describe the current state of the channel.

The hashes returned in a response are all those whose posts comprise the channel
state (defined above), excluding chat messages.

If `historic` is set to `1`, the requester expects to receive the hashes
of *all* posts that make up the current channel state from the perspective of
the responding client.

`updates` MUST be `>= 0`. The requester expects to receive up to `updates`
post hashes, as they are produced in the future, which further alter this channel's
state. Set `updates` to 0 to not receive any live / future state changes.

Responses from a peer will keep coming until

1. This request is concluded (see above for conditions), or both of
    1. `updates` hashes are returned (if `updates >= 1`), and
    2. all known historic hashes are returned (if `historic == 1`)   <---- revise me TODO

##### 6.2.2.6 Request Channel List

Request a list of known channels from peers.

field          | type               | desc
---------------|--------------------|-----------------------------------
`offset`       | `varint`           | number of channel names to skip (`0` to skip none)
`limit`        | `varint`           | maximum number of channel names to return

Its `msg_type` MUST be set to `6`.

This request returns zero or more `Channel List Response` messages.

A `limit` of 0 indicates a desire to receive the full set of known channels
from a peer at the time of requesting.

The `offset` field can be combined with the `limit` field to allow clients to
paginate through the list of all channel names known by a peer.

#### 6.2.3 Responses
Multiple responses may be generated for a single request and results trickle in from peers.

Every response MUST begin with the above message header.

Responses containing an unknown `req_id` SHOULD be ignored.

A response MUST have its `req_id` set to the same `req_id` of the request they are responding to.

##### 6.2.3.1 Hash Response

Respond with a list of hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`hash_count` | `varint`            | number of hashes in the response
`hashes`     | `u8[hash_count*32]` | BLAKE2b hashes concatenated together

Its `msg_type` MUST be set to `0`.

A `Hash Response` message with `hash_count=0` indicates that a peer does not
intend to return any further data for the given request ID (`req_id`).

##### 6.2.3.2 Post Response

Respond with a list of post contents in response to a `Request Post`.

field        | type                | desc
-------------|---------------------|--------------------------
`post0_len`  | `varint`            | length of first post
`post0_data` | `u8[post0_len]`     | first post
`post1_len`  | `varint`            | length of second post
`post1_data` | `u8[post1_len]`     | second post
`...`        |                     |
`postN_len`  | `varint`            | length of Nth post
`postN_data` | `u8[postN_len]`     | Nth post

Its `msg_type` MUST be set to `1`.

A recipient reads zero or more (`post_len`,`post_data`) pairs until a
`post_len` set to 0 is encountered.

Clients SHOULD hash an entire post to check whether it is post that it was
expecting (i.e. had sent out a `Request Post` for).

Each post SHOULD contain the complete and valid body of a known post type, as
specified below.

##### 6.2.3.3 Channel List Response

Respond with a list of names of known channels.

field          | type                | desc
---------------|---------------------|-------------------------------------
`channel0_len` | `varint`            | length in bytes of the first channel name
`channel0`     | `u8[channel_len]`   | the first channel name (UTF-8)
`...`          |                     |
`channelN_len` | `varint`            | length in bytes of the Nth channel name
`channelN`     | `u8[channel_len]`   | the Nth channel name (UTF-8)

Its `msg_type` MUST be set to `7`.

A recipient reads the zero or more (`channel_len`,`channel`) pairs until
`channel_len` is 0.

In order for pagination to work properly, clients are RECOMMENDED to use a
stable sort method for the names of channels.

### 6.3 Posts

#### 6.3.1 Header
Every post MUST begin with the following 6-field header:

field        | type              | desc
-------------|-------------------|-------------------------------------------------------
`public_key` | `u8[32]`          | Ed25519 key that authored this post
`signature`  | `u8[64]`          | Ed25519 signature of the fields that follow
`num_links`  | `varint`          | how many BLAKE2b hashes this post links back to (0+)
`links`      | `u8[32*num_links]`| BLAKE2b hashes of the latest messages in this channel/context
`post_type`  | `varint`          | see custom post type sections below
`timestamp`  | `varint`          | seconds since UNIX Epoch

More fields follow for different post types below.

The post type sections below document the fields that follow these initial
fields depending on the `post_type`. Refer to each `post/*` type for what to
expect.

The `post_type` is a varint, so if the post types below are inadequate, one can
create additional types using unused numbers (`>= 256`). A client SHOULD NOT
define its own custom post types below 256, since these may be used in the
future to expand the core protocol.

Specify `num_links=0` if there is nothing to link to.

Clients SHOULD ignore posts with a `post_type` that they don't understand or
support.

All fields specified in the subsequent subsections MUST be present for a post
of a given `post_type`.

#### 6.3.2 `post/text`

Post a chat message to a channel.

field          | type               | desc
---------------|--------------------|---------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`text_len`     | `varint`           | length of the text field
`text`         | `u8[text_len] `    | message content (UTF-8)

Its `post_type` MUST be set to `0`.

The `text` body of a chat message MUST be a valid UTF-8 string. Its length MUST
NOT to exceed 4 kibibytes (4096 bytes).

#### 6.3.3 `post/delete`

Request that peers encountering this post delete the referenced posts by their
hashes from their local storage, and not store the referenced posts in the
future.

field           | type                   | desc
----------------|------------------------|-------------------------
`num_deletions` | `varint`               | how many hashes of posts there are to be deleted
`hash`          | `u8[32*num_deletions]` | BLAKE2b hashes of posts to be deleted

Its `post_type` MUST be set to `1`.

A client interpreting this post MUST only perform a local deletion of the
referenced posts if the author (`post.public_key`) matches the author of the
post to be deleted (i.e. only the user who authored a post may delete it).

#### 6.3.4 `post/info`

Set public information about one's self.

field        | type               | desc
-------------|--------------------|-------------------------
`key1_len`   | `varint`           | length of the first key to set
`key1`       | `u8[key_len]`      | name of the first key to set (UTF-8)
`value1_len` | `varint`           | length of the first value to set, belonging to `key1`
`value1`     | `u8[value_len]`    | value of the first key:value pair
...          |                    |
`keyN_len`   | `varint`           | length of the Nth key to set
`keyN`       | `u8[key_len]`      | name of the Nth key to set (UTF-8)
`valueN_len` | `varint`           | length of the Nth value to set, belonging to `keyN`
`valueN`     | `u8[value_len]`    | value of the Nth key:value pair

Its `post_type` MUST be set to `2`.

Several key/value pairs can be set at once. A post MUST indicate it is done
setting pairs by setting a final `keyN_len` of zero.

Keys
1. MUST be UTF-8 strings.
2. MUST be between 1 and 128 codepoints in length.

The valid bytes for a value depends on the key. See the table below.

A value field MUST NOT exceed 4096 bytes (4 kibibytes) in length.

The following keys SHOULD be supported:

key       | value format | desc
----------|--------------|---------------------------------------
`name`    | UTF-8        | handle this user wishes to use as a pseudonym

A valid `name` field MUST be a valid UTF-8 string, between 1 and 32 codepoints.

To save space, a client may wish to discard from disk older versions of these
messages from a particular user.

#### 6.3.5 `post/topic`

Set a topic for a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)
`topic_len`    | `varint`           | length of the topic field
`topic`        | `u8[topic_len] `   | topic content

Its `post_type` MUST be set to `3`.

A `topic` field MUST be a valid UTF-8 string, between 0 and 512 codepoints. A
topic of length zero SHOULD be considered as the current topic being cleared.

#### 6.3.6 `post/join`

Publically announce membership in a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)

Its `post_type` MUST be set to `4`.

#### 6.3.7 `post/leave`

Publically announce termination of membership in a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name as a string of text (UTF-8)

Its `post_type` MUST be set to `5`.

## 7. Security Considerations

### 7.1 Out of scope Threats
1. Attacks on the transport layer by non-members of the cabal. This covers the *confidentiality* of a connection between peers, and prevent *eavesdropping* and *man-in-the-middle attacks*, as well as message reading/deleting/modifying.

2. Attacks that attempt to gain illicit entry to a cabal, by posing as a member (*peer entity authentication*).

3. Actors capable of deriving an Ed25519 private key from the public key via brute force, which would break data integrity and permit data insertion/deletion/modification of the compromised user's posts.

4. Attacks that stem from how cable data ends up being stored locally. For example, an attacker with access to the user's machine being able to access their stored private key or chat history on disk.

### 7.2 In-scope Threats

Documented here are attacks that can come from *within* a cabal — by those who are technically legitimate members and can peer freely with other members. It is currently assumed (until something like a version of Cabal's subjective moderation system is designed & implemented) that those who are proper members of a cabal are trusted to not cause problems for other users, but even a future moderation design would benefit from a clear outline of the attack surface.

### 7.2.1 Susceptibilities
#### 7.2.1.1 Inappropriate Use
1. An attacker could issue `post/topic` posts to edit channel topics to garbage text, offensive content, or malicious content (e.g. phishing). Since most chat programs have channel topics controlled by "moderators" or "admins", this could cause confusion if users do not realize that anyone can set these strings.
2. The list of channels in the `Channel List Response` message could be falsified to include channels that do not exist (i.e. no users have posted to them) or to omit the names channels that do exist.
    1. A possible future mitigation to this might be inclusion of an explicit `post/channel` post type, to denote channel creation, which `Channel List Response` responders would need to cite the hashes of. However, even in this scenario an attacker could trivially produce 1000s or more legitimate channels to this same end.

#### 7.2.1.2 Spoofing
1. An attacker could issue a `post/info` to alter their display name to be the same as another user, causing confusion as to which user is authoring certain chat messages.
    1. Client-side mitigation options exist, such as colourizing names by the public key of the user, or displaying a short hash digest of their key next to their name when there are multiple users sharing a name.

#### 7.2.1.3 Denial of Service
1. Authoring very large posts (gigabytes or larger) and/or a large number of smaller posts, and sharing them with others to download.
2. Making a large quantity of expensive requests (e.g. a time range request on a channel with a long chat history that covers its entire lifetime, repeatedly).
    1. Clients could implement per-connection rate limiting on requests, to prevent a degradation of service from network participants.
3. Creating an excessively large number of new channels (by writing at least one `post/text` post to each). Since channels can only be created and not removed, this has the potential to make a cabal somewhat unusable by legitimate users, if there are so many garbage channels they cannot locate real ones.
    1. New channel creation could be rate-limited, although even at a limit of 1 channel/day, it still would not take long to produce high levels of noise.
    2. Future moderation capabilities could curtail channels discovered to be garbage by issuing moderation posts that delete such channels.
4. Providing a `Post Response` with large amounts of bogus data. Ultimately the content hashes from the requested hash and the data will not match, but the machine may expend a great deal of time and computational power determining each data block's legitimacy.

#### 7.2.1.4 Repudiation
1. While all posts are cryptographically signed, a user can still claim that their private signing key was stolen, making reliable non-repudiation infeasible.
    1. A limited mitigation could involve a user posting a new `post/tombstone` post that informs other peers that this identity has been compromised, and that it should no longer be trusted as legitimate henceforth.

#### 7.2.1.5 Message Omission
1. While a machine can not issue a `post/delete` to erase another user's posts, they could choose to omit post hashes from responses to requests made to them by others. This attack is only viable if the machine is a client's only means of accessing certain data (e.g. the client was unable to directly connect to any non-attacker machines). Once that client connects to other, non-malicious machines, they will be able to "fill the gaps" of missing data within the time window & channels in which they are interested.

#### 7.2.1.6 Attacker Expulsion
1. An attacker causing problems by means of spoofing, denial of service, passive listening, or inappropriate use cannot, as per the current protocol design, be expelled from the cabal group chat. Legitimate users have no means of recourse beyond starting a new cabal and not inviting the attacker.
    1. Future moderation capabilities added to the protocol could render the attacker unable to connect with a significant portion of the cabal using e.g. transitive blocking capabilities, mitigating their inappropriate use.

### 7.2.2 Protections
#### 7.2.2.1 Replay Attacks
1. Posts that have already been ingested will be deduplicated (i.e. not re-ingested) by their content hash, so resending hashes or data does no harm in itself.

#### 7.2.2.2 Data Integrity
1. All posts are crytographically signed, and cannot be altered or forged unless a user's private key has been compromised.
2. Certain posts have implicit authorization (e.g. a `post/info` post can only alter its author's display name, and cannot be used to change another user's name), which is carried out by clients following the specification re: post ingestion logic.

#### 7.2.2.3 Privilege Escalation
Cabal has no privilege levels beyond that of a) member and b) non-member. Non-members have zero privileges (not even able to participate at the wire protocol level), and all members hold the same privileges.

### 7.3 Future Work
Future work is planned around the outer layers of cable security:

1. **Against non-member active & passive attackers**: having transport security (to prevent non-members of cabals from reading, modifying, or otherwise interacting with data sent between members) via a mechanism with end-to-end encryption
2. **Against unauthorized access**: having a handshake protocol, to prevent non-members from gaining illicit access
3. **Against inappropriate use by members**: having a system for moderation and write-access controls internal to a cabal, so that users can mitigate and expel attacks from those who have already gained legitimate membership.

## 8. Normative References
- [Vogels, W. "Eventually consistent." *Communications of the ACM*, 52, 40–44, Janury 2009](https://dl.acm.org/doi/pdf/10.1145/1435417.1435432)
- [Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", RFC 2119, July 2003](https://www.rfc-editor.org/rfc/rfc2119)
- [Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", RFC 8174, May 2017](https://www.rfc-editor.org/rfc/rfc8174)
- [BLAKE2b](https://www.blake2.net/blake2.pdf)
- [Ed25519](https://ed25519.cr.yp.to/)
- [Unicode 15.0.0](https://www.unicode.org/versions/Unicode15.0.0/)

## 9. Informative References
- [Cabal][Cabal]
- [hypercore][hypercore]
- [libsodium](https://libsodium.org)
- [Tor][Tor]
- [I2P][I2P]

[hypercore]: https://github.com/holepunchto/hypercore
[Cabal]: https://cabal.chat
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174
[BCP 14]: https://www.rfc-editor.org/bcp/bcp14
[Tor]: https://www.torproject.org
[I2P]: https://geti2p.net/en/
