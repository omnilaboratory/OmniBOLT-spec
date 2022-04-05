# OmniBOLT #1: Basic Protocol and Terminology

## Terminology

* `OBD`: OmniBOLT Daemon.
* `channel`: A channel refers to Poon-Dryja channel in lightning network. Channel is denoted by `[Alice, USDT, Bob]`, which means Alice and Bob build a channel and fund it by USDT.
* `property`: refers to tokens issued on Omnilayer, the same to "asset".
* `RSMC`: Revocable Sequence Maturity Contract is composed to punish malicious peers, who broadcasts elder commitment transactions to get more refund than what's exactly in his balance.
* `HTLC`: Hashed Time-Lock Contract chains multiple channels for transferring tokens from one peer to another, betweem whom there is no direct channel established.
* `Commitment Transaction`: is created but not broadcast, and may be invalidated by next commitment transaction.
* `BR`: Breach Remedy transaction is used in RSMC, that if Alice cheats by broadcasting an elder commmitment transaction, BR will send all her money to Bob.
* `RD`: Revocable Delivery transaction pays out from the 2-2 P2SH transaction output, when Alice broadcast the latest legitimate commitment transaction. It sends money to Bob immediatly and will send money to Alice after relatively, say 100 blocks, from current block height. 
* `HED`:  HTLC Execution Delivery
* `HT`: HTLC Timeout
* `HBR`: HTLC Breach Remedy, the breach remedy transaction in HTLC
* `HTRD`: HTLC Timeout Revocable Delivery, the revocable delievery transaction in HTLC
* `HTBR`: HTLC Timeout Breach Remedy, punishes Alice who broadcasts the elder hash time-locked transaction during the time lock period. 
* `Atomic Swap`: Atomic swap technology enables the exchange of one cryptocurrency for another without using centralized intermediaries, such as exchanges. 
* `HTLSC`: Hashed TimeLock Swap Contract, which consists of two seperate HTLCs with extra specified exchange rate of tokens and time lockers.

## Data types

`byte_array`: an array of bytes, the first 2 bytes indicate the length of the array. the maximum length is 65535  
`32*byte`: a fixed length version of `byte_array`, where the decimal value of the first 2 bytes is 32.   
`64*byte`: a fixed length version of `byte_array`, where the decimal value of the first 2 bytes is 64.   
`float64`: 8 bytes represent a float, [8 decimal is allowed and can be recognized](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#string-to-int64)
 
