# OmniBOLT #2: RSMC and OmniLayer Transactions

Sometimes we use "money" instead of Omni assets for illustration. Readers just image what we fund, transfer or trade is USDT, an important asset issued on Omnilayer.

## The `funding_created` and `funding_signed` Message 

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
    |       |                              |       |
    |       |                              |       |
    |       |                              |       |
    |       |--(5)--- funding_locked  ---->|       |
    |       |<-(6)--- funding_locked  -----|       |
    +-------+                              +-------+

    - where node Alice is 'funder' and node Bob is 'fundee', 
After the first time deposit from Alice, Alice and Bob both can deposite in this channel any times. 
And of course, each of them can withdraw money if the counterparty agrees, 
as long as the two parties sign the correct Revocable Sequence Maturity Contracts for these onchain transactions.  

```

**Proposal 1: 2-2 P2SH**:
 
In order to avoid malicious counterparty who rejects to sigh any payment out of the P2SH transaction, so that the money is forever locked in the channel, we construct a Commitment Transaction where one is able to revoke a transaction. This is the first place we introduce the Revocable Sequence Maturity Contract (RSMC), invented by Poon and Dryja in their white paper, in this specification.

So the `funding_created` message does not mean both parties really deposite money into the channel. The first round communication is just simpley setup a P2SH address, and construct a RSMC, after that, Alice and Bob can transfer Omni assets  
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

 
## The `withdraw` Message 
 
**Rationale**
 
This message indicates how to withdraw money from a channel, without the need of closing a channel. It comprises the following steps:

1. Allice raises a request, to withdraw her money in P2SH, with her proof of her balance.
2. in 2-3 P2SH model, Satoshi verify Alice's request and proof, 
2.1 if they are correct, Alice and Satoshi raise a transaction paying from S2SH to Alice's omni address.
2.2 if they are wrong/incorrect, Satoshi reject the request and notify Bob
