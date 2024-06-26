# Cable Handshake

Version: 1.0-draft8

Published: February 2024

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
* [3. Noise Handshake](#3-noise-handshake)
  + [3.1 Static Keypair](#31-static-keypair)
  + [3.2 Process](#32-process)
* [4. Post-Handshake Operation](#4-post-handshake-operation)
  + [4.1 Pseudocode functions](#41-pseudocode-functions)
  + [4.2 Message encoding & transmission](#42-message-encoding--transmission)
    - [4.2.1 Fragmentation](#421-fragmentation)
    - [4.2.2 Encryption and Authentication](#422-encryption-and-authentication)
    - [4.2.3 Message length](#423-message-length)
    - [4.2.4 Message transmission](#424-message-transmission)
  + [4.3 Message decoding](#43-message-decoding)
  + [4.4 End of stream](#55-end-of-stream)
* [5. Security considerations](#5-security-considerations)
  + [5.1 Out-of-scope attacks](#51-out-of-scope-attacks)
  + [5.2 In-scope attacks](#52-in-scope-attacks)
    - [5.2.1 Susceptible](#521-susceptible)
    - [5.2.2 Protected](#522-protected)
* [6. References](#6-references)
  + [6.1 Normative References](#61-normative-references)
  + [6.2 Informative References](#62-informative-references)

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
A Cable Handshake is comprised of 2 phases, which, in a successful handshake,
are progressed through in sequence:

1. Noise Handshake
2. Post-Handshake Operation

At a high level, the (1) Noise Handshake establishes a secure channel between
hosts, which the (2) Post-Handshake Operation phase uses to send encrypted and
authenticated further outgoing and incoming Cable Wire Protocol messages.

A successful Cable Handshake will resemble the following exchange of
*handshake messages*:

```
INITIATOR                                 RESPONDER         STEP
========================================================================
  Noise ephemeral key ------------------------->            (1)

  <------------ Noise ephemeral key + static key            (2)

  Noise static key ---------------------------->            (3)

  <----- Bidirectional encrypted messages ----->            (4)
```
*Figure 1.0      Handshake message exchange overview*

Steps 1-3 are part of the Noise Handshake phase. Upon completion of steps 1-3,
the protocol enters into the final phase, Post-Handshake Operation, in step 4.

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

## 3. Noise Handshake

### 3.1 Static Keypair
Each user in a cabal is identified by a static public/private Ed25519 key pair.

This keypair SHOULD be generated when a user first joins or creates a cabal,
and SHOULD be persisted in some manner, so that it can be re-used for the
handshake of every peer connection made. For security reasons, the keypair MUST
be unique to that cabal, and MUST NOT be shared across other cabals. (See the Wire
Protocol's Security Considerations section for a more detailed explanation.)

The keypair is used to both authenticate connections and to sign posts in the
Cable Wire Protocol. The same keypair SHOULD be used for both.

### 3.2 Process
The Noise Handshake phase is performed by following the listed steps in the
Noise specification, under *5. Processing rules*, which MUST be executed:

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
- The ASCII-encoded string `"CABLE/1.0"` MUST be used as the `prologue` in `Initialize()`. The number "1.0" in the prologue is so because this version of the protocol is 1.0. The definitive bytes of this, in hexadecimal, are `43 41 42 4c 45 2f 31 2e 30`.
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

## 4. Post-Handshake Operation
Once the Noise Handshake phase is complete, the Cable Handshake is in the
Post-Handshake Operation phase, where Cable Wire Protocol messages MAY be
transmitted and received. There is a final set of rules, described here, for
how incoming and outgoing data speaking the Cable Wire Protocol must be encoded
and decoded.

At a high level, all Cable Wire Protocol messages need to be passed through
Noise for encryption, and then prefixed with an encrypted length indicator.
Incoming Cable Wire Protocol messages will also be length-prefixed, and the
message bodies will be encrypted as *ciphertext*s, and must be run through
Noise for decryption. There are additional steps to handle message
fragmentation and the end of the stream, described in the next subsection.

The Noise function `Split()`, run at the end of the Noise Handshake, returns a
pair of `CipherState` objects `(c1, c2)` to be used as follows:
- For the initiator, `c1` MUST be used for encryption, and `c2` for decryption.
- For the responder, `c2` MUST be used for encryption, and `c1` for decryption.

Specifically, to exchange messages during Post-Handshake Operation, the listed
steps in the Noise specification, under *5. Processing rules*, MUST be
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

### 4.1 Pseudocode functions
For the remainder of this section, define the following pseudocode elements:

- Let `ZERO` be an empty sequence of bytes.
- Let `|` be byte-wise concatenation.
- Let `EncryptWithAd()` and `DecryptWithAd()` be the Noise functions of the
  same names.
- Let `WriteBytes(bytes)` be a function that writes `bytes` bytes over the
  network to the other host.
- Let `bytes = ReadBytes(len)` be a function that reads `len` bytes over the
  network from the other host, and returns those bytes as `bytes`.
- Let `bytes.slice(start, length)` be a function on a byte sequence that
  returns a slice of a sequence of bytes, starting at position `start`, and
  including the next `length` bytes.
- Let `bytes.length` be a property of a byte sequence that returns the length
  of the byte sequence `bytes`, in bytes.
- Let `min(a, b)` be a function that returns the smaller of two numbers, `a`
  and `b`.

### 4.2 Message encoding & transmission

#### 4.2.1 Fragmentation
The maximum length of a Noise payload is 65535 bytes. This does not include the
message authentication code for the encrypted data, which is 16 bytes, leaving
65519 bytes available per Noise payload for message data.

Messages to be sent with a length exceeding 65519 bytes MUST by divided into
`n > 1` segments such that

```
message = S₁ | ... | Sₙ
```

prior to transmission, such that the first `n - 1` segments are 65519 bytes in
length, and the final segment is of a length constituting the remaining bytes.
Messages with a length less than or equal to 65519 bytes MUST be sent without
any fragmentation.

For example, a message of length 155719 bytes would be fragmented into `n = 3`
segments, where `S₁.length = 65519` bytes, `S₂.length = 65519` bytes, and the
final segment `S₃.length = 155719 - 65519 * 2 = 24681` bytes.

#### 4.2.2 Encryption and Authentication
Each segment MUST be encrypted with a MAC using the Noise function `EncryptWithAd`.

In pseudocode, this would look like calling this function on each segment, Sₖ,
such that a ciphertext, Cₖ is produced:

```
Cₖ = EncryptWithAd(ZERO, Sₖ)
```

This results in an equal number of ciphertexts as there were segments, C₁...Cₙ.

#### 4.2.3 Message length
The total length of a sequence of message segments, S₁...Sₙ can be computed as

```
totalLen = (n - 1) * 65535 + (Sₙ.length + 16)
```

The number `totalLen` is then encoded as a 4-byte little endian integer, and
finally encrypted with a MAC:

```
lenEncrypted = EncryptWithAd(ZERO, len)
```

`lenEncrypted` MUST be computed first, followed by each ciphertext in sequence,
C₁ through Cₙ. The order of encryption is essential, since `EncryptWithAd` is a
stateful function.

#### 4.2.4 Message transmission
Using the values produced from the preceding subsections, the final message
MUST be transmitted in the following sequence:

1. Write the encrypted ciphertexts' length: `WriteBytes(lenEncrypted)`

2. Write all ciphertexts in order: `WriteBytes(C₁); ... WriteBytes(Cₙ)`

### 4.3 Message decoding
Message decoding reverses the preceding steps:

1. Read 20 bytes from the network (4 bytes of length data, plus 16 bytes for the MAC):

```
lenEncrypted = ReadBytes(20)
```

2. Decrypt the total ciphertexts' length:

```
totalLen = DecryptWithAd(ZERO, lenEncrypted)
```

3. Read the ciphertexts from the network, and concatenate their decrypted
   plaintexts' together to form the original Cable Wire Protocol message,
   `message`:

```
let message = ZERO
while (totalLen > 65535) {
  let ciphertext = ReadBytes(65535)
  let segment = DecryptWithAd(ZERO, ciphertext)
  message = message | segment
  totalLen -= 65535
}
let ciphertext = ReadBytes(totalLen)
let segment = DecryptWithAd(ZERO, ciphertext)
message = message | segment
```

### 4.4 End of stream
When a host has decided to terminate the exchange of messages, they MUST send
a message of length zero to indicate this intention, and MUST NOT send any
further messages. The zero-length message is known as an end-of-stream marker.

The host receiving an end-of-stream marker SHOULD respond with an
end-of-stream marker of its own to indicate it has also finished writing. An
implementation SHOULD have a time-out of some kind in case the other side does
not transmit an end-of-stream marker or the marker is truncated by an attacker.

When a host has both sent and received a zero-length message, it is then safe
for the underlying transport to execute its own disconnect logic, if any.

## 5. Security considerations
### 5.1 Out-of-scope attacks
Attacks on the inner Cable Wire Protocol are not considered here. See the
Security Considerations section on its specification [[4](#ref4)].

### 5.2 In-scope attacks
#### 5.2.1 Susceptible
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

This document does not provide any mandates on how the cabal key is stored. If
stored on disk in plaintext, it would be vulnerable to any unauthorized access
to the device storing it.

Implementations that fail to generate a sufficiently random cabal key are more
susceptible to it being guessed. ChaCha20 also has similar risks around
generating its nonce, and ensuring not to use the same nonce for multiple
sessions.

#### 5.2.2 Protected
The `XX` Noise handshake pattern always uses fresh ephemeral keypairs to
initialize the handshake, so every Cable session will have a unique secret
shared key for that session. If that secret key is discovered after a session
ends, it cannot be used to break future communications.

The ChaCha20-Poly1305 streaming cipher provides confidentiality via the
ChaCha20 cipher, and data integrity via the Poly1305 authenticator. The use of
a counter in the streaming cipher allows dropped, inserted, or replayed
messages to be detected.

Message lengths are encrypted and authenticated, hiding message boundaries and
hindering fingerprinting efforts compared to plaintext message length prefixes.

If there were no cabal key, communicating hosts would be vulnerable to a
man-in-the-middle attack. However, a man-in-the-middle without knowledge of the
cabal key would be unable to successfully handshake with either host.

The cabal key is never transmitted over the network. It could only be leaked
over an out-of-band channel, e.g. someone posting it onto a website.

Diffie-Hellman secret key derivation performed by Noise prevents a passive
attacker from learning the shared secret, despite public keys being exchanged
in the clear.

## 6. References
### 6.1 Normative References
- <a id="ref1"></a>**[1]** [Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words", RFC 8174, May 2017](https://www.rfc-editor.org/rfc/rfc8174)
- <a id="ref2"></a>**[2]** [Perrin, T., "The Noise Protocol Framework", 11 July 2018](vendor/noise_34.pdf)
- <a id="ref3"></a>**[3]** [McQuistin, S., Band, V., Jacob, D., and C. Perkins, "Describing Protocol Data Units with Augmented Packet Header Diagrams", Work in Progress, Internet-Draft, draft-mcquistin-augmented-ascii-diagrams-10, 7 March 2022][Header]

### 6.2 Informative References
1. <a id="ref4"></a>**[4]** [Cable Wire Protocol](readme.md)

[noise-spec]: vendor/noise_34.pdf
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174
[BCP 14]: https://www.rfc-editor.org/bcp/bcp14
[Header]: https://datatracker.ietf.org/doc/html/draft-mcquistin-augmented-ascii-diagrams-10

