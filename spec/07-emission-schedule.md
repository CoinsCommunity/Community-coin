# Section 7: Emission Schedule and Block Reward Split

## 7.1 Overview

Community uses a two-phase emission model. Phase 1 distributes coins broadly through active mining participation using a smooth exponential decay curve. Phase 2 is a permanent fixed tail emission that keeps miners incentivized indefinitely. There is no hard supply cap. The network never reaches zero block rewards.

---

## 7.2 Definitions

**Base block reward** — the total number of coins created by the protocol in a given block, before splitting between destinations. The base block reward varies by phase and block height.

**PHASE_1_END_BLOCK** — the block height at which Phase 1 ends and Phase 2 begins. A genesis parameter.

**R_0** — the base block reward at block height 1, the starting reward. A genesis parameter.

**TAIL_EMISSION** — the fixed base block reward paid in every Phase 2 block. A genesis parameter.

**DECAY_FACTOR** — the per-block multiplier applied to the Phase 1 reward curve. A value between 0 and 1. A genesis parameter.

---

## 7.3 Phase 1: Initial Distribution

Phase 1 runs from block 1 through block PHASE_1_END_BLOCK inclusive.

The base block reward at block height n during Phase 1 is:

R(n) = TAIL_EMISSION + (R_0 - TAIL_EMISSION) * DECAY_FACTOR^n

where n is the block height counted from 1, and DECAY_FACTOR^n means DECAY_FACTOR raised to the power n.

At block 1 the reward approaches R_0. As n increases the reward decays smoothly toward TAIL_EMISSION. The curve never drops below TAIL_EMISSION during Phase 1 because DECAY_FACTOR is positive and (R_0 - TAIL_EMISSION) is positive.

DECAY_FACTOR is chosen so that by block PHASE_1_END_BLOCK the computed reward differs from TAIL_EMISSION by less than one indivisible unit of CMTY. The transition to Phase 2 is therefore imperceptible in practice.

There are no step changes, halvings, or discontinuities in the Phase 1 reward curve. The reward decreases by a consistent factor from one block to the next throughout Phase 1.

---

## 7.4 Phase 2: Permanent Tail Emission

Phase 2 begins at block PHASE_1_END_BLOCK + 1 and runs forever.

The base block reward at every Phase 2 block is:

R(n) = TAIL_EMISSION

TAIL_EMISSION is fixed and does not decrease over time. It is the same value at block PHASE_1_END_BLOCK + 1 as it is at block 10,000,000,000.

The annual inflation rate produced by TAIL_EMISSION decreases every year because total supply grows while annual emission remains constant. The inflation rate approaches zero asymptotically but never reaches it. Mining remains economically viable indefinitely because the base reward never goes to zero.

---

## 7.5 Block Reward Split

Every base block reward is divided among three destinations before being paid out. The split ratios are genesis parameters.

| Destination | Parameter | Description |
|---|---|---|
| Miner | MINER_REWARD_SHARE | Fraction paid to the miner who found the PoW solution |
| Stakers | STAKER_REWARD_SHARE | Fraction paid to the staking panel that approved the block |
| Faucet pool | FAUCET_POOL_RATE | Fraction allocated to the faucet pool (see Section 8) |

These three shares must sum to exactly 1:

MINER_REWARD_SHARE + STAKER_REWARD_SHARE + FAUCET_POOL_RATE = 1

MINER_REWARD_SHARE is the largest of the three shares, reflecting the greater resource commitment of proof-of-work mining.

STAKER_REWARD_SHARE is smaller than MINER_REWARD_SHARE but meaningful enough to incentivize continuous staker participation and availability.

---

## 7.6 Miner Reward Calculation

The miner reward for block n is:

MINER_REWARD(n) = floor(R(n) * MINER_REWARD_SHARE)

where floor() rounds down to the nearest indivisible unit of CMTY.

The miner who finds the valid proof-of-work solution for block n receives MINER_REWARD(n) in the coinbase transaction of block n. The miner reward is paid unconditionally once the PoW solution is valid, regardless of staker vote outcome. However, if the staker panel rejects the block the entire block including the coinbase transaction is invalid and the miner receives nothing. This is defined further in Section 6.

---

## 7.7 Staker Reward Calculation

The staker reward pool for block n is:

STAKER_POOL(n) = floor(R(n) * STAKER_REWARD_SHARE)

STAKER_POOL(n) is divided equally among the members of the staking panel that approved block n. The staking panel composition is defined in Section 6. Each panel member receives:

STAKER_INDIVIDUAL_REWARD(n) = floor(STAKER_POOL(n) / PANEL_SIZE)

where PANEL_SIZE is the number of stakers in the voting panel as defined in Section 6.

Rounding losses from floor division on STAKER_POOL(n) are destroyed and do not accumulate. Rounding losses from floor division on STAKER_INDIVIDUAL_REWARD(n) are similarly destroyed.

Stakers who were selected for the panel but voted to reject a block receive no reward for that block. Only stakers whose approval vote contributed to a supermajority approval receive the individual reward. The behavior of stakers who fail to vote within the required window is defined in Section 6.

---

## 7.8 Faucet Pool Allocation

The faucet pool allocation for block n is:

FAUCET_ALLOCATION(n) = floor(R(n) * FAUCET_POOL_RATE)

This amount is transferred to the faucet pool automatically with every confirmed block. The faucet pool allocation behavior, drop triggering, and claiming rules are defined in Section 8.

---

## 7.9 Transaction Fees

Transaction fees are paid by users submitting transactions. Every transaction specifies a fee amount. The minimum acceptable fee is a network-enforced parameter defined in Section 3.

Transaction fees collected in block n are split between the miner and the staking panel as follows:

| Destination | Parameter | Description |
|---|---|---|
| Miner | FEE_MINER_SHARE | Fraction of total fees in block n paid to the miner |
| Stakers | FEE_STAKER_SHARE | Fraction of total fees in block n split among the approving panel |

FEE_MINER_SHARE + FEE_STAKER_SHARE = 1

FEE_MINER_SHARE and FEE_STAKER_SHARE are genesis parameters.

The miner fee payment is:

MINER_FEE(n) = floor(TOTAL_FEES(n) * FEE_MINER_SHARE)

where TOTAL_FEES(n) is the sum of all transaction fees included in block n.

The staker fee pool is:

STAKER_FEE_POOL(n) = floor(TOTAL_FEES(n) * FEE_STAKER_SHARE)

STAKER_FEE_POOL(n) is divided equally among the approving staking panel members using the same floor division logic as the base staker reward.

Transaction fees are not allocated to the faucet pool. The faucet pool is funded exclusively by the base block reward allocation.

Claim transactions as defined in Section 8 carry no fee. They are exempt from minimum fee requirements.

---

## 7.10 Total Block Payment

The total coins paid out in block n, including base reward and fees, is:

TOTAL_PAYOUT(n) = MINER_REWARD(n) + MINER_FEE(n) + STAKER_POOL(n) + STAKER_FEE_POOL(n) + FAUCET_ALLOCATION(n)

The net new supply created in block n is:

NEW_SUPPLY(n) = R(n) + TOTAL_FEES(n) - TOTAL_FEES(n)
              = R(n)

Transaction fees are not new supply. They are collected from existing coin and redistributed. Only the base block reward R(n) represents new coins entering circulation. The faucet pool allocation is part of R(n) and represents coins created in that block that are held in the faucet pool rather than immediately distributed.

---

## 7.11 Rounding and Precision

All reward calculations use floor division to the smallest indivisible unit of CMTY. Fractions smaller than one indivisible unit are destroyed rather than accumulated or carried forward.

The coinbase transaction in each block must exactly match the computed miner reward and staker rewards for that block height. A block whose coinbase transaction pays more than the computed reward is invalid and rejected by the network regardless of all other validity conditions.

---

## 7.12 Genesis Parameters Summary

The following parameters are defined at genesis and cannot be changed:

| Parameter | Description |
|---|---|
| R_0 | Base block reward at block height 1 |
| TAIL_EMISSION | Fixed base block reward in Phase 2, permanent |
| DECAY_FACTOR | Per-block decay multiplier for Phase 1 reward curve |
| PHASE_1_END_BLOCK | Block height at which Phase 2 begins |
| MINER_REWARD_SHARE | Fraction of base reward paid to miner |
| STAKER_REWARD_SHARE | Fraction of base reward split among approving stakers |
| FAUCET_POOL_RATE | Fraction of base reward allocated to faucet pool |
| FEE_MINER_SHARE | Fraction of block transaction fees paid to miner |
| FEE_STAKER_SHARE | Fraction of block transaction fees split among approving stakers |

Exact values for all genesis parameters are determined during the final specification review period and published with the genesis block. They are not changed after publication.
