# OmniBOLT #5: Atomic Swap Protocol among Channels

>*An atomic swap is a smart contract technology that enables the exchange of one cryptocurrency for another without using centralized intermediaries, such as exchanges.*

> -- https://www.investopedia.com/terms/a/atomic-swaps.asp
  
In general, atomic swaps take place between different blockchains, for exchanging tokens with no trust in each other. Channels defined in OmniBOLT can be funded by any token issued on OmniLayer. If one needs to trade his tokens, say USDT, for someone else's Bitcoins, both parties are required to acknowledge receipt of funds of USDT and BTC, within a specified timeframe using a cryptographic hash function. If one of the involved parties fails to confirm the transaction within the timeframe, then the entire transaction is voided, and funds are not exchanged, but refunded to the original account. The latter action ensures to the removal of counterparty risk.  

The standard swap procedure between channels is:

```
    Alice creats HTLC 1                                           Bob creats HTLC 2
[Alice --- X USDT---> Bob]                                    [Alice <--- Y BTC--- Bob]
    +----------------+                                           +----------------+
    |                |                                           |                |
    |     create     |                                           |                |
    |     HTLC 1     |----(1)---  tell Bob Tx 1 created   -----> |     create     |
    |                |                                           |     HTLC 2     | 
    |                |                                           |                |
    |     Locked     |<---(2)--  Acknowledge and create Tx 2 --- |     Locked     |
    |       by       |                                           |       by       |
    |     Hash(R),   |                                           |      Hash(R)   |
    | Time-Locker t1 |                                           | Time-Locker t2 |
    |       and      |                                           |       and      |
    |   Bob's Sig    |                                           |   Alice's Sig  |
    |                |                                           |     t2 < t1    |
    |                |                                           |                |
    |                |                                           |                |
    |                |                                           |                |
    |                |----(3)----   Send R to get BTC     -----> | Alice + Y BTC  |
    | Bob + X USDT   |<---(4)----   Use R to get USDT     ------ |                |
    |                |                                           |                |
    |                |                                           |                |
    |                |----(5)----     or time out,               |                |
    |                |		   refund to both sides   -----> |                |
    |                |                                           |                |
    +----------------+                                           +----------------+

    - where `HTLC 1` transfers X USDT to Bob in channel `[Alice, USDT, Bob]`, locked by `Hash(R)`, `t1` and Bob's signature. 
    - `HTLC 2` transfers Y BTC to Alice in channel `[Alice, BTC, Bob]`, locked by `Hash(R)`, `t2`(`t2 < t1`) and Alice's signature . 
    

```  
  
Apparently it is not necessary that Alice and Bob have a direct channel between them:


```
   Alice creats HTLC 1                                           Bob creats HTLC 2
[Alice --- X USDT---> David_1]                              [Bob --- Y BTC---> Carol_1]
    +----------------+                                           +----------------+  
    |     HTLC 1     |----(1)---  tell Bob Tx 1 created   -----> |                |
    |                |<---(2)--  Acknowledge and create Tx 2 --- |     HTLC 2     | 
    +----------------+                                           +----------------+
            .                                                             .  
            .                                                             .  
            .                                                             .  
            .                                                             .  
            .                                                             .     

[David_n --- X USDT---> Bob]                                [Carol_m ---Y BTC---> Alice] 
    +----------------+                                           +----------------+
    |     HTLC 1     |                                           |     HTLC 2     |
    +----------------+                                           +----------------+
 
                                                                 +----------------+
         Alice        ----(3)----Send R to Bob to get BTC -----> | Alice + Y BTC  |
                                                                 +----------------+
    +----------------+
    | Bob + X USDT   |<---(4)----   Use R to get USDT    -------        Bob
    +----------------+ 
                                          
    +----------------+                                           +----------------+
    |                |----(5)----     or time out,               |                |
    |                |		   refund to both sides   -----> |                | 
    +----------------+                                           +----------------+


```  

## Hashed TimeLock Swap Contract

Hashed TimeLock Swap Contract (HTLSC), which defines a swap, consists of two separate HTLCs with an extra specified exchange rate of tokens and time lockers.  

Simply there are 5 steps in a swap.   

**Step 0**: Alice creats `HTLC 1` to pay Bob X USDT. `HTLC 1` is locked by `Hash(R)` and time locker `t1`.  
**Step 1**: Alice notifies Bob of the details of `HTLC 1`.  
**Step 2**: Bob either acknowledges and creates `HTLC 2` to pay Alice Y BTC or ignores the message, in which case `HTLC 1` will be automatically canceled after timeframe `t1`.  `HTLC 2` is also locked by `Hash(R)`.  
**Step 3**: In this step, Alice applies `R` in her BTC channel to get her BTC fund. She has to check the redeem script `HTLC 2`, to see whether or not the `HTLC 2` is locked by `Hash(R)`. If Bob is cheating, he could use a faked `Hash(R')` in redeem script. Therefore when Alice applies `R` to unlock her fund in the BTC channel, she will get nothing but expose the secrete `R` to Bob.   
**Step 4**: After Alice exposes `R` to Bob, Bob then can use `R` to get his fund in his USDT channel.  
**Step 5**: If Alice changes her mind, and refuses to apply `R` to get her fund in BTC, then after a timeframe, funds in BTC channels and USDT channels are all refunded to the original accounts. No one in this swap will take a loss.   
 

No participant can cheat. After inputting `R` in each channel, the `HTLC 1` and `2` transform into general commitment transactions, which is the same procedure that how an [HTLC transforms to a commitment transaction](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#terminate-htlc-off-chain).  

In chain `[Alice, USDT, David_1] --> ... --> [David_n, USDT, Bob]`, Alice creates `HTLC 1` and its mirror transactions on Bob side, with time locker `t1`, which in the diagram is 3 days as an example.  

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png">
</p>

At the same time, Bob creates `HTLC 2` in the chain `[Bob, BTC, Carol_1] --> ... --> [Carol_n, BTC, Alice]` and its mirror transactions on Alice's side, sending the agreed number of BTCs to Alice. Time locker `t2` is set to be 2 days, less than `t1=3` days.  

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy-BTC-channel.png">
</p>



## `swap`

`swap` specifies the asset that one peer needs to transfer.

1. type: -80 (swap)
2. data:
  * [`channel_id`:`channel_id_from`]: Alice initiate the swap procedure by creating an HTLSC.
  * [`channel_id`:`channel_id_to`]: Bob who Alice want to trade token with.
  * [`u64`:`property_sent`]: Omni asset (id), which is sent out to Bob.
  * [`u64`:`property_receieved`]: the Omni asset (id), which is required to the counter party (Bob) 
  * [`u64`:`amount`]: ammout of the property that is sent.
  * [`float64`:`exchange_rate`]: `= property_sent/property_receieved`. For example, sending out X USDT in exchange of Y BTC, the exchange rate is X/Y.
  * [`string`:`transaction_id`]: HTLSC transaction ID, which is the one sending asset in `channel_id_from`. 
  * [`u64`:`hashed_R`]: Hash(R).     
  * [`u64`:`time_locker`]: For example, 3 days. 
 

## `swap_accepted`

`swap_accepted` specifies the asset that one peer needs to transfer.

1. type: -81 (swap_accepted)
2. data:
  * [`channel_id`:`channel_id_from`]: Alice initiate the swap procedure by creating an HTLSC.
  * [`channel_id`:`channel_id_to`]: Bob who Alice want to trade token with.
  * [`u64`:`property_sent`]: Omni asset (id), which is sent out to Bob.
  * [`u64`:`property_receieved`]: the Omni asset (id), which is required to the counter party (Bob) 
  * [`u64`:`amount`]: ammout of the `property_receieved` that is sent in `channel_id_to`.
  * [`float64`:`exchange_rate`]: `= property_sent/property_receieved`. For example, sending out X USDT in exchange of 1 BTC, the exchange rate is X/1.
  * [`string`:`transaction_id`]: HTLSC transaction ID, which is the one sending asset in `channel_id_to`. 
  * [`u64`:`hashed_R`]: Hash(R).     
  * [`u64`:`time_locker`]: For example, 2 days, which must be less than the `time_locker` in message `swap`. 

Bob in `channel_id_to` has to monitor the `transaction_id` in channel `channel_id_from`, to see whether or not the corresponding transactions, like RD, HED, HTRD, etc, have been correctly created. After he validates the transactions, he will create HTLSC according to the arguments `amount` in channel `channel_id_to` and then reply to Alice with the message `swap_accepted`.


Alice receives the message `swap_accepted`. If anything is not exactly correct, Alice will not send R to get her assets in `channel_id_to`, hence Bob is not able to get this asset in `channel_id_from`. After a timeframe, the two channels revock to their previous state.

## Remark
Atomic swap is the foundation of lots of blockchain applications. [Next chapter](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) will see some examples, which are intuitive and will help our readers to build much more complex applications for real-world businesses. 


 
 
