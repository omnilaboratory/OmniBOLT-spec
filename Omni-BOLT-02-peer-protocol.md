# Omni-BOLT #2: Peer Protocol for Channel Management

The peer channel protocol has three phases: establishment, normal operation, and closing.

The basic oprations are the same to [BOLT 02](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md). 


# Channel

## [`Channel ID`](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#definition-of-channel_id)
Some messages use a `channel_id` to identify the channel. It's derived from the funding transaction by combining the `funding_txid` and the `funding_output_index`, using big-endian exclusive-OR (i.e. `funding_output_index` alters the last 2 bytes).

Prior to channel establishment, a `temporary_channel_id` is used, which is a random nonce.

Note that as duplicate `temporary_channel_id`s may exist from different peers, APIs which reference channels by their channel id before the funding transaction is created are inherently unsafe. The only protocol-provided identifier for a channel before funding_created has been exchanged is the (`source_node_id`, `destination_node_id`, `temporary_channel_id`) tuple. Note that any such APIs which reference channels by their channel id before the funding transaction is confirmed are also not persistent - until you know the script pubkey corresponding to the funding output nothing prevents duplicative channel ids.

## [Channel Establishment]()
After authenticating and initializing a connection ([BOLT #8](https://github.com/lightningnetwork/lightning-rfc/blob/master/08-transport.md) and [BOLT #1](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md), respectively), channel establishment may begin. This consists of the funding node (funder) sending an `open_channel` message, followed by the responding node (fundee) sending `accept_channel`. With the channel parameters locked in, the funder is able to create the funding transaction and both versions of the commitment transaction, as described in [Omni-BOLT #3]. The funder then sends the outpoint of the funding output with the `funding_created` message, along with the signature for the fundee's version of the commitment transaction. Once the fundee learns the funding outpoint, it's able to generate the signature for the funder's version of the commitment transaction and send it over using the `funding_signed` message.

```
    +-------+                              +-------+
    |       |--(1)---  open_channel  ----->|       |
    |       |<-(2)--  accept_channel  -----|       |
    |       |                              |       |
    |   A   |--(3)--  funding_created  --->|   B   |
    |       |<-(4)--  funding_signed  -----|       |
    |       |                              |       |
    |       |--(5)--- funding_locked  ---->|       |
    |       |<-(6)--- funding_locked  -----|       |
    +-------+                              +-------+

    - where node A is 'funder' and node B is 'fundee'

```

If this fails at any stage, or if one node decides the channel terms offered by the other node are not suitable, the channel establishment fails.

Note that multiple channels can operate in parallel, as all channel messages are identified by either a `temporary_channel_id` (before the funding transaction is created) or a `channel_id` (derived from the funding transaction).

**The `open_channel` Message**

This message contains information about a node and indicates its desire to set up a new Omni aware channel. This is the first step toward creating the funding transaction and both versions of the commitment transaction. In order to differ from the BOLT `open_channel` message, we use specified `chain_hash:chain_hash` to mark the Omni assets specific messages:

1. type: -32 (open_channel)
2. data: 
    * [`chain_hash`:`chain_hash`] 
    * [`32*byte`:`temporary_channel_id`]
    



