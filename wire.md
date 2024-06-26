# cable

Version: 1.0-draft8

Author: Kira Oakley

Contributors: nthia, Alexander Cobleigh, Henry (cryptix), Noelle Leigh, Aljoscha Meyer, Jan Winkelmann

## Abstract

This document describes the network protocol "cable", used to facilitate
peer-to-peer group chat rooms.

## Table of Contents
* [0. Background](#0-background)
* [1. Introduction](#1-introduction)
  + [1.1 Terminology](#11-terminology)
* [2. Scope](#2-scope)
* [3. Definitions](#3-definitions)
* [4. Cryptographic Parameters](#4-cryptographic-parameters)
  + [4.1 BLAKE2b](#41-blake2b)
  + [4.2 Ed25519](#42-ed25519)
* [5. Data Model](#5-data-model)
  + [5.1 Posts](#51-posts)
    - [5.1.2 Links](#512-links)
      * [5.1.2.1 Setting links](#5121-setting-links)
    - [5.1.3 Causal Sorting](#513-causal-sorting)
    - [5.1.4 Ingesting a New Post](#514-ingesting-a-new-post)
    - [5.1.5 Keeping & Discarding Posts](#515-keeping--discarding-posts)
  + [5.2 Requests & Responses](#52-requests--responses)
    - [5.2.1 Lifetime of a Request](#521-lifetime-of-a-request)
    - [5.2.2 Limits](#522-limits)
  + [5.3 Users](#53-users)
    - [5.3.1 Names](#531-names)
    - [5.3.2 State](#532-state)
  + [5.4 Channels](#54-channels)
    - [5.4.1 Names](#541-names)
    - [5.4.2 Topics](#542-topics)
    - [5.4.3 User Membership](#543-user-membership)
    - [5.4.4 State](#544-state)
    - [5.4.5 Synchronization](#545-synchronization)
* [6. Wire Formats](#6-wire-formats)
  + [6.1 Field tables](#61-field-tables)
  + [6.2 Post Formats](#62-post-formats)
    - [6.2.1 Header](#621-header)
    - [6.2.2 `post/text`](#622-posttext)
    - [6.2.3 `post/delete`](#623-postdelete)
    - [6.2.4 `post/info`](#624-postinfo)
    - [6.2.5 `post/topic`](#625-posttopic)
    - [6.2.6 `post/join`](#626-postjoin)
    - [6.2.7 `post/leave`](#627-postleave)
  + [6.3 Message Formats](#63-message-formats)
    - [6.3.1 Message Header](#631-message-header)
    - [6.3.2 Requests](#632-requests)
      * [6.3.2.1 Post Request](#6321-post-request)
      * [6.3.2.2 Cancel Request](#6322-cancel-request)
      * [6.3.2.3 Channel Time Range Request](#6323-channel-time-range-request)
      * [6.3.2.4 Channel State Request](#6324-channel-state-request)
      * [6.3.2.4.1 Special Case of Transmission of Causal Chains](#63241-special-case-of-transmission-of-causal-chains)
      * [6.3.2.5 Channel List Request](#6325-channel-list-request)
    - [6.3.3 Responses](#633-responses)
      * [6.3.3.1 Hash Response](#6331-hash-response)
      * [6.3.3.2 Post Response](#6332-post-response)
      * [6.3.3.3 Channel List Response](#6333-channel-list-response)
* [7. Security Considerations](#7-security-considerations)
  + [7.1 Out of scope Threats](#71-out-of-scope-threats)
  + [7.2 In-scope Threats](#72-in-scope-threats)
  + [7.2.1 Susceptibilities](#721-susceptibilities)
    - [7.2.1.1 Inappropriate Use](#7211-inappropriate-use)
    - [7.2.1.2 Spoofing](#7212-spoofing)
    - [7.2.1.3 Denial of Service](#7213-denial-of-service)
    - [7.2.1.4 Repudiation](#7214-repudiation)
    - [7.2.1.5 Message Omission](#7215-message-omission)
    - [7.2.1.6 Attacker Expulsion](#7216-attacker-expulsion)
  + [7.2.2 Protections](#722-protections)
    - [7.2.2.1 Replay Attacks](#7221-replay-attacks)
    - [7.2.2.2 Data Integrity](#7222-data-integrity)
    - [7.2.2.3 Privilege Escalation](#7223-privilege-escalation)
* [8. Normative References](#8-normative-references)
* [9. Informative References](#9-informative-references)

## 0. Background
Cable is a peer-to-peer protocol for private group chats, called *cabals*.

Cable operates differently from the traditional server-client model, where the
server is a centralized authority. Instead, in Cable, every node in the network
is equal to each other. Nodes of a cabal share data with each other in order to
build a view of the state of that cabal.

The purpose of the Cable Wire Protocol is to facilitate the creation and sync
of cabals, by allowing peers to exchange cryptographically signed documents
with each other, such as chat messages, spread across various user-defined
channels.

Cabal's original protocol was based off of [hypercore][hypercore], which was
found to have limitations and trade-offs that didn't suit Cabal's needs well.
These limitations and trade-offs were the impetus for the creation of the cable
protocol.

## 1. Introduction
The purpose of the cable wire protocol is to facilitate the members of a group
chat to exchange cryptographically signed documents with each other, such as
chat messages, spread across various user-defined channels.

cable is designed to be:

* fairly simple to implement in any language with minimal dependencies
* general enough to be used across different network transports
* useful, even if written as a partial implementation
* efficient in its use of network resources, by
    * syncing only the relevant subsets of the full dataset, and
    * being compact over the wire
* not specific to any particular kind of database backend

### 1.1 Terminology
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [BCP 14][BCP 14], [RFC 2119][RFC 2119], [RFC
8174][RFC 8174] when, and only when, they appear in all capitals, as shown
here.

## 2. Scope
This protocol focuses on the over-the-wire bytes that get sent between peers
that enable the exchange of chat messages and user and channel information.

This protocol does not specify encryption nor authentication of the connection,
nor a mechanism for the discovery of network peers. For encryption and
authentication, it is RECOMMENDED to utilize the [Cable Handshake
Protocol](handshake.md).

It is assumed that peers speaking cable are already authorized to access its
data. Section [Security Considerations](#7-security-considerations) includes an
analysis of the types of anticipated attacks that authorized members may still
carry out.

## 3. Definitions
**host**: A computer running the Cable Wire Protocol.

**cabal**: An instance of a private group chat, whose host participants communicate using the Cable Wire Protocol.

**peer**: A host, whom the referred host is communicating with via the Cable Wire Protocol.

**[Ed25519][Ed25519]**: A public-key cryptographic signature system.

**hash**: A 32-byte BLAKE2b digest of a particular sequence of bytes.

**`BLAKE2b(...)`**: The application of the BLAKE2b hashing function, producing a hexadecimal string of 32 characters.

**public key**: An Ed25519 key that can be known by others.

**private key**: An Ed25519 key, used for signing data. Kept secret to all but whoever controls it.

**signature**: A unique sequence of binary data that can be produced by combining a private key and an input text. The signature can then be verified by the private key's corresponding public key.

**user**: A pair of Ed25519 keys -- a public key and private key -- identifying a distinct person or program in a cabal. A user is known by others by their public key.

**post**: A data structure with various fields, conforming to a particular format. Always contains a signature from the private key of the user who created it. See [6. Wire Formats](#6-wire-formats) for more details.

**link**: A post, P, is said to "link" to another post, Q, if the `links` field of P contains the BLAKE2b hash Q.

**`P → Q`**: A short-hand convention for indicating that the post P links to the post Q.

**`hasChain(P, Q)`**: True if the host knows of some sequence of posts `R₀...Rₙ`, where `n >= 0`, such that `P → R₀ → ... → Rₙ → Q`; false otherwise.

**head**: A post Q is a "head" if there is no post P known to the host such that `P → Q`.

**UNIX epoch**: Midnight UTC on January 1st, 1970.

**timestamp**: A point in time represented by the number of milliseconds since the UNIX epoch.

**`now()`**: A function that returns the host's current local timestamp.

**causal sort**: A specific ordering of a set of posts. See [5.1.3 Causal Sorting](#513-causal-sorting) for a full explanation.

**latest**: When used in the context of ordering a set of posts, this refers to the post that, from a particular host's perspective at a given moment in time, has the highest causal sort value.

**channel**: A conceptual object with its own unique name, that users can participate in by authoring posts that reference the channel.

**request**: A binary payload originating from a particular host.

**response**: A binary payload, traversing the network from a particular host back to the host who issued the original request.

**message**: Either a request or a response.

**`req_id`**: A 64-bit number that identifies a particular request traversing in the network, and any corresponding responses.

**requester**: A host authoring a request, to be sent to other peers.

**responder**: A host authoring a response, in reference to a request sent to them.

**varint**: a variable-length unsigned integer, encoded as [Unsigned LEB128](https://en.wikipedia.org/wiki/LEB128#Unsigned_LEB128).

**Unicode**: The [Unicode 15.0.0 standard](https://www.unicode.org/versions/Unicode15.0.0/).

**UTF-8**: The "UTF-8" encoding scheme outlined in [Chapter 2 Section 5](https://www.unicode.org/versions/Unicode15.0.0/ch02.pdf#G11165) of the Unicode 15.0.0 specification.

## 4. Cryptographic Parameters

### 4.1 BLAKE2b
The following are the general parameters to be used with BLAKE2b.

- Digest byte length: 32
- Key byte length (not used): 0
- Salt (hexadecimal): `5b6b 41ed 9b34 3fe0`
- Personalization (hexadecimal): `5126 fb2a 3740 0d2a`

### 4.2 Ed25519

Cable uses the Ed25519-SHA-512 variant of [Ed25519][Ed25519]:

- *b* = 256
- *H* = SHA-512
- *q* = 2<sup>255</sup> - 19
- The 255-bit encoding of **F**<sub>2<sup>255</sup>-19</sub> is the usual little-endian encoding of { 0, 1, ..., 2<sup>255</sup> - 20 }
- ℓ = 2<sup>252</sup> + 27742317777372353535851937790883648493
- *d* = -121655 / 121666 ∈ **F**<sub>q</sub>
- *B* is the unique point (*x*,4/5) ∈ *E* for which *x* is positive

Henceforth, consider use of the function **`verify(P)`**, which returns true or
false, to refer the result of verifying the Ed25519 signature of a post, P,
against its contents. This is done by checking `P.signature` against all fields
in P following (but not including) the `signature` field. The fields that a
post may contain are described in the next section.

## 5. Data Model

### 5.1 Posts
A cabal is comprised of posts, made by its participants.

All posts have, at minimum, the following fields:

field        | desc
-------------|-------------------------------------------------------
`public_key` | public key that authored this post
`signature`  | signature of the fields that follow
`num_links`  | how many hashes this post links back to (0+)
`links`      | hashes of the latest posts in this channel/context
`post_type`  | see custom post type sections below
`timestamp`  | milliseconds since UNIX Epoch

All posts are cryptographically signed, in order to prove the identity of its
author and that it has not been tampered with.

A post's author is identified by the `public_key` field, and the `signature`
field provides a cryptographic signature of all of the other fields in the
post.

Depending on the `post_type`, a post will have additional fields. Those fields,
as well as the others mentioned above, are described in much more detail in
Section 6.

When a user "makes a post", they are only writing to some local storage,
perhaps indexed by the hash of the content for easy querying. Posts are only
sent to other peers in response to queries about them (e.g. chat messages
within some time range).

Cable defines the following post types, which are described in much more detail
in Section 6:

| post type    | description |
|--------------|-------------|
| `post/text`  | a textual chat message, posted to a channel |
| `post/delete`| the deletion of a previously published post |
| `post/info`  | set or clear informative key/value pairs on a user  |
| `post/topic` | set or clear a channel's topic string |
| `post/join`  | announce membership to a channel |
| `post/leave` | announce cessation of membership to a channel |

#### 5.1.2 Links
Any post can be referenced by its hash: a BLAKE2b hash of a post in its
entirety.

Every post has a field called `links`: a list of BLAKE2b hashes. This fields
allows a post to refer to 0 or more other posts by means of specifying those
posts' hashes.

The hash for a post is produced by putting a post's verbatim binary content,
including the post header, through the BLAKE2b function. (The structure of
posts is described below, in Section 6.)

Referencing a post by its hash provides a causal proof: it demonstrates that a
post must have occurred after all of the other posts referenced. This property
can be useful for ordering chat messages, since a timestamp alone can cause
ordering problems if a host's hardware clock is skewed, or a timestamp is
spoofed.

##### 5.1.2.1 Setting links
Each post made to a channel should link to all known heads in that channel.
This is explained in detail below.

Let `linkableTypes` be the set of strings `{ 'post/text', 'post/topic', 'post/join', 'post/leave' }`.

When a post 𝑃 is created such that `𝑃.type ∈ linkableTypes`, it SHOULD link to
all other posts 𝑄ᵢ known to the host that meet the following criteria:

1. `𝑄ᵢ.type ∈ linkableTypes`
2. `𝑃.channel == 𝑄ᵢ.channel`
3. There exists no known post 𝑅 such that `BLAKE2b(𝑄ᵢ) ∈ 𝑅.links` (i.e. that 𝑄ᵢ is a head)

#### 5.1.3 Causal Sorting
Causal sorting is the act of sorting a set of posts in ascending order.

Let the `>` and `<` binary operators, when used to compare two strings, refer to a lexicographic comparison of the two strings, in ascending order.

The comparison of two posts, Q and P, happens in a series of comparisons:

1. If `hasChain(Q, P)` is true, then `Q > P`. Otherwise, continue.

2. If `hasChain(P, Q)` is true, then `P > Q`. Otherwise, continue.

3. If `Q.timestamp > P.timestamp`, then `Q > P`. Otherwise, continue.

4. If `Q.timestamp == P.timestamp` and `BLAKE2b(Q) > BLAKE2b(P)`, then `Q > P`. Otherwise, continue.

5. `P > Q`.

At a high level, the highest priority for sorting is the existence of a chain
of links between two posts such that one post must have happened before the
other. Failing that, timestamp is used to determine order. Finally, failing
that, the hexadecimal encoding of the post hashes are compared.

#### 5.1.4 Ingesting a New Post
When a host receives any new post, P, in a Post Response message, it MUST pass
the following criteria to be accepted and stored by said host:

1. `verify(P)` is true. That is, the signature of P matches the contents of P.

2. P is well-formed. This means it is formatted according to [6.2 Post Formats](#62-post-formats). This also means posts with an unknown `post_type` are discarded.

3. `P.timestamp < now() + ONE_WEEK` (604800000 milliseconds). Posts that are a
   week or more into the future are discarded.

#### 5.1.5 Keeping & Discarding Posts
A host MAY discard any post at any time, whether in the interest of saving disk
space, processing time, or not wanting to contribute to the propagate of
certain content.

However, in the interest of keeping synchronization robust and effective, it is
suggested that hosts prefer to KEEP posts in the following circumstances:

1. Posts representing channel state that are latest, second-latest, or
   third-latest. It is preferred to hold onto second-latest and third-latest in
   case the latest post is deleted, so there is another older, known value to
   fall back to.

2. `post/text` posts newer than the oldest history for a channel of which they
   are a member. (e.g. Choosing to discard chat messages older than 6 months.)

3. All `post/delete` posts.


### 5.2 Requests & Responses
Requests and responses are the two types of messages in the Cable Wire
Protocol.

A request is created by a host, and sent to one or more peers. Under certain
conditions (described below in 5.2.2), those peers can then forward that
request to other additional peers. A request may be forwarded again like this
several times, and reach several hosts in the network.

Each host in this tree of peers that a request passes through may produce one
or more responses in reply to the request. A request and a response are set to
share the same unique value in their `req_id` field, so hosts know that a
response is regarding a particular request.

Responses travel along the reciprocal path of the original request that was
sent. So, if host `A` sends a request to `B`, and `B` forwards that request to
`C`, then `B` could issue responses back to `A` directly, while any responses
from `C` would route back through `B`, who would route that response back to
`A`.

Below is a brief listing of the request and response types defined. A more
detailed normative section follows in Section 6.

| request name                  | description |
|-------------------------------|-------------|
| Post Request                  | request posts identified by a particular set of hashes |
| Cancel Request                | abort a previously issued request |
| Channel Time Range Request    | request post hashes of chat messages and deletions within a time range |
| Channel State Request         | request post hashes describing a given channel's state |
| Channel List Request          | list all known channel names |

| response name                 | description |
|-------------------------------|-------------|
| Hash Response                 | list hashes, pertaining to some request |
| Post Response                 | list posts, pertaining to some request |
| Channel List Response         | list channel names |

#### 5.2.1 Lifetime of a Request
Any given request on any host is identified by its `req_id`, and is either
considered to be "alive" or "concluded". When a request becomes concluded from
a host's perspective, that request and its `req_id` may be safely forgotten.

In the lifetime of a given request, there are two mutually exclusive roles an
involved host may have, each with their own criteria for considering a request
concluded.

The first role is the **requester**, who is the host who allocated the new
request and has sent it to a peer. That request is considered concluded when
any of the following criteria are true:

1. Received a response message indicating there are no further responses to be
   expected. (See Sections 6.3.3.1 through 6.3.3.3 for details on specific
   terminating responses.)
2. Sent the responder a Cancel Request with the original request's `req_id`
   specified.
3. The connection to the peer was lost.

The second role is the **responder**, who is the host receiving the request,
and may send one or more responses back to the requester, depending on the
response type. A given request is considered concluded when any of the
following criteria are true:

1. Sent a response message indicating there are no further responses to be
   expected. (Again, see Sections 6.3.3.1 through 6.3.3.3.)
2. Received a Cancel Request with the original request's `req_id` specified.
3. The connection to the peer was lost.

Further, any host who receives an incoming request with a `req_id` equal to a
known alive request's `req_id` SHOULD be discarded.

#### 5.2.2 Limits
Some requests have a `limit` field specifying an upper bound on how many hashes
a host wishes to receive in response. A peer responding to such a request
MUST honour that limit by counting how many hashes they send back to the
requester, including hashes received through other peers that the responding
host has forwarded that request to.

For example, assume `A` sends a request to `B` with `limit = 50`. `B` forwards
the request to `C` and `D`. `B` may send back 15 hashes to `A` at first, which
means there are now a maximum of `50 - 15 = 35` hashes left for `C` and `D`
combined for `B` to potentially send back.

A requester receiving more than `limit` hashes MAY choose to discard the
extraneous ones.

### 5.3 Users

#### 5.3.1 Names
- A valid user name MUST be a UTF-8 string.
- A valid user name MUST between 1 and 32 codepoints.

Codepoints are used instead of bytes to try to offset some of the social
privilege embedded in the UTF-8 encoding. Latin-derived languages get a big
advantage in UTF-8 when it comes to how many characters they can encode in a
fixed number of bytes: one can encode 64 English letters in 64 bytes, but only
21 hiragana or sanskrit characters.

#### 5.3.2 State
A user has the following properties:

1. a public key
2. a set of key/value pairs

A user's public key is known because it is included in every post made by them.

The set of key/value pairs comes from the latest known `post/info` post made by
that user, which contains key/value pairs.

### 5.4 Channels
A channel is a named collection consisting of the following:

1. chat messages (`post/text`)
2. user joins and leaves (`post/join` or `post/leave`).
3. a topic string (`post/topic`)

A user writing a chat message or join to a channel implies that that named
channel has now been created, if it hasn't already been, and thus MUST be returned
in future Channel List Requests.

#### 5.4.1 Names
- A valid channel name MUST be a UTF-8 string.
- A valid channel name MUST between 1 and 64 codepoints.
- Channel names are case insensitive. (e.g. "cAbAL" is the same channel as "cabal")

#### 5.4.2 Topics
- A valid channel topic MUST be a UTF-8 string.
- A valid channel topic MUST be between 0 and 512 codepoints.
- If there is no known topic set for a channel, it MUST be considered the empty string (`""`).
- A channel topic string set by a user to the empty string MUST be considered as there being no topic currently set.

#### 5.4.3 User Membership
A user makes a post "to a channel" if they have set the `channel` field on said
post to the name of that channel.

A user is a member of a channel at a particular point in time if and only if,
from a host's perspective, that user has issued a `post/join`, `post/text`, or
`post/topic` to that channel and has not issued a `post/leave` since.

Hosts SHOULD issue a `post/join` post before issuing any other posts to a
channel, and SHOULD issue a `post/leave` post when the user of the program
expresses a desire to leave a channel.

#### 5.4.4 State
A channel at any given moment, from the perspective of a host, is fully
described by the following:

1. The latest `post/info` post of all members.
2. The latest of all users' `post/join` or `post/leave` posts to the channel.
3. The latest `post/topic` post made to the channel.
4. All known `post/text` posts made to channel.

#### 5.4.5 Synchronization
There are two types of requests that allow a host to track the state of any
channels that a host is interested in: the Channel State Request allows
tracking general state (who is in the channel, its topic, information about
users in the channel), and the Channel Time Range Request tracks the history of
chat messages within that channel and any deletions of those chat messages.

When building a Channel Time Range Request, hosts SHOULD set the `time_end` and
`time_start` fields in the following manner, to reliably track recent channel
chat history:

```
time_end = now()
time_start = now() - WINDOW_WIDTH
```

where `now()` is the current system timestamp, and `WINDOW_WIDTH` is the size
of the "rolling window" to track chat messages within, in milliseconds. Hosts are
RECOMMENDED to use a `WINDOW_WIDTH` of one week (25,200,000 milliseconds).

This value MAY be customized, since there are network environments where users
may be offline for up to several months at a time, and a wider rolling window
would be necessary to ensure those chat messages are synchronized when such a
user connects once again to other peers in the cabal.

Other `time_start`/`time_end` values are potentially useful, such as querying
farther back in time to acquire a more complete set of chat message history.

## 6. Wire Formats

### 6.1 Field tables
The following tables of fields describe how to encode binary payloads using a
combination of varints and fixed-length unsigned byte sequences. Each table
also defines the exact byte order required to produce a particular type of
payload, where the first field describes the first set of bytes and so on.
Taken together, field tables describe how to produce and decode binary payloads
from and into the different post and message types described in subsequent
sections.

field      | type     | desc
-----------|----------|-------------------------------------------------------------
`foo`      | `u8`     | description of the field `foo`
`bar`      | `u8[4]`  | description of the field `bar`

The above example describes a 5 byte binary payload. The payload's first byte
is defined to be `foo`. The 1 byte `foo` is followed immediately by `bar`,
which is defined to be 4 bytes long.

If `foo = 17` and `bar = [3,6,8,64]`, the binary payload would be as follows:

```
 foo   bar
 0x11  0x03 0x06 0x08 0x40
 ^^^^  ^^^^^^^^^^^^^^^^^^^
 |     ╰-------------------- bar = [3, 6, 8, 64]
 |
 ╰-------------------------- foo = 17
```

The following data types are used:
- `u8`: a single unsigned byte.
- `u8[N]`: a sequence of exactly `N` unsigned bytes.
- `varint`: a variable-length unsigned integer. 

### 6.2 Post Formats

#### 6.2.1 Header
Every post MUST begin with the following 6-field header:

field        | type              | desc
-------------|-------------------|-------------------------------------------------------
`public_key` | `u8[32]`          | public key that authored this post
`signature`  | `u8[64]`          | signature over the fields that follow
`num_links`  | `varint`          | how many hashes this post links back to (0+)
`links`      | `u8[32*num_links]`| hashes of the latest posts in this channel/context
`post_type`  | `varint`          | see custom post type sections below
`timestamp`  | `varint`          | milliseconds since UNIX Epoch

The `signature` for a post is produced by signing the concatenation of all fields of the post
immediately after the `signature` field, including the `post_type`-specific
fields specified in the sections that follow, using the Ed25519 signature
scheme.

The post type sections below document the fields that MUST follow these initial
fields, depending on the `post_type`.

The protocol MAY be extended by implementers by creating additional
`post_type`s. Implementers MUST only use `post_type > 255`. The
first 256 are reserved for core protocol use.

All fields specified in the subsequent subsections MUST be present for a post
of a given `post_type`.

#### 6.2.2 `post/text`

Post a chat message to a channel.

field          | type               | desc
---------------|--------------------|---------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name (UTF-8)
`text_len`     | `varint`           | length of the text field, in bytes
`text`         | `u8[text_len] `    | chat message text (UTF-8)

`post_type` MUST be set to `0`.

The `text` body of a chat message MUST be a valid UTF-8 string. Its length MUST
NOT exceed 4 kibibytes (4096 bytes).

#### 6.2.3 `post/delete`

Request that peers encountering this post delete the referenced posts from
their local storage, and not store the referenced posts in the future.

field           | type                   | desc
----------------|------------------------|-------------------------
`num_deletions` | `varint`               | how many hashes of posts there are to be deleted
`hashes`        | `u8[32*num_deletions]` | concatenated hashes of posts to be deleted

`post_type` MUST be set to `1`.

A host interpreting this post MUST only perform a local deletion of the
referenced posts if the author (`post.public_key`) matches the author of the
post to be deleted (i.e. only the user who authored a post may delete it).

#### 6.2.4 `post/info`

Set public information about one's self.

field        | type               | desc
-------------|--------------------|-------------------------
`num_keypairs`|`varint`           | how many key/value pairs follow
`key1_len`   | `varint`           | length of the first key to set, in bytes
`key1`       | `u8[key1_len]`     | name of the first key to set (UTF-8)
`value1_len` | `varint`           | length of the first value to set, belonging to `key1`, in bytes
`value1`     | `u8[value_len]`    | value of the first key:value pair
...          |                    |
`keyN_len`   | `varint`           | length of the Nth key to set, in bytes
`keyN`       | `u8[keyN_len]`     | name of the Nth key to set (UTF-8)
`valueN_len` | `varint`           | length of the Nth value to set, belonging to `keyN`, in bytes
`valueN`     | `u8[value_len]`    | value of the Nth key:value pair

`post_type` MUST be set to `2`.

Several key/value pairs MAY be set at once.

A `post/info` post is a complete description of a user's self-published
information. The latest `post/info` post by a user fully replaces any
previously known versions. They are not additive. If a key is set in one
`post/info` but not the subsequent one, that key MUST be treated as being set
to its default. See the table of keys/values below for each key's default
value.

Keys MUST be UTF-8 strings, and MUST be between 1 and 128 codepoints in length.

A value field MUST NOT exceed 4096 bytes (4 kibibytes) in length.

The valid bytes for a value depends on the key. See the table below.

The following keys MUST be supported:

key             | value format | desc
----------------|--------------|---------------------------------------
`name`    | UTF-8        | the name this user wishes to be known as. The default value is a string containing the hexadecimal encoding of the user's public key.
`accept-role`   | varint       | determines whether this user accepts moderation roles being assigned them. Default value is 1 (accepts roles).

The following keys are RECOMMENDED to be supported:

key             | value format | desc
----------------|--------------|---------------------------------------
`accept-role`   | varint       | determines whether this user accepts moderation roles being assigned them. Default value is 1 (accepts roles).


A post MAY contain other keys, even if they are unknown to an implementation at the time.

To save space, a host MAY discard older versions of a `post/info` for a user.

#### 6.2.5 `post/topic`

Set a topic for a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name (UTF-8)
`topic_len`    | `varint`           | length of the topic field, in bytes
`topic`        | `u8[topic_len] `   | topic content (UTF-8)

`post_type` MUST be set to `3`.

A `topic` field MUST be a valid UTF-8 string, between 0 and 512 codepoints. A
topic of length zero MUST be considered as the current topic being cleared to
the empty string, "".

#### 6.2.6 `post/join`

Publicly announce membership in a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name (UTF-8)

`post_type` MUST be set to `4`.

#### 6.2.7 `post/leave`

Publicly announce termination of membership in a channel.

field          | type               | desc
---------------|--------------------|-------------------------------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name (UTF-8)

`post_type` MUST be set to `5`.

### 6.3 Message Formats

#### 6.3.1 Message Header

All messages MUST begin with the following header fields:

field         | type     | desc
--------------|----------|-------------------------------------------------------------
`msg_len`     | `varint` | number of bytes in rest of message, not including the `msg_len` field
`msg_type`    | `varint` | a type identifier for the message, which controls which fields follow this header
`req_id`      | `u8[8]`  | unique id of this request (random)

Message-specific fields follow after the `req_id`.

Each request and response type has a unique `msg_type` (see below), which
controls which fields will immediately follow this header.

Hosts encountering a `msg_type` they do not know how to parse MUST ignore and
discard it.

The request ID, `req_id`, is a 64-bit number, generated randomly by the
requester. It is used to uniquely identify the request during its lifetime
across the peers who may handle it.

When a host forwards a request to further peers, the `req_id` MUST NOT be
changed, so that routing loops can be more easily detected by peers in the
network.

The protocol MAY be extended by implementers by creating additional
`msg_type`s. Implementers MUST only use `msg_type > 255`. The
first 256 are reserved for core protocol use.

#### 6.3.2 Requests

##### 6.3.2.1 Post Request

Request a set of posts, given their hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`hash_count` | `varint`            | number of hashes to request
`hashes`     | `u8[32*hash_count]` | hashes, concatenated together

`msg_type` MUST be set to `2`.

Results are provided by one or more Post Response messages.

The responder SHOULD immediately return what data is locally available, rather
than holding on to the request in anticipation of perhaps seeing the requested
hashes in the future.

Hosts MUST be able to handle receiving posts of unexpected types appearing in
responses, and MAY choose for themselves whether to discard them or not.

##### 6.3.2.2 Cancel Request

Conclude a given `req_id` and stop receiving responses for that request.

This request can be used to terminated previously sent, long-lived requests.

field        | type                | desc
-------------|---------------------|-------------------------------------
`cancel_id`  | `u8[4]`             | the `req_id` of the request to be cancelled

Receiving this request indicates that any further responses sent back with a
`req_id` matching the given `cancel_id` will be ignored and discarded.

`msg_type` MUST be set to `3`.

`cancel_id` MUST be set to the `req_id` of the request to be cancelled.

There is no expected response to this request.

Like any other request, this request MUST have its own unique `req_id` in order
to function as intended. `cancel_id` is used to set the request identifier to
cancel, not the `req_id`.

A peer receiving a Cancel Request SHOULD forward it along the same route and
peers it forwarded the original message with `req_id = cancel_id`, to the same
peers as the original request, so that all peers who know of the original
request are notified.

##### 6.3.2.3 Channel Time Range Request

Request chat messages and chat message deletions written to a channel between a
start and end time, optionally subscribing to future chat messages.

field          | type               | desc
---------------|--------------------|----------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes
`channel`      | `u8[channel_len] ` | channel name (UTF-8)
`time_start`   | `varint`           | milliseconds since UNIX Epoch (inclusive)
`time_end`     | `varint`           | milliseconds since UNIX Epoch (exclusive)
`limit`        | `varint`           | maximum number of hashes to return

`msg_type` MUST be set to `4`.

A responder receiving this request MUST respond with 1 or more Hash Response
messages.

`time_start` is the post with the oldest timestamp the requester is interested
in; `time_end` is the newest.

A responder SHOULD include the hashes of all known `post/text` and
`post/delete` posts made to a channel between `time_start` and `time_end`.

Only when `time_end` is set to 0 SHOULD the responder keep this request alive
even after all known hashes in the range `time_start` to `now()` are provided,
and continue to receive any new chat messages that the responder learns of in
the future, so long as this request is still alive.

A responder SHOULD respond with all known chat messages within the requested
time range, though they may desire not to in certain circumstances,
particularly if a channel has a very long history and the responding host lacks
sufficient resources at the time to return thousands or hundreds of thousands
of chat message hashes.

A responder is RECOMMENDED to send posts in reverse chronological order by post
timestamp, so that, if the requester sees that they received a number of hashes
equal to their `limit`, they can be assured that they now have all posts newer
than the oldest post hash the responder did send, and can make subsequent
requests to paginate backwards in time.

A `limit` of 0 MUST be understood as having no maximum on the number of hashes
the requester wishes to receive.

##### 6.3.2.4 Channel State Request

Request posts that describe the current state of a channel and its members, and
optionally subscribe to future state changes.

field          | type               | desc
---------------|--------------------|-----------------------------------
`channel_len`  | `varint`           | length of the channel's name, in bytes 
`channel`      | `u8[channel_len] ` | channel name (UTF-8)
`future`       | `varint`           | whether to include live / future state hashes

`msg_type` MUST be set to `5`.

A responder receiving this request MUST respond with 1 or more Hash Response
messages, with only posts that relate to the current state of the channel.
Requesters MAY discard hashes mapping to posts that do not contain relevant
information.

See Section 5.4.4 for context on what comprises channel state. Chat messages
SHOULD NOT be included in responses to this request.

`future` MUST be set to either `1` or `0`.

If `future = 1`, the responder SHOULD respond with future channel state changes
as they become known to the responder, and the request SHOULD be held open
indefinitely on both the requester and responder side until a Cancel Request is
issued by the requester, or the responder elects to end the request by sending
a Hash Response with `hash_count = 0`.

If `future = 1` and a post that is part of the latest state for a channel is
deleted, the responder MUST immediately send the hash of the next-latest
piece of state of that same type as a Hash Response. For example, if the latest
`post/topic` setting the channel's topic string is deleted by its author with a
`post/delete` post, the hash of the second-latest `post/topic` for that channel
SHOULD be sent.

If `future = 0`, only the latest state posts will be included, and the request
MUST NOT be held open.

##### 6.3.2.4.1 Special Case of Transmission of Causal Chains
There is an additional requirement of responders to this request, to be
followed when one of the hashes returned by a responder belongs to a particular
post, P, such that the following is true about any other post, Q:

```
hasChain(P, Q) && Q.timestamp > P.timestamp
```

That is, when there is an earlier known post Q in the causal chain starting at
P that has a later timestamp than P (the latest known post). This can happen
because of natural clock skew -- someone's clock is several hours ahead or
behind due to not syncing to internet time services -- or intentional
inappropriate use.

In order to ensure sync operates consistently in these cases, when the above is
true about a post P, the responder SHOULD also include the hashes of all posts
that make up the causal chain `P -> ... -> Q` (including Q).

This is very important, because without these post hashes, there may be, for
example, a causal chain like `P -> R -> Q`, and if a host knows only P and Q
but not R, they would lack a causal chain and conclude that Q is newer than P,
since `Q.timestamp > P.timestamp`.

##### 6.3.2.5 Channel List Request

Request a list of known channels from peers.

field          | type               | desc
---------------|--------------------|-----------------------------------
`offset`       | `varint`           | number of channel names to skip (`0` to skip none)
`limit`        | `varint`           | maximum number of channel names to return

`msg_type` MUST be set to `6`.

This request returns zero or more Channel List Response messages.

If `limit` is 0, the responder MUST respond with all known channels (after
skipping the first `offset` entries).

The `offset` field can be combined with the `limit` field to allow hosts to
paginate through the list of all channel names known by a peer.

#### 6.3.3 Responses
Multiple responses MAY be generated for a single request, where results trickle
in from the set of responding peers.

Every response MUST begin with the message header detailed in Section 6.3.1,
followed by bytes specific to the response `msg_type`, detailed in the sections
that follow.

Responses containing an unknown `req_id` SHOULD be ignored.

Responders MUST set a response's `req_id` set to the same `req_id` of the
request they are responding to.

##### 6.3.3.1 Hash Response

Respond with a list of zero or more hashes.

field        | type                | desc
-------------|---------------------|-------------------------------------
`hash_count` | `varint`            | number of hashes to follow
`hashes`     | `u8[hash_count*32]` | hashes, concatenated together

`msg_type` MUST be set to `0`.

A responder MUST send a Hash Response message with `hash_count = 0` to indicate
that they do not intend to return any further hashes for the given `req_id` and
they have concluded the request on their side.

##### 6.3.3.2 Post Response

Respond with a list of posts, in response to a Post Request.

field        | type                | desc
-------------|---------------------|--------------------------
`post0_len`  | `varint`            | length of first post, in bytes
`post0_data` | `u8[post0_len]`     | first post
`post1_len`  | `varint`            | length of second post, in bytes
`post1_data` | `u8[post1_len]`     | second post
`...`        |                     |
`postN_len`  | `varint`            | length of Nth post, in bytes
`postN_data` | `u8[postN_len]`     | Nth post

`msg_type` MUST be set to `1`.

A recipient reads zero or more (`post_len`,`post_data`) pairs until a
`post_len` set to 0 is encountered.

A responder MUST send a Post Response message with `post0_len = 0` to indicate
that they do not intend to return any further posts for the given `req_id` and
they have concluded the request on their side.

Hosts SHOULD hash an entire post to check whether it is post that it was
expecting (i.e. had sent out a Post Request for). However, misbehaving peers
may end up providing posts that are still coincidentally useful to the host,
so hosts MAY elect to keep certain posts.

Each post MUST contain the complete and valid body of a known post type
(Section 6.2).

##### 6.3.3.3 Channel List Response

Respond with a list of names of known channels.

field          | type                | desc
---------------|---------------------|-------------------------------------
`channel0_len` | `varint`            | length in bytes of the first channel name, in bytes
`channel0`     | `u8[channel_len]`   | the first channel name (UTF-8)
`...`          |                     |
`channelN_len` | `varint`            | length in bytes of the Nth channel name, in bytes
`channelN`     | `u8[channel_len]`   | the Nth channel name (UTF-8)

`msg_type` MUST be set to `7`.

A recipient reads the zero or more (`channel_len`,`channel`) pairs until
`channel_len = 0`.

The channel names SHOULD be sorted lexicographically in ascending order, so that requesters can effectively sent subsequent requests to paginate through the results.

Unlike Hash Response and Post Response, only a single Channel List Response may
be sent. Sending this response concludes the request for the sender.

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
2. The list of channels in the Channel List Response message could be falsified to include channels that do not exist (i.e. no users have posted to them) or to omit the names channels that do exist.
    1. A possible future mitigation to this might be inclusion of an explicit `post/channel` post type, to denote channel creation, which Channel List Response responders would need to cite the hashes of. However, even in this scenario an attacker could trivially produce 1000s or more legitimate channels to the same end.

#### 7.2.1.2 Spoofing
1. An attacker could issue a `post/info` to alter their display name to be the same as another user, causing confusion as to which user is authoring certain chat messages.
    1. Host-side mitigation options exist, such as colourizing names by the public key of the user, or displaying a short hash digest of their key next to their name when there are multiple users sharing a name.

2. Posts are signed by their author, but do not contain a reference to the cabal they were written to. An attacker could perform a replay attack by sharing a post made on one cabal into another cabal, and it would appear authentic. This is mitigated by the Handshake Protocol specification having an explicit advisory to never re-use identity keys between cabals, allowing for users and hosts to assume that any public keys that are the same across cabals are completely coincidental and not the same person.

3. Messages are not signed. If a message is forwarded from one peer to another, that peer has the opportunity to rewrite that message. Note that this only applies to authorized peers, who have completed a Cable Handshake, and not any intermediary network node.

#### 7.2.1.3 Denial of Service
1. Authoring very large posts (gigabytes or larger) and/or a large number of smaller posts, and sharing them with others to download.
2. Making a large quantity of expensive requests (e.g. a time range request on a channel with a long chat history that covers its entire lifetime, repeatedly).
    1. Hosts could implement per-connection rate limiting on requests, to prevent a degradation of service from network participants.
3. Creating an excessively large number of new channels (by writing at least one `post/text` post to each). Since channels can only be created and not removed, this attack has the potential to make a cabal somewhat unusable by legitimate users, if there are so many garbage channels they cannot locate real ones.
    1. New channel creation could be rate-limited, although even at a limit of 1 channel/day, it still would not take long to produce high levels of noise.
    2. Future moderation capabilities could curtail channels discovered to be garbage by issuing moderation posts that delete such channels.
4. Providing a Post Response with large amounts of bogus data. Ultimately the content hashes from the requested hash and the data will not match, but the machine may expend a great deal of time and computational power determining each data block's legitimacy.

#### 7.2.1.4 Repudiation
1. While all posts are cryptographically signed, a user can still claim that their private signing key was stolen, making reliable non-repudiation infeasible.
    1. A limited mitigation could involve a user posting a new, hypothetical `post/tombstone` type post that informs other peers that this identity has been compromised, and that it should no longer be trusted as legitimate henceforth.

#### 7.2.1.5 Message Omission
1. While a machine can not issue a `post/delete` to erase another user's posts, they could choose to omit post hashes from responses to requests made to them by others. This attack is only viable if the machine is a host's only means of accessing certain data (e.g. the host was unable to directly connect to any non-attacker machines). Once that host connects to other, non-malicious machines, they will be able to "fill the gaps" of missing data within the time window & channels in which they are interested.

#### 7.2.1.6 Attacker Expulsion
1. An attacker causing problems by means of spoofing, denial of service, passive listening, or inappropriate use cannot, as per the current protocol design, be expelled from the cabal group chat. Legitimate users have no means of recourse beyond starting a new cabal and not inviting the attacker.
    1. Future moderation capabilities added to the protocol could render the attacker unable to connect with a significant portion of the cabal using e.g. transitive blocking capabilities, mitigating their inappropriate use.

### 7.2.2 Protections
#### 7.2.2.1 Replay Attacks
1. Posts that have already been ingested will be deduplicated (i.e. not re-ingested) by their content hash, so resending hashes or data does no harm in itself.

#### 7.2.2.2 Data Integrity
1. All posts are cryptographically signed, and cannot be altered or forged unless a user's private key has been compromised.
2. Certain posts have implicit authorization (e.g. a `post/info` post can only alter its author's display name, and cannot be used to change another user's name), which is carried out by hosts following the specification re: post ingestion logic.

#### 7.2.2.3 Privilege Escalation
Cabal has no privilege levels beyond that of a) member and b) non-member. Non-members have zero privileges (not even able to participate at the wire protocol level), and all members hold the same privileges.

## 8. Normative References
- [Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", RFC 2119, July 2003](https://www.rfc-editor.org/rfc/rfc2119)
- [Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", RFC 8174, May 2017](https://www.rfc-editor.org/rfc/rfc8174)
- [BLAKE2b](https://www.blake2.net/blake2.pdf)
- [Ed25519][Ed25519]
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
[Ed25519]: https://ed25519.cr.yp.to/
