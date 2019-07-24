# OmniBOLT: In-Progress Specifications

Based on the fundamental theory of Lightning network, OmniBOLT specification describes how to enable OmniLayer assets to be transferred via ligtning channels, and how can OmniLayer assets benefit from the noval quick payment theory. According to the layer-2 protocol [BOLT (Basis of Lightning Technology) ](https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md) specification for off-chain bitcoin transfer, we propose this OmniLayer specific protocol to expand the horizons of the basic theory, to support wider perspective of assets.    

We name this new specification OmniBOLT, in order to avoid possible conflicts with BOLT. We break down our task into two major steps: 

>the first step, we run nodes that are omni assets aware: for example, users can creat channels for USDT, which is issued on Omnilayer and BTC netowrk, then they will be able to transfer USDT more quick and more cheaper. 

>the second step, we will try to be compatible with existing Lightning Nodes around the world, and we invite experts working on BOLT to work together with us. 

To be self-contained, any messages or definitions that differ from what are defined in original BOLT specification, we will include both new and old parameters to form complete messages. For those messages that are the same to BOLT, we just link to the original address where they are defined.    

# Chapters

We not only just list messages and parameters that are used in our implementation, we also complete content that explains why we do so. Most of this spec is strictly follow the rules defined in lightning white paper. The original paper may be hard to read for our programmers, so we draw some diagrams for better understanding. Hope it helps :-)

[OmniBOLT #01:]() Base Protocol

[OmniBOLT #02:](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-02-peer-protocol.md) peer-protocol, channel open

[OmniBOLT #03:](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md) RSMC and OmniLayer Transactions 

[OmniBOLT #04:](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-04-HTLC-and-Payment-Routing.md) HTLC and payment Routing

OmniBOLT #05: Improve liquidity by using USDT

# Implementation

Implementation of OmniBOLT specification can be found in this repository [LightningOnOmnilayer](https://github.com/LightningOnOmnilayer/LightningOnOmni)


# Contribution

OmniBolt specification is initiated by [Omnilayer.foundation](https://github.com/OmniLayer).
Any proposal, please submit issues in this repo.
