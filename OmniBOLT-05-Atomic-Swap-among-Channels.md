# OmniBOLT #5: Atomic Swap Protocol among Channels

>*An atomic swap is a smart contract technology that enables the exchange of one cryptocurrency for another without using centralized intermediaries, such as exchanges.*

> -- https://www.investopedia.com/terms/a/atomic-swaps.asp
  
In general, atomic swaps take place between different block chains, for exchanging tokens with no trust of each other. Channels defined in OmniBOLT can be funded by any token issued on OmniLayer. If one needs to trade his tokens, say USDT, for some one else's Bitcoins, both parties are required to acknowledge receipt of funds of USDT and BTC, within a specified timeframe using a cryptographic hash function. If one of the involved parties fails to confirm the transaction within the timeframe, then the entire transaction is voided, and funds are not exchanged, but refunded to original account. The latter action ensures to remove counterparty risk.  

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
    | Bob + X USDT   |<---(4)----   Send R to get USDT    ------ |                |
    |                |                                           |                |
    |                |                                           |                |
    |                |----(5)----     or time out,               |                |
    |                |		   refund to both sides   -----> |                |
    |                |                                           |                |
    +----------------+                                           +----------------+

    - where HTLC 1 transfers X USDT to Bob in channel `[Alice, USDT, Bob]`, locked by Hash(R), t1 and Bob's signature. 
    - HTLC 2 transfers Y BTC to Alice in channel `[Alice, BTC, Bob]`, locked by Hash(R), t2(`t2 < t1`) and Alice's signature . 
    

```  
  
Apparently it is not necessary that Alice and Bob have direct channels between them:


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

Hashed TimeLock Swap Contract (HTLSC), which defines a swap, consists of two seperate HTLCs with extra specified exchange rate of tokens and time lockers.  

Simply there are 5 steps in a swap. In step 3, Alice sends R to Bob, hence she can unlock HTLC 2 to get her Y BTC in the channel `[Alice, BTC, Bob]`. Therefor Bob knows R, and use R to unlock his X USDT in the channel `[Alice, USDT, Bob]`.  

No participant is able to cheat. After inputting R in each channel, the transaction 1 and 2 turn into general commitment transactions, which is the same procedure that how an [HTLC transforms to a commitment transaction](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-05-Atomic-Swap-among-Channels.md#terminate-htlc-off-chain).

In channel `[Alice, USDT, Bob]`, Alice create an HTLC and its mirror transactions on Bob side, with time locker `t1`, which in the diagram is 3 days as an example.

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy.png">
</p>

At the same time, Bob creates another HTLC in the channle `[Alice, BTC, Bob]` and its mirror transactions on Alice side, sending the agreed number of BTCs to Alice. Time locker `t2` is set to be 2 days, less than `t1=3` days.

<p align="center">
  <img width="768" alt="HTLC with full Breach Remedy transactions" src="https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/imgs/HTLC-diagram-with-Breach-Remedy-BTC-channel.png">
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

Bob in `channel_id_to` has to monitor the `transaction_id` in channel `channel_id_from`, to see whether or not the corresponding transactions, like RD, HED, HTRD, etc, have been correctly created. After he validates the transactions, he will create HTLSC according to the arguments `amount` in channel `channel_id_to`, and then reply Alice with message `swap_accepted`.


Alice receives the message `swap_accepted`. If anything is not exactly correct, Alice will not send R to get her assets in `channel_id_to`, hence Bob is not able to get this asset in `channel_id_from`. After a timeframe, the two channels revock to their previous state.

## Remark
Atomic swap is a foundation of lots of blockchain applications. [Next chapter](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-06-Mortgage-Loan-Contracts-for-Crypto-Assets.md) will see some examples, which are intuitive and will help our readers to build much more complex applications for real world businesses. 


 
 
