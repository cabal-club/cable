# Cable Protocol

Cable is a peer-to-peer protocol for private group chats, called *cabals*.

Cable operates differently from the traditional server-client model, where the
server is a centralized authority. Instead, in Cable, every node in the network
is equal to each other. Nodes of a cabal share data with each other in order to
build a view of the state of that cabal.

The purpose of the Cable Protocol is to facilitate the creation and sync of
cabals, by allowing peers to exchange cryptographically signed documents with
each other, such as chat messages, spread across various user-defined channels.
All of this happens over an encrypted channel between each peer.

Cable consists of two protocols:

1. The [Cable Handshake](./handshake.md), which determines if two peers are compatible and establishes a secure channel between them.
2. The [Cable Wire Protocol](./wire.md), which allows two peers to request and exchange signed data (*posts*) from each other by their content hashes.

Cable is designed to be:

1. fairly simple to implement in any language, with minimal dependencies
2. general enough to be used across different network transports
3. useful, even if written as a partial implementation
4. efficient in its use of network resources, by
    1. syncing only the relevant subsets of the full dataset, and
    2. being compact over the wire
5. not specific to any particular kind of database backend

While the overall structure of these protocols is unlikely to change, these
documents are still under active development, and should not be considered
stable.

