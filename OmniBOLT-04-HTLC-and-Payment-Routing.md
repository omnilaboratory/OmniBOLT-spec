# OmniBOLT #4: HTLC and Payment Routing

>*"A bidirectional payment channel only permits secure transfer of funds inside a channel. To be able to construct secure transfers using a network of channels across multiple hops to the final destination requires an additional construction, a Hashed Timelock Contract (HTLC)."*

> -- Poon & Dryja, The Bitcoin Lightning Network: Scalable Off-chain Instant Payments
  

A common misleading explanation in chaining the channels using HTLC is that if Alice wants to pay 10 USDT to David, she can use 2 hops to reach David:

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

The redeem script is:

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


1. type: -40 (AddHTLC)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`byte`:`IsPayInvoice`]: bool type  
  * [`float64`:`amount`]:   
  * [`float64`:`amount_to_payee`]: amount to payee.   
  * [`byte_array`:`memo`]: memo to the payee.   
  * [`64*byte`:`h`]: Hash(R) used to lock the payment.    
  * [`int32`:`cltv_expiry`]: expiry blocks.  
  * [`byte_array`:`last_temp_address_private_key`]: private key of temp address generated in last RSMC: give up the previous commitment tx.  
  * [`byte_array`:`htlc_sender_signed`]: HTLC sub contracts and partially signed by the sender.    
  * [`byte_array`:`routing_packet`]:   
  * [`byte_array`:`payer_commitment_tx_hash`]:  payer's commitment transaction hash.  
  * [`byte_array`:`PayerNodeAddress`]: 
  * [`byte_array`:`PayerPeerId`]:   
	 
Data format of `htlc_sender_signed`:  
  
  	* [`byte_array`:`curr_rsmc_temp_address_pub_key`]: pub key of temp address used to accept "pay to RSMC" in current C(n)x.  
  	* [`byte_array`:`curr_htlc_temp_address_pub_key`]: pub key of temp address used to accept "pay to HTLC" in current C(n)x.  
  	* [`byte_array`:`curr_htlc_temp_address_for_ht1a_pub_key`]: pub key of temp address used to accept "pay to RSMC" in Ht1a.  
  	* [`byte_array`:`cna_counterparty_partial_signed_data`]:   partially signed data from payer in C(n)x, e.g. C(3)a.  
  	* [`byte_array`:`cna_rsmc_partial_signed_data`]:   partially signed RSMC data from payer in C(n)x, e.g. C(3)a. 
  	* [`byte_array`:`cna_htlc_partial_signed_data`]:   partially signed HTLC data from payer in C(n)x, e.g. C(3)a.  

 
1. type: -41 (HTLCReceiverSigned)
2. data: 
  * [`channel_id`:`channel_id`]:
  * [`byte_array`:`payer_commitment_tx_hash`]:  payer's commitment transaction hash.  
  * [`byte_array`:`htlc_reciever_signed`]: HTLC sub contracts signed by the reciever, the symmetric transactions created on the reciever side and signed. 
  * [`byte_array`:`PayerNodeAddress`]: 
  * [`byte_array`:`PayerPeerId`]:   

Data format of `htlc_reciever_signed`:  

  	* [`byte_array`:`cna_htlc_temp_address_for_ht_pub_key`]:    
  	* [`byte_array`:`payee_commitment_tx_hash`]:  payee's commitment transaction hash.   
 	* [`byte_array`:`payee_curr_rsmc_temp_address_pub_key`]:  payee's current RSMC temp address pub_key.   
  	* [`byte_array`:`payee_curr_htlc_temp_address_pub_key`]:  payee's current HTLC temp address pub_key.   
  	* [`byte_array`:`payee_curr_htlc_temp_address_for_he_pub_key`]:  payee's current pub_key of HTLC temp address for HE.  
  	* [`byte_array`:`payee_last_temp_address_private_key`]:  payee's private_key of the last temp address.  
  	* [`byte_array`:`cna_complete_signed_rsmc_hex`]:    
  	* [`byte_array`:`cna_complete_signed_counterparty_hex`]:  
  	* [`byte_array`:`cna_complete_signed_htlc_hex`]:  
  	* [`byte_array`:`cna_rsmc_rd_partial_signed_data`]:  
  	* [`byte_array`:`cna_htlc_ht_partial_signed_data`]:  
  	* [`byte_array`:`cna_htlc_hlock_partial_signed_data`]:  
  	* [`byte_array`:`cnb_rsmc_partial_signed_data`]:  
  	* [`byte_array`:`cnb_counterparty_partial_signed_data`]:  
  	* [`byte_array`:`cnb_htlc_partial_signed_data`]:   



  
1. type: -42 (PayerCompletlySigned)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`byte_array`:`payer_commitment_tx_hash`]:  payer's commitment transaction hash.   
  * [`byte_array`:`payee_commitment_tx_hash`]:  payee's commitment transaction hash.   
  * [`byte_array`:`payer_completely_signed`]: payer completely signed HTLC sub contracts.  
  * [`byte_array`:`PayerNodeAddress`]: 
  * [`byte_array`:`PayerPeerId`]:   

Data format of `payer_completely_signed`:  

  	* [`byte_array`:`cna_htlc_temp_address_for_ht_pub_key`]:  pub key of temp address in HT for HTLC of C(n)a, e.g. C(3)a.  
  	* [`byte_array`:`cnb_complete_signed_rsmc_hex`]: completly signed RSMC hex in cnb, , e.g. C(3)b.  
  	* [`byte_array`:`cnb_complete_signed_counterparty_hex`]: completly signed counterparty's hex in cnb, , e.g. C(3)b.  
  	* [`byte_array`:`cnb_complete_signed_htlc_hex`]: completly signed HTLC hex in cnb, e.g. C(3)b.   
  	* [`byte_array`:`cnb_rsmc_rd_partial_data`]: partially signed RD in RSMC in cnb, e.g. C(3)b.   
  	* [`byte_array`:`cnb_htlc_htd_partial_data`]: partially signed HTD in HTLC in cnb, e.g. C(3)b.   
  	* [`byte_array`:`cnb_htlc_hlock_partial_data`]:  
  	* [`byte_array`:`cna_htlc_ht_hex`]: hex of HT in HTLC of C(n)a.   
  	* [`byte_array`:`cna_htlc_htrd_partial_data`]:  HTRD data of C(n)a. 
  	* [`byte_array`:`cna_htlc_htbr_partial_data`]:  HTBR data of C(n)a. 
  	* [`byte_array`:`cna_htlc_hed_raw_data`]:  HED data of C(n)a.  
 
 
1. type: -43 (Completly signed HTLC) send back the completely signed HT1a in C(n)a
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`byte_array`:`cna_htlc_htrd_complete_signed_hex`]:  
  * [`byte_array`:`cna_htlc_hed_partial_data`]:   
		 
		 

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



