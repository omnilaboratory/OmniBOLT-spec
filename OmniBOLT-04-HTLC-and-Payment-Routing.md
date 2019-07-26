# OmniBOLT #4: HTLC and Payment Routing

>*"A bidirectional payment channel only permits secure transfer of funds inside a channel. To be able to construct secure transfers using a network of channels across multiple hops to the nal destination requires an additional construction, a Hashed Timelock Contract (HTLC)."*

> -- Poon & Dryja, The Bitcoin Lightning Network: Scalable Off-chain Instant Payments
  

A big and common misleading explanation in chaining the channels using HTLC is that if Alice wants to pay 10 USDT to David, she can use 2 hops to reach David:

```
Alice ---(10 USDT)---> Bob ---(10 USDT)---> Carol ---(10 USDT)---> David
```

It is confusing, because there is no concept of personal account in ligtning. The only building block lighning uses is channel. So the correct hops are:

```
[Alice --(10 USDT)--> Bob] ==chain== [Bob --(10 USDT)--> Carol] ==chain== [Carol --(10 USDT)--> David]

[A B] stands for the channel built by A and B
```

Alice transfers 10 USDT to Bob inside the `[Alice Bob]` channel, then Bob transfers 10 USDT to Carol inside the `[Bob Carol]` channel, and finally Carol transfers 10 USDT to David in `[Bob Carol]`.


