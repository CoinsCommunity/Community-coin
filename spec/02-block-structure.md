# Section 2: Block Structure and Validation

## 2.1 Overview

A block is the fundamental unit of the Community ledger. Each block contains a header committing to the block's contents and metadata, a coinbase transaction distributing the block reward, a set of staker vote transactions approving the block, and a set of user-submitted transactions. A block is only added to the canonical chain after satisfying both proof-of-work validity and supermajority staker approval as defined in Sections 5 and 6.

---

## 2.2 Block Header

The block header is a fixed-size serialized structure containing the following fields in order:

| Field | Size | Description |
|---|---|---|
| VERSION | 4 bytes | Protocol version number. Current value: 1 |
| HEIGHT | 8 bytes | Block height. Genesis block is height 0 |
| PREV_HASH | 32 bytes | Hash of the previous finalized block header |
| MERKLE_ROOT | 32 bytes | Merkle root of all transactions in this block |
| TIMESTAMP | 8 bytes | Unix timestamp in seconds |
| DIFFICULTY_TARGET | 32 bytes | The difficulty target T this block must satisfy |
| NONCE | 8 bytes | Miner-varied field used in PoW search |

All integer fields are encoded in little-endian byte order.

The block header hash is computed as:

BLOCK_HASH = HASH(HASH(serialized_header))

where HASH is the protocol hash function defined in Section 9 and double-hashing follows the same convention as the transaction hash defined in Section 3.

The BLOCK_HASH is used as the PREV_HASH in the next block, as the seed input for staker panel selection as defined in Section 6, and as the faucet drop trigger input as defined in Section 8.

---

## 2.3 Block Body

The block body follows the block header in serialization order. The block body contains:

1. The coinbase transaction (exactly one, always first)
2. Vote transactions approving this block (zero or more, in any order)
3. User transactions (zero or more, in any order within this group)

The Merkle root in the block header commits to all transactions in the block body in the order they appear. Reordering transactions produces a different Merkle root and invalidates the block header.

**Maximum block size.** The total serialized size of the block body must not exceed MAX_BLOCK_SIZE bytes. MAX_BLOCK_SIZE is a genesis parameter. A block exceeding this limit is invalid regardless of all other validity conditions.

---

## 2.4 Transaction Types

Community supports the following transaction types. Each transaction carries a TYPE field identifying which type it is. The full structure of each type is defined in Section 3.

| Type byte | Name | Description |
|---|---|---|
| 0x01 | Coinbase | Block reward distribution. One per block, always first |
| 0x02 | Transfer | Standard private coin transfer between wallets |
| 0x03 | Stake | Lock coins into a staking commitment |
| 0x04 | Unstake | Return matured staking commitment coins to spendable balance |
| 0x05 | Vote | Staker approval or rejection vote on a candidate block |
| 0x06 | Claim | Faucet drop claim |
| 0x07 | Slash | Report of conflicting staker votes with evidence |

---

## 2.5 Coinbase Transaction

Every block contains exactly one coinbase transaction. It must be the first transaction in the block body. The coinbase transaction has no inputs. It creates coins from nothing up to the amount permitted by the emission schedule for that block height.

The coinbase transaction outputs are:

1. Miner reward output — paid to the miner's address. Amount as computed in Section 7.
2. Staker reward outputs — one output per staker who submitted a valid APPROVE vote for this block. Amount per staker as computed in Section 7. These outputs appear in the order the corresponding vote transactions appear in the block.
3. Faucet pool output — paid to the faucet pool. Amount as computed in Section 7.

If a drop event is triggered by this block's hash as defined in Section 8, the coinbase transaction does not pay out the drop. Drop payouts occur through Claim transactions submitted in subsequent blocks during the claiming window.

The coinbase transaction is invalid if:

- The miner reward output exceeds the computed miner reward for this block height.
- Any staker reward output exceeds the computed individual staker reward for this block height.
- The faucet pool output does not exactly equal the computed faucet allocation for this block height.
- The sum of all outputs exceeds R(n) as computed in Section 7 for block height n.
- A staker reward output is paid to an address not in the confirmed voting panel for this block.

A block whose coinbase transaction is invalid is itself invalid and rejected.

---

## 2.6 Block Validity Rules

A block is valid if and only if all of the following conditions are satisfied. A block failing any condition is invalid and rejected. Invalid blocks are not relayed to other nodes.

**Header validity:**

1. VERSION is a recognized protocol version.
2. HEIGHT equals the HEIGHT of PREV_HASH's block plus one.
3. PREV_HASH is the hash of a finalized block in the canonical chain.
4. MERKLE_ROOT correctly commits to the transactions in the block body in their serialized order.
5. TIMESTAMP satisfies the timestamp rules defined in Section 5.6.
6. DIFFICULTY_TARGET equals the computed difficulty for this block height as defined in Section 5.5.
7. The PoW solution is valid: MINING_OUTPUT(HASH(serialized_header || NONCE)) < DIFFICULTY_TARGET.

**Body validity:**

8. The first transaction is a coinbase transaction and it is valid as defined in Section 2.5.
9. No other transaction in the block is a coinbase transaction.
10. All vote transactions in the block are valid vote transactions for this block as defined in Section 6.6.
11. All transfer, stake, unstake, claim, and slash transactions are valid as defined in Section 3.
12. No transaction appears more than once in the block.
13. No key image appears more than once in the block or in any ancestor block on the canonical chain.
14. The total serialized block size does not exceed MAX_BLOCK_SIZE.
15. The block has received supermajority staker approval as defined in Section 6.7.

Rule 13 enforces double-spend prevention. Key images are defined in Section 4. Each key image uniquely identifies a specific coin being spent. A key image appearing twice means the same coin is being spent twice, which is rejected.

Rule 15 means a block is not fully valid until staker approval is received. Nodes may hold a block with a valid PoW solution and valid body in a pending state awaiting staker votes. The block transitions from pending to valid when supermajority approval is achieved. It is discarded if the voting window closes without supermajority approval.

---

## 2.7 Merkle Tree Construction

The Merkle tree is constructed from the transaction hashes of all transactions in the block body in their order of appearance.

Each leaf node of the Merkle tree is the hash of a serialized transaction:

LEAF(i) = HASH(serialized_transaction(i))

Internal nodes are computed by hashing the concatenation of their two children:

NODE(left, right) = HASH(left || right)

If the number of transactions is odd at any level of the tree, the last transaction hash is duplicated to produce an even number of inputs for that level. This convention matches standard Merkle tree construction.

The Merkle root is the root node of the fully constructed tree.

---

## 2.8 Block Propagation

When a miner finds a valid PoW solution it broadcasts the complete block (header and body) to all connected peers immediately. Nodes that receive a valid pending block (valid PoW, valid body, awaiting staker votes) relay it to their peers. Nodes do not relay blocks with invalid PoW solutions or invalid bodies.

When a staker submits a vote transaction it is broadcast as a standalone transaction and included in the next available block by miners. Staker votes do not require a special relay path.

When a block achieves supermajority approval, nodes update their canonical chain and relay the finalized block to peers who may not have seen all the vote transactions that finalized it.

---

## 2.9 Genesis Block

The genesis block is height 0. It has no predecessor so its PREV_HASH field is set to 32 zero bytes.

The genesis block body contains:

1. A coinbase transaction with a single output allocating GENESIS_FAUCET_ALLOCATION coins to the faucet pool and the first block's miner reward to the miner who found the genesis block.
2. No vote transactions. The genesis block requires no staker approval.
3. No user transactions.

The genesis block is hardcoded in all compliant implementations. Its hash is the root of the only valid Community chain.

---

## 2.10 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| MAX_BLOCK_SIZE | Maximum serialized block body size in bytes |
