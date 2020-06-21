# OmniBOLT: Facilitates smart assets lightning transactions
[![](https://img.shields.io/badge/license-MIT-brightgreen)](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/LICENSE) [![](https://img.shields.io/badge/made%20by-Omni%20Foundation-blue)]() [![](https://img.shields.io/badge/project-OmniBOLT%20Daemon-orange)](https://github.com/omnilaboratory/obd)

* `Contact`: Neo Carmack(neocarmack@omnilab.online), Ben Fei(benfei@omnilab.online)

OmniBOLT is a lightning network specification, enabling faster and lower cost transactions of smart crypto assets, providing more flexible contracts for upper layer decentralized finance applications. [OmniBOLT daemon](https://github.com/omnilaboratory/obd) is a golang implementation of this specification, an open source, off-chain decentralized platform, build upon BTC/OmniLayer network, implements basic HTLC payment, atomic swap of multi-currencies, and more off-chain contracts on the network of smart [assets enabled lightning channels](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-02-peer-protocol.md#omni-address).  
 
 
## Advantages

Not only BTC circulation is supported as current implementation of [BOLT](https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md), but also:  
 
* Instant payment of smart assets issued on OmniLayer and Etherium(soon future). 
* Cross channel atomic swap for various crypto assets.
* Decentralized exchange on top of stable coin enabled lightning channels. 
* Collateral Lending Contract and more flexible contracts for various DeFi scenarios based on atomic swap, without any extra cost of transaction fee or any intermediary;  
	* Interested readers shall directly go to [chapter 6: DEX, Collateral Lending Contract, online store ...](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) to seek more examples.
 

## Why OmniBOLT

Decentralized finance industry requires a much more flexible, extensible and cheaper smart assets circulation solution to solve the main chain scalability problem. Lightning network is a solid technology to this problem, but currently only BTC(and some of its forks) is supported. 

Meanwhile, Omnilayer is an onchain smart assets issuance technology, which is proven secure and stable. So that constructing lightning channels on top of it automatically acquires the ability of issuing assets, temper resistant, and on-chain settlement, with which OmniBOLT greatly widens the perspective of the original lightning theory and technology.  
 

## How it works:

<p align="center">
  <img width="500" alt="OmniBOLT-Protocol-Suite" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/OmniBOLT-Protocol-Suite.png">
</p>

1. On-chain protocol is Omnilayer, which is the issuance and settlement level;  
2. OmniBOLT 2, 3, 4 form the main network protocols;   
3. Applications level consists of contracts for various applications; 

OmniBOLT itself does not issue tokens. All tokens are issued on Omnilayer, and enter the OmniBOLT network through P2(W)SH backed channels, being locked on the main chain, and can be redeemed on the Omnilayer main chain at any time.  
 

## Chapters and Protocol Suite

We not only just list messages and arguments that are used in our implementation, but also complete content that explains why we do so. Most of this spec is strictly follow the rules/logics defined in the lightning white paper. The original paper may be hard to read for our programmers, so we draw some diagrams for better understanding. Hope it helps :-)

[OmniBOLT #01](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-01-basic-protocol-and-Terminology.md): Base Protocol and Terminology

[OmniBOLT #02:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-02-peer-protocol.md) peer-protocol, Poon-Dryja channel open

[OmniBOLT #03:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md) RSMC and OmniLayer Transactions 

[OmniBOLT #04:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-04-HTLC-and-Payment-Routing.md) HTLC and payment Routing

[OmniBOLT #05:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md) Atomic Swap Protocol among Channels

[OmniBOLT #06:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) DEX, Collateral Lending Contract, online store and more applications

OmniBOLT #07: Construct transactions on OmniLayer



## Technology Guide
[OmniBOLT Technology Guide Part I](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/docs/OmniBOLT-Technology-guide-part-I-2020-05-01_en.pdf) offers quick understanding of the rationale, concepts, architecture of OmniBOLT.  

## Implementation and API for wallet

Implementation of OmniBOLT specification can be found in this repository [LightningOnOmnilayer](https://github.com/omnilaboratory/obd), as well as the API online documentation can be found [here](https://api.omnilab.online).

Javascript API: [here](https://github.com/omnilaboratory/DebuggingTool/blob/master/js/obdapi.js).

GUI debugging tool: [here](https://github.com/omnilaboratory/DebuggingTool).
 


## Contribution

OmniBolt specification is initiated by [Omnilayer.foundation](https://github.com/OmniLayer).
Any proposal, please submit issues in this repo.
