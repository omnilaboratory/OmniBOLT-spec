# OmniBOLT: Facilitates smart assets lightning transactions

[![](https://img.shields.io/badge/license-MIT-brightgreen)](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/LICENSE) [![](https://img.shields.io/badge/made%20by-Omni%20Foundation-blue)]() [![](https://img.shields.io/badge/project-OmniBOLT%20Daemon-orange)](https://github.com/omnilaboratory/obd)

* `Contact`: Neo Carmack(neocarmack@omnilab.online), Ben Fei(benfei@omnilab.online)

Based on the fundamental theory of the Lightning network, the OmniBOLT specification describes how to enable OmniLayer assets to be circulated via lightning channels, and how can smart assets benefit from this novel quick payment theory.

In addition, OmniBOLT provides more flexible contracts for upper-layer decentralized applications. [OmniBOLT daemon](https://github.com/omnilaboratory/obd) is a golang implementation of this specification, an open source, off-chain decentralized platform, build upon BTC/OmniLayer network, implements basic HTLC payment, atomic swap of multi-currencies, and more off-chain contracts on the network of smart [assets enabled lightning channels](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-02-peer-protocol.md#omni-address).

 
## Features  
 
* Instant payment of smart assets issued on OmniLayer and Etherium(soon future). 
* Cross channel atomic swap for various crypto assets.
* Decentralized exchange on top of stable coin enabled lightning channels. 
<!--
* Collateral Lending Contract and more flexible contracts for various DeFi scenarios based on atomic swap, without any extra cost of transaction fee or any intermediary;  
	* Interested readers shall directly go to [chapter 6: DEX, Collateral Lending Contract, online store ...](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) to seek more examples.
 -->
* Automatic market maker and liquidity pool for DEX.
 
## Why OmniBOLT

The decentralized finance industry requires a much more flexible, extensible, and cheaper smart assets circulation solution to solve the main chain scalability problem. The Lightning Network is a solid technology for this problem. According to the layer-2 protocol [BOLT (Basis of Lightning Technology)](https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md) specification for off-chain bitcoin transfer, we need a protocol to support a wider range of assets for upper-layer applications: payment, game, finance, or scenarios that need stablecoins.  

Meanwhile, Omnilayer is an on-chain smart assets issuance technology, which is proven secure and stable. Constructing lightning channels on top of it automatically acquires the ability to issue assets, tamper resistance, and on-chain settlement. This is where OmniBOLT is built upon.

## How it works:

<p align="center">
  <img width="500" alt="OmniBOLT-Protocol-Suite" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/OmniBOLT-Protocol-Suite.png">
</p>

1. On-chain protocol is Omnilayer, which is the issuance and settlement level;  
2. OmniBOLT 2, 3, and 4 form the main network protocols;   
3. Applications level consists of contracts for various applications; 

OmniBOLT itself does not issue tokens. All tokens are issued on Omnilayer, and enter the OmniBOLT network through P2(W)SH backed channels, being locked on the main chain, and can be redeemed on the Omnilayer main chain at any time.  
 

## Chapters and Protocol Suite

We not only just list messages and arguments that are used in our implementation, but also complete content that explains why we do so. Most of this spec strictly follows the rules/logic defined in the lightning white paper. The original paper may be hard to read for our programmers, so we draw some diagrams for better understanding. Hope it helps :-)

[OmniBOLT #01](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-01-basic-protocol-and-Terminology.md): Base Protocol and Terminology

[OmniBOLT #02:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-02-peer-protocol.md) peer-protocol, Poon-Dryja channel open

[OmniBOLT #03:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md) RSMC and OmniLayer Transactions 

[OmniBOLT #04:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-04-HTLC-and-Payment-Routing.md) HTLC and payment Routing

[OmniBOLT #05:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md) Atomic Swap Protocol among Channels

<!--
[OmniBOLT #06:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) DEX, Collateral Lending Contract, online store and more applications
-->
[OmniBOLT #06:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Automatic-Market-Maker-and-DEX.md) Automatic Market Maker, Liquidity Pool and DEX

[OmniBOLT #07:](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-07-Hierarchical-Deterministic-(HD)-wallet.md)  Hierarchical Deterministic (HD) wallet, Invoice Encoding



## Technology Guide
[OmniBOLT Technology Guide Part I](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/docs/OmniBOLT-Technology-guide-part-I-2020-05-01_en.pdf) offers quick understanding of the rationale, concepts, architecture of OmniBOLT.  

## Implementation and API for client applications

Implementation of OmniBOLT specification can be found in this repository [OmniBOLT Daemon](https://github.com/omnilaboratory/obd).  
 
## Quick Start

It is recommended to start with a graphic tool to play with OmniBOLT: [https://github.com/omnilaboratory/obd#quick-start](https://github.com/omnilaboratory/obd#quick-start)


## Contribution

OmniBolt specification is incubated by [Omnilayer.foundation](https://github.com/OmniLayer).
Any proposal, please submit issues in this repo.
