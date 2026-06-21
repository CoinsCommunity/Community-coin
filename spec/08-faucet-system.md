# Section 8: Faucet System

## 8.1 Overview

The faucet system is a protocol-level coin distribution mechanism that operates autonomously without human intervention. It is funded by a fixed percentage of every block reward and releases coins at unpredictable intervals in variable amounts determined entirely by block hashes. No party can trigger, delay, suspend, or modify faucet behavior. It runs as long as the network produces blocks.

---

## 8.2 Definitions

**Faucet pool** — a protocol-controlled balance accumulated from block reward allocations. It has no private key. It is not an address in the conventional sense. The only mechanism that can spend from the faucet pool is a valid faucet drop as defined in this section.

**Drop event** — a faucet release triggered when a block hash falls within a defined threshold range. At most one drop event occurs per block.

**Drop tier** — the classification of a drop event determining its payout amount. Four tiers exist: Common, Uncommon, Rare, and Legendary.

**Claiming window** — the fixed period of blocks following a drop event during which eligible wallets may submit claim transactions.

**Claim transaction** — a special transaction type that transfers a fixed amount from the faucet pool to a claiming wallet. Claim transactions are free, they require no fee.

**Drop ID** — a unique identifier assigned to each drop event, derived from the block hash of the block that triggered the drop.

---

## 8.3 Faucet Pool Funding

Every block reward is split according to the allocation schedule defined in Section 7. Of that reward, a fixed percentage denoted FAUCET_POOL_RATE is allocated to the faucet pool automatically by the protocol.

FAUCET_POOL_RATE is a genesis parameter. It is fixed at genesis and cannot be changed.

The faucet pool accumulates continuously. If no drop event occurs for an extended period the pool grows. If the pool balance falls below the amount required for a triggered drop tier, that drop does not occur and no event is logged. The trigger is simply skipped for that block.

---

## 8.4 Drop Triggering

Every block, the network evaluates whether a drop event has been triggered. The evaluation uses the block hash of the newly confirmed block.

The block hash is interpreted as an unsigned 256-bit integer H.

Four threshold values are defined as genesis parameters:

| Parameter | Description |
|---|---|
| THRESHOLD_LEGENDARY | Upper bound for a Legendary drop |
| THRESHOLD_RARE | Upper bound for a Rare drop |
| THRESHOLD_UNCOMMON | Upper bound for an Uncommon drop |
| THRESHOLD_COMMON | Upper bound for a Common drop |

These thresholds satisfy the following ordering:

THRESHOLD_LEGENDARY < THRESHOLD_RARE < THRESHOLD_UNCOMMON < THRESHOLD_COMMON < 2^256

The evaluation logic is:

1. If H < THRESHOLD_LEGENDARY: a Legendary drop event is triggered.
2. Else if H < THRESHOLD_RARE: a Rare drop event is triggered.
3. Else if H < THRESHOLD_UNCOMMON: an Uncommon drop event is triggered.
4. Else if H < THRESHOLD_COMMON: a Common drop event is triggered.
5. Else: no drop event occurs for this block.

At most one drop event occurs per block. Tiers are evaluated from rarest to most common. A block hash falling below THRESHOLD_LEGENDARY triggers only a Legendary drop, not a Rare or Common drop.

The threshold values are calibrated to produce approximate drop frequencies. Target frequencies are:

| Tier | Target frequency |
|---|---|
| Common | Approximately 1 in 50 blocks |
| Uncommon | Approximately 1 in 500 blocks |
| Rare | Approximately 1 in 5,000 blocks |
| Legendary | Approximately 1 in 50,000 blocks |

These are probabilistic targets. The faucet system has no memory and no scheduling. Each block is an independent evaluation. Actual intervals between drops of any tier are random and may vary significantly from the targets.

---

## 8.5 Drop Amounts

Each drop tier has a fixed payout amount per claim. Payout amounts are genesis parameters.

| Parameter | Description |
|---|---|
| AMOUNT_COMMON | Coins paid per claim in a Common drop |
| AMOUNT_UNCOMMON | Coins paid per claim in an Uncommon drop |
| AMOUNT_RARE | Coins paid per claim in a Rare drop |
| AMOUNT_LEGENDARY | Coins paid per claim in a Legendary drop |

Payout amounts must satisfy AMOUNT_COMMON < AMOUNT_UNCOMMON < AMOUNT_RARE < AMOUNT_LEGENDARY.

All claimants in a given drop event receive the same payout amount for that tier. The total payout from a drop event depends on how many wallets successfully claim before the claiming window closes.

---

## 8.6 Drop ID

When a drop event is triggered in block B, a Drop ID is assigned as follows:

DROP_ID = HASH(block_hash(B) || drop_tier)

where HASH is the standard protocol hash function, block_hash(B) is the hash of block B, drop_tier is a single byte encoding the tier (0x01 for Common, 0x02 for Uncommon, 0x03 for Rare, 0x04 for Legendary), and || denotes concatenation.

Drop IDs are unique per drop event. The same block cannot produce two drop events so no two drop events share a Drop ID.

---

## 8.7 Claiming Window

When a drop event is triggered in block B, a claiming window opens immediately. The claiming window remains open for CLAIM_WINDOW_BLOCKS blocks.

CLAIM_WINDOW_BLOCKS is a genesis parameter.

The claiming window for a drop event triggered in block B is open during blocks B through B + CLAIM_WINDOW_BLOCKS inclusive. A claim transaction submitted in block B + CLAIM_WINDOW_BLOCKS + 1 or later is invalid and the network rejects it.

---

## 8.8 Claim Transactions

A claim transaction specifies:

- DROP_ID: the identifier of the drop event being claimed
- CLAIMING_ADDRESS: the wallet address receiving the payout

A claim transaction is valid if and only if all of the following conditions are met:

1. The DROP_ID corresponds to a drop event that has occurred.
2. The current block height is within the claiming window for that DROP_ID.
3. The CLAIMING_ADDRESS has not previously submitted a valid claim for this DROP_ID.
4. The faucet pool balance is greater than or equal to the payout amount for the drop tier.

If any condition is not met the network rejects the claim transaction. Rejected claim transactions are not included in blocks.

Valid claim transactions transfer the drop tier payout amount from the faucet pool to CLAIMING_ADDRESS. The faucet pool balance decreases by that amount. The CLAIMING_ADDRESS balance increases by that amount.

Claim transactions are free. They require no transaction fee. Miners and stakers process claim transactions without receiving a fee for doing so.

---

## 8.9 One Claim Per Wallet Per Drop

The network maintains a claimed set for each drop event. The claimed set records every CLAIMING_ADDRESS that has submitted a valid claim for that DROP_ID.

When a claim transaction is evaluated, the network checks whether CLAIMING_ADDRESS is already in the claimed set for DROP_ID. If it is, the claim transaction is rejected.

If the claim is valid, CLAIMING_ADDRESS is added to the claimed set for DROP_ID before the transaction is finalized.

The one-claim-per-wallet rule is enforced entirely by the protocol. It does not require identity verification. A wallet holder can create multiple wallets and claim once per wallet per drop. The economic limit on this behavior is that each claiming wallet must exist and be capable of submitting a transaction, which requires some minimum coin balance for operational purposes in a typical network environment.

---

## 8.10 Unclaimed Amounts

When the claiming window for a drop event closes, any portion of the potential payout not distributed through valid claims remains in the faucet pool. Unclaimed amounts are not destroyed. They accumulate and are available for future drop events.

There is no mechanism to redistribute unclaimed amounts from a specific drop event. They simply remain in the pool and are paid out in future drops.

---

## 8.11 Pool Depletion Handling

If the faucet pool balance at the time of a triggered drop is less than the payout amount for that tier, the drop does not occur. The trigger is skipped silently. No drop event is logged, no claiming window opens, and no Drop ID is assigned for that block.

The pool continues accumulating from subsequent block rewards. A future triggered drop of any tier will proceed normally once the pool balance is sufficient for that tier's payout amount.

---

## 8.12 Genesis Faucet Allocation

The genesis block includes an initial faucet pool allocation. This allocation funds the faucet pool before any block rewards have been earned. The genesis faucet allocation is a genesis parameter denoted GENESIS_FAUCET_ALLOCATION.

The faucet system is active from block 1. There is no activation delay or minimum pool threshold before the first drop can occur.

---

## 8.13 Genesis Parameters Summary

The following parameters are defined at genesis and cannot be changed:

| Parameter | Description |
|---|---|
| FAUCET_POOL_RATE | Percentage of each block reward allocated to the faucet pool |
| THRESHOLD_LEGENDARY | Block hash upper bound for Legendary drop trigger |
| THRESHOLD_RARE | Block hash upper bound for Rare drop trigger |
| THRESHOLD_UNCOMMON | Block hash upper bound for Uncommon drop trigger |
| THRESHOLD_COMMON | Block hash upper bound for Common drop trigger |
| AMOUNT_COMMON | Payout per claim in a Common drop |
| AMOUNT_UNCOMMON | Payout per claim in an Uncommon drop |
| AMOUNT_RARE | Payout per claim in a Rare drop |
| AMOUNT_LEGENDARY | Payout per claim in a Legendary drop |
| CLAIM_WINDOW_BLOCKS | Number of blocks a claiming window remains open |
| GENESIS_FAUCET_ALLOCATION | Initial faucet pool balance at genesis |

Exact values for all genesis parameters are determined during the final specification review period and published with the genesis block. They are not changed after publication.
