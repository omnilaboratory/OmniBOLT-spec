# OmniBOLT #6: DEX, Collateral Lending Contract, online store and more Applications

Following examples use multi-stage atomic swaps for specific scenarios. The procedures shall be implemented as a peiece of program written in a turing complete language, like Javascript or Solidity, calling obd api to complete the foundamental tasks. All participants shall run the programs to check if all the transactions are valid and if the counterparties are honest.  



## Collateral Lending Contract (CLC)
Collateral Lending Contract serves this certain purpose:  

>“
You deposit something valuable as mortgage in an escrow account, and I lend money to you according to a proper LTV(Loan to Value). If you repay the loan within agreed deadline, i will return your mortgage. If you don’t, your mortgage will be mine.
”


Actually an HTLSC creates an escrow accounts for participants in a loan. We assume a normal scenario: 

>Bob wants to borrow 900 USDT from Alice, he use his 1 BTC as collateral. 

So Bob initiates a swap (HTLSC 1):

1) Bob --> Alice: swap(amount = 1 BTC, property = USDT, exchange_rate = 900, time_locker = 30 days, Hash(R1), ...).  
This creates HTLSC in channel `[Alice, BTC, Bob]`.

2) Alice --> Bob: swap_accepted(amount = 900 USDT, exchange_rate = 900, time_locker = 20 days, Hash(R1), ...).  
This creates HTLSC in channel `[Alice, USDT, Bob]`.


Meanwhile, Bob needs to create the redeem swap (HTLSC 2) to get his 1 BTC back:

1) Bob --> Alice: swap(amount = 900 USDT, property = BTC, exchange_rate = 1/900, time_locker = 60 days, Hash(R2), ...).  
This creates HTLSC in channel `[Alice, USDT, Bob]`.

1) Alice --> Bob: swap_accepted(amount = 1 BTC, exchange_rate = 1/900, time_locker = 50 days, Hash(R2), ...).  
This creates HTLSC in channel `[Alice, BTC, Bob]`.

Only when the participants accepted the two swaps, and their OBDs helps to create all the corresponding transaction required by HTLSC, Bob is able to use R1 to get his 900 USDT by HTLSC 1 in channel `[Alice, USDT, Bob]`, hence Alice gets 1 BTC as collateral from Bob. 

And after a timeframe, Bob wants to redeem his 1 BTC. He use R2 in HTLSC 2 to get his 1 BTC back by HTLSC 2 in channel `[Alice, BTC, Bob]`, hence Alice get her 900 USDT back in channel `[Alice, USDT, Bob]`.

Ofcourse, Alice can require a loan rate according to her knowledge of the price of BTC. For example she require Bob to create a swap with exchange rate 1/905. Then she will get 905 USDT when Bob redeems his BTC.

```
                                 
     [Alice,USDT,Bob]                               [Alice,BTC,Bob]
    +----------------+                           +------------------+
    |                |                           |                  |
    |                |                           |                  |
    |                |<---(1)---  swap 1  -----> |     HTLSC 1      |
    |                |                           |  Bob + 900 USDT  |
    |                |                           | Alice - 900 USDT |
    | Alice + 1 BTC  |                           |                  | 
    |  Bob - 1 BTC   |                           |                  |
    |                |                           |                  |
    |                |                           |                  |
    |                |<---(2)--   swap 2  -----> |     HTLSC 2      |
    | Bob + 1 BTC    |                           |                  |
    | Alice - 1 BTC  |                           |                  |
    |                |                           | Bob - 900 USDT   |
    |                |                           | Alice + 900 USDT |
    |                |                           |                  | 
    |                |                           |                  |
    |                |                           |                  |
    |                |<---(3)----or time out,    |                  |
    |                |	refund to both sides --> |                  |
    |                |                           |                  |
    +----------------+                           +------------------+
 
```  



## Online Pet Store

This example is one stage swap, which is quite straight farward:  

1) Alice issues smart asset "PET" on Omnilayer, for each token represents a crypto cat.  
2) Bob establishes an USDT channel and a PET channel with Alice, and funds the USDT channel.  
3) Bob creates a HTLSC to pay Alice 100 USDT for one cat.  

That's it :-)  


