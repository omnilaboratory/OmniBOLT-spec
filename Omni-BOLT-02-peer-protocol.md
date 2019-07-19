# OmniBOLT #2: Peer Protocol for Channel Management

The peer channel protocol has three phases: establishment, normal operation, and closing.

The basic oprations are the same to [BOLT 02](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md), but with essential updates in messages to be compatible with OmniLayer protocol.


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
    * [`chain_hash`:`chain_hash`]: The `chain_hash` value denotes the exact blockchain that the opened channel will reside within. This is usually the genesis hash of the respective blockchain. We use in this case the hash of the ["Exodus Address"](https://github.com/OmniLayer/spec#initial-token-distribution-via-the-exodus-address) for Omnilayer, which is 1EXoDusjGwvnjZUyKkxZ4UHEf77z6A5S4P, from which the first Mastercoins were generated during the month of August 2013. PS: `chain_hash` allows nodes to open channels across many distinct blockchains as well as have channels within multiple blockchains opened to the same peer (if it supports the target chains).
    * [`32*byte`:`temporary_channel_id`]: used to identify this channel on a per-peer basis until the funding transaction is established, at which point it is replaced by the `channel_id`, which is derived from the funding transaction. 
    * [`u64`:`funding_satoshis`]: the amount the sender is putting into the channel.
    * [`u64`:`push_msat`]: an amount of initial funds that the sender is **unconditionally** giving to the receiver. 
    * [`u64`:`dust_limit_satoshis`]: the threshold below which outputs should not be generated for this node's commitment or HTLC transactions (i.e. HTLCs below this amount plus HTLC transaction fees are not enforceable on-chain). This reflects the reality that tiny outputs are not considered standard transactions and will not propagate through the Bitcoin/OmniLayer network.
    * [`u64`:`max_htlc_value_in_flight_msat`]: is a cap on total value of outstanding HTLCs, which allows a node to limit its exposure to HTLCs.
    * [`u64`:`channel_reserve_satoshis`]: the minimum amount that the other node is to keep as a direct payment.
    * [`u64`:`htlc_minimum_msat`]: indicates the smallest value HTLC this node will accept. 
    * [`u32`:`feerate_per_kw`]: indicates the initial fee rate in satoshi per 1000-weight (i.e. 1/4 the more normally-used 'satoshi per 1000 vbytes') that this side will pay for commitment and HTLC transactions, as described in [BOLT #3: fee-calculation](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#fee-calculation) (this can be adjusted later with an `update_fee` message).
    * [`u16`:`to_self_delay`]: the number of blocks that the other node's to-self outputs must be delayed, using OP_CHECKSEQUENCEVERIFY delays; this is how long it will have to wait in case of breakdown before redeeming its own funds.
    * [`u16`:`max_accepted_htlcs`]: similar to `max_htlc_value_in_flight_msat`, this value limits the number of outstanding HTLCs the other node can offer, **NOT** the total value of HTLCs. 
    * [`point`:`funding_pubkey`]: the public key in the 2-of-2 multisig script of the funding transaction output.
    
    **basepoint**: The various `_basepoint` fields are used to derive unique keys as described in [BOLT #3: key-derivation](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#key-derivation) for each commitment transaction. Varying these keys ensures that the transaction ID of each commitment transaction is unpredictable to an external observer, even if one commitment transaction is seen; this property is very useful for preserving privacy when outsourcing penalty transactions to third parties.
    
    * [`point`:`revocation_basepoint`]:  
    * [`point`:`payment_basepoint`]:  
    * [`point`:`delayed_payment_basepoint`]:  
    * [`point`:`htlc_basepoint`]:  
    * [`point`:`first_per_commitment_point`]: the per-commitment point to be used for the first commitment transaction,
    
    
    * [`byte`:`channel_flags`]: Only the least-significant bit of `channel_flags` is currently defined: `announce_channel`. This indicates whether the initiator of the funding flow wishes to advertise this channel publicly to the network, as detailed within [BOLT #7: p2p-node-and-channel-discovery](https://github.com/lightningnetwork/lightning-rfc/blob/master/07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery).
    * [`u16`:`shutdown_len`] (option_upfront_shutdown_script): 
    * [`shutdown_len*byte`:`shutdown_scriptpubkey`] (option_upfront_shutdown_script): allows the sending node to commit to where funds will go on mutual close, which the remote node should enforce even if a node is compromised later.
    
    **[ FIXME: Describe dangerous feature bit for larger channel amounts. ]**



# [Requirement]

Requirement is the same to [BOLT-02-requiremnets](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#requirements).

# [Rationale]

Rationale is the same to [BOLT-02-rationale](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#rationale).

# [Future]

Future is the same to [BOLT-02-future](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#future).

It would be easy to have a local feature bit which indicated that a receiving node was prepared to fund a channel, which would reverse this protocol.

**The `accept_channel` Message**

This message contains information about a node and indicates its acceptance of the new channel. This is the second step toward creating the funding transaction and both versions of the commitment transaction.

1. type: -33 (accept_channel)
2. data:
    * [`32*byte`:`temporary_channel_id`]
    * [`u64`:`dust_limit_satoshis`]
    * [`u64`:`max_htlc_value_in_flight_msat`]
    * [`u64`:`channel_reserve_satoshis`]
    * [`u64`:`htlc_minimum_msat`]
    * [`u32`:`minimum_depth`]
    * [`u16`:`to_self_delay`]
    * [`u16`:`max_accepted_htlcs`]
    * [`point`:`funding_pubkey`]
    * [`point`:`revocation_basepoint`]
    * [`point`:`payment_basepoint`]
    * [`point`:`delayed_payment_basepoint`]
    * [`point`:`htlc_basepoint`]
    * [`point`:`first_per_commitment_point`]
    * [`u16`:`shutdown_len`] (option_upfront_shutdown_script)
    * [`shutdown_len*byte`:`shutdown_scriptpubkey`] (option_upfront_shutdown_script)
    

#[Requirements](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#requirements-1)


**The `funding_created` and `funding_signed` Message**

This message describes the outpoint which the funder has created for the initial commitment transactions. After receiving the peer's signature, via `funding_signed`, it will broadcast the funding transaction to the BTC/Omnilayer network.

We focus on omni assets in founding creation. Here are two proposals:
**Prop 1**. Alice and Bob create a 2-2 P2SH payment to `scriptPubKey`, which is the hash of the **Redeem Script**.
**Prop 2**. Alice, Bob and Satoshi create a 2-3 P2SH payment to `scriptPubKey`, where Satoshi takes the responsibility to check/verify all the transactions within the channel. The advantage of this proposal is that when Alice wants to withdraw money from the channel, she does not need the signature from Bob, the only thing she has to do is provide her signature and notify Satoshi to check the channel ledger, and if the money belongs to Alice, Satoshi will provide his signature so that Alice get her money on Omnilayer network.

Proposal 2 has possible vulnerability under the **conspiracy attack**. We will address this issue in the following chapters.

```
    +-------+                              +-------+
    |       |--(1)---  open_channel  ----->|       |
    |       |<-(2)--  accept_channel  -----|       |
    |       |                              |       |
    |   A   |--(3)--  funding_created  --->|   B   |
    |       |<-(4)--  funding_signed  -----|       |
    |       |                              |       |
    |       |         channel_A_B          |       | 
    |       |(4.1)-->deposit_X_USDT        |       |
    |       |        deposit_Y_USDT <-(4.2)|       |
    |       |            ......            |       |
    |       |                              |       |
    |       |                              |       |
    |       |--(5)--- funding_locked  ---->|       |
    |       |<-(6)--- funding_locked  -----|       |
    +-------+                              +-------+

    - where node Alice is 'funder' and node Bob is 'fundee', 
but after the first time deposit from Alice, Alice and Bob both can deposite in this channel any times. And of course, each of them can withdraw money if the counterparty agree.

```

**Proposal 1: 2-2 P2SH**:

1. type: -34 (funding_created)
2. data:
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
    * [`32*byte:funder_pubKey`]: the omni address of funder Alice.
    * [`signature`:`funder_signature`]: signature of funder Alice.
    
 
1. type: -35 (funding_signed)
2. data:
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
    * [`32*byte`:`funder_pubKey`]: the omni address of funder Alice.
    * [`signature`:`funder_signature`]: signature of funder Alice.
    * [`32*byte`:`fundee_pubKey`]: the omni address of fundee Bob.
    * [`signature`:`fundee_signature`]: signature of funder Bob.
    * [`64*byte???`:`redeemScript`]: redeem script used to generate P2SH address.
    * [`32*byte`:`p2sh_address`]: hash of redeemScript.
    * [`channel_id`:`channel_id`]: final global channel id generated.
  
 
**deposit** message can be sent by both funder and fundee, as long as the amount of assets is blow `max_htlc_value_in_flight_msat` in the `open_channel` message. 
    
1. type: -351 (deposit)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`32*byte:p2sh_address`]: the p2sh address generated in `funding_signed` message.
    * [`32*byte`:`deposite_transaction_id`]: deposite transaction id.
    * [`32*byte`:`when`]: the time of the transaction get confirmed.
    * [`32*byte`:`where`]: from which omni address this transaction is raised. 
    * [`signature`:`funder_signature`]: the signature of the funder to prove himself.
    
 In 2-3 P2SH, Satoshi will record every deposit, and broadcast it to Alice and Bob for confirmation.
 
 
1. type: -352 (get_balance)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`32*byte:p2sh_address`]: the p2sh address generated in `funding_signed` message.
(**Deprecated**)* [`32*byte`:`who`]: the channel owner, Alice or Bob, can query the balance.
    * [`signature`:`signature`]: the signature of Alice or Bob.
    
1. type: -353 (get_balance_respond)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`32*byte`:`property_id`]: the asset id generated by Omnilayer protocol.
    * [`32*byte`:`name`]: the name of the asset.
    * [`unsigned_64byte_integer`:`balance`]: balance in this channel.
    * [`unsigned_64byte_integer`:`reserved`]: currently not in use.
    * [`unsigned_64byte_integer`:`frozen`]: currently not in use.

 
**The `withdraw` Message**
 
# Rationale
 
This message indicates how to withdraw money from a channel, without the need of closing a channel. It comprises the following steps:

1. Allice raises a request, to withdraw her money in P2SH, with her proof of her balance.
2. in 2-3 P2SH model, Satoshi verify Alice's request and proof, 
2.1 if they are correct, Alice and Satoshi raise a transaction paying from S2SH to Alice's omni address.
2.2 if they are wrong/incorrect, Satoshi reject the request and notify Bob