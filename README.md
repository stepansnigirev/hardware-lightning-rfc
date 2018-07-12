# Lightning node to Hardware wallets communication protocol

This document attempts to describe a data flow between Lightning node and HSM device. Lightning network is coming to town and number of nodes in the network rapidly increases. One important issue that face people trying to run a mainnet lightning node is security of their private keys.

Currently none of hardware wallets support Lightning Network and none of Lightning implementation fully support HSMs. This gives us a chance to develop a common standard for communication between the lightning software and hardware device in a way that minimizes security risks for the user and provides all necessary information for user confirmation.

## Rationale

All the secrets controlling the funds of the lightning node should be stored on the secure device. Lightning node setted up to work with HSM should not be able to do anything with the funds without talking to HSM. It includes both on-chain and off-chain transactions. Secrets used for communication and channel announcements may be stored on the node itself to provide watch-only functionality.

Any HSM request for on-chain or off-chain transaction should provide all necessary information to the HSM module such that hardware device can verify and display information to the user.

## On-chain transaction

### Funding transaction

When openning the channel we need to make sure that:

- one of the two public keys of the multisig transaction is our own
- the other public key corresponds to the node we are opening a channel with

On-chain public key of the other node is different from the node public key, so we need to provide a proof that one of public keys is actually a public key of the second node.

For unsigned transactions format we can use [PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) that supports additional fields we can utilize for lightning-specific purposes. We can include a proof of the other node's public key in additional fields of the PSBT.

Data sent to the HSM in PSBT form:

- raw unsigned transaction
- key derivation path for signing
- key derivation path for change address
- script for multisig address we are funding
- proof that the second pubkey is from node we are opening channel to

## Channel updates and routing

### Direct payments

Storing all channel secrets on the HSM device doesn't make sense as hardware wallets normally have very limited resources and memory. Instead we can encrypt secrets on the HSM and pass encrypted data to the lightning node to store them in the database.

**TODO: careful!** we need to keep track of the channel state updates somehow to prevent compromized node to broadcast previous state of the channel. Then we lose all the money. Unilateral close tx MUST stay on the hardware device.

### Routing

**TODO: careful!** If HSM is not aware of the channel being closed unilaterely by the other party it can be fooled and lose money providing a route between two nodes.
**TODO:** describe attack in mode details.
