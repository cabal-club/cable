# Cable Handshake

Version: 1.0-draft4

Published: October 2023

Author: Kira Oakley

## Table of Contents
* [1. Introduction](#1-introduction)
  + [1.1 Terminology](#11-terminology)
  + [1.2 Document versioning](#12-document-versioning)
  + [1.3 Protocol overview](#13-protocol-overview)
* [2. Key concepts](#2-key-concepts)
  + [2.1 Initiator and responder](#21-initiator-and-responder)
  + [2.3 Noise](#23-noise)
    - [2.3.1 Protocol name](#231-protocol-name)
    - [2.3.2 General operation](#232-general-operation)
  + [2.4 Cabal key](#24-cabal-key)
* [3. Version Exchange](#3-version-exchange)
  + [3.1 Protocol versioning scheme](#31-protocol-versioning-scheme)
  + [3.2 Protocol Version Message exchange](#32-protocol-version-message-exchange)
* [4. Noise Handshake](#4-noise-handshake)
* [5. Post-Handshake Operation](#5-post-handshake-operation)
  + [5.1 Pseudocode functions](#51-pseudocode-functions)
  + [5.2 Fragmentation](#52-fragmentation)
  + [5.3 Message encoding](#53-message-encoding)
  + [5.4 Message decoding](#54-message-decoding)
* [6. Security considerations](#6-security-considerations)
  + [6.1 Out-of-scope attacks](#61-out-of-scope-attacks)
  + [6.2 In-scope attacks](#62-in-scope-attacks)
    - [6.2.1 Susceptible](#621-susceptible)
    - [6.2.2 Protected](#622-protected)
* [7. References](#7-references)
  + [7.1 Normative References](#71-normative-references)
  + [7.2 Informative References](#72-informative-references)

## 1. Introduction
Cable is a peer-to-peer protocol, communicated between a pair of hosts over an
arbitrary full-duplex binary transport. It allows compatible peers to exchange
data such as chat messages over an encrypted channel.

Cable consists of two protocols:

1. The Cable Handshake (explained in this document)
2. The Cable Wire Protocol [[4]](#ref4)

The Cable Handshake protocol MUST be executed first. Upon successful completion
of the handshake, the pair of hosts may then exchange messages using the Cable
Wire Protocol.

The purpose of the Cable Handshake is three-fold:

1. To ensure both hosts' implementation versions are compatible.
2. For each host to prove to the other that they know the secret *cabal key*.
2. To establish an encrypted channel between the two hosts, allowing Cable Wire
   Protocol messages to flow.

### 1.1 Terminology
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [BCP 14][BCP 14], [RFC 2119][RFC 2119], [RFC
8174][RFC 8174] when, and only when, they appear in all capitals, as shown
here.[[1](#ref1)]

### 1.2 Document versioning
This document is versioned together with the Cable Wire Protocol specification.
A valid implementation of Cable MUST follow both documents at the same version.

For example, if version 1.0 of the Cable Handshake is implemented, version 1.0
of the Cable Wire Protocol must also be implemented in order to form a valid
Cable implementation.

### 1.3 Protocol overview
A Cable Handshake is comprised of 3 phases, which, in a successful handshake,
are progressed through in sequence:

1. Version Exchange
2. Noise Handshake
3. Post-Handshake Operation

At a high level, the (1) Version Exchange exists to ensure protocol version
compatibility between both hosts. For hosts that are compatible, the (2) Noise
handshake then allows the exchange of ephemeral public keys, used with
Diffie-Hellman to produce a shared secret key for the session. That secret key,
in the (3) Post-Handshake Operation phase, is used to encrypt/decrypt and
authenticate all further outgoing and incoming Cable Wire Protocol messages.

A successful Cable Handshake will resemble the following exchange of
*handshake messages*:

```
INITIATOR                                 RESPONDER         STEP
========================================================================
  [MAJOR].[MINOR] version ---------------------->           (1)
  
  <---------------------- [MAJOR].[MINOR] version           (2)

  Noise ephemeral key ------------------------->            (3)

  <------------ Noise ephemeral key + static key            (4)

  Noise static key ---------------------------->            (5)

  <----- Bidirectional encrypted messages ----->            (6)
```
*Figure 1.0      Handshake message exchange overview*

Steps 1 and 2 are part of the Version Exchange phase of the handshake, and
steps 3-5 belong to the Noise Handshake phase. Upon completion of steps
1-5, the protocol enters into Post-Handshake Operation phase in step 6.

## 2. Key concepts

### 2.1 Initiator and responder
In a Cable Handshake, one host takes on the role of *initiator* and the other
takes on the role of *responder*. This is a requirement of the *Noise Protocol
Framework*, which is discussed further in the next subsection.

Since different transports have different properties, a single rule cannot be
provided as to which host takes on which role. However, for transport protocols
where there are well-defined *client* and *server* roles, such as TCP/IP,
implementations SHOULD regard the client as the initiator and the server as the
responder.

The role chosen only decides the order of messages sent during the handshake
process and does not affect how the Cable Wire Protocol operates.

### 2.3 Noise
The Noise Protocol Framework is a framework for building cryptographically
secure protocols. It is not a protocol in itself, but rather a framework for
producing protocols specific to an application's cryptographic needs.

Cable uses Revision 34 (2018-07-11) of Noise. It can be found mirrored in this
repository and is referenced in this document as the "Noise specification"
[[2](#ref2)]. It is mirrored here because its current URL is not locked to a
specific version, and may change: https://noiseprotocol.org/noise.html

This document uses terminology defined in the Noise specification. Names and
specification sections will be *italicized*, and references to functions or
other pseudocode defined in the Noise specification will appear in `code
blocks`.

#### 2.3.1 Protocol name
A specific Noise protocol is defined by a unique string -- the *protocol name*
-- that describes i) the messages exchanged by each host, ii) whether
pre-shared keys are used (and when they are incorporated), and iii) the suite
of cryptographic primitives to be used.

The *protocol name* used by Cable is `Noise_XXpsk0_25519_ChaChaPoly_BLAKE2b`.
This can be broken down into the following components:

- `XX` is a Noise *handshake pattern* indicating that each host will be securely transmitting a static public key to the other party.
- `psk0` implies that a pre-shared 32-byte key must be known by both hosts
  wishing to communicate, mixed into the cryptographic state as the first step.
- `25519` implies the use of Curve25519 key pairs and X25519 for
  Diffie-Hellman.
- `ChaChaPoly1305` is the combination of the ChaCha20 streaming cipher and the
  Poly1305 authenticator.
- `BLAKE2b` is the hashing algorithm to be used.

Specifics of these cryptographic dependencies can be found in the Noise
specification, including input parameters and references to relevant RFCs,
under *12. DH functions, cipher functions, and hash functions*.

#### 2.3.2 General operation
The Noise specification explains the general operation of a Noise protocol in
*5. Processing rules*. This subsection briefly summarizes the process.

Noise provides relatively high-level functions like `WriteMessage()`, which,
during the Noise Handshake phase of this protocol will produce the proper
outgoing binary payloads which can be understood by the Noise implementation on
the other host.

Similarly, `ReadMessage()` handles parsing and incorporating handshake data
from the other host. The only responsibility of a Cable Handshake
implementation is to "ferry" these Noise-produced messages between itself and
the host with whom it is handshaking: what `WriteMessage()` produces is to be
sent over the network transport to the other host, and what comes in over the
network transport is fed into `ReadMessage()`.

This repeats until the handshake is complete, after which two `CipherState`
objects are returned -- one for encryption and one for decryption -- which
marks the end of the Noise Handshake, and the beginning of Post-Handshake
Operation, where these keys can then be used for encrypting and decrypting
Cable Wire Protocol messages to and from the other host.

The section *5. Processing rules* of the Noise specification goes into more
detail, and may be worth reading through for implementors.

### 2.4 Cabal key
A collection of peers sharing subsets of a single dataset is called a *cabal*.
In other messaging software, this might be called a "server" or "instance". A
cabal contains a collection of channels, of which users can be members, and
chat messages may be posted.

Each cabal is identified by a 32-byte key, which is ideally generated using the
best means available for random number generation on the host system. This is
called the *cabal key*, and functions as the pre-shared key (`psk0` in the
aforementioned Noise handshake pattern).

Since the cabal key is mixed into Noise handshake state, every handshake with
another host under this protocol only permits access to a specific cabal. Only
a pair of hosts using the same pre-shared key in Noise will be able to
successfully complete a Noise handshake.

The cabal key effectively acts as a "secret passphrase": only hosts who know
the key can successfully handshake and then exchange Cable Wire Protocol
messages. If either party doesn't know the key, Noise will indicate handshake
failure. This key is only ever used locally and is never sent over the network
transport. Members of a cabal can share the cabal key over various out-of-band
means (e.g. other chat programs, written on paper, etc.)

The pre-shared key MUST be mixed into the handshake state as per the rules in
*9. Pre-shared symmetric keys* of the Noise specification.

## 3. Version Exchange
### 3.1 Protocol versioning scheme
Each revision of the Cable specification has a major and minor version
associated with it. For example, `1.3`. An implementation of the Cable
specification shares the same version name as the specification version it
implements. So, an implementation of version `3.5` of the Cable specification
would consider its version to also be `3.5`.

A version is comprised of two components:

1. Major Version
2. Minor Version

A new *major version* of the specification indicates a breaking change. When
two implementations have differing major versions, they would be unable to
properly communicate with each other, or it is unsafe for them to. This could
be for various reasons, such as a change in the ciphersuite used for
encryption, a change in how messages are structured, or a new security
constraint that has been added.

A new *minor version* of the specification indicates a non-breaking change. Two
implementations with differing minor versions will still be able to communicate
properly with each other, although there may be features available to an
implementation with a newer version that the older may not be able to take
advantage of. Examples of changes necessitating a new minor version include new
Cabal Wire Protocol message or post types that older versions can safely
ignore, or changes to "SHOULD" or "MAY" directives in the protocol that do not
break interoperability.

A specific Cable version can be expressed by writing "`[MAJOR].[MINOR]`". For
example, `4.7` or `9.1`. Each version component is limited to the range `0 -
255`.

The minor version is reset to zero upon the release of a new major version.

### 3.2 Protocol Version Message exchange
The first phase of the handshake is an exchange of messages containing each
host's implementation version. This is done to ensure compatibility between the
hosts' implementations, and constitutes the first two steps from Figure 1.0
above:

```
INITIATOR                           RESPONDER       STEP
=========================================================
  [MAJOR].[MINOR] version ---------------->          (1)
  
  <---------------- [MAJOR].[MINOR] version          (2)
```
Figure 2.0  The first two messages sent in a Cable Handshake

Each of these messages is a *Protocol Version Message*. This is a 2-byte binary
message with the following structure, using the style described in [[3](#ref3)]:

```
     0                   1             
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Major Version |  Minor Version  |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

where:

**Major Version: 1 unsigned byte**. This is an implementation's major version.

**Minor Version: 1 unsigned byte**. This is an implementation's minor version.

The handshake protocol begins with Step 1: the initiator MUST send a Protocol
Version Message to the responder after a connection is established.

Step 2: When this message is received by the responder, the responder compares its
major version to the major version received.

If the major version of its implementation differs from the one received, the
responder SHOULD respond with its own Protocol Version Message, and then MUST
terminate the connection to the other host. In the case of a major version
mismatch, it is valid for the responder to not send a Protocol Version Message
reply, but discouraged, since the initiator will not be able to distinguish
major version incompatibility from e.g. a connection being accidentally lost.

If the major versions are the same, the responder MUST send a Protocol Version
Message to the initiator. The responder now enters the Noise Handshake phase.

Upon receiving the responder's Protocol Version Message, the initiator compares
its major version to the one received. If they do not match, the initiator MUST
terminate the connection: the implementations are not compatible. If the major
versions match, the initiator also moves to the Noise Handshake phase.

NOTE: A nice quality of this approach is that if we ever wanted to add more
features to the handshake, like the exchange of feature flags (e.g.
compression), this could be added before or after the Noise Handshake in a
breaking major version change. This means we only need to commit upfront to
this maximally simple version exchange mechanism now, and the overall mechanism
could change greatly in the future to allow us to respond to unanticipated
needs.

## 4. Noise Handshake

### 4.1 Static Keypair
Each user in a cabal is identified by a static public/private Curve25519 key
pair.

This keypair SHOULD be generated when a user first joins or creates a cabal,
and SHOULD be persisted in some manner, so that it can be re-used for the
handshake of every peer connection made. For security reasons, the keypair MUST
be unique to that cabal, and MUST NOT shared across other cabals. (See the Wire
Protocol's Security Considerations section for a more detailed explanation.)

The keypair is used to both authenticate connections and to sign posts in the
Cable Wire Protocol. The same keypair SHOULD be used for both.

### 4.2 Process
The Noise Handshake phase is performed by following the listed steps in the
Noise specification, under *6. Processing rules*, which MUST be executed:

> To execute a Noise protocol you `Initialize()` a `HandshakeState`. During
> initialization you specify the handshake pattern, any local key pairs, and
> any public keys for the remote host you have knowledge of. After
> `Initialize()` you call `WriteMessage()` and `ReadMessage()` on the
> `HandshakeState` to process each handshake message. If any error is signaled
> by the `DECRYPT()` or `DH()` functions then the handshake has failed and
> the `HandshakeState` is deleted.
> 
> Processing the final handshake message returns two `CipherState` objects, the
> first for encrypting transport messages from initiator to responder, and the
> second for messages in the other direction.

Different implementations of Noise may choose different names for functions,
structures, and parameters, so this document attempts to keep to the functions,
structures, and parameters that the Noise specification explicitly defines.

The following constraints also apply:
- The string `"CABLE"` MUST be used as the `prologue` in `Initialize()`.
- The string `"XXpsk0"` MUST be used as the `handshake_pattern` in
  `Initialize()`.
- The initiator MUST set `initiator` to `true` in `Initialize()`. Otherwise, it
  MUST be set to `false`.
- The cabal key MUST be mixed into the `SymmetricState` as described in *9.
  Pre-shared symmetric keys* of the Noise specification.
- If an error is signaled by the `DECRYPT()` or `DH()` functions, the
  connection MUST also be terminated.

At the end of a successful Noise Handshake, both hosts will have a pair of
`CipherState` objects, to be used in the final phase, Post-Handshake Operation.

## 5. Post-Handshake Operation
Once the Version Exchange and Noise Handshake phases are complete, the Cable
Handshake is in the Post-Handshake Operation phase, where Cable Wire Protocol
messages MAY be transmitted and received. There are a final set of rules,
described here, for how incoming and outgoing data speaking the Cable Wire
Protocol must be encoded and decoded.

At a high level, all Cable Wire Protocol messages need to be passed through
Noise for encryption, and then prefixed with an encrypted length indicator.
Incoming Cable Wire Protocol messages will also be length-prefixed, and the
message bodies will be encrypted as *ciphertext*s, and must be run through
Noise for decryption. There are additional steps to handle message
fragmentation, described in the next subsection.

The Noise function `Split()`, run at the end of the Noise Handshake, returns a
pair of `CipherState` objects `(c1, c2)` to be used as follows:
- For the initiator, `c1` MUST be used for encryption, and `c2` for decryption.
- For the responder, `c2` MUST be used for encryption, and `c1` for decryption.

Specifically, to exchange messages during Post-Handshake Operation, the listed
steps in the Noise specification, under *6. Processing rules*, MUST be
followed:

> Transport messages are then encrypted and decrypted by calling
> `EncryptWithAd()` and `DecryptWithAd()` on the relevant `CipherState` with
> zero-length associated data. If `DecryptWithAd()` signals an error due to
> `DECRYPT()` failure, then the input message is discarded. The application may
> choose to delete the `CipherState` and terminate the session on such an
> error, or may continue to attempt communications. If `EncryptWithAd()` or
> `DecryptWithAd()` signal an error due to nonce exhaustion, then the
> application must delete the `CipherState` and terminate the session.

In this context, "transport messages" are Cable Wire Protocol messages. If
`DecryptWithAd()` signals an error due to `DECRYPT()` failure, the client
MUST terminate the connection.

### 5.1 Pseudocode functions
For the remainder of this section, define the following pseudocode elements:

- Let `ZERO` be an empty sequence of bytes.
- Let `EncryptWithAd()` and `DecryptWithAd()` be the Noise functions of the
  same names.
- Let `WriteBytes(bytes)` be a hypothetical function that writes `bytes` bytes
  over the network to the other host.
- Let `bytes = ReadBytes(len)` be a hypothetical function that reads `len`
  bytes over the network from the other host, and returns those bytes as
  `bytes`.
 - Let `bytes.slice(start, length)` be a hypothetical function on a byte
   sequence that returns a slice of a sequence of bytes, starting at position
   `start`, and including the next `length` bytes.
 - Let `result = bytes.concat(bytes2)` be a hypothetical function on a byte
   sequence that concatenates the byte sequence `bytes2` onto the existing byte
   sequence `bytes`, producing the new byte sequence `result`.
 - Let `bytes.length` be a hypothetical property of a byte sequence that
   returns the length of the byte sequence `bytes`, in bytes.
- Let `min(a, b)` be a hypothetical function that returns the smaller number of
  `a` and `b`.

### 5.2 Fragmentation
The maximum Noise message length is 65535 bytes, so any input `plaintext`
exceeding this length MUST be fragmented as specified below in order to
facilitate encrypted transport.

At a high level, this is done by the following steps:

1. If the remaining unsent bytes are less than 65536 bytes in length, encrypt
   and send it over the network; done.
2. Otherwise, encrypt and send the first 65535 bytes; return to step 1.

See the subsequent subsections for concrete details.

### 5.3 Message encoding
This subsection defines pseudocode function `WriteMsg(plaintext)` that takes
the full Cable Wire Protocol message payload, `plaintext`, as bytes, and writes
them to the network, performing encryption and fragmentation:

```js
function WriteMsg (plaintext) {
  let written = 0
  while (written < plaintext.length) {
    let segmentLen = min(65535, plaintext.length - written)
    let bytes = plaintext.slice(written, segmentLen)
    let ciphertext = EncryptWithAd(ZERO, bytes)
    WriteBytes(ciphertext)
    written += bytes.length
  }
}
```

If, for example, a `plaintext` of length 90200 were to be encoded & written,
the first 65535 bytes would first be encrypted and written, followed by the
remaining 24665 bytes being encrypted and written.

When a Cable Wire Protocol message, `plaintext` is to be sent, it MUST follow
these steps:

1. Compute the length of the payload, `plaintext`, in bytes, `len`.

2. Encrypt `len` as a two-byte unsigned little endian integer: `cipherlen = EncryptWithAd(ZERO, len)`.

3. `WriteBytes(cipherlen)`

4. `WriteMsg(plaintext)`

### 5.4 Message decoding
This subsection defines pseudocode function `plaintext = ReadMsg(len)` that
reads a ciphertext message of length `len`, performing de-fragmentation and
decryption:

```js
function ReadMsg (len) {
  let plaintext = ZERO
  let bytesRemaining = len
  while (bytesRemaining > 0) {
    let segmentLen = min(65535, bytesRemaining)
    let ciphertext = ReadBytes(segmentLen)
    let segment = DecryptWithAd(ZERO, ciphertext)
    plaintext = plaintext.concat(segment)
    bytesRemaining -= segmentLen
  }
}
```

Reading a Cable Wire Protocol message MUST follow these steps:

1. `cipherlen = ReadBytes(2)`, interpreting these 2 bytes as a little-endian
   unsigned integer.

2. `len = DecryptWithAd(ZERO, cipherlen)`

4. `plaintext = ReadMsg(len)`

5. The resulting bytes `plaintext` may then be parsed as a Cable Wire Protocol
   message.

## 6. Security considerations
### 6.1 Out-of-scope attacks
Attacks on the inner Cable Wire Protocol are not considered here. See the
Security Considerations section on its specification [[4](#ref4)].

### 6.2 In-scope attacks
#### 6.2.1 Susceptible
The most significant security concern is regarding the secrecy of the cabal
key. The privacy of a cabal hinges entirely on the secrecy of the cabal key
associated with it. If a cabal's key is leaked to unintended recipients or made
public, the members of that cabal will have no means of distinguishing intended
peer from unintended peer, and may leak data that was not intended to be
shared, such as historic and new chat messages, channel names, and published
user data. Further, such an intrusion may not be detectable, as an intruder
could listen in on the cabal without authoring any posts of their own
indefinitely. Even if the intrusion is known, there is currently no way to
"re-key" a cabal, other than starting a new cabal, so there is no means of
expelling unwanted parties from a cabal. There are plans for a moderation
system that operates within cabals, which could allow unauthorized members with
the cabal key to be blocked from interacting with its members.

Denial of Service attacks are possible. A large amount of incomplete handshake
attempts could cause a large number of Noise-related data structures to be
allocated and held in memory, allowing for potential memory or port exhaustion,
not unlike a TCP SYN flood attack. Implementations using transports like TCP/IP
may be able to mitigate by refusing connections from IPs that have been opening
an unreasonable number of connections.

A passive attacker could sniff packets to determine what version of Cabal
various hosts are running, since the version exchange happens in the clear. For
hosts running older versions with known security vulnerabilities, this
information could be used to then explicitly target that host. It's worth
mentioning that, at time of writing, no such vulnerabilities are known to
exist.

An active attacker could manipulate the Protocol Version messages such that two
hosts are handshake due to maliciously changed version number, making each host
appear to have sent an incompatible major version number.

This document does not provide any mandates on how the cabal key is stored. If
stored on disk in plaintext, it would be vulnerable to any unauthorized access
to the device storing it.

Implementations that fail to generate a sufficiently random cabal key are more
susceptible to it being guessed. ChaCha20 also has similar risks around
generating its nonce, and ensuring not to use the same nonce for multiple
sessions.

#### 6.2.2 Protected
The `XX` Noise handshake pattern always uses fresh ephemeral keypairs to
initialize the handshake, so every Cable session will have a unique secret
shared key for that session. If that secret key is discovered after a session
ends, it cannot be used to break future communications.

The ChaCha20-Poly1305 streaming cipher provides confidentiality via the
ChaCha20 cipher, and data integrity via the Poly1305 authenticator. The use of
a counter in the streaming cipher allows dropped, inserted, or replayed
messages to be detected.

If there were no cabal key, communicating hosts would be vulnerable to a
man-in-the-middle attack. However, a man-in-the-middle without knowledge of the
cabal key would be unable to successfully handshake with either host.

The cabal key is never transmitted over the network. It could only be leaked
over an out-of-band channel, e.g. someone posting it onto a website.

Diffie-Hellman secret key derivation performed by Noise prevents a passive
attacker from learning the shared secret, despite public keys being exchanged
in the clear.

## 7. References
### 7.1 Normative References
- <a id="ref1"></a>**[1]** [Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", RFC 8174, May 2017](https://www.rfc-editor.org/rfc/rfc8174)
- <a id="ref2"></a>**[2]** [Perrin, T., "The Noise Protocol Framework", 11 July 2018](vendor/noise_34.pdf)
- <a id="ref3"></a>**[3]** [McQuistin, S., Band, V., Jacob, D., and C. Perkins, "Describing Protocol Data Units with Augmented Packet Header Diagrams", Work in Progress, Internet-Draft, draft-mcquistin-augmented-ascii-diagrams-10, 7 March 2022][Header]

### 7.2 Informative References
1. <a id="ref4"></a>**[4]** [Cable Wire Protocol](readme.md)

[noise-spec]: vendor/noise_34.pdf
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174
[BCP 14]: https://www.rfc-editor.org/bcp/bcp14
[Header]: https://datatracker.ietf.org/doc/html/draft-mcquistin-augmented-ascii-diagrams-10

