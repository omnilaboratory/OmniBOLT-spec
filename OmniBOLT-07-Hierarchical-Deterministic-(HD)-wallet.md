# OmniBOLT #7: Hierarchical Deterministic(HD) wallet, Non Custodial Daemon, Invoice Encoding.

* `correct me`: neocarmack@omnilab.online  or join our slack channel to discuss:  https://omnibolt.slack.com/archives/CNJ0PMUMV

# Table of Contents
 * [Hierarchical Deterministic(HD) wallet](#hierarchical-deterministichd-wallet)
 	* [Motivation](#motivation)
 	* [Mneminic word and hardened HD chain](#mneminic-word-and-hardened-hd-chain) 
	* [Client SDK Implementation](#client-sdk-implementation)
 * [Non Custodial OmniBOLT Daemon](#non-custodial-omnibolt-daemon)
 

 * [Invoice encoding](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md#invoice-encoding)
 	* [Human readable part](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md#human-readable-part)
 * [Reference](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md#reference)
 

## Hierarchical Deterministic(HD) wallet

### Motivation

The life cycle of an HTLC includes 20+ transactions and generates many temporary addresses to receive, store, and send assets. Each of these transactions constructed to transfer assets from these addresses requires the signature by private keys of the transaction owner. And more importantly, by the design of the lightning network, constructing a new transaction requires handing over the private key of the previous commitment transaction, it is necessary to specify a proper key generation process, to simplify interoperability between light clients and obd nodes.  
 


### Mneminic word and hardened HD chain

The only thing for a user needed to be recorded in some safe places is the mnemonic words(12 words), which are used to generate all the pub/priv key pairs during the life cycle of channels owned by the user. Mnemonic words are generated for the first time when the user creates his wallet on an OBD.  

The 12 words code is generated on the client-side, and so are all the child, grandchild key pairs, and so on. OBD in non-custodial mode has no access to these keys.  


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

Non-hardened child keys will [compromise the parent keys](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#security). Since The private key of a commitment transaction shall be handed over to the counterparty when a new commitment transaction is created, we apply a hardened key, generated up to the account level, for every temporary multi-sig address.  


### Client SDK Implementation

[Javascript SDK:](https://github.com/omnilaboratory/DebuggingTool/tree/master/sdk). This JS SDK exposes an API set that manages mnemonic codes and pub/priv key generation. Plus it helps developers to construct and sign transactions required by the lightning channels.  


## Non Custodial OmniBOLT Daemon
 

```
					  ______                             
				      ___/-     \____
				     /     -   /     \___-------- | tracker |:records the quality of nodes' service.  
				  __|         /            \-------------------| tracker |  
				 |       obd network        \   
			  -------|___________________________|-------   
                         /                                           \   
                        /                                             \   
                | reomote obd |                                  | remote obd |   
                       |		                              |    
                       |		                              |    
        -----------------------------                  -------------------------------  
        |              |	    |   	       |              |              |  
        |              |	    |   	       |              |              |  
    +---------+    +--------+    +--------+         +--------+    +--------+     +--------+  
    | desktop |    | mobile |    | mobile |	    | mobile |    | desktop|     | mobile |  
    | wallet  |    | wallet |    | wallet |	    | wallet |    | wallet |     | wallet |  
    +---------+    +--------+    +--------+         +--------+    +--------+     +--------+  
    | obd SDK |                                                                  | obd SDK |  
    |  seeds  |                       ....................                       |  seeds  |  
    |  keys   |                                                                  |  keys   |  

```

If OBD(OmniBOLT daemon) runs in non-custodial mode, then the clients' seeds and keys are not generated by or stored in OBD, but in their own local storage managed by the client SDK. When a client(wallet) needs to sign a transaction, it receives the transaction's hex from the obd it connects, uses corresponding private keys to sign, and sends the signature back to the OBD node to verify. Even if the OBD server has been compromised, users' transaction data are still secure: hackers can not make any transaction to steal without users' keys.  


Users don't need to trust any obd node, even those nodes they deploy by themselves.



## Exclusive mode
The exclusive mode works in the same way as lnd. Every user MUST run his private obd node, which manages and stores all his keys. For application integrators, obd exposes GRPC API to interact with and the tech stack is the same as lnd.   

## Invoice encoding

The difference between an OBD invoice and LND invoice is that obd invoice has specified asset id in human readable part. Remaining parts are the same to [specified in LND](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md#bolt-11-invoice-protocol-for-lightning-payments).


### Human readable part

The human-readable part of an obd invoice consists of three sections:

prefix: ob + [SLIP-0173 registered Human Readable Part (prefix)](https://github.com/satoshilabs/slips/blob/master/slip-0173.md#registered-human-readable-parts):  
<!-- obo for Omni/BTC mainnet, obto for Omni/BTC testnet, obort for Omni/BTC regtest   -->

|          |  Omni/BTC Mainnet  |  Omni/BTC Testnet  |  Omni/BTC Regtest  |
|----------|  ----------------  |  ----------------  |  ----------------  |
|  prefix  |       obo 		| 	obto 	     |       obort 	  |
 

asset ID: the omni asset id.   

amount: optional, number in that currency, followed by an optional multiplier letter. The unit encoded here is one token, for example: 1 usdt or 0.1 usdt.  

## Reference

* [BIP-0044: Multi-Account Hierarchy for Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
* [slip-0173: registered-human-readable-parts](https://github.com/satoshilabs/slips/blob/master/slip-0173.md#registered-human-readable-parts)
* [OLE-300: human-readable-part](https://github.com/OmniLayer/Documentation/blob/master/OLEs/ole-300.adoc#human-readable-part)
* [BOLT-11: payment encoding](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md#bolt-11-invoice-protocol-for-lightning-payments)
* [BIP-0032: security](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#security)
 
