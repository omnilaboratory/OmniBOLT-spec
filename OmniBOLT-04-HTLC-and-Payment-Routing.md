# OmniBOLT #4: HTLC and Payment Routing

>*"A bidirectional payment channel only permits secure transfer of funds inside a channel. To be able to construct secure transfers using a network of channels across multiple hops to the final destination requires an additional construction, a Hashed Timelock Contract (HTLC)."*

> -- Poon & Dryja, The Bitcoin Lightning Network: Scalable Off-chain Instant Payments
  

A big and common misleading explanation in chaining the channels using HTLC is that if Alice wants to pay 10 USDT to David, she can use 2 hops to reach David:

```
Alice ---(10 USDT)---> Bob ---(10 USDT)---> Carol ---(10 USDT)---> David
```

It is confusing, because there is no concept of personal account in lightning. The only building block lightning uses is channel. So the correct hops are:

```
[Alice --(10 USDT)--> Bob] ==(Bob has two channels)== [Bob --(10 USDT)--> Carol] ==(Carol has two channels)== [Carol --(10 USDT)--> David]

[A, USDT, B] stands for the channel built by A and B, funded by USDT.
```

Alice transfers 10 USDT to Bob inside the `[Alice, USDT, Bob]` channel, then Bob transfers 10 USDT to Carol inside the `[Bob, USDT, Carol]` channel, and finally Carol sends 10 USDT to David in `[Bob, USDT, Carol]`.

## Hashed TimeLock Contract

An HTLC implements this procedure:

If Bob can tell Alice `R`, which is the pre-image of `Hash(R)` that some one else (Carol) in the chain shared with Bob 3 days ago in exchange of 10 USDT in the channel `[Bob, USDT, Carol]`, then Bob will get the 10 USDT fund inside the channel `[Alice, USDT, Bob]`, otherwise Alice gets her 10 USDT back. 

Simply put: 

```
in a particular channel, the one who wants the money, tells the counterparty the `R`. 
```
So the script is simple:

```
OP_IF
    OP_HASH256 <HASH 256(R)> OP_EQUALVERIFY
    2 <Alice2> <Bob2> OP_CHECKMULTISIG
OP_ELSE
    2 <Alice1> <Bob1> OP_CHECKMULTISIG
OP_ENDIF
```

Equipted with HTLC, the internal transfer of fund `[Alice --(10 USDT in HTLC)--> Bob]` is then an extra unbroadcasted output from funding transaction embeded together with [RD1a/BR1a](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#the-commitment_tx-and-commitment_tx_signed-message).

<!-- 
![HTLC](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png "HTLC")
-->

<p align="center">
  <img width="750" alt="HTLC with full Breach Remedy transactions" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png">
</p>

**HED1a**: HTLC Execution Delivery  
**HT1a**: HTLC Timeout  
**HBR1a**: HTLC Breach Remedy  
**HTRD1a**: HTLC Timeout Revocable Delivery  
**HTBR1a**: HTLC Timeout Breach Remedy  

## messages

```
    +-------+                                                             +-------+
    |       |--(1)-------------  update_add_htlc_1 (40)  ---------------->|       |  
    |       |         partially signed toB, toRSMC, toHTLC in C(3)a       |       |  
    |       |                                                             |       |
    |       |<-(2)-------------  update_add_htlc_2 (41)  -----------------|       |
    |   A   |           signed toB, toRSMC, toHTLC in C(3)a               |   B   | 
    |       |	              partially signed RD, HTLa, HLock            |       |
    |       |        partially signed toB, toRSMC, toHTLC in C(3)b        |       |
    |       |                                                             |       |
    |       |                                                             |       |
    |       |--(3)-------------  update_add_htlc_3 (42)   --------------->|       |  
    |       |              signed toB, toRSMC, toHTLC in C(3)b            |       |
    |       |   partially signed (RD in toRSMC, HTD1b in toHTLC, HLock)   |       |
    |       |   partially signed HTRD1a, HTBR1a, HED1a in HT1a in C(3)a   |       |
    |       |                                                             |       |
    |       |                                                             |       |
    |       |<-(4)-------------- update_add_htlc_4 (43)   ----------------|       |
    |       |          send back the completely signed HT1a in C(3)a      |       |
    |       |                                                             |       |  
    |       |                                                             |       |  
    |       |                                                             |       |  
    +-------+                                                             +-------+
   
```

## update_add_htlc 

`update_add_htlc` consists of two rounds of signing sub-transactions to forward an HTLC to the remote peer. Users should compare it to [`update_add_htlc`](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc) in `lightning-rfc`.  

Please note that the payment infomation are all encoded in transaction hex.  

`byte_array`: an array of bytes, the first 4 bytes indicate the length of the array.   
`32*byte`: a fixed length version of `byte_array`, where the decimal value of the first 4 bytes is 32.   

1. type: -40 (AddHTLC)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`int32`:`hop_id`]: auto increase by 1 when forwards an HTLC.
  * [`int32`:`PropertyId`]:  
  * [`byte`:`IsPayInvoice`]:  
  * [`32*byte`:`PayerCommitmentTxHash`]:  payer's commitment transaction hash.  
  * [`32*byte`:`PayerPeerId`]:  
  * [`32*byte`:`PayerNodeAddress`]:  
  * [`byte_array`:`C3aRsmcPartialSignedData`]:  
  * [`byte_array`:`C3aCounterpartyPartialSignedData`]:  
  * [`64*byte`:`C3aHtlcPartialSignedData`]:  
  	 
 
1. type: -41 (HTLCSigned)
2. data: 
  * [`channel_id`:`channel_id`]:
  * [`int32`:`hop_id`]: auto increase by 1 when forwards an HTLC. 
  * [`int32`:`PropertyId`]:  
  * [`32*byte`:`latestCommitmentTxID`]:  Commitment Tx ID.  
  * [`66*byte`:`RSMCTempAddressPubKey`]:   payer's temp address pubkey.  
  * [`32*byte`:`RSMCMultiAddress`]: Multi-sig address for RSMC.  
  * [`byte_array`:`RSMCRedeemScript`]:  Redeem script for RSMC.  
  * [`byte_array`:`RSMCMultiAddressScriptPubKey`]:  ScriptPubKey for RSMC.  
  * [`byte_array`:`RSMCTxHex`]:  Transaction hex for RSMC.  
  * [`32*byte`:`RSMCTxid`]:  Transaction ID.  
  * [`int32`:`AmountToRSMC`]:  Amount of property to RSMC address.  

 
  
1. type: -42 (PayeeCreateHTRD1a)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`int32`:`hop_id`]: auto increase by 1 when forwards an HTLC. 
  * [`int32`:`PropertyId`]:  
  * [`32*byte`:`latestCommitmentTxID`]:  Commitment Tx ID.  
  * [`32*byte`:`RSMCTempAddressPubKey`]:  Payee's current  Htlc temp address pubKey .  
  * [`32*byte`:`RSMCMultiAddress`]: Payee's multi-sig address for RSMC.  
  * [`byte_array`:`RSMCMultiAddressScriptPubKey`]:  ScriptPubKey for Payee's RSMC.  
  * [`byte_array`:`RSMCRedeemScript`]:  Redeem script for Payee's RSMC.  
  * [`byte_array`:`RSMCTxHex`]:  data from message 41 RSMCTxHex.  
  * [`32*byte`:`RSMCTxid`]:  Transaction ID.  
  * [`int32`:`AmountToRSMC`]:  Amount of property to RSMC address.  
 
1. type: -43 (PayerSignHTRD1a) send back the completely signed HT1a in C(3)a
2. data: 
  * [`channel_id`:`channel_id`]
  * [`int32`:`hop_id`]: auto increase by 1 when forwards an HTLC. 
  * [`int32`:`PropertyId`]:  
  * [`32*byte`:`latestCommitmentTxID`]:  Commitment Tx ID.  
  * [`32*byte`:`RSMCTempAddressPubKey`]:  Payee's current  Htlc temp address pubKey .  
  * [`32*byte`:`RSMCMultiAddress`]: Payee's multi-sig address for RSMC.  
  * [`byte_array`:`RSMCMultiAddressScriptPubKey`]:  ScriptPubKey for Payee's RSMC.  
  * [`byte_array`:`RSMCRedeemScript`]:  Redeem script for Payee's RSMC.  
  * [`byte_array`:`RSMCTxHex`]:  data from message 41 RSMCTxHex.Hex.  
  * [`32*byte`:`RSMCTxid`]:  Transaction ID.  
  * [`int32`:`AmountToRSMC`]:  Amount of property to RSMC address.   
		 
		 

### Requirements

to be added. 



## Terminate HTLC off-chain
After Bob telling Alice the `R`, the balance of this channel shall be updated, and the current HTLC shall be terminated. OBD then creates a new commitment transactions `C3a/C3b` for this purpose.

<p align="center">
  <img width="768" alt="Commitment Transaction to Terminate an HTLC" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/C3a-Terminate-a-HTLC-both-sides.png">
</p>

Terminating an HTLC: `update_fulfill_htlc`, `update_fail_htlc`

To supply the preimage:

1. type: -130 (update_fulfill_htlc)
2. data:
  * [`channel_id`:`channel_id`]
  * [`u64`:`hop_id`]
  * [`u64`:`property_id`]: the Omni asset id. 
  * [`u64`:`private_key_Alice_4`]: the private key of Alice 4, encrypted by Bob's public key when sending this message to Bob.
  * [`u64`:`private_key_Alice_5`]: the private key of Alice 5, encrypted by Bob's public key when sending this message to Bob.
  * [`32*byte`:`payment_preimage`]: the R

`private_key_Alice_4` and `private_key_Alice_5` are used in breach remedy transactions created in C2. If Bob find out Alice is cheating, Bob can broad these BR transactions to get the money in temporary multi-sig addresses as punishments to Alice. The procedure are the same to RSMC in chaper 2.

For a timed out or route-failed HTLC:

1. type: -131 (update_fail_htlc)
2. data:
  * [`channel_id`:`channel_id`]
  * [`u64`:`id`]
  * [`u16`:`len`]
  * [`len*byte`:`reason`]

### Requirements
the same to [requirement of Removing an HTLC](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#requirements-10). 



