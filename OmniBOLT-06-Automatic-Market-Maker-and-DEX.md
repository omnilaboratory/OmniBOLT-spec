# OmniBOLT #6: Automatic Market Maker model, Liquidity Pool and DEX on Lightning Network 

* This work is in progress. 

## introduction

Automatic market maker (AMM for short) model on lightning network holds significant advantages comparing to onchain AMM[1,6] exchanges:  

1. There is no gas fee for each swap because of the off-chain nature.  
2. Token swap is quick.  
3. Liquidity is for both payment and trading.  

Uniswap[2], Curve[3], and Balancer[4], which operate on an automated market maker (AMM) model, proof an efficient way that a dex can be. Their smart contracts hold liquidity reserves of various token pairs and traders execute swaps directly against these reserves. In this model, prices are set automatically according to a constant product `x*y=k` model, where `x` is the number of token A and `y` is the number of token B in pool. When a trader sells out `x'` token A for `y'` token B, the amount of token A in pool increases and the amount of token B decreases,  but the product remains the same: `(x+x')*(y-y')=x*y=k`. We don't bring transaction fee into calculation yet, but will add this part later in this paper.  

Liquidity providers are incentivized by receiving the transaction fee (0.3% in general).  

AMM on lightning network operates on the same constant product model, but the infrastructure is completely different than the onchain AMM.  

This paper outlines the core mechanics of how AMM on LN works. We suppose readers are familiar with both LN and AMM, so that we will omit basic concepts introductions. For Bitcoin lightning network, we refer to lnd, and for smart asset lightning network, we refer to Omnibolt.


## liquidity pool

LN already has funded channels to support multi-hop HTLC payment. Channels funded by a certain token form a logical network, where Alice is able to pay Bob even if they don't have direct channel. Hops on the payment path offer liquidity and receive a portion of fee if the payment is success.  

In AMM model, liquidity providers play a similar roll: if a swap succeed, one who deposits his tokens into the contract will get a commission according to the proportion of his contribution in the token pool.  

Naturally, funded channels in lightning network form a global liquidity pool, the difference is that the whole lightning network is a pool, every node maintains a portion of liquidity, while onchain AMM uses a contract address to collect tokens: all tokens are deposited together in one address.  

We use **AMM balance** and **payment balance** to distinguish the two attributes of channel funds.  

```
for a 100 USDT channel: [Alice, 100 USDT, Bob]
    e.g. Alice's local balance: 30 USDT as AMM balance, and 25 USDT as payment balance.  
         Bob's local balance: 45 USDT as payment balance. 
``` 

For each payment, use the funds from payment balance first, and then use the funds from AMM balance if the funds in the payment balance is not enough.

After paying to Bob 30 USDT, the channel balance is like: 
```
for a 100 USDT channel: [Alice, 100 USDT, Bob]
    e.g. Alice's local balance: 25 USDT as AMM balance, and 0 USDT as payment balance.  
         Bob's local balance: 70 USDT as payment balance, 5 USDT as AMM balance.  
```

The global AMM liquidity will not change, if Bob don't manually mark the 5 USDT as payment balance.  


## trackers and peer discovery 

Trackers, in the design of OmniBOLT network, play an important role in maitaining the global constant product model.  

When an OmniBOLT node is online, it has to announce itself to the network via a tracker it connects, as the rendezvous, notifying its neighbors to updates its token type, amount of channels and liquidity reserves. Omnibolt applies tracker network to register nodes, update status of nodes graph. Any tracker can be a rendezvous[5] that allows nodes to connect.  

If you run your own tracker, it needs time to collect all the nodes information in the network, communicate with them to build the graph. The longer the nodes are online, the more complete the graph topology information is.  


## token trades

Then a tracker maintains all nodes' balances and hence it is able to calculate the token price for a trade:  

Alice plans to sell out `x'` token A for certain number of token B at fee rate `0.3%`. So that the pool will have `(x+x')` token A and `(y-y')` token B. The price after trade will be updated to `price(B) = (x+x')/(y-y')`. 

**Step 1**. Alice queries a tracker for the liquidity of both token A and token B networks. The tracker calculate the `y'` hence decide how much token B that Alice can get:

```
fee = x * 0.3%  
A pool = x + x' - 0.3%x'  
B pool = x*y/(A pool)  

y' = y - B pool  

exchange_rate = x'/y'  
``` 


**Step 2**: The tracker seeks multiple nodes, which together can provide `y'` token B, returns Alice the payment paths.  

**Step 3**. Alice create atomic [swaps](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#swap)[7] and send them to these nodes repectively, where `exchange_rate` is the price of token B at the moment of trade. 

**Step 4**. Finish the atomic swaps with all these nodes, get `y'` tokens in **payment balance**.  

```
Suppose x'/y' = 2, Alice sells out 10 A:  

          |--atomic swap---> Bob: 2 A for 4 B
          |
Alice ----|--atomic swap---> David: 4.5 A for 9 B 
          |
          |--atomic swap---> Carol: 3.5 A for 7 B

```

Fees are allocated to all nodes on the paths, and are still in **AMM balance** of each node:  

```
A pool = x + x'
B pool = x*y/(x + x' - 0.3%x')

new invariant = (A pool) * (B pool) = (x + x')*x*y/(x + x' - 0.3%x')
```

The invariant increases after every trade. 

**Like the characteristics of lightning network, AMM is suitable for small transactions.**  




## fee structure

Token A to token B trads: 0.3% fee paid in A. This fee will be apportioned among the nodes in [atomic swap payment path](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md)[7], which means although the pool has many many liquidity providers, for each transaction, not all the providers will get paid.  

Trackers balance the workload of the whole network: If one node is busy handling HTLC, another node will be selected to be a hop and earns the swap fee. 

If a liquidity provider think the tracker he connects is not fair enough, he may choose another one. Each tracker shall publish its path/node selection policy.  


## adding liquidity

Adding liquidity to lightning network is simple: just open a channel with your counterparty and fund it. The lightning network discovers new channels and updates the network graph so that your channels will contribute to the global payment liquidity.  

But adding liquidity to AMM pool is different. Not all the tokens funded in channels can be liquidity reserves. Adding liquidity must use the current exchange rate at the moment of deposit[6]. The exchange rate is calculate from global `x/y`, feed by the tracker:  

**Step 1**: Suppose Alice fund her BTC channel `x'`, then she should fund her USDT channel `y'= y(x+x')/x  - y`.  

**Step 2**: If she fund more or less USDT, the extra tokens, USDT or BTC, will be marked as payment liquidity reserve, which is not a "donation" as Uniswap designs.  

**Step 3**: Alice sync her funding to her trackers, which built connections with her and record Alice's balance of BTC and USDT.  

The first AMM liquidity provider could deposite any amount of BTC and USDT, the tracker will calculate how much BTC or USDT will be marked as AMM liquidity according to price feed from oracle.  



Adding liquidity costs BTC gas fee.  

## removing liquidity

There are two ways to remove liquidity:  
1. close channel and withdraw tokens to the mainchain.  
2. pay tokens to someone else: the token paid will be in another investor's balance. But if the investor chooses to keep these token in pool, the global liquidity will not change.

Trackers calculate the remaining tokens in the liquidity reserve, and the extra tokens will be marked payment liquidity reserve, according to exchange rate at the moment of closing channel:  

Suppose Alice closes her channel of `x'` BTCs, then the BTC-USDT pair will have redundant USDT in pool. Its tracker randomly selects some USDT channels in the graph, marks a portion of channel fund, `y'` to be payment liquidity reserve,to make sure the global price `x/y = BTC/USDT` unchanged, where `y' = y - y(x-x')/x`.  

There is no protocol fee taken during closing a channel. Only gas fee in BTC occurs.  


## oracle
To feed the real time external price for trading. To be done.  


## reference

1. Vitalik Buterin.The x*y=k market maker model. https://ethresear.ch/t/improving-front-running-resistance-of-x-y-k-market-makers.  
2. Uniswap. https://uniswap.org/
3. Curve. https://www.curve.fi/
4. Balancer. https://balancer.finance/
5. [Connect to a tracker](https://omnilaboratory.github.io/obd/#/OBD-README?id=step-2-connect-to-a-tracker): https://omnilaboratory.github.io/obd/#/OBD-README?id=step-2-connect-to-a-tracker  
6. Uniswap whitepaper. https://uniswap.org/whitepaper.pdf
7. Atomic swap. https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#swap
 

 

 

 
 
