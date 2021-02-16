# OmniBOLT #6: Automatic Market Maker model and DEX on Lightning Network 

* this work is not finalized yet. 

## introduction

Automatic market maker (AMM for short) model on lightning network holds significant advantage comparing to onchain AMM[1] exchanges:  

1. There is no gas fee for each swap because of the off-chain nature.  
2. Token swap is quick.  

Uniswap, Curve, and Balancer, which operate on an automated market maker (AMM) model, proof an efficient way that a dex can be. Their smart contracts hold liquidity reserves of various token pairs and traders execute swaps directly against these reserves. In this model, prices are set automatically according to a constant product `x*y=k` model, where `x` is the number of token A and `y` is the number of token B in pool. When a trader sells out `x'` token A for `y'` token B, the amount of token A in pool increases and the amount of token B decreases,  but the product remains the same: `(x+x')*(y-y')=x*y=k`. We don't bring transaction fee into calculation yet, but will add this part later in this paper. 

Liquidity prividers are Incentivized by receiving the transaction fee (0.3% in general).  

AMM on lightning network operates on the same constant product model, but the infrastructure are completely different than the onchain AMM. This paper outlines the core mechanics of how AMM on LN works. We suppose readers are familiar with both LN and AMM, so that we will omit basic concepts introductions. For Bitcoin lightning network, we refer to lnd, and for Omnilayer asset lightning network, we refore to Omnibolt.


## liquidity pool

LN already has funded channels to support multi-hop HTLC payment. Channels funded by a certain token form a logical network, where Alice is able to pay Bob even if they don't have direct channel. Hops on the payment path offer liquidity and receive a portion of fee if the payment is success.  

In AMM model, liquidity providers play a similar roll: if a swap succeed, one who deposits his tokens into the contract will get a commission according to the proportion of his contribution in the token pool.  

Naturally, funded channels in lightning network form a liquidity pool, the only difference is that the whole lightning network is a pool, every node maintains a portion of liquidity, while onchain AMM uses a contract address to collect tokens: all tokens are deposited together in one address.  
 

## peer discovery and token trades

When a OmniBOLT node is online, it has to announce itself to the network, let the neighbors to know its token type, amount of channels and liquidity. Omnibolt applies tracker network to register nodes, update status of nodes graph. Any tracker can be a rendezvous[2].

Then a tracker maintains all nodes' balances and hence it is able to calculate the token price for a trade:  

Alice plans to sell out `x'` token A at `price(B) = x/y`. So that the pool will have `(x+x')` token A and `(y-y')` token B. The price will be updated to `price(B) = (x+x')/(y-y')`. 

Step 1. Alice queries a tracker for the liquidity of both token A and token B networks. The tracker calculate the `price(B)` hence decide how much token B that Alice can get.  

Step 2. Alice create an atomic [swap](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#swap) and send it to Bob, where `exchange_rate` is the price of token B calculated above. 

Step 3. Finish the atomic swap. 



## Adding liquidity

Not all the tokens funded in channel can be liquidity reserves. Adding liquidity must according to 

 

## Removing liquidity

Removing liquidity is simple: close channel and withdraw tokens to the mainchain. Trackers will discover the inactivity of a closed channel and will update the 

## reference

1. VitalikButerin.Thex*y=kmarketmakermodel.https://ethresear.ch/t/improving-front-running-resistance-of-x-y-k-market-makers.  
2. [Connect to a tracker](https://omnilaboratory.github.io/obd/#/OBD-README?id=step-2-connect-to-a-tracker): https://omnilaboratory.github.io/obd/#/OBD-README?id=step-2-connect-to-a-tracker  
 

 

 

 
 
