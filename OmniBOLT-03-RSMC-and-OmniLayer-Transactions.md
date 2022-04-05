# OmniBOLT #3: RSMC and OmniLayer Transaction  

Sometimes we use "money" instead of Omni assets for illustration purpose. Readers just image that what we fund, transfer or trade is USDT, an important asset issued on Omnilayer.

From this chapter on, our context is Omnilayer, not only bitcoin any more.

RMSC pays tokens to the counterparty directly without any locker. The receiver is only passively receiving, and does not need any unlocking or confirming actions. This protocol can be applied to the following classic scenarios:  

1. peer-to-peer instant message system      
2. Send red packets in social networks  
3. air drop

# Table of Contents
 * [Omnilayer Class C Transaction](#Omnilayer-Class-C-Transaction)
 	* [op return](#op_return)
 	* [payload](#payload)  
 	* [string to int64](#string-to-int64)
 * [BTC_funding and asset_funding](#The-btc_funding_created-btc_funding_signed-asset_funding_created-and-asset_funding_signed-Messages)
 * [Commitment_tx, revoke and acknowledge](#The-commitment_tx-and-revoke-and-acknowledge-Message)
	 * [Diagram and messages](#diagram-and-messages)  
	 * [Omni RSMC transaction construction](#OMNI-RSMC-transaction-construction)
	 * [Funding and to_remote transaction](#Funding-and-to-remote-transactions)
	 * [Commitment transaction](#Commitment-Transaction)
	 * [Message data](#message-data)
 * [Cheat and Punishment](#Cheat-and-Punishment)
 * [Close_channel](#The-close_channel-Message )  
 * [Unit test vectors](#unit-tests)
 * [References](#references)
  
## Omnilayer Class C Transaction 

Omnibolt defines and transfers tokens in lightning network subject to the [omni specification version 0.7](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc). Specifically, [the class C transaction](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#65-class-c-transactions-op_return-method) is mainly used in the omnibolt. The golang implementation is under the [`omnicore`](https://github.com/omnilaboratory/obd/tree/master/omnicore) directory in the obd project. An example of constructing simple send raw transaction([the class c tx in omni protocol](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#65-class-c-transactions-op_return-method)) offline is [here](https://github.com/omnilaboratory/obd/tree/master/omnicore#construct-simple-send-transaction).  

Validators (e.g the counterparty) must use omnicore(integrated by tracker) full nodes to verify the correctness of received transactions.  

### op_return 
By definition, [the class C transaction](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#65-class-c-transactions-op_return-method) [embeds a payload in an OP_RETURN output](https://github.com/omnilaboratory/obd/blob/master/omnicore/CreateRawOmniTransactionOpreturn.go#L170-L208), prefixed with a transaction marker "omni", to a raw bitcoin transaction: 
```
vin: [...]  
out: [
    {op_return_out:{value:0, pkScript:opReturn_encode}},
    {normal_out:{value:546, pkScript: pkHash}}
]
```
Where `op_return_out` has to be less than 83 bytes and is encoded as: 
```go
  	/* 
	 * Embeds a payload in an OP_RETURN output, prefixed with a transaction marker "omni".
  	 *
  	 * The request is rejected, if the size of the payload with marker is larger than
   	 * the allowed data carrier size ("-datacarriersize=n").
   	 */
  
  	//"omni"
  	vchOmBytes := []byte{0x6f, 0x6d, 0x6e, 0x69}
  
	s := make([][]byte, 2)

	s[0] = vchOmBytes
	s[1] = payload_bytes

	sep := []byte("")
	omni_payload := bytes.Join(s, sep)

	if (uint(len(omni_payload))) > nMaxDatacarrierBytes {
		fmt.Println("omni_payload is too long, only 83 bytes are allowed")
		return nil
	}
	 
	//add op code, which is OP_RETURN in this case, and data .
	op_return_data := make([][]byte, 2)
	op_return_data[0] = []byte{OP_RETURN}
	op_return_data[1] = addOpCodeData(omni_payload)
	op_return := bytes.Join(op_return_data, sep)

	return op_return

```

The [byte array payload_bytes](https://github.com/omnilaboratory/obd/blob/master/omnicore/rpcpayload.go#L94-L114) defines the property ID and the amount to be paid:   

### payload

Transaction type 0 is used in:  
* funding asset to a channel, 
* pay the `to_remote` output and in RSMC and HTLC,  
* construct breach remedy transactions,  

 

|   size   |   Field               |      type          |  Example           |   
| -------- |-----------------------|  ----------------  |  ----------------  |   
|  2 bytes |  Transaction version  |  [Transaction version](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#field-transaction-version) |   0   |   
|  2 bytes |  Transaction type     |  [Transaction type](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#field-transaction-type) 	    |   0   |   
|  4 bytes |  Currency identifier  |  [Currency identifier](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#field-currency-identifier) | 1(omni) | 
|  8 bytes |  Amount to transfer   |  [Amount](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#field-number-of-coins) 	            |   100 |   


And the [transaction type 7](https://gist.github.com/dexX7/1138fd1ea084a9db56798e9bce50d0ef) is used in commitment trsactions that constructs 2 or 3 outputs in RSMC and HTLC respectively:  
 
| size		|	Field				|		Type		|	Example		 	|  
| -------- 	|	-----------------------		|  	-------------------	|  -------------------	 	|   
| 2 bytes	|	Transaction version		|	Transaction version	|	0			|
| 2 bytes	|	Transaction type		|	Transaction type	|	7 (= Send-to-Many)	|
| 4 bytes	|	Token identifier to send	|	Token identifier	|	31 (= USDT )	 	|
| 1 byte 	|	Number of outputs		|	Integer-one byte	|	3		 	|
| 1 byte 	|	Receiver output #		|	Integer-one byte	|	1 (= vout 1)		|
| 8 bytes	|	Amount to send			|	Number of tokens	|	20 0000 0000 (= 20.0)	|
| 1 byte 	|	Receiver output #		|	Integer-one byte	|	2 (= vout 2)	 	|
| 8 bytes	|	Amount to send			|	Number of tokens	|	15 0000 0000 (= 15.0)	|
| 1 byte 	|	Receiver output #		|	Integer-one byte	|	4 (= vout 4)	 	|
| 8 bytes	|	Amount to send			|	Number of tokens	|	30 0000 0000 (= 30.0)	|
 
The above may changes according to the update of omnicore.


The transaction version used by OmniBOLT are both 0. 

If on little endian systems, use `SwapByteOrder16(...)`, `SwapByteOrder32(...)`, `SwapByteOrder64(...)` to swap the bit order: 
```go
func SwapByteOrder16(us uint16) uint16 {
	if IsLittleEndian() {
		us = (us >> 8) |
			(us << 8)
	}
	return us
}

func SwapByteOrder32(ui uint32) uint32 {
	if IsLittleEndian() {
		ui = (ui >> 24) |
			((ui << 8) & 0x00FF0000) |
			((ui >> 8) & 0x0000FF00) |
			(ui << 24)
	}
	return ui
}

func SwapByteOrder64(ull uint64) uint64 {
	if IsLittleEndian() {
		ull = (ull >> 56) |
			((ull << 40) & 0x00FF000000000000) |
			((ull << 24) & 0x0000FF0000000000) |
			((ull << 8) & 0x000000FF00000000) |
			((ull >> 8) & 0x00000000FF000000) |
			((ull >> 24) & 0x0000000000FF0000) |
			((ull >> 40) & 0x000000000000FF00) |
			(ull << 56)
	}

	return ull
}

```

And use `Uint16ToBytes(...)`, `Uint32ToBytes(...)`, `Uint64ToBytes(...)` to transform int to byte array to be embeded:  
```go
func Uint16ToBytes(n uint16) []byte {
	return []byte{
		byte(n),
		byte(n >> 8),
	}
}
func Uint32ToBytes(n uint32) []byte {
	return []byte{
		byte(n),
		byte(n >> 8),
		byte(n >> 16),
		byte(n >> 24),
	}
}
func Uint64ToBytes(n uint64) []byte {
	return []byte{
		byte(n),
		byte(n >> 8),
		byte(n >> 16),
		byte(n >> 24),
		byte(n >> 32),
		byte(n >> 40),
		byte(n >> 48),
		byte(n >> 56),
	}
}
```

Ties together to create a payload. For type 0:  

```go

func OmniCreatePayloadSimpleSend(property_id uint32, amount uint64) []byte {

	var messageType uint16 = 0
	var messageVer uint16 = 0
	messageType = SwapByteOrder16(messageType)
	messageVer = SwapByteOrder16(messageVer)
	property_id = SwapByteOrder32(property_id)
	amount = SwapByteOrder64(amount)

	len := 4
	s := make([][]byte, len)

	s[0] = Uint16ToBytes(messageType)
	s[1] = Uint16ToBytes(messageVer)
	s[2] = Uint32ToBytes(property_id)
	s[3] = Uint64ToBytes(amount)

	sep := []byte("")
	return bytes.Join(s, sep)

}

```

And for [type 7](https://github.com/omnilaboratory/obd/blob/lnd/omnicore/rpcpayloadSendToMany.go#L50):  

```
	var messageType uint16 = 7
	var messageVer uint16 = 0
	messageType = SwapByteOrder16(messageType)
	messageVer = SwapByteOrder16(messageVer)
	property_id = SwapByteOrder32(property_id)

	var outputs_count byte = 0
	var total_value_out int64 = 0 
	
	receivers := make([][]byte, 0)

	for j := 0; j < len(receivers_array); j++ {
		outputs_count += 1
		amount := StrToInt64(receivers_array[j].Amount, divisible)
		amount_uint64 := uint64(amount)
		size := 2
		one_receiver := make([][]byte, size)
		one_receiver[0] = OneByte(receivers_array[j].Output)
		one_receiver[1] = Uint64ToBytes(SwapByteOrder64(amount_uint64))

		seperator := []byte("")
		one_receiver_in_byts := bytes.Join(one_receiver, seperator)

		receivers = append(receivers, one_receiver_in_byts)
		total_value_out = total_value_out + AmountFromValue(receivers_array[j].Amount)
	}

	len := 4 + outputs_count
	s := make([][]byte, len)

	s[0] = Uint16ToBytes(messageVer)
	s[1] = Uint16ToBytes(messageType)
	s[2] = Uint32ToBytes(property_id)
	s[3] = OneByte(outputs_count)

	for i := byte(4); i < len; i++ {
		s[i] = receivers[i-4]
	}

	sep := []byte("")

	payload := bytes.Join(s, sep)
	return payload, HexStr(payload)
	
```

### string to int64
The token amount in floating number is represented in a string, which has to be [converted to a usable int64](https://github.com/omnilaboratory/obd/blob/master/omnicore/ParseString.go#L21-L95). 8 decimal is allowed and can be recognized. If the decimal is less than 8, pad zeros on right. If there are too many decimals, truncate after 8:   


```go
	pos := strings.Index(strAmount, ".")
	if pos == -1 {
		// no decimal point but divisible so pad 8 zeros on right 
		pad_eight_zero := "00000000"
		strAmount = strings.Join([]string{strAmount, pad_eight_zero}, "")

	} else {
		// check for existence of second decimal point, if so invalidate amount
			 
		posSecond := strings.LastIndex(strAmount, ".")
		if posSecond != pos {
			return 0
		}

		if (len(strAmount) - pos) < 9 {
			// there are decimals either exact or not enough, pad as needed

			strRightOfDecimal := strAmount[pos+1 : len(strAmount)]
			zerosToPad := 8 - len(strRightOfDecimal)
			//fmt.Println(strRightOfDecimal)

			// do we need to pad?
			if zerosToPad > 0 {
				for it := 0; it != zerosToPad; it++ {
					strAmount += "0"
				}
			}
		} else {
			// there are too many decimals, truncate after 8
			// strAmount = strAmount.substr(0, pos + 9);
			strAmount = strAmount[0 : pos+9]
		}
		str1 := strAmount[0:pos]
		str2 := strAmount[pos+1 : len(strAmount)]
		strAmount = strings.Join([]string{str1, str2}, "")
	}
		 

	nAmount, _ = strconv.ParseInt(strAmount, 10, 64)
 ```


## The `btc_funding_created`, `btc_funding_signed`, `asset_funding_created` and `asset_funding_signed` Messages 

The four messages describe the outpoint which the funder has created for the initial commitment transactions. After receiving the peer's signature, via `funding_signed`, it will broadcast the funding transaction to the Omnilayer network.

We focus on omni assets in founding creation: Alice and Bob create a 2-2 P2SH payment to `scriptPubKey`, which is the hash of the **Redeem Script**.  

All transactions, besides the btc funding transaction, are omni class C transaction.  
 

```
    +-------+                                                  +-------+
    |       |---(1)-------------   open_channel     ---------->|       |
    |       |<--(2)------------  accept_channel    ------------|       |
    |       |                                                  |       |
    |   A   |---(3)------- btc_funding_created(3400 ) -------->|   B   |
    |       |<--(4)-------  btc_funding_signed(3500)  ---------|       |
    |       |                                                  |       |
    |       |---(5)-------- asset_funding_created(34) -------->|       |
    |       |<--(6)--------  asset_funding_signed(35) ---------|       |
    |       |                                                  |       |
    |       |---(7)------------  funding_locked  ------------->|       |
    |       |<--(8)------------  funding_locked  --------------|       |
    |       |                                                  |       |
    |       |<----------------   wait for close  ------------->|       |
    +-------+                                                  +-------+

    - where node Alice is 'funder' and node Bob is 'fundee'. Same to BOLT, the fundee is not allowed to fund the channel. 
    This is because the limitation of current BTC implementation. 
    - Of course, each of them can withdraw money if the counterparty agrees, as long as the two parties sign the correct Revocable Sequence Maturity Contracts for these onchain transactions.  

```

**2-2 P2SH**:
 
In order to avoid malicious counterparty who rejects to sign any payment out of the P2SH transaction, so that the money is forever locked in the channel, funder must construct a Commitment Transaction by which he is able to revoke his funding transaction. This is the first place we introduce the Revocable Sequence Maturity Contract (RSMC), invented by Poon and Dryja in their white paper, in this specification.

So the `funding_created` message does not mean both parties really deposite money into the channel. The first round communication is just simply setup a P2SH address, construct an unbroadcast funding transaction, construct a RSMC and exchange signatures. After that, Alice or Bob can broadcast the funding transaction to transfer real Omni assets into the channel.

The following diagram shows the steps we MUST do before any participants broadcast the funding/commitment transactions. BR1a (Breach Remedy) can be created later before the next commitment transaction is contructed.


<p align="center">
  <img width="500" alt="RSMC-C1a-RD1a" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-C1a-RD1a.png">
</p>

Alice's OBD constructs a promised transaction and a refund transaction: C1a/RD1a (Revocable Delivery), which pays out from the 2-2 P2SH transaction output:

step 1: Alice constructs a temporary 2-2 multi-sig address using Alice's temporary private key Alice2 and waiting Bob's signature: Alice2 & Bob.

step 2: Alice constructs a promise payment C1a out of Alice & Bob, one output is 60 USDT to `Alice2 & Bob`, and the other output is 40 USDT to Bob.

step 3: RD1a is the first output of C1a, which pays Alice 60 USDT, but with a sequence number preventing immediate payment if Alice cheats.

step 4: Bob signs C1a and RD1a, constructs the symmetric C1b/RD1b, hands to Alice for signature.

step 5: Alice signs C1b/RD1b and send back to bob. 

Both sides verifies the signed the transactions, if all are correct, Alice and Bob update their local database. Alice broadcasts the funding transaction. C1a and C1b cost the same output, only one can enter the blockchain.  

Any side broadcasts C1a/C1b, the couterparty immediatly gets the output1 of C1a/C1b, but has to wait for a seq=1000 to get his fund. If C1a/C1b are revocked but broadcasted, the counterparty can broadcast BR1a or BR1b immediately and get the remaining funds in the channel immediately. 
  

1. type: -3400 (btc_funding_created)
2. data:
    * [`32*byte`:`amount`]: total BTC funded by Alice.  
    * [`32*byte`:`funding_btc_hex`]: BTC funding transaction hex, used by Bob to validate the transaction.  
    * [`byte_array`:`funding_redeem_hex`]: funding redeem hex.   
    * [`sha256`:`funding_txid`]: funding transaction id.
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
 
Alice notifies Bob that she created the BTC funding transaction by payloads packed in this message -3400. 


1. type: -3500 (btc_funding_signed)
2. data:
    * [`byte`:`approval`]: true to confirm the btc funding transaction. 
    * [`byte_array`:`funding_redeem_hex`]: funding redeem hex.   
    * [`sha256`:`funding_txid`]: funding transaction id.   
    * [`32*byte`:`temporary_channel_id`]: the `temporary_channel_id` which will be replaced by channel_id in the following messeges.
  

Bob's OBD verifies the btc funding transaction by its hex, and replies Alice that he knows the funding of BTC by message -3500. 


1. type: -34 (asset_funding_created)
2. data:
    * [`byte_array`:`c1a_rsmc_hex`]: The constructed first rsmc transaction hex.  
    * [`byte_array`:`funding_omni_hex`]: The funding omni asset transaction hex, with which Bob can extract all the asset information, including funding output indx, amount, token, etc. Bob then is able to verify the funding transaction. Be noticed that in [BOLT-02-funding_created message](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#the-funding_created-message), funder sends fundee a funding_output_index for validation.  
    * [`32*byte`:`rsmc_temp_address_pub_key`]: Internal multisig address <Alice2 & Bob> used by c1a.
    * [`32*byte`:`temporary_channel_id`]: the same as the `temporary_channel_id` in the `open_channel` message.
    
    
Alice creats the asset funding transaction and asks Bob to verify and sign this transaction.
 
1. type: -35 (asset_funding_signed)
2. data:
    * [`channel_id`:`channel_id`]: generated by exclusive-OR of the funding_txid and the funding_output_index from the asset_funding_created message. 
    * [`byte_array`:`rd_hex`]: The hex of revockable delivery(RD1a) transaction.
    * [`32*byte`:`fundee_pubKey`]: the omni address of fundee Bob.
    * [`byte_array`:`rsmc_signed_hex`]: rsmc signed by Bob.
     
  
Bob signs, and send `asset_funding_signed` message back to Alice, hence Alice knows the 2-2 P2SH address has been created, but not broadcasted. 
   
   
## The `commitment_tx` and `revoke and acknowledge` Message

The two messages describe a RSMC payment inside one channel created by Alice and Bob. We introduce HTLC and corresponding messages in the next chapter.  

### diagram and messages
```
    +-------+                                             +-------+
    |       |--(1)-------  commitment_tx(351)  ---------->|       |  
    |       |<-(2)----- commitment_tx_signed (352) -------|       |
    |   A   |             Revoke and Acknowledge,         |   B   | 
    |       |           construct BR1a, C2a and RD2a      |       | 
    |       |            and the mirror transactions      |       |
    |       |                  on Bob's OBD               |       | 
    |       |                                             |       |  
    |       |--(3)-------------- (353) ------------------>|       |
    |       |      send back signed transactions in C2B   |       |
    +-------+                                             +-------+
   
```
     
<!-- ![RSMC](https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-diagram.png "RSMC") -->

<p align="center">
  <img width="750" alt="RSMC-diagram" src="https://github.com/omnilaboratory/OmniBOLT-spec/blob/master/imgs/RSMC-diagram.png">
</p>

There are two outputs of a commitment transaction:  
to_rsmc([to local](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_local-output)): 0. Alice2 & Bob 60,  
[to remote](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_remote-output): 1. Bob 40.  

In revockable delivery(RD) branch, the `to_rsmc` output sends funds back to the owner of this commitment transaction and thus must be timelocked using (for example) sequence number =1000, and must has breach remedy(BR) transaction for Bob in the case that Alice broadcasts a revocked commitment transaction.  

The RD and BR are written in a redeem script, which locks the to_local output. Put the redeem script and the omni class C transaction together to constructe the commitment transaction:   


### OMNI RSMC transaction construction  

#### Funding and `to remote` transactions   

```
version: 1  
locktime: 0 
tx input:
	* outpoint: the vout of funding transaction.  
	* <payee's signature> <payer's signature>: to spend the funding transaction.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* receiver/reference:{value:dust, pkScript: pubkey script},    
	* change:{value:change in satoshis, pkScript: the channel pubkey script }    
```
  
Where:  
`opReturn_encode`: the [encoded tx version( = 0 ), tx type( = 0 ), token id and amount](#payload), prefixed by "omni".  
 
`change`: change = satoshis in channel - dust - miner fee. By default, we set dust 546 satoshis.  

Fees are calculated as in [fee calculation](https://github.com/lightning/bolts/blob/master/03-transactions.md#fee-calculation), and must add the `op_return` byte numbers.  

The outputs are sorted into the order by omnicore spec.   

#### Commitment Transaction  
```
version: 1  
locktime: 0 
tx input:
	* outpoint: the vout of funding transaction.  
	* <payee's signature> <payer's signature>: to spend the funding transaction.  

tx output:
	* op_return:{value:0, pkScript:opReturn_encode},  
    	* to_rsmc/reference1:{value:dust, pkScript: RSMC redeem script},  
	* to_remote/reference2:{value:dust, pkScript: pubkey script},  
	* change:{value:change satoshis, pkScript: the channel pubkey script }    
```
Where:  
`opReturn_encode`: the [encoded tx version( = 0 ), tx type( = 7 ), token id and amount](#payload), prefixed by "omni".  

The payload on sender side is:

| size		|	Field				|	Example		 	|  
| -------- 	|	-----------------------		|    -------------------	|   
| 2 bytes	|	Transaction version		|	0			|
| 2 bytes	|	Transaction type		|	7 (= Send-to-Many)	|
| 4 bytes	|	Token identifier to send	|	31 (= USDT )	 	|
| 1 byte 	|	Number of receivers		|	2		 	|
| 1 byte 	|	Receiver output #		|	1 (= vout 1)		|
| 8 bytes	|	Amount to send			|	to_rsmc (e.g. 45)	|
| 1 byte 	|	Receiver output #		|	2 (= vout 2)	 	|
| 8 bytes	|	Amount to send			|	to_remote (= 55)	| 
 
The payload on receiver side is:  

| size		|	Field				|	Example		 	|  
| -------- 	|	-----------------------		|    -------------------	|   
| 2 bytes	|	Transaction version		|	0			|
| 2 bytes	|	Transaction type		|	7 (= Send-to-Many)	|
| 4 bytes	|	Token identifier to send	|	31 (= USDT )	 	|
| 1 byte 	|	Number of receivers		|	2		 	|
| 1 byte 	|	Receiver output #		|	1 (= vout 1)		|
| 8 bytes	|	Amount to send			|	to_rsmc (e.g. 55)	|
| 1 byte 	|	Receiver output #		|	2 (= vout 2)	 	|
| 8 bytes	|	Amount to send			|	to_remote (= 45)	| 
 

`to_rsmc` and `to_remote` are two outputs that allocates the balance of the channel. `to_remote` is locked by the pubkey script and `to_rsmc` is locked by the following `redeem script`:  

```bat
OP_IF
    # Breach Remedy(BR) branch, execute the penalty transaction
    <revocationpubkey>
OP_ELSE
    # Revockable Delivery(RD) branch
    `to_self_delay`
    OP_CHECKSEQUENCEVERIFY
    OP_DROP
    <local_delayedpubkey>
OP_ENDIF
OP_CHECKSIG
```  

`change`: change = satoshis in channel - dust - miner fee. By default, we set dust 546 satoshis.  

In the script, readers should notice that the 2-of-2 multisig as the revocation mechanism was replaced with the elliptic curve point multiplication technique. The updated version is of the form:
```
<revocationpubkey> OP_CHECKSIG
```

The outputs are sorted into the order by omnicore spec.   

### message data

Alice must send the hex `rsmc_hex` of the transaction based on `to_rsmc output` to Bob to verify and sign. The transaction based on `to_remote output` is named `to_counterparty_tx`, and Alice must send the hex `to_counterparty_tx_hex` to Bob to sign as well. In message `-352`, the signed arguments are `signed_to_counterparty_tx_hex` and `signed_rsmc_hex` respectively.  

Bob constructs the symmetric transaction C2b and hands it back to Alice for signing. 


1. type: -351 (commitment_tx)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`32*byte`:`amount`]: amount of the payment.
    * [`byte_array`:`commitment_tx_hash`]: C(n+1)a's commitment transaction hash, if Alice is the payer.
    * [`32*byte`:`last_temp_address_private_key`]: When C(n+1)a created, the payer shall give up the private key of the previous temp multi-sig address in C(n)a. <Alice2, Bob> as an example in the above diagram. 
    * [`byte_array`:`rsmc_hex`]: the hex of RSMC transaction. From the hex, payment information can be extracted and verified by the counterparty.  
    * [`byte_array`:`to_counterparty_tx_hex`]: The hex of the transaction that pays the counterparty money.
 
For example, Alice pays Bob `amount` of omni asset by sending `rsmc_Hex`. Her OBD constructs BR(1)a(Breach Remedy) and send it to Bob. Bob checks the Alice2's signature in BR1a to verify if the private key is correct. If it is, Bob signs C2a/RD2a.

 
1. type: -352 (Revoke and Acknowledge Commitment Transaction)
2. data:
    * [`32*byte`:`channel_id`]: the global channel id.
    * [`byte`:`approval`]: payee accepts or rejects this payment.
    * [`32*byte`:`amount`]: amount of the payment.  
    * [`byte_array`:`commitment_tx_hash`]: Payer's commitement tx hash.  
    * [`32*byte`:`last_temp_address_private_key`]: payee gives up the private key of his previous temp multi-sig address in C(n)b.
    * [`32*byte`:`curr_temp_address_pub_key`]: payee's current temp address pubkey.
    * [`byte_array`:`signed_to_counterparty_tx_hex`]: payee signs the `to_counterparty_tx_hex` sent by payer.
    * [`byte_array`:`signed_rsmc_hex`]: payee signs the `rsmc_hex` sent by payer.
    * [`byte_array`:`rsmc_hex`]: payee's rsmc transaction hex, used in constructing mirror transaction on payee's obd. 
    * [`byte_array`:`to_counterparty_tx_hex`]: payee's to_counterparty_tx_hex, used in constructing mirror transaction on payee's obd. 
    * [`byte_array`:`payer_rd_hex`]: payee signs based on `signed_rsmc_hex`, and send it back to payer to construct the RD transaction.
 

1. type: -353 (send back signed transactions in C2B)  
2. data: to be added  

Alice can only update the local state database after receiving all the signatures of C(n)a from Bob. Otherwise, if any message in the process is interrupted, Alice must cancel the established transaction and return to the state of the previous commitment tx. On Bob's side, the same logic is applied, the local state database is updated only after all Alice's signatures are received. 

## Cheat and Punishment

All penalties are implemented through breach remedy transactions and Seq timelocks. If Alice tries to broadcast C1a(the older transaction) to claim more money after she pays by C2a，then by broadcasting breach remedy transaction, this money will be sent to the Bob without delay.    

In the above [diagram](#diagram-and-messages), the payer Alice's OBD constructs C2a and simultaneously, Alice gives up the ownership of C1a by sending her temporary private key of Alice2 to Bob. This is a critical step for Bob to be able to punish fraudulent behavior. 

There has to be a daeman process that monitors Alice's behaviar. If it detects that Alice broadcasts C1a, it has to notify Bob to broadcast the punishment transaction BR1a using Alice2's private key. If Bob does not broadcast BR1a before the sequence number expires, Alice will be success in cheating, and get the 60 USDT.
 
 
## The `close_channel` Message 
 
**Rationale**
 
This message indicates how to withdraw money from a channel, by broadcasting a `C(n)a`, which is the latest commitment transaction in this channel. It comprises the following steps:

1. Allice raises a request, to withdraw her money in P2SH, with her proof of her balance.  
2. Alice waits OBD to approval:  
2.1 if they are correct, Alice raises a transaction paying from S2SH to Alice's omni address.  
2.2 if they are wrong/incorrect, OBD rejects the request and notify Bob.  


Before closing a channel, all HTLCs pending in this channel shall be removed, after which `close_channel` can be successfully executed.

1. type: -38 (close_channel)  
2. data:
    * [`channel_id`:channel_id]
    * [`u16`:`len`]
    * [`len`*`byte:scriptpubkey`]
    * [`signature`:`signature`]: the signature of Alice or Bob.

## unit tests

### Class C transaction test vectors 
Sender: "1K6JtSvrHtyFmxdtGZyZEF7ydytTGqasNc"  
Receiver: "1Njbpr7EkLA1R8ag8bjRN7oks7nv5wUn3o"  
Sender pays 0.1 token(2) to the receiver, where miner fee is 0.0006, and changes return to the sender.  

```
unspent outputs for sender:
[
    ...,
    {
        "txid" : "c23495f6e7ba24705d43583edd69ff25a354c18e69fd8514c07ec6f47cb995de",
        "vout" : 0,
        "address" : "1K6JtSvrHtyFmxdtGZyZEF7ydytTGqasNc",
        "account" : "",
        "scriptPubKey" : "76a914c6734676a08e3c6438bd95fa62c57939c988a17b88ac",
        "amount" : 0.00100000,
        "confirmations" : 0,
        "spendable" : true
    },
    {
        "txid" : "ee1673b09b0edaf7aaf8eb0bfd53a5a2757eb3e342e731bfc960b869aa0ab6b3",
        "vout" : 2,
        "address" : "1K6JtSvrHtyFmxdtGZyZEF7ydytTGqasNc",
        "account" : "",
        "scriptPubKey" : "76a914c6734676a08e3c6438bd95fa62c57939c988a17b88ac",
        "amount" : 0.00835660,
        "confirmations" : 1416,
        "spendable" : true
    }
]



payload:

|   size   |   Field               |   value   |    
| -------- |-----------------------|  -------  | 
|  16bits  |  Transaction version  |     0     |  
|  16bits  |  Transaction type     |     0     |  
|  32bits  |  Currency identifier  |     2     | 
|  64bits  |  Amount to transfer   |    0.1    |   


expected encoded payload: 00000000000000020000000000989680   

expected transaction hex: 0100000002de95b97cf4c67ec01485fd698ec154a325ff69dd3e58435d7024bae7f69534c20000000000ffffffffb3b60aaa69b860c9bf31e742e3b37e75a2a553fd0bebf8aaf7da0e9bb07316ee0200000000ffffffff036a5a0d00000000001976a914c6734676a08e3c6438bd95fa62c57939c988a17b88ac0000000000000000166a146f6d6e690000000000000002000000000098968022020000000000001976a914ee692ea81da1b12d3dd8f53fd504865c9d843f5288ac00000000

```

### RSMC test vectors

to be added.  

## references
 
 1. Omni specification version 0.7, [https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#omni-protocol-specification](https://github.com/OmniLayer/spec/blob/master/OmniSpecification.adoc#omni-protocol-specification)
 2. Omni specification for sendtomany, [https://gist.github.com/dexX7/1138fd1ea084a9db56798e9bce50d0ef](https://gist.github.com/dexX7/1138fd1ea084a9db56798e9bce50d0ef)
 3. Fee calculation, [https://github.com/lightning/bolts/blob/master/03-transactions.md#fee-calculation](https://github.com/lightning/bolts/blob/master/03-transactions.md#fee-calculation)
 
 
