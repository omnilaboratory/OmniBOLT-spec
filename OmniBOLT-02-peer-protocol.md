# OmniBOLT #2: Peer Protocol for Channel Management

# Omni Address 
One most important point is that addresses in OmniBOLT and its implementation must be Omnilayer addresses created by omnicore. Bitcoin-only address can not know any assets information. When users fund Omni assets to a channel with bitcoin addresses that do not support Omni Layer transactions, it can be difficult or impossible to recover the transferred Omni assets.  

So this is the main reason that current lnd channels can not be OmniBOLT channels.

In the soon future, OmniBOLT will update to ["Omni Layer Safe Segregated Witness Address Format"](https://github.com/OmniLayer/Documentation/blob/master/OLEs/ole-300.adoc#motivation).  
 

# Channel
The peer Poon-Dryja channel protocol has three phases: establishment, normal operation (commitment transactions, funding transactions, HTLCs, Collateral lending, etc), and closing.

The basic logics are the same to [BOLT 02](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md), but with some updates in messages to be compatible with OmniLayer protocol. The fields are similar to what are defined in BOLT 02, but [OmniBOLT (chapter 7) does not involve any third party watch tower](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md) for monitoring and punishing, the key generation mechanism is different, so that the various `_basepoint` fields are not used when creates channels. During our implementation, these fields may be changed.  
  

## `Channel ID` 
The concept of channel ID is the same to the definition in [BOLT](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#definition-of-channel_id). It is a global unique identification for a channel, used by wallets to locate and connect to users' account. Before a final channel id is created, a temporary id normally used before funding real BTC and tokens into the channel to be established.

The temporaty ids may be duplicated. In current implementation, one instance of OBD will not generate duplicated temp id, and one OBD may manage thousands of light clients(users). Only a channel id has been finalized, it can be braodcast, and can be used in operations from other OBD instances. 

## [Channel Establishment]()

Since OmniBOLT uses channels to transfer and exchange tokens, and on bitcoin network, only BTC can be the transaction fee, here comes the difference between OmniBOLT and other lightning specifications: We need extra messages to deposit miner fees into a channel to be established.

Opening a channel seeks remote an OmniBOLT node, to create a channel according to the requests from local node.  

The process consists of:

* funding node (funder) sends an `open_channel` message, followed by the responding node (fundee) sending `accept_channel`. 
* The funder creates the funding BTC transaction and get the approval from fundee. 
* The funder creates the funding token transaction and the corresponding commitment transaction, as described in [OmniBOLT #3](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md). 
* The funder then sends the hex of the funding transaction with the `asset_funding_created` message, along with the hex of the first commitment transaction in the channel, to the fundee. 
* Once the fundee learns the funding transaction have been created, and validate the hex, it's able to generate the signature for the funder's version of the commitment transaction, create his mirror commitment transaction, and send it back using the `asset_funding_signed` message.
* Once the funder recieved the signed feedback message, he has to broadcast his asset funding transaction, so that the channel will hold the assets. 


During funding creation, OmniBOLT adds extra arguments (e.g. `[32*byte:property_id]`) to specify which omni asset is funded in creating this channel. 


```
    +-------+                                  +-------+
    |       |---(1)----  open_channel    ----->|       |
    |       |<--(2)---  accept_channel   ------|       |
    |       |                                  |       |
    |   A   |---(3)-- funding BTC created ---->|   B   |
    |       |<--(4)--  funding BTC signed  ----|       |
    |       |                                  |       |
    |       |---(5)- funding Tokens created -->|       |
    |       |<--(6)-- funding Tokens signed ---|       |
    |       |                                  |       |
    |       |---(7)----  funding_locked  ----->|       |
    |       |<--(8)----  funding_locked  ------|       |
    |       |                                  |       |
    |       |<-------   wait for close   ----->|       |
    +-------+                                  +-------+

    - where node A is 'funder' and node B is 'fundee'. 
    - extra BTC funding is needed as transaction fee, from where OmniBOLT differs BOLT

```

If this fails at any stage, or if one node decides the channel terms offered by the other node are not suitable, the channel establishment fails.

Note that multiple channels can operate in parallel, as all channel messages are identified by either a `temporary_channel_id` (before the asset funding transaction is created) or a `channel_id` (derived from the funding transaction).

### The `open_channel` Message 

This message contains information about a node and indicates its desire to set up a new Omni aware channel. This is the first step toward creating the funding transaction and both versions of the commitment transaction. In order to recognize a OmniBOLT channel, we use specified `chain_hash:chain_hash` to mark the OmniBOLT specific messages.


We don't specify which asset will be in this channel during creating, so there is no need to modify the existing design. Comments of this message comes from the original [BOLT #2](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#the-open_channel-message) spec, readers may check if the two are consistent:  

1. type: -32 (open_channel)
2. data: 
    * [`chain_hash`:`chain_hash`]: The `chain_hash` value denotes the exact blockchain that the opened channel will reside within. This is usually the genesis hash of the respective blockchain. We use in this case the hash of the ["Exodus Address"](https://github.com/OmniLayer/spec#initial-token-distribution-via-the-exodus-address) for Omnilayer, which is `1EXoDusjGwvnjZUyKkxZ4UHEf77z6A5S4P`, from which the first Mastercoins were generated during the month of August 2013. PS: `chain_hash` allows nodes to open channels across many distinct blockchains as well as have channels within multiple blockchains opened to the same peer (if it supports the target chains).
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
    * [`point`:`funding_pubkey`]: the public key in the two-of-two multisig script of the funding transaction output.  

 
    **basepoint is ignored**: The [BOLT #3: key-derivation](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#key-derivation) uses the various `_basepoint` fields to derive unique keys for each commitment transaction. This property is used for preserving privacy when outsourcing penalty transactions to third parties. But OmniBOLT does not involve third party watch towers, we apply Hierarchical Deterministic(HD) pathes to generate pub/priv key pairs for all the transactions and there is no outsourcing of monitoring revockable transactions and punishing cheating activities. Reader shall go to [chapter #7 Hierarchical Deterministic wallet](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md) for the motivation and mechanism.  
 
    * [`point`:`revocation_basepoint`]: 
    * [`point`:`payment_basepoint`]: 
    * [`point`:`delayed_payment_basepoint`]: 
    * [`point`:`htlc_basepoint`]: 
    * [`point`:`first_per_commitment_point`]: 
  
    * [`byte`:`channel_flags`]: Only the least-significant bit of `channel_flags` is currently defined: `announce_channel`. This indicates whether the initiator of the funding flow wishes to advertise this channel publicly to the network, as detailed within [BOLT #7: p2p-node-and-channel-discovery](https://github.com/lightningnetwork/lightning-rfc/blob/master/07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery).
    * [`u16`:`shutdown_len`] (option_upfront_shutdown_script): 
    * [`shutdown_len*byte`:`shutdown_scriptpubkey`] (option_upfront_shutdown_script): allows the sending node to commit to where funds will go on mutual close, which the remote node should enforce even if a node is compromised later.

** Requirement 

Requirement is the same to [BOLT-02-requiremnets](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#requirements).

** Rationale 

Rationale is the same to [BOLT-02-rationale](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#rationale).

** Future 

Future is the same to [BOLT-02-future](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#future).

 

### The `accept_channel` Message 

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


 
