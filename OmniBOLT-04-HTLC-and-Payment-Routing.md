# OmniBOLT #4: HTLC and Payment Routing

>*"A bidirectional payment channel only permits secure transfer of funds inside a channel. To be able to construct secure transfers using a network of channels across multiple hops to the final destination requires an additional construction, a Hashed Timelock Contract (HTLC)."*

> -- Poon & Dryja, The Bitcoin Lightning Network: Scalable Off-chain Instant Payments
  

A big and common misleading explanation in chaining the channels using HTLC is that if Alice wants to pay 10 USDT to David, she can use 2 hops to reach David:

```
Alice ---(10 USDT)---> Bob ---(10 USDT)---> Carol ---(10 USDT)---> David
```

It is confusing, because there is no concept of personal account in ligtning. The only building block lightning uses is channel. So the correct hops are:

```
[Alice --(10 USDT)--> Bob] ==(Bob has two channels)== [Bob --(10 USDT)--> Carol] ==(Carol has two channels)== [Carol --(10 USDT)--> David]

[A B] stands for the channel built by A and B
```

Alice transfers 10 USDT to Bob inside the `[Alice Bob]` channel, then Bob transfers 10 USDT to Carol inside the `[Bob Carol]` channel, and finally Carol 10 USDT to David in `[Bob Carol]`.

## Hashed TimeLock Contract

An HTLC implements this procedure:

If Bob can tell Alice R, which has hash image Hash(R) that Alice shared with Bob 3 days ago, together with Alice's commitment transaction whoch pays Bob 5 USDT inside their channel `[Alice Bob]`, then Bob will get the 5 USDT fund, otherwise Alice gets her 10 USDT back. So the script is simple:

```
OP_IF
    OP_HASH256 <HASH 256(R)> OP_EQUALVERIFY
    2 <Alice2> <Bob2> OP_CHECKMULTISIG
OP_ELSE
    2 <Alice1> <Bob1> OP_CHECKMULTISIG
OP_ENDIF
```

Equipted with HTLC, the internal transfer of fund `[Alice --(10 USDT in HTLC)--> Bob]` is then an extra unbroadcasted output from funding transaction embeded together with [RD1a/BR1a](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/OmniBOLT-03-RSMC-and-OmniLayer-Transactions.md#the-commitment_tx-and-commitment_tx_signed-message).

![HTLC](https://github.com/LightningOnOmnilayer/Omni-BOLT-spec/blob/master/imgs/HTLC-diagram.png "HTLC")

**HED1a**: HTLC Execution Delivery
**HT1a**: HTLC Timeout
**HTRD1a**: Timeout Revocable Delivery


