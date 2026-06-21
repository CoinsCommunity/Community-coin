# Section 6: Consensus — PoS Staker Selection, Voting, and Finality

## 6.1 Overview

Community uses a hybrid Proof-of-Work and Proof-of-Stake consensus mechanism. The PoW layer determines who proposes blocks. The PoS layer determines whether proposed blocks are accepted. Both layers must cooperate for a block to be added to the chain. This section covers the PoS layer in full.

The PoW layer, mining algorithm, and difficulty adjustment are defined in Section 5.

---

## 6.2 Definitions

**Stake** — CMTY coins locked into a staking commitment by a participant wishing to serve as a validator.

**Staker** — a participant who has an active staking commitment with a balance above the minimum staking threshold.

**Staking commitment** — a time-locked transaction that removes coins from a wallet's spendable balance and registers them as active stake. Coins in a staking commitment cannot be spent until the commitment matures.

**Active stake** — stake that is currently locked and eligible for panel selection. Stake that is maturing, pending, or withdrawn is not active stake.

**Voting panel** — the set of stakers pseudorandomly selected to approve or reject a specific candidate block.

**Panel size** — the number of stakers in a voting panel. Defined by the genesis parameter PANEL_SIZE.

**Supermajority** — the minimum number of panel approvals required to confirm a block. Defined by the genesis parameter SUPERMAJORITY_THRESHOLD.

**Voting window** — the number of blocks during which selected panel members may submit their vote on a candidate block. Defined by the genesis parameter VOTE_WINDOW_BLOCKS.

**Candidate block** — a block with a valid PoW solution that has been broadcast to the network and is awaiting staker approval.

**Finalized block** — a candidate block that has received supermajority approval and been added to the canonical chain.

---

## 6.3 Becoming a Staker

Any participant holding CMTY may become a staker by submitting a staking commitment transaction.

A staking commitment transaction specifies:

- STAKE_AMOUNT: the number of CMTY coins to lock. Must be greater than or equal to MIN_STAKE, a genesis parameter.
- STAKER_ADDRESS: the address receiving staking rewards when selected.

When a staking commitment transaction is included in a finalized block, the coins are removed from the sender's spendable balance and registered as active stake under STAKER_ADDRESS. The coins remain locked for STAKE_LOCK_PERIOD blocks from the block in which the commitment was confirmed. STAKE_LOCK_PERIOD is a genesis parameter.

A staker may hold multiple staking commitments simultaneously. Each commitment is tracked independently with its own lock period and maturity date. The total active stake attributed to a staker address is the sum of all active commitments under that address.

---

## 6.4 Stake Weighting

Panel selection is weighted by active stake. A staker's probability of selection for any given panel is proportional to their share of total active stake across all stakers at the time of selection.

Formally, for a staker S with active stake STAKE(S), and total active stake across all stakers of TOTAL_STAKE:

SELECTION_PROBABILITY(S) = STAKE(S) / TOTAL_STAKE

A staker with more coins is more likely to be selected but faces the same pseudorandom selection process as every other staker. No staker can guarantee selection for any specific block regardless of stake size.

---

## 6.5 Panel Selection

When a miner broadcasts a candidate block B, the network deterministically computes the voting panel for B using a pseudorandom selection process seeded by values that were finalized before block B was found.

The panel seed is computed as:

PANEL_SEED(B) = HASH(block_hash(parent(B)) || block_height(B))

where parent(B) is the most recently finalized block before B, block_hash(parent(B)) is its hash, and block_height(B) is the height of the candidate block.

The PANEL_SEED is deterministic given the parent block and the candidate block height. It is unpredictable before the parent block is finalized because the parent block hash cannot be known until the parent block is mined.

Panel selection proceeds as follows:

1. Compute PANEL_SEED(B).
2. Derive a pseudorandom sequence from PANEL_SEED(B) using an expand function defined in Section 9.
3. Using the pseudorandom sequence, perform a weighted sampling without replacement from the set of all active stakers, weighted by STAKE(S) / TOTAL_STAKE.
4. The first PANEL_SIZE unique stakers drawn constitute the voting panel for block B.

A staker who is not online or fails to vote does not affect panel composition. Panel membership is determined at the moment the candidate block is broadcast and does not change if a staker goes offline.

If the number of registered active stakers is less than PANEL_SIZE, all active stakers form the voting panel regardless of PANEL_SIZE. This condition is expected only in the very early days of network operation.

---

## 6.6 Voting

Each member of the voting panel for candidate block B must submit a vote transaction within VOTE_WINDOW_BLOCKS blocks of the block height at which B was broadcast.

A vote transaction specifies:

- CANDIDATE_BLOCK_HASH: the hash of the candidate block being voted on.
- VOTE: either APPROVE or REJECT.
- VOTER_ADDRESS: the staker address of the panel member submitting the vote.

A vote transaction is valid if and only if:

1. VOTER_ADDRESS is a member of the correctly computed voting panel for CANDIDATE_BLOCK_HASH.
2. The current block height is within VOTE_WINDOW_BLOCKS of the broadcast height of the candidate block.
3. VOTER_ADDRESS has not previously submitted a vote for this CANDIDATE_BLOCK_HASH.

The network rejects invalid vote transactions. They are not included in blocks.

Each panel member may vote only once per candidate block. A second vote transaction from the same VOTER_ADDRESS for the same CANDIDATE_BLOCK_HASH is invalid regardless of the vote value.

---

## 6.7 Block Confirmation

A candidate block B is confirmed and added to the chain when the number of valid APPROVE votes submitted for B reaches SUPERMAJORITY_THRESHOLD.

SUPERMAJORITY_THRESHOLD is defined as a genesis parameter expressed as a minimum count of approvals. SUPERMAJORITY_THRESHOLD must be greater than half of PANEL_SIZE to ensure that no two conflicting blocks can simultaneously achieve supermajority approval.

Once a candidate block reaches SUPERMAJORITY_THRESHOLD approvals it is immediately finalized. The block is added to the chain. Block rewards are paid. No further votes on that block are processed.

---

## 6.8 Block Rejection

A candidate block B is rejected if any of the following conditions occur:

1. The number of REJECT votes submitted for B exceeds PANEL_SIZE minus SUPERMAJORITY_THRESHOLD. This makes it mathematically impossible for B to reach supermajority approval with remaining votes.

2. The voting window closes, meaning VOTE_WINDOW_BLOCKS blocks have passed since B was broadcast, and B has not reached SUPERMAJORITY_THRESHOLD approvals.

When a candidate block is rejected:

- The block is discarded. It is never added to the chain.
- The miner receives no reward. The PoW solution yields nothing.
- Stakers who voted APPROVE receive no reward for this block.
- Stakers who voted REJECT receive no reward for this block.
- Stakers who did not vote receive no reward for this block.
- The network continues processing the next candidate block at the same height.

The miner may immediately attempt to mine a new candidate block at the same height. Other miners are also competing at the same height. The first new candidate block to achieve supermajority approval at that height is added to the chain.

---

## 6.9 Staker Rewards

Stakers who voted APPROVE on a successfully finalized block receive their share of the staker reward as defined in Section 7.

Stakers who voted REJECT on a finalized block receive no reward for that block, regardless of whether their rejection was correct or incorrect.

Stakers who were selected for a panel but failed to submit any vote within the voting window receive no reward for that block.

Stakers who submitted a valid APPROVE vote that contributed to a supermajority receive the individual staker reward as computed in Section 7.

Rewards are paid in the coinbase transaction of the finalized block itself. The coinbase transaction lists each rewarded staker address and the corresponding payment.

---

## 6.10 Unstaking

When a staking commitment reaches its maturity date, meaning STAKE_LOCK_PERIOD blocks have passed since the commitment was confirmed, the staker may submit an unstaking transaction to return the locked coins to their spendable balance.

An unstaking transaction specifies the commitment being matured. The network verifies the commitment has reached its maturity block height and transfers the locked coins back to the spendable balance of STAKER_ADDRESS.

A staker may not unstake early. There is no early withdrawal mechanism. Coins locked in an active staking commitment are not accessible until the commitment matures regardless of circumstances.

Coins in a maturing commitment remain as active stake until the unstaking transaction is confirmed. A staker who does not submit an unstaking transaction continues to be eligible for panel selection and reward until they explicitly unstake.

---

## 6.11 Nothing-at-Stake Mitigation

The nothing-at-stake problem describes a scenario in a pure PoS system where stakers vote on multiple competing chain forks simultaneously because there is no cost to doing so.

Community mitigates this in two ways.

First, the hybrid PoW/PoS design means an attacker must also control majority hashrate to sustain a fork. Staker votes alone cannot produce a valid chain.

Second, a staker who submits conflicting votes on two candidate blocks at the same height within the same voting window is penalized. The network detects conflicting votes when both are broadcast. The detection rule is: if two valid vote transactions from the same VOTER_ADDRESS reference different CANDIDATE_BLOCK_HASH values at the same block height, both votes are invalid and the staker's active staking commitment is slashed.

**Slashing** — when a staker's commitment is slashed, the locked coins are removed from the staker's balance permanently. They are not redistributed. They are destroyed. The slashed staker receives no further rewards and their stake is removed from the active staker set immediately.

Slashing requires cryptographic proof of the conflicting votes. Any network participant may submit a slash transaction containing both conflicting vote transactions as evidence. The network verifies the evidence and executes the slash automatically if valid. The participant submitting a valid slash transaction receives a slash bounty of SLASH_BOUNTY_RATE as a percentage of the slashed amount. SLASH_BOUNTY_RATE is a genesis parameter.

---

## 6.12 Genesis Parameters Summary

The following parameters are defined at genesis and cannot be changed:

| Parameter | Description |
|---|---|
| MIN_STAKE | Minimum CMTY required to register as a staker |
| STAKE_LOCK_PERIOD | Number of blocks a staking commitment remains locked |
| PANEL_SIZE | Number of stakers selected per voting panel |
| SUPERMAJORITY_THRESHOLD | Minimum APPROVE votes required to finalize a block |
| VOTE_WINDOW_BLOCKS | Number of blocks panel members have to submit votes |
| SLASH_BOUNTY_RATE | Percentage of slashed stake paid to the slash reporter |

Exact values for all genesis parameters are determined during the final specification review period and published with the genesis block. They are not changed after publication.
