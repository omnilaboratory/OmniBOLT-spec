# OmniBOLT #7: Hierarchical Deterministic(HD) wallet, Non Custodial Daemon, Invoice Encoding.

* `correct me`: neocarmack@omnilab.online 

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

The life cycle of an HTLC includes 20+ transactions and generates many temporary addresses to receive, store, and send assets. Each of these transactions constructed to transfer assets from these addresses requires signature by private keys of the transaction owner. And more importantly by the design of lightning network, constructing new transaction requires to hand over the private key of previous commitment transaction, it is necessary to specify a proper key generation process, to simplify interoperability between light clients and obd nodes.  
 


### Mneminic word and hardened HD chain

The only thing for a user needed to be recorded in some safe places is the mnemonic words(12 words), which is used to generate all the pub/priv key pairs during the life cycle of channels owned by the user. Mnemonic words are generated at the first time when the user creates his wallet on an OBD.  

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


### Client SDK Implementation

[Javascript SDK:](https://github.com/omnilaboratory/DebuggingTool/tree/master/sdk). This JS SDK exposes API set that manages mnemonic codes and pub/priv key generation. Plus it helps developers to construct and sign transactions required by lightning channel.  


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

OBD(OmniBOLT daemon) runs in non-custodial mode, which means clients' seeds and keys are not generated by or stored in obd, but in their own local storage, managed by client SDK. Every time a client(wallet) needs to sign a transaction, it receives hex from the obd it connects, uses corresponding private keys to sign, and send the signature back to obd. The keys will never expose to the network. OBD, together with omnicore, validates the signature, but has no knowledge of clients' private information. So that even if the obd server has been hacked, users' transaction data are still secure: hackers can not make any transaction to steal without users' seed or key. 


Users don't need to trust any obd node, even those nodes they deploy by themselves.

OBD is designed for both liquidity providers and individual users: 


**liquidity provider**  

The business model for a liquidity providers is straight forword. They connect to as many clients as possible, and earn channel fees by providing funded channels as relays of payments. Liquidity providers shall make sure the service quality of their nodes, especially the quality of being constantly online in order to earn more money during their service. With this motivation, we can expect that these liquidity providers bring sufficient funded channels hence better connectivity to the network. 

More importantly, when you(the wallet client) go offline, these kind of nodes keep monitoring you counterparties activities, and execute penalty for punishing cheating activities if there are any, which are basic functions of a lignting node.  

We recommend users to connect to liquidity providers' nodes and regularly backup channel data to local device.


**individual user**  

If you have complete knowledge of how block chain and lightning network works, you will try to run your own full obd node. Which means you shall install omnicore/btccore, syc chain data, manage the network security and availability. 


**remark**

(07.09.2020)  

Currently being released version of obd does not specify a role of third party watch towers, because the business model for independent watch tower is not quite convincing yet. Running a full node of obd and keep it secure, being constantly online and serving tons of anonimours outsourcing need professional skills and will be costly. How much fee shall be charged, how to evaluate service quality, these problems requires community consensus. If watch tower maintainer can not make enough money to cover his operation cost, he will not be motivated. 


You can simply connect to liquidity providers' nodes, who serve as relays of payments, construct payment paths for individual clients, and earn channel fee. This part of channel fee will be significantly huge than merely monitoring cheating activities. They are already motivated to offer high quality services, including protecting clients from cheating activities, so that they can make more money during their serivce life cycle.  

For this reason, we just consider watch tower a value added service of liquidity provider, which can be almost free, comparing to huge amount of channel fee that they are able to earn. 


Another option is that users' community can crowd fund a cloud server to run a full obd node. Your community will be charged fee by the cloud computing company. And it is the only fee the community shall pay, which is cheap and transparent. This trustless obd node will also handle all your daily transactions. Because of limited number person in your community, there won't be huge amount of transactions to be managed, and less cheating will be punished, maintaining a node for your small group will not suppose to be challenging.

If you have better suggestions or other convincing points, please join our slack channel to discuss:  
https://omnibolt.slack.com/archives/CNJ0PMUMV



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

* [slip-0173: registered-human-readable-parts](https://github.com/satoshilabs/slips/blob/master/slip-0173.md#registered-human-readable-parts)
* [OLE-300: human-readable-part](https://github.com/OmniLayer/Documentation/blob/master/OLEs/ole-300.adoc#human-readable-part)
* [BOLT-11: payment encoding](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md#bolt-11-invoice-protocol-for-lightning-payments)
 
