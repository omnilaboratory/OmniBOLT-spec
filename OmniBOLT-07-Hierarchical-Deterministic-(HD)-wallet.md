# OmniBOLT #7: Hierarchical Deterministic(HD) wallet, Invoice Encoding.

## Motivation

An HTLC includes 20+ transactions and generates many temporary addresses to receive, store, and send assets. Each of transactions constructed to transfer assets from these addresses requires signature by private key of the owner. And more importantly by the design of lightning network, constructing new commitment transaction requires to hand over the private key of previous commitment transaction, it is necessary to specify the standard key generation will simplify interoperability between light clients and obd nodes.  

OmniBOLT does not outsource trustless watching for revoked transactions and penalty to any third party watch tower. Peers shall moinitor for themselves. 


## Mneminic word and hardened HD chain

The only thing for a user to be recorded in some safe places is the mnemonic words(12 words), which is used to generate all the pub/priv key pairs during the life cycle of channels owned by the user. Mnemonic words are generated at the first time when the user creates his wallet on an OBD.  

The 12 words code is generated on client side, so are all the child, grand child key pairs and so on. OBD has no knowledge of these keys.  


```
    +-------------------------------+  
    |       mnemonic code word:     |  
    |   brave van defence carry     |  
    |   true garbage echo media     |  
    |   make branch morning dance   |  
    +-------------------------------+  
                |
		| (1) key streching function 
		| PBKDF2 using SHA-512
		|
		|
		| (2) 2048 rounds 
		| 
    +-------------------------+           +-------------------+                                
    |      512 bit Seed       |---(3)-->  |  master key K_m   |
    |  5b5a7e5......4988c0570 |           +-------------------+
    +-------------------------+                     |
 					child keys  | 
						    | 
   				+--------+     +--------+      +--------+
   				|  K_m1  |     |  K_m2  |      |  K_m3  |	
   				+--------+     +--------+      +--------+
    				     |		    | 		   |  
 		grand child keys     |		    |   	   |  
			        +----------+    +---------+    +---------+ 
				| k,k,k,k  |    | k,k,k,k |    | k,k,k,k |	
				+----------+    +---------+    +---------+
					.....................
```  

According to BIP44, which uses hardening/private derivation on most levels:  

`m / purpose' / coin_type' / account' / change / address_index`

but not on the change and address index level.  

Non-hardened child key will [compromise the parant keys](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#security). Since The private key of a commitment transaction shall be handed over to the counterparty when new commitment transaction is created, we apply hardened key, generated up to the account level, for every temporary multi-sig address.  


## Invoice encoding

The difference between an OBD invoice and LND invoice is that obd invoice has specified asset id in human readable part. Remaining parts are the same to [specified in LND](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md#bolt-11-invoice-protocol-for-lightning-payments).

### Human readable part

The human-readable part of an obd invoice consists of three sections:

prefix: ob + SLIP-0173 registered Human Readable Part (prefix):  
obo for Omni/BTC mainnet, obto for Omni/BTC testnet, obort for Omni/BTC regtest  

asset ID: the asset id.   
amount: optional, number in that currency, followed by an optional multiplier letter. The unit encoded here is one token, for example: 1 usdt or 0.1 usdt.  

### Reference

* [slip-0173: registered-human-readable-parts](https://github.com/satoshilabs/slips/blob/master/slip-0173.md#registered-human-readable-parts)
* [OLE-300: human-readable-part](https://github.com/OmniLayer/Documentation/blob/master/OLEs/ole-300.adoc#human-readable-part)
 
