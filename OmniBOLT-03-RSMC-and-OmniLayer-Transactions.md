# OmniBOLT #3: RSMC and OmniLayer Transactions

Sometimes we use "money" instead of Omni assets for illustration purpose. Readers just image that what we fund, transfer or trade is USDT, an important asset issued on Omnilayer.

From this chapter on, our context is Omnilayer, not only bitcoin any more.

## The `btc_funding_created`, `btc_funding_signed`, `asset_funding_created` and `asset_funding_signed` Messages 

The four messages describe the outpoint which the funder has created for the initial commitment transactions. After receiving the peer's signature, via `funding_signed`, it will broadcast the funding transaction to the Omnilayer network.

We focus on omni assets in founding creation. Here are two proposals:

**Prop 1**. Alice and Bob create a 2-2 P2SH payment to `scriptPubKey`, which is the hash of the **Redeem Script**.

`Deprecated`
**Prop 2**. Alice, Bob and Satoshi create a 2-3 P2SH payment to `scriptPubKey`, where Satoshi takes the responsibility to check/verify all the transactions within the channel. The advantage of this proposal is that when Alice wants to withdraw money from the channel, she does not need the signature from Bob, the only thing she has to do is provide her signature and notify Satoshi to check the channel ledger, and if the money belongs to Alice, Satoshi will provide his signature so that Alice get her money on Omnilayer network.

Proposal 2 has possible vulnerability under the **conspiracy attack**. We will address this issue in the following chapters.



```
    +-------+                                                  +-------+
    |       |---(1)-------------   open_channel     ---------->|       |
    |       |<--(2)------------  accept_channel    ------------|       |
    |       |                                                  |       |
    |   A   |---(3)------- btc_funding_created(3400 ) -------->|   B   |
    |       |<--(4)-------  btc_funding_signed(3500)  ---------|       |
    |       |                                                  |       |
    |       |---(5)-------- asset_funding_created(34) -------->|       |
    |       |<--(6)--------  asset_funding_signed(35) ---------|       |
    |       |                                                  |       |
    |       |---(7)------------  funding_locked  ------------->|       |
    |       |<--(8)------------  funding_locked  --------------|       |
    |       |                                                  |       |
    |       |<----------------   wait for close  ------------->|       |
    +-------+                                                  +-------+

    - where node Alice is 'funder' and node Bob is 'fundee'. Same to BOLT, the fundee is not allowed to fund the channel. 
    This is because the limitation of current BTC implementation. 
    - Of course, each of them can withdraw money if the counterparty agrees, as long as the two parties sign the correct Revocable Sequence Maturity Contracts for these onchain transactions.  

```

**Proposal 1: 2-2 P2SH**:
 
In order to avoid malicious counterparty who rejects to sign any payment out of the P2SH transaction, so that the money is forever locked in the channel, we construct a Commitment Transaction where one is able to revoke a transaction. This is the first place we introduce the Revocable Sequence Maturity Contract (RSMC), invented by Poon and Dryja in their white paper, in this specification.

So the `funding_created` message does not mean both parties really deposite money into the channel. The first round communication is just simpley setup a P2SH address, construct funding transaction but unbroadcasted and construct a RSMC. After that, Alice or Bob can broadcast the funding transaction to transfer real Omni assets into the channel.

The following diagram shows the steps we MUST do before any participants broadcast the funding/commitment transactions. BR1a (Breach Remedy) can be created later before the next commitment transaction is contructed.


<p align="center">
  <img width="500" alt="RSMC-C1a-RD1a" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-C1a-RD1a.png">
</p>

Alice's OBD constructs refund transaction: C1a/RD1a (Revocable Delivery), which pays out from the 2-2 P2SH transaction output:

step 1: Alice constructs a temporary 2-2 multi-sig address using Alice's temporary private key Alice2 and waiting Bob's signature: Alice2 & Bob.

step 2: Alice constructs a promise payment C1a out of Alice & Bob, one output is 60 USDT to `Alice2 & Bob`, and the other output is 40 USDT to Bob.

step 3: RD1a is the first output of C1a, which pays Alice 60 USDT, but with a sequence number preventing immediate payment if Alice cheat.

step 4: Bob signs C1a and RD1a, sends back to Alice.

step 5: Alice's OBD constructs refund transaction: C1b/RD1b.
 
`byte_array`: an array of bytes, the first 4 byte indicate the length of the array.   
`32*byte`: a fixed length version of `byte_array`, where the decimal value of the first 4 bytes is 32.   


1. type: -3400 (btc_funding_created)
2. data:
    * [`32*byte`:`amount`]: total BTC funded by Alice.  
    * [`32*byte`:`funding_btc_hex`]: BTC funding transaction hex, used by Bob to validate the transaction.  
    * [`byte_array`:`funding_redeem_hex`]: funding redeem hex.   
    * [`sha256`:`funding_txid`]: funding transaction id.
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
 
Alice's OBD notify Bob's OBD that she created the BTC funding transaction by payloads packed in this message -3400. 


1. type: -3500 (btc_funding_signed)
2. data:
    * [`byte`:`approval`]: true to confirm the btc funding transaction. 
    * [`byte_array`:`funding_redeem_hex`]: funding redeem hex.   
    * [`sha256`:`funding_txid`]: funding transaction id.   
    * [`32*byte`:`temporary_channel_id`]: the `temporary_channel_id` which will be replaced by channel_id in the following messeges.
  

Bob's OBD verifies the btc funding transaction by its hex, and replies Alice that he knows the funding of BTC by message -3500. 


1. type: -34 (asset_funding_created)
2. data:
    * [`byte_array`:`c1a_rsmc_hex`]: The constructed first rsmc transaction hex.  
    * [`byte_array`:`funding_omni_hex`]: The funding omni asset transaction hex, with which Bob can extract all the asset information, including funding output indx, amount, token, etc. Bob then is able to verify the funding transaction. Be noticed that in [BOLT-02-funding_created message](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#the-funding_created-message), funder sends fundee a funding_output_index for validation.  
    * [`32*byte`:`rsmc_temp_address_pub_key`]: Internal multisig address <Alice2 & Bob> used by c1a.
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
    
    
Alice creats the asset funding transaction and asks Bob to verify and sign this transaction.
 
1. type: -35 (asset_funding_signed)
2. data:
    * [`channel_id`:`channel_id`]: generated by exclusive-OR of the funding_txid and the funding_output_index from the asset_funding_created message. 
    * [`byte_array`:`rd_hex`]: The hex of revockable delivery(RD1a) transaction.
    * [`32*byte`:`fundee_pubKey`]: the omni address of fundee Bob.
    * [`byte_array`:`rsmc_signed_hex`]: rsmc signed by Bob.
     
  
Bob signs, and send `asset_funding_signed` message back to Alice, hence Alice knows the 2-2 P2SH address has been created, but not broadcasted. 
   
   
## The `commitment_tx` and `revoke and acknowledge commitment transaction` Message

The two messages describe a payment inside one channel created by Alice and Bob, upon which HTLC makes it possible for quick payment between any two peers, who do not has a direct channel yet. We introduce HTLC and corresponding messages in the next chapter.  

### diagram and messages
```
    +-------+                                             +-------+
    |       |--(1)-------  commitment_tx(351)  ---------->|       |  
    |       |<-(2)----- commitment_tx_signed (352) -------|       |
    |   A   |                                             |   B   | 
    |       |           construct BR1a, C2a and RD2a      |       | 
    |       |            and the mirror transactions      |       |
    |       |                  on Bob's OBD               |       | 
    |       |                                             |       |  
    |       |--(3)-------------- (353) ------------------>|       |
    |       |      send back signed transactions in C2B   |       |
    +-------+                                             +-------+
   
```
     
<!-- ![RSMC](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-diagram.png "RSMC") -->

<p align="center">
  <img width="750" alt="RSMC-diagram" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-diagram.png">
</p>

There are two outputs of a commitment transaction:  
[to local](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_local-output): 0. Alice2 & Bob 60,  
[to remote](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_remote-output): 1. Bob 60.  

`to local output` sends funds back to the owner of this commitment transaction and thus must be timelocked using for example sequence number =1000. By breach remedy transaction, this money will be sent to the counterparty(Bob) immediatly(without delay), if Alice tries to broadcast C1a to claim more money after she pays by C2a.  

Alice must send the hex `rsmc_hex` of the transaction based on `to local output` to Bob to verify and sign. The transaction based on `to remote output` is named `counterparty_tx`, and Alice must send the hex `to_counterparty_tx_hex` to Bob to sign as well. In message `-352`, the signed arguments are `signed_to_counterparty_tx_hex` and `signed_rsmc_hex` respectively.  

1. type: -351 (commitment_tx)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`32*byte`:`amount`]: amount of the payment.
    * [`byte_array`:`commitment_tx_hash`]: C(n+1)a's commitment transaction hash, if Alice is the payer.
    * [`32*byte`:`last_temp_address_private_key`]: When C(n+1)a created, the payer shall give up the private key of the previous temp multi-sig address in C(n)a. <Alice2, Bob> as an example in the above diagram. 
    * [`byte_array`:`rsmc_hex`]: the hex of RSMC transaction. From the hex, payment information can be extracted and verified by the counterparty.  
    * [`byte_array`:`to_counterparty_tx_hex`]: The hex of the transaction that pays the counterparty money.
 
For example, Alice pays Bob `amount` of omni asset by sending `rsmc_Hex`. Her OBD constructs BR(1)a(Breach Remedy) and send it to Bob. Bob checks the Alice2's signature in BR1a to verify if the private key is correct. If it is, Bob signs C2a/RD2a.

 
1. type: -352 (Revoke and Acknowledge Commitment Transaction)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`byte`:`approval`]: payee accepts or rejects this payment.
    * [`32*byte`:`amount`]: amount of the payment.  
    * [`byte_array`:`commitment_tx_hash`]: Payer's commitement tx hash.  
    * [`32*byte`:`last_temp_address_private_key`]: payee gives up the private key of his previous temp multi-sig address in C(n)b.
    * [`32*byte`:`curr_temp_address_pub_key`]: payee's current temp address pubkey.
    * [`byte_array`:`signed_to_counterparty_tx_hex`]: payee signs the `to_counterparty_tx_hex` sent by payer.
    * [`byte_array`:`signed_rsmc_hex`]: payee signs the `rsmc_hex` sent by payer.
    * [`byte_array`:`rsmc_hex`]: payee's rsmc transaction hex, used in constructing mirror transaction on payee's obd. 
    * [`byte_array`:`to_counterparty_tx_hex`]: payee's to_counterparty_tx_hex, used in constructing mirror transaction on payee's obd. 
    * [`byte_array`:`payer_rd_hex`]: payee signs based on `signed_rsmc_hex`, and send it back to payer to construct the RD transaction.
 

1. type: -353 (send back signed transactions in C2B)  
2. data: to be added  
	

## Cheat and Punishment

In the above diagram, the payer Alice's OBD constructs C2a. Simultaneously, Alice sends her temporary private key of Alice2 to Bob. After Bob sends back the signed messsage, she is able to construct RD2a. If she cheats by broadcasting C1a, where she has more money in her balance, Bob will immedialtly get 60 USDT from address `Alice2 & Bob`. There has to be a daeman process that monitors Alice's behaviar. If it detects that Alice broadcasts C1a, it has to notify Bob to broadcast the punishment transaction BR1a using Alice2's private key. If Bob does not broadcast BR1a before the sequence number expires, Alice will be success in cheating, and get the 60 USDT.
 
 
## The `close_channel` Message 
 
**Rationale**
 
This message indicates how to withdraw money from a channel, by broadcasting a `C(n)a`, which is the latest commitment transaction in this channel. It comprises the following steps:

1. Allice raises a request, to withdraw her money in P2SH, with her proof of her balance.  
2. Alice waits OBD to approval:  
2.1 if they are correct, Alice raises a transaction paying from S2SH to Alice's omni address.  
2.2 if they are wrong/incorrect, OBD rejects the request and notify Bob.  


Before closing a channel, all HTLCs pending in this channel shall be removed, after which `close_channel` can be successfully executed.

1. type: -38 (close_channel)  
2. data:
    * [`channel_id`:channel_id]
    * [`u16`:`len`]
    * [`len`*`byte:scriptpubkey`]
    * [`signature`:`signature`]: the signature of Alice or Bob.

 
 
