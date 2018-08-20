# Hardware wallets for Lightning Network

This document attempts to describe potential integration of HSM devices with Lightning nodes.

## Motivation

Lightning network is going mainstream and number of nodes in the network rapidly increases. One important issue that people face trying to run a mainnet lightning node is security of their private keys.

Currently none of hardware wallets support Lightning Network and none of Lightning implementation fully support HSMs. This gives us a chance to develop a common standard for communication between the lightning software and hardware device in a way that minimizes security risks for the user and provides all necessary information for user confirmation.

## Architecture

All the secrets controlling the funds of the lightning node should be stored on the secure device. Lightning node setted up to work with HSM should not be able to do anything with the funds without talking to HSM. It includes funding, closing and commitment transactions. Secrets used for communication and channel announcements may be stored on the node itself to provide watch-only functionality. Storing `node_key` on the host has certain side-effects that we address in the [Appendix A](#appendix-a-sideffects-from-keeping-the-node_key-on-the-host), but still can be used in some cases.

Any request to the HSM should provide all necessary information such that it can decide whether to ask for user confirmation or not. If user interaction is required HSM should be able to display everything to convince the user that the action he requested is what will actually happen.

In the description below we consider that our host (lightning node) is completely compromized. In the next sections we describe general overview of the data requirements for different types of requests.

In most cases we just need to sign a transactions. [PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) might be a good choice. It supports arbitrary fields that we can use for lightning-specific purposes.

## Opening a channel

When we are opening a channel we need to check three things:

- channel amount is correct
- channel output is a 2-of-2 multisignature address with two keys
  - one of these keys is controlled by us
  - another one corresponds to the remote node with certain `node_id`
- we have the first commitment transaction signed by remote node

Host interaction with the HSM then splits into steps:

- Host receives `funding_pubkey` from remote node signed with the corresponding `node_id`
- Host constructs the funding transaction and sends it to the HSM together with signed remote `funding_pubkey`. Host should also provide all derivation pathes for the keys.
- HSM asks user for confirmation to open a channel with `node_id` and `amount`
- User confirms, HSM signs the transaction but doesn't send it to the host yet
- HSM sends txid to the host and waits for the signed commitment transaction
- Host gets the signed commitment transaction from the remote node and sends it to HSM
- In return HSM sends to the host a signed funding transaction to broadcast
- When funding transaction gets into blockchain Host provides a proof of inclusion in the block
- HSM starts accepting channel updates only after it gets this proof.

## Channel updates and routing

Storing all channel secrets and commitment transactions on the HSM device doesn't make sense as hardware wallets normally have very limited resources and memory. Instead we can encrypt secrets and transactions on the HSM and pass encrypted data back to the host to store them in the database.

## Routing

If HSM is not aware of the channel being closed unilaterely by the other party it can be fooled and lose money providing a route between two nodes with one of the channels closed.

To prevent this HSM should check this either with another remote watchtower or by checking all blocks. We can require from the host to provide block headers with txids and run it through the filter to determine whether this block contains unilateral closing transactions.

## Blockchain data

Many lightning transactions (HTLC, commitment) have timelocks. This means that HSM should track the blockchain height and time somehow. The easiest way is to include real-time-clock to the hardware design and require the host to send all the block headers to the HSM. If the host delays the blocks it can be determined by comparing the timestamps in the blocks with the value of RTC on the HSM. It should not differ from current time by more than 2 hours.

HSM can check the blocks for old states, funding and unilateral closing transactions to make sure that HSM is aware of current channel states. It doesn't need a full block for that - index of txids included in the block would be enough.

## Appendix A. Sideffects from keeping the `node_key` on the host

If we decide to give out `node_id` to the host it can do several annoying things when the host is compromized.

### Mutual close may become impossible

When **opening a channel** node can provide a `shutdown_scriptpubkey` that will be used for mutual channel close. If attacker replaces the `shutdown_scriptpubkey` to the key that is not owned by the node then mutual close becomes impossible - HSM will never sign a closing transaction with outputs to `shutdown_scriptpubkey` if it doesn't own this key. The other party will never sign a closing transaction if the output is not equal to `shutdown_scriptpubkey`. This means that such channel can be closed only unilaterelly.

### Invoices can send money to the attacker by channel replacement

When someone is openning a channel with us we provide to the remote node our `funding_pubkey`. If our node is compromised and `node_key` is stored on the host attacker can fool the remote nodes and replace our `funding_pubkey` with his key. 

All invoices can be also signed by the attacker with our `node_key` and then customers will pay to the attacker instead of us.

If we use our node only to provide liquidity to the network, collect fees from multihops and not accepting any payments, then leaving the `node_key` on the host makes sense. Otherwise `node_id` should be stored in HSM.