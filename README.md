# OmniBOLT: In-Progress Specifications

Based on the fundamental theory of Lightning network, Omni-BOLT specification describes how to enable OmniLayer assets to be transferred via ligtning channels, according to the [BOLT (Basis of Lightning Technology) ](https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md) specification, which is a layer-2 protocol for off-chain bitcoin transfer by mutual cooperation.

We name this new specification OmniBOLT, in order to avoid possible conflicts with BOLT. We break down our task into two major steps: 

>the first step, we run nodes that are omni assets aware: for example, users can creat channels for USDT, which is issued on Omnilayer and BTC netowrk, then they will be able to transfer USDT more quick and more cheaper. 

>the second step, we will try to be compatible with existing Lightning Nodes around the world, if experts working on BOLT can work together with us. 

To be self-contained, any messages or definitions that differ from what are defined in original BOLT specification, we will include both new and old parameters to form complete messages. For those messages that are the same to BOLT, we just link to the original address where they are defined.    

# Chapters

[OmniBOLT #01:]() Base Protocol

[OmniBOLT #02:](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/Omni-BOLT-02-peer-protocol.md) peer-protocol

[OmniBOLT #03:](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/Omni-BOLT-03-OmniLayer%20Transaction%20and%20Script%20Formats.md) OmniLayer Transaction and Script Formats 

# Implementation

Implementation of Omni-BOLT specification can be found in this repository [LightningOnOmnilayer](https://github.com/LightningOnOmnilayer/LightningOnOmni)


# Contribution

OmniBolt specification is initiated by [Omnilayer.foundation](https://github.com/OmniLayer).
Any proposal, please raise an issue in this repo.
