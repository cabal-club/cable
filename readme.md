# cable (cabal protocol)

This document describes the bytes over the wire to speak the cable protocol.

You will need libsodium bindings or their equivalents for:

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

# messages

All messages begin with a `msg_len` and a `msg_type` varint:

_**note**: cable uses protobuf-style
[varints](https://developers.google.com/protocol-buffers/docs/encoding#varints). For an example
implementation of varint encoding/decoding, see the
[nodejs varint package](https://www.npmjs.com/package/varint)_

field     | type   | desc
----------|--------|-------------------------------------------------------------
msg\_len  | varint | number of bytes in rest of message, i.e. not including the `msg_len` field
msg\_type | varint | see fields below

More fields follow after the `msg_type`.

Requests and responses all have a unique `msg_type`.

Clients may experiment with custom message types beyond the ids used by this specification
(where `msg_type>=64`). Or for more application-specific tasks, clients can use a sub-protocol.

Requests and responses may set a `circuit_id` to establish a path between nodes so that intermediate
nodes can more intelligently route requests without duplicating messages to multiple peer
connections. Otherwise intermediate nodes would need to reply to multiple connected peers with
results even if those peers are not involved in a query.

## responses

There are 3 types of responses:

* hash response - a list of hashes (most queries return this)
* data response - a list of data chunks (in response to a hash query)
* protocol response - response with a custom application-specific payload

Multiple responses may be generated for a single request and results trickle in from peers.

## requests

Each request begins with these two fields (after the `msg_type` applicable to all messages):

field       | type     | desc
------------|----------|--------------------------------------------------------------
msg\_len    | varint   | number of bytes in the message after this field
msg\_type   | varint   |
req\_id     | u8[4]    | unique id of this request (random)
circuit\_id | u8[4]    | circuit for an established path or `[0,0,0,0]` for no circuit
ttl         | varint   | number of hops remaining

More fields follow for different request types below.

When `ttl > 0`, a peer can choose to forward a request along to other peers and forward the response
back to the peer that made the request. Each peer performing this action should decrement the `ttl`
by one.

When forwarding a request, do not change the `req_id` so that routing loops can be more easily
detected by peers.

## message types

This section describes the list of core cable message types.

### hash response (`msg_type=0`)

Respond with a list of hashes.

field       | type                | desc
------------|---------------------|------------------------------------------------
msg\_len    | varint              | number of bytes in the message after this field
msg\_type   | varint (=0)         |
req\_id     | u8[4]               | id this is in response to
hash\_count | varint              | number of hashes in the response
hashes      | u8[hash\_count\*32] | blake2b hashes concatenated together

### data response (`msg_type=1`)

Respond with a list of results for data lookups by hash.

field       | type          | desc
------------|---------------|--------------------------------------------------------------
msg\_len    | varint        | number of bytes in the message after this field
msg\_type   | varint        | 1
req\_id     | u8[4]         | id this is in response to
circuit\_id | u8[4]         | circuit for an established path or `[0,0,0,0]` for no circuit
data\_len   | varint        | length of data field
data        | u8[data\_len] | response payload

Receive (`data_len`,`data`) pairs until `data_len` is 0.

### request by hash (`msg_type=2`)

Request data for a set of hashes.

field       | type                | desc
------------|---------------------|--------------------------------------------------------------
msg\_len    | varint              | number of bytes in this message after this field
msg\_type   | varint              | 2
req\_id     | u8[4]               | unique id of this request (random)
circuit\_id | u8[4]               | circuit for an established path or `[0,0,0,0]` for no circuit
ttl         | varint              | number of hops remaining
hash\_count | varint              | number of hashes to request
hashes      | u8[32\*hash\_count] | blake2b hashes concatenated together

Results are provided by a data response (`msg_type=1`).

### cancel request (`msg_type=3`)

Stop receiving results for a request.

Some requests stay open and wait for data to arrive. You can close these long-running subscriptions
using a cancel request.

field       | type   | desc
------------|--------|--------------------------------------------------------------
msg\_len    | varint | number of bytes in this message after this field
msg\_type   | varint | 3
req\_id     | u8[4]  | stop receiving results for this id
circuit\_id | u8[4]  | circuit for an established path or `[0,0,0,0]` for no circuit

If a peer is forwarding results for this request, the message should be passed
upstream accordingly.

### public protocol request (`msg_type=4`)

Embed a cleartext sub-protocol request on top of cable or delegate encryption to the sub-protocol.
Additional fields or alternatively-structured data may be appended after `protocol_type` so long as
the length of the message in `msg_len` is accurate.

field          | type   | desc
---------------|--------|---------------------------------------------------------------
msg\_len       | varint | number of bytes in this message after this field
msg\_type      | varint | 4
req\_id        | u8[4]  | unique id of this request (random)
circuit\_id    | u8[4]  | circuit for an established path or `[0,0,0,0]` for no circuit
protocol\_type | varint | unique numeric code for sub-protocol

### public protocol response (`msg_type=5`)

Embed a cleartext sub-protocol response on top of cable or delegate encryption to the sub-protocol.
Some protocol responses may want to use the built-in hash requests and data responses but others may
need to embed additional or differently-structured data which this message type supports.

field          | type   | desc
---------------|--------|--------------------------------------------------------------
msg\_len       | varint | number of bytes in this message after this field
msg\_type      | varint | 5
req\_id        | u8[4]  | id this is in response to
circuit\_id    | u8[4]  | circuit for an established path or `[0,0,0,0]` for no circuit
protocol\_type | varint | unique numeric code for sub-protocol

### shared secret request (`msg_type=6`)

Establish a shared secret with an existing circuit.

field       | type   | desc
------------|--------|---------------------------------------------------------------
msg\_len    | varint | number of bytes in this message after this field
msg\_type   | varint | 6
req\_id     | u8[4]  | unique id of this request (random)
circuit\_id | u8[4]  | circuit for an established path or `[0,0,0,0]` for no circuit

###


