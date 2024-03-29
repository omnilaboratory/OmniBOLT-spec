# OmniBOLT #4: HTLC and Payment Routing

>*"A bidirectional payment channel only permits secure transfer of funds inside a channel. To be able to construct secure transfers using a network of channels across multiple hops to the final destination requires an additional construction, a Hashed Timelock Contract (HTLC)."*

> -- Poon & Dryja, The Bitcoin Lightning Network: Scalable Off-chain Instant Payments
  

The building block lightning defines is channel， chained by HTLCs:

```
[Alice --(10 USDT)--> Bob] ==(Bob has two channels)== [Bob --(10 USDT)--> Carol] ==(Carol has two channels)== [Carol --(10 USDT)--> David]. 

where: 

[A, USDT, B] stands for the channel built by A and B, funded by USDT.
```

Alice transfers 10 USDT to Bob inside the `[Alice, USDT, Bob]` channel, then Bob transfers 10 USDT to Carol inside the `[Bob, USDT, Carol]` channel, and finally, Carol sends 10 USDT to David in `[Carol, USDT, David]`.  

# Table of Contents
 * [Hashed TimeLock Contract](#Hashed-TimeLock-Contract) 
 * [OMNI HTLC transaction construction](#OMNI-HTLC-transaction-construction) 
	* [Commitment Transaction and Concurrency](#Commitment-Transaction-and-Concurrency)
	* [HTLC success HED1a](#HTLC-Success-HED1a )
	* [HTLC timeout HT1a](#HTLC-Timeout-HT1a)
	* [HTLC breach remedy](#HTLC-Breach-Remedy)
	* [Fee rate](#Fee-rate)
 * [Messages](#messages)
 	* [Update_add_htlc](#update_add_htlc)  
 	* [Commitment signed](#update_add_htlc)
 	* [Revock and acknowledge](#update_add_htlc)    
 * [Terminate HTLC off-chain](#Terminate-HTLC-off-chain)
 * [Unit test vectors](#unit-tests)
 * [References](#references)
  


## Hashed TimeLock Contract

An HTLC implements this procedure:

If Bob can tell Alice `R`, which is the pre-image of `Hash(R)` that someone else (Carol) in the chain shared with Bob 3 days ago in an exchange of 15 USDT in the channel `[Bob, USDT, Carol]`, then Bob will get the 15 USDT funds inside the channel `[Alice, USDT, Bob]`, otherwise Alice gets her 15 USDT back. 
 

<!-- 
![HTLC](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png "HTLC")
-->

<p align="center">
  <img width="750" alt="HTLC with full Breach Remedy transactions" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png">
</p>

Internal transactions in the above diagram are explained in [chapter 1](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-01-basic-protocol-and-Terminology.md#terminology).  

There are three outputs of a commitment transaction:
`to_rsmc(to local)`: 0. Alice3 & Bob 45,  
`to remote`: 1. Bob 40,  
`to_htlc`: 2. Alice4 & Bob 15  

`to_htlc` has three branches to handle the situations of time out, breach remedy, and htlc success. We put the payload and outputs together to construct the HTLC transaction:  

## OMNI HTLC transaction construction

### Commitment Transaction and Concurrency

On sender side: 
```
version: 2  
locktime: 0 
tx input:
	* outpoint: the vout of funding transaction.  
	* <payee's signature> <payer's signature>: to spend the funding transaction.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* to_rsmc/reference1:{value:dust, pkScript: RSMC redeem script},  
	* to_remote/reference2:{value:dust, pkScript: pubkey script},  
	* to_htlc_1/reference3:{value:dust, pkScript: offered htlc script}, 
	* to_htlc_2/reference4:{value:dust, pkScript: offered htlc script}, 
	* ...
	* to_htlc_n/reference_n:{value:dust, pkScript: offered htlc script}, 
	* change:{value:change satoshis, pkScript: the channel pubkey script }    
```
Where:  
`opReturn_encode`: the [encoded tx version( = 0 ), tx type( = 7 ), token id and array of receivers](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#payload), prefixed by "omni".  

payload in output 0:  

| size		|	Field				|	Sender Value	 	|  
| -------- 	|	-----------------------		|	-------------------	|   
| 2 bytes	|	Transaction version		|	0			|
| 2 bytes	|	Transaction type		|	7 (= Send-to-Many)	|
| 4 bytes	|	Token identifier to send	|	31 (e.g. USDT )	 	|
| 1 byte 	|	Number of outputs		|	3		 	|
| 1 byte 	|	Receiver output #		|	1 (= vout 1)		|
| 8 bytes	|	Amount to send			|	to_rsmc (e.g. 45)	|
| 1 byte 	|	Receiver output #		|	2 (= vout 2)	 	|
| 8 bytes	|	Amount to send			|	to_remote (e.g. 40)	|
| 1 byte 	|	Receiver output #		|	3 (= vout 3)	 	|
| 8 bytes	|	Amount to send			|	offered_htlc_1 (e.g. 10)|
| 1 byte 	|	Receiver output #		|	4 (= vout 4)	 	|
| 8 bytes	|	Amount to send			|	offered_htlc_2 (e.g.  5)|
| ...    	|	...     			|	...			|
| 1 byte 	|	Receiver output #		|	8 (= vout 8)	 	|
| 8 bytes	|	Amount to send			|	offered_htlc_6 (e.g. 5) | 


`op_return` has maximum 83 bytes space, so that concurrently the maximum outputs is 8, including `to_rsmc`, `to_remote`, and 6 `offered_htlc_i` outputs.  

Concurrency means a channel can accept or send HTLCs at the same time. Because of the `op-return` space limitation, if a node needs a bigger throughput, it has to build more channels with more counterparties. Channels are vertical scaling: they work in parallel and will not affect each other, and adding more channels improves the TPS linearly. This is a useful feature for liquidity providers who build thousands of channels to gain a high TPS and earn channel fees by providing liquidity to the network.  

On receiver side: 
```
version: 2  
locktime: 0 
tx input:
	* outpoint: the vout of funding transaction.  
	* <payee's signature> <payer's signature>: to spend the funding transaction.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* to_remote/reference1:{value:dust, pkScript: pubkey script},  
	* to_local/reference2:{value:dust, pkScript: RSMC redeem script},  
	* to_htlc_1/reference3:{value:dust, pkScript: received htlc script}, 
	* to_htlc_2/reference4:{value:dust, pkScript: received htlc script}, 
	* ...
	* to_htlc_n/reference_n:{value:dust, pkScript: received htlc script},  
	* change:{value:change satoshis, pkScript: the channel pubkey script }    
```

payload in output 0: 

| size		|	Field				|	Receiver Value	 	 |  
| -------- 	|	-----------------------		|	-------------------	 |   
| 2 bytes	|	Transaction version		|	0			 |
| 2 bytes	|	Transaction type		|	7 (= Send-to-Many)	 |
| 4 bytes	|	Token identifier to send	|	31 (e.g. USDT )	 	 |
| 1 byte 	|	Number of outputs		|	3		 	 |
| 1 byte 	|	Receiver output #		|	1 (= vout 1)		 |
| 8 bytes	|	Amount to send			|	to_remote (e.g. 45)	 |
| 1 byte 	|	Receiver output #		|	2 (= vout 2)	 	 |
| 8 bytes	|	Amount to send			|	to_rsmc (e.g. 40)	 |
| 1 byte 	|	Receiver output #		|	3 (= vout 3)	 	 |
| 8 bytes	|	Amount to send			|	received_htlc_1 (e.g. 10)|
| 1 byte 	|	Receiver output #		|	4 (= vout 4)	 	 |
| 8 bytes	|	Amount to send			|	received_htlc_2 (e.g.  5)|
| ...    	|	...     			|	...			 |
| 1 byte 	|	Receiver output #		|	8 (= vout 8)	 	 |
| 8 bytes	|	Amount to send			|	received_htlc_6 (e.g. 5) | 

`change`: change = satoshis in channel - dust - miner fee. By default, we set dust 546 satoshis.  

The outputs are sorted into the order by omnicore spec:  `op_return` payload is in output 0 and the receivers are ordered as in the payload definition.   

On both sides, `to_rsmc` and `to_remote` are locked by redeem script and pubkey script as in chapter 3 RSMC transaction sector. `to_htlc` on the sender side is locked by the `offered HTLC` and on the receiver side is locked by `received HTLC` as in [BOLT 3](https://github.com/lightning/bolts/blob/master/03-transactions.md#offered-htlc-outputs):  

```bat
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_NOTIF
        # To local node via HTLC-timeout transaction (timelocked).
        OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
        # To remote node with preimage.
        OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```  


The remote node can redeem the HTLC with the witness:
```
<remotehtlcsig> <preimage R>
```



On receiver side, the received HTLC is the same to `received HTLC` in BOLT 3:  
```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_IF
        # To local node via HTLC-success transaction.
        OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
        # To remote node after timeout.
        OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
        OP_CHECKSIG
    OP_ENDIF
OP_ENDIF
```

### HTLC Success HED1a 
 
HED(n)(a/b) spends the `to_htlc` output by receiver's signature and preimage R:

```
version: 2  
locktime: 0 
tx input:
	* outpoint: tx id of parent commitment transaction and the `to_htlc` output is 3.  
	* witness stack: 0 <remotehtlcsig> <localhtlcsig> <payment_preimage>.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* receiver/reference1:{value:dust, pkScript: receiver's pubkey script},  
	* change:{value:change satoshis, pkScript: the channel pubkey script }    
``` 

### HTLC Timeout HT1a 

If timed out, payer should construct HT1a to get this HTLC payment back. 

```
version: 2  
locktime: 0 
tx input:
	* outpoint: tx id of parent commitment transaction and the `to_htlc` output is 3.  
	* witness stack: 0 <remotehtlcsig> <localhtlcsig>.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* receiver/reference1:{value:dust, pkScript: RSMC redeem script},   
	* change:{value:change satoshis, pkScript: the channel pubkey script }    
```

Where in both HTLC success and timeout:  
`opReturn_encode`: the [encoded tx version( = 0 ), tx type( = 0 ), token id and amount](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#payload), prefixed by "omni".   

The amount is the amount of vout3 in the `op_return` payload of commitment transaction.  

change satoshis = channel satoshis - dust.  

RSMC redeem script is the same as in [RSMC commitment transaction](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#commitment-transaction)



### HTLC Breach Remedy 

The HTLC BR transaction can be constructed and broadcast without delay when the counter party broadcasts a revocked commitment.  

```
version: 2  
locktime: 0 
tx input:
	* outpoint: the `to_htlc` output is 3.  
	*  witness stack: <revockation key> to spend the to_htlc output.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* receiver/reference:{value:dust, pkScript: pubkey script},    
	* change:{value:change in satoshis, pkScript: the channel pubkey script }    
```
  
Where:  
`opReturn_encode`: the [encoded tx version( = 0 ), tx type( = 0 ), token id and amount](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#payload), prefixed by "omni".   


The receiver is Bob if Alice cheats.  

The amount is the number in `vout=3`( receiver `to_htlc`) of the commitment transaction `op_return` payload.  

### Fee rate

For liquidity providers, when relaying an HTLC, a channel fee could be charged in the token(not in satoshis). Generally, the fee rate is known to the network when a node announces itself and a lower fee rate will attract more  HTLC traffic.  

For a fee rate example of 50 basis points, an incoming HTLC is 15 USDT, then the outgoing HTLC is `15-15*0.5% = 14.925` USDT.  


## Messages

```
    +-------+                                                             +-------+
    |       |--(1)-------------   update_add_htlc  (40)  ---------------->|       |  
    |       |         		    for asset 1       		  	  |       |  
    |       |                                                             |       |
    |       |--(1)-------------   update_add_htlc  (40)  ---------------->|       |  
    |       |         		    for asset 2       		 	  |       |   
    |       |                                                             |       |  
    |       |<-(1)-------------   update_add_htlc  (40)  -----------------|       |
    |   A   |           	    for asset 3              		  |   B   |  
    |       |                                                             |       |
    |       |--(2)-------------  commitment signed (41)   --------------->|       |  
    |       |              	    for asset 1         		  |       |  
    |       |              	                             		  |       |  
    |       |<-(3)-----------  revock and acknowledge (42)  --------------|       |
    |       |              	    for asset 1         		  |       |  
    |       |                                                             |       |
    |       |<-(2)-------------- commitment signed (41)   ----------------|       |
    |       |          		    for asset 3      			  |       |
    |       |                                                             |       |  
    |       |--(2)-------------  commitment signed (41)   --------------->|       |  
    |       |              	    for asset 2         		  |       |  
    |       |                                                             |       | 
    |       |<-(3)-----------  revock and acknowledge (42)  --------------|       |
    |       |              	    for asset 2         		  |       |   
    |       |                                                             |       |  
    |       |--(3)-----------  revock and acknowledge (42)  ------------->|       |
    |       |              	    for asset 3         		  |       |    
    +-------+                                                             +-------+
   
```

A node has multiple asset channels, it can simultaneously send out transactions， and wait for the counterparty to respond.

## update_add_htlc 

`update_add_htlc` consists of two rounds of signing sub-transactions to forward an HTLC to the remote peer. Users should compare it to [`update_add_htlc`](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc) in `lightning-rfc`.  

Please note that the payment information is all encoded in transaction hex.  


1. type: -40 (AddHTLC)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`4 bytes`:`asset id`]: 
  * [`byte`:`IsPayInvoice`]: bool type  
  * [`float64`:`amount`]:    
  * [`sha256`:`payment_hash`]: the hash(R) used to lock the payment. 
  * [`byte_array`:`memo`]: memo to the payee.   
  * [`int32`:`cltv_expiry`]: expiry blocks.  
  * [`1366*byte`:onion_routing_packet]
	 

 
1. type: -41 (commitment signed)
2. data: 
  * [`channel_id`:`channel_id`]:
  * [`4 bytes`:`asset id`]: 
  * [`signature`:`signature`]: sender's signature.  
  * [`u16`:num_htlcs]: number of HTLCs for this `asset id`.  
  * [`num_htlcs*signature`:htlc_signature]: htlc signatures. 

 

  
1. type: -42 (revock and acknowledge)
2. data: 
  * [`channel_id`:`channel_id`]:  
  * [`4 bytes`:`asset id`]: 
  * [`32*byte`:per_commitment_secret]
  * [`point`:next_per_commitment_point]

if `asset_id = 0`, then OmniBOLT processes bitcoin lightning network, and is compatible to current bitcoin only lightning network. 

### Requirements

TO DO(Neo Carmack, Ben Fei): Complete the asset related "MUST to-do" list.  

A node MUST:

1. If a node receives a commitment transaction for a certain asset, which is not the asset(ID) that the channel(ID) is built for, then the node has to close the asset(ID) channel with the remote party.  
2. Check if the asset id in the incoming HTLC is the same in the transaction `op_return` payload for every HTLC. If not, the node has to close the asset(ID) channel.  
3. Validate the signature. If incorrect, then the node has to close the asset(ID) channel with the remote party.   


## Terminate HTLC off-chain
After Bob tells Alice the `R`, the balance of this channel shall be updated, and the current HTLC shall be terminated. OBD then creates a new commitment transactions `C3a/C3b` for this purpose.

<p align="center">
  <img width="768" alt="Commitment Transaction to Terminate an HTLC" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/C3a-Terminate-a-HTLC-both-sides.png">
</p>

Terminating an HTLC: `update_fulfill_htlc`, `update_fail_htlc`

To supply the preimage:

1. type: -130 (update_fulfill_htlc)
2. data:
  * [`channel_id`:`channel_id`]
  * [`u64`:`hop_id`]
  * [`4 bytes`:`asset id`]: 
  * [`u64`:`private_key_Alice_4`]: the private key of Alice 4, encrypted by Bob's public key when sending this message to Bob.
  * [`u64`:`private_key_Alice_5`]: the private key of Alice 5, encrypted by Bob's public key when sending this message to Bob.
  * [`32*byte`:`payment_preimage`]: the R

`private_key_Alice_4` and `private_key_Alice_5` are used in breach remedy transactions created in C2. If Bob finds out Alice is cheating, Bob can broadcast these BR transactions to get the money in temporary multi-sig addresses as punishments to Alice. The procedure is the same as RSMC in chapter 2. 

For a timed-out or route-failed HTLC:

1. type: -131 (update_fail_htlc)
2. data:
  * [`channel_id`:`channel_id`]
  * [`u64`:`id`]
  * [`u16`:`len`]
  * [`len*byte`:`reason`]

### Requirements
the same to [requirement of Removing an HTLC](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#requirements-10). 

## Unit tests
Omni HTLC transaction testing vectors will be added soon.  

## References
1. BOLT 3 transactions, [https://github.com/lightning/bolts/blob/master/03-transactions.md](https://github.com/lightning/bolts/blob/master/03-transactions.md)
2. BOLT 2 peer protocol, [https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md)
3. Omni specification for sendtomany, [https://gist.github.com/dexX7/1138fd1ea084a9db56798e9bce50d0ef](https://gist.github.com/dexX7/1138fd1ea084a9db56798e9bce50d0ef)
