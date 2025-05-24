---
layout: post
title: "Bitcoin"
excerpt_separator: <!--more-->
date: 2016-04-24 00:00:00 +0800
author: xiangxiang
categories: bitcoin
tags: [bitcoin]
---
Mastering bitcoin

<!--more-->
* auto-gen TOC:
{:toc}

## 0x00 Outline
- Two basic problems for digital money
- How bitcoin works \(the big picture\)
- Transactions \(How to transfer money\)
- The Blockchain \(How to store money\)
- Mining and Consensus \(How to create money\)

## 0x01 Two basic problems
1. Can I trust the money is authentic and not counterfeit?

2. Can I be sure that no one else can claim that this money belongs to them and not me? \(Aka the “double\-spend” problem\.\)

## 0x02  Bitcoin’s four key innovations
- A decentralized peer\-to\-peer network \(the bitcoin protocol\)
- A public transaction ledger \(the blockchain\)
- A decentralized mathematical and deterministic currency issuance \(distributed mining\)
- A decentralized transaction verification system \(transaction script\)

## 0x03 Transactions
- Transaction Lifecycle
- Transaction Structure
- Transaction Outputs and Inputs
- Transaction Scripts and Script Language
- Standard Transactions

### 3.1 Transaction Lifecycle
- Creating Transactions \(writing a paper check\)
  * signature
  * with no confidential information

- Broadcasting Transactions to the Bitcoin Network
  * publicly broadcast with any kind of network \(4G\, wifi\, Bluetooth\, NFC\, …\)

- Propagating Transactions on the Bitcoin Network
  * the transaction will be validated by the node **independently**
  * if valid\, the node will propagate it to other nodes
  * otherwise\, the node will reject it

---
To prevent spamming, denial-of-service attacks, or other nuisance attacks against the bitcoin system, 
every node independently validates every transaction before propagating it further.

Validation checklist,ch8-tx-verification


### 3.2 Transaction Structure

| Size | Field | Description |
| :-: | :-: | :-: |
| 4 bytes | Version | Specifies which rules this transaction follows |
| __1–9 bytes \(VarInt\)__ | __Input Counter__ | How many inputs are included |
| __Variable__ | __Inputs__ | One or more transaction inputs |
| __1–9 bytes \(VarInt\)__ | Output Counter | How many outputs are included |
| __Variable__ | __Outputs__ | __One or more transaction outputs__ |
| __4 bytes__ | __Locktime__ | __A Unix timestamp or block number__ |

---

A transaction is a data structure that encodes a transfer of value from a source of funds, called an input, to a destination, called an output.
Transaction inputs and outputs are not related to accounts or identities

### 3.3 Transaction Inputs
- Inputs =  unspent transaction output\( <span style="color:#ff0000">UTXO</span> \)
- UTXO
  * indivisible chunks of bitcoin currency locked to a specific owner
  * recorded on the blockchain
  * recognized as currency units by the entire network
- no accounts or balances in bitcoin
- only unspent transaction outputs \(UTXO\) scattered in the blockchain
- Transactions consume UTXO by unlocking it with the signature of the current owner and  create UTXO by locking it to the bitcoin address of the new owner\.

---

coinbase transaction; The user’s wallet application will typically select from the user’s available UTXO various units to compose an amount greater than or equal to the desired transaction amount.


### 3.4 The structure of a transaction input

| Size | Field | Description |
| :-: | :-: | :-: |
| 32 bytes | Transaction Hash | __Pointer to the transaction containing the UTXO to be spent__ |
| __4 bytes__ | Output Index | The index number of the UTXO to be spent; first one is 0 |
| __1\-9 bytes \(VarInt\)__ | Unlocking\-Script Size | __Unlocking\-Script length in bytes\, to follow__ |
| Variable | Unlocking\-Script | A script that fulfills the conditions of the UTXO locking script\. |
| 4 bytes | Sequence Number | Currently disabled Tx\-replacement feature\, set to 0xFFFFFFFF |

---

In simple terms, transaction inputs are pointers to UTXO

Transaction Outputs



* Transaction outputs consist of two parts:
  * An amount of bitcoin\, denominated in satoshis\, the smallest bitcoin unit
  * A locking script\, also known as an "encumbrance" that "locks" this amount by specifying the conditions that must be met to spend the output
* The structure of a transaction output


| Size | Field | Description |
| :-: | :-: | :-: |
| 8 bytes | Amount | __Bitcoin value in satoshis \(10__  __\-8__  __bitcoin\)__ |
| __1–9 bytes \(VarInt\)__ | __Locking\-Script Size__ | How many inputs are included |
| __Variable__ | __Locking\-Script__ | __A script defining the conditions needed to spend the output__ |

### 3.5 A principle example of a Bitcoin transaction with 1 input and 1 output

Input:

Previous tx: f5d8ee39a430901c91a5917b9f2dc19d6d1a0e9cea205b009ca73dd04470b9a6

Index: 0

<span style="color:#ff0000">scriptSig</span> : 304502206e21798a42fae0e854281abd38bacd1aeed3ee3738d9e1446618c4571d10

90db022100e2ac980643b0b82c0e88ffdfec6b64e3e6ba35e7ba5fdd7d5d6cc8d25c6b241501

Output:

Value: 5000000000

<span style="color:#ff0000">scriptPubKey</span> : OP\_DUP OP\_HASH160 404371705fa9bd789a2fcd52d2c580b65d35549d

OP\_EQUALVERIFY OP\_CHECKSIG

### 3.6 Standard Transactions
- Pay\-to\-Public\-Key\-Hash \(P2PKH\)
- Pay\-to\-Public\-Key
- Multi\-Signature
- Data Output \(OP\_RETURN\)
- Pay\-to\-Script\-Hash \(P2SH\)

#### 3.6.1 Scripting Language
- a Forth\-like reverse\-polish notation  <span style="color:#ff0000">stack</span> \-based execution language
- Turing Incompleteness \(no loops or goto\)
- Stateless Verification
- Simple example
  * Locking script: 3 OP\_ADD 5 OP\_EQUAL
  * Unlocking script: 2
  * Combining Unlocking script and Locking script:  <span style="color:#ff0000">2 3 OP\_ADD 5 OP\_EQUAL</span>

- Script example: 2 3 OP_ADD 5 OP_EQUAL

```
[] = stack
2->[2]
3->[2 3]
OP_ADD->[5]
5->[5 5]
OP_EQUAL->[TRUE]
```

#### 3.6.2 Pay-to-Public-Key-Hash (P2PKH)
- Locking script

```
OP_DUP OP_HASH160 <Public Key Hash> OP_EQUAL OP_CHECKSIG
<Signature> <Public Key>
```

- Unlocking script

```
<Signature> <Public Key> OP_DUP OP_HASH160
<Public Key Hash> OP_EQUAL OP_CHECKSIG
```

#### 3.6.3 Pay-to-Public-Key
- Locking script

```
<Public Key A> OP_CHECKSIG
<Signature from Private Key A>
```

- Unlocking script

```
<Signature from Private Key A> <Public Key A> OP_CHECKSIG
```

#### 3.6.4 Pay-to-Public-Key
- Locking script
  * The prefix OP_0 is required because of a bug in the original implementation of CHECKMULTISIG where one item too many is popped off the stack. It is ignored by CHECKMULTISIG and is simply a placeholder.

```
M <Public Key 1> <Public Key 2> ... <Public Key N> N OP_CHECKMULTISIG
OP_0 <Signature B> <Signature C>
```

- Unlocking script

```
A locking script example (2-3)
2 <Public Key A> <Public Key B> <Public Key C> 3 OP_CHECKMULTISIG
```

#### 3.6.5 Data Output (OP_RETURN)
- Bitcoin’s distributed and timestamped ledger, the blockchain, has potential uses far beyond payments. 
- `OP_RETURN <data>`

- The data portion is limited to 40 bytes and most often represents a hash, such as the output from the SHA256 algorithm (32 bytes)
- Many applications put a prefix in front of the data to help identify the application
- no "unlocking script"

#### 3.6.6 Pay-to-Script-Hash (P2SH)
- Introduced in 2012
- Cons of Multi-Signature: too many public keys (high fees) 
- RIPEMD160也是一个密码上的hash函数；原来的unloching script变成了locking script的一部分;redeem赎回

```
Multi-Signature
2 <PK1> <PK2> <PK3> <PK4> <PK5> 5 OP_CHECKMULTISIG (Redeem Script)
<Sig1> <Sig3>
RIPEMD160{SHA256(Locking script)} -> 20-byte hash value
P2SH locking:  OP_HASH160 <20-byte hash> OP_EQUAL
Unlocking: <Sig1> <Sig3> 2 PK1 PK2 PK3 PK4 PK5 5 OP_CHECKMULTISIG
```

Benefits of pay-to-script-hash
- Complex scripts are replaced by shorter fingerprints in the transaction output\, making the transaction smaller\.
- Scripts can be coded as an address\, so the sender and the sender’s wallet don’t need complex engineering to implement P2SH\.
- P2SH shifts the burden of constructing the script to the recipient\, not the sender\.
- P2SH shifts the burden in data storage for the long script from the output \(which is in the UTXO set\) to the input \(stored on the blockchain\)\.
- P2SH shifts the burden in data storage for the long script from the present time \(payment\) to a future time \(when it is spent\)\.
- P2SH shifts the transaction fee cost of a long script from the sender to the recipient\, who has to include the long redeem script to spend it\.

## 0x04 The Blockchain
- The structure of a Block

| Size | Field | Description |
| :-: | :-: | :-: |
| 4 bytes | Block Size | __The size of the block\, in bytes\, following this field__ |
| __80 __  __bytes__ | Block Header | Several fields form the block header |
| __1\-9 bytes \(VarInt\)__ | Transaction Counter | __How many transactions follow__ |
| Variable | Transactions | The transactions recorded in this block |

---

In simple terms, transaction inputs are pointers to UTXO

### 4.1 The structure of the block header

| Size | Field | Description |
| :-: | :-: | :-: |
| 4 bytes | Version | __A version number to track software/protocol upgrades__ |
| __32 __  __bytes__ | Previous Block Hash | A reference to the hash of the previous \(parent\) block in the chain |
| __32 __  __bytes __ | Merkle Root | __A hash of the root of the merkle tree of this block’s transactions__ |
| 4 bytes | Timestamp | The approximate creation time of this block \(seconds from Unix Epoch\) |
| 4 bytes | Difficulty Target | The proof\-of\-work algorithm difficulty target for this block |
| 4 bytes | Nonce | A counter used for the proof\-of\-work algorithm |

---

In simple terms, transaction inputs are pointers to UTXO

### 4.2 Block Identifiers
- Block Header Hash
  * Why only header?
  * 000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f
  * \(first block\, aka the genesis block
- Block Height
  * 0 \(first block\)
---

Why only header?Merkle Root


## 0x05 Mining
- New bitcoin is added to the money supply
- Secure the bitcoin system
- Miners validate new transactions and record them on the global ledger
- Miners receive two types of rewards for mining: new coins created with each new block\, and transaction fees from all the transactions included in the block\.

---

Finally, after 13.44 million blocks, in approximately 2140, almost 2,099,999,997,690,000 satoshis, or almost 21 million bitcoins, will be issued.
every 4 years,1/2;init=50

### 5.1 Mining steps
1. Aggregating Transactions into Blocks, While searching for a solution to the block  1
2. Once upon receiving block 1 and validating it, check all the transactions in the memory pool, remove any that were included in block 1
3. Immediately constructs a new empty block(a candidate for block 2)
---

step 1. After validating transactions->a candidate block.

### 5.2 Transaction Age, Fees, and Priority
- Priority = Sum \(Value of input \* Input Age\) / Transaction Size

- The first 50 kilobytes of transaction space in a block are set aside for high\-priority transactions
- Then fills the rest of the block up to the maximum block size, with transactions that carry at least the minimum fee, prioritizing those with the highest fee per kilobyte of transaction\.
- If there is any space remaining in the block\, might choose to fill it with no\-fee transactions\.

---

To construct the candidate block, Jing’s bitcoin node selects transactions from the memory pool by applying a priority metric to each transaction and adding the highest priority transactions first

### 5.3 The Generation Transaction(coinbase transaction)
- The first transaction added to the block is a special transaction
- Total Fees = Sum\(Inputs\) \- Sum\(Outputs\)
- Rewards = Block\_value\(now 25\) \+ total\_fees

### 5.4 Transaction Structure (Recall)

| Size | Field | Description |
| :-: | :-: | :-: |
| 4 bytes | Version | Specifies which rules this transaction follows |
| __1–9 bytes \(VarInt\)__ | __Input Counter__ | How many inputs are included |
| __Variable__ | __Inputs__ | One or more transaction inputs |
| __1–9 bytes \(VarInt\)__ | Output Counter | How many outputs are included |
| __Variable__ | __Outputs__ | __One or more transaction outputs__ |
| __4 bytes__ | __Locktime__ | __A Unix timestamp or block number__ |


The structure of a normal transaction input

| Size | Field | Description |
| :-: | :-: | :-: |
| 32 bytes | Transaction Hash | __Pointer to the transaction containing the UTXO to be spent__ |
| __4 bytes__ | Output Index | The index number of the UTXO to be spent; first one is 0 |
| __1\-9 bytes \(VarInt\)__ | Unlocking\-Script Size | __Unlocking\-Script length in bytes\, to follow__ |
| Variable | Unlocking\-Script | A script that fulfills the conditions of the UTXO locking script\. |
| 4 bytes | Sequence Number | Currently disabled Tx\-replacement feature\, set to 0xFFFFFFFF |

In simple terms, transaction inputs are pointers to UTXO, exception coinbase


The structure of a generation transaction input

| Size | Field | Description |
| :-: | :-: | :-: |
| 32 bytes | Transaction Hash | <span style="color:#ff0000">All bits are zero: Not a transaction hash reference</span> |
| __4 bytes__ | Output Index | <span style="color:#ff0000">The index number of the UTXO to be spent; first one is 0</span> |
| __1\-9 bytes \(VarInt\)__ | Coinbase Data Size | <span style="color:#ff0000">Length of the coinbase data\, from 2 to 100 bytes</span> |
| Variable | Unlocking\-Script | <span style="color:#ff0000">Arbitrary data used for extra nonce and mining tags in v2 blocks\, must begin with block height</span> |
| 4 bytes | Sequence Number | set to 0xFFFFFFFF |

---

In simple terms, transaction inputs are pointers to UTXO, exception coinbase


The structure of the block header

| Size | Field | Description |
| :-: | :-: | :-: |
| 4 bytes | Version | A version number to track software/protocol upgrades |
| __32 bytes__ | __Previous Block Hash__ | A reference to the hash of the previous \(parent\) block in the chain |
| __32 bytes__ | __Merkle Root__ | A hash of the root of the merkle tree of this block’s transactions |
| __4 bytes__ | Timestamp | The approximate creation time of this block \(seconds from Unix Epoch\) |
| __4 bytes__ | __Difficulty Target__ | __The proof\-of\-work algorithm difficulty target for this block__ |
| __4 bytes__ | __Nonce__ | __A counter used for the proof\-of\-work algorithm__ |

### 5.5 Mining the Block
- Mining is the process of hashing the block header repeatedly\, changing one parameter\, until the resulting hash matches a specific target
- Desired difficulty :in bits\, how many of the leading bits must be zero
- Proof\-Of\-Work Algorithm
  * for nonce in xrange\(max\_nonce\):
  * hash\_result = sha256\(header\)\.hexdigest\(\)
  * if long\(hash\_result\,16\) < target:
  * print hash\_result\, nonce
  * print ‘fail’
---

Brute-force;随着时间，target原来越小

### 5.6 Successfully Mining the Block
- Miner transmits the block to all its peers
- They receive\, validate\, and then propagate the new block

## 0x06 Research Topics
- Programmable money
- Proof of work
- Blockchain


## refs
- [https://github.com/bitcoinbook/bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)