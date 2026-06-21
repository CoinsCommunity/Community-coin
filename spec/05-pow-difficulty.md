# Section 5: Consensus — PoW Algorithm and Difficulty Adjustment

## 5.1 Overview

Community's Proof-of-Work layer determines who may propose blocks. It uses a memory-hard, CPU-optimized mining algorithm designed to make specialized mining hardware economically unattractive. Difficulty adjusts every block using a linearly weighted moving average that responds quickly to real hashrate changes while resisting manipulation.

---

## 5.2 Definitions

**Hash function** — the cryptographic hash function used to produce the PoW output. Defined in Section 9.

**Mining algorithm** — the full computational procedure a miner executes to produce a valid PoW solution. Defined in Section 5.3.

**Difficulty** — a 256-bit target value T. A PoW solution is valid if and only if the algorithm output interpreted as an unsigned 256-bit integer is less than T. Lower T means higher difficulty.

**Solve time** — the elapsed time in seconds between the timestamp of a block and the timestamp of its parent block.

**Target block time** — the desired average solve time in seconds. A genesis parameter denoted TARGET_BLOCK_TIME.

**LWMA window** — the number of recent blocks used in the difficulty adjustment calculation. A genesis parameter denoted LWMA_WINDOW.

---

## 5.3 Mining Algorithm

Community's mining algorithm is a memory-hard, CPU-optimized proof of work built on the following design requirements. The precise algorithm construction is specified in a companion cryptographic document subject to independent review before genesis. This section defines the requirements the algorithm must satisfy.

**Requirement 1 — Large random-access dataset.** The algorithm operates over a dataset large enough that it must reside in CPU L3 cache to execute at full speed. The minimum dataset size is DATASET_SIZE, a genesis parameter. Implementations that reduce the dataset to fit in smaller memory experience proportional performance degradation. This creates a structural advantage for commodity CPUs over ASICs, which have limited cache memory relative to their compute throughput.

**Requirement 2 — Unpredictable memory access pattern.** The sequence of memory addresses accessed during algorithm execution must be determined by intermediate computation results rather than fixed at compile time. An observer cannot precompute or prefetch memory accesses without executing the algorithm.

**Requirement 3 — Sequential dependency.** Each step of the algorithm depends on the output of the previous step. Steps cannot be parallelized within a single algorithm execution. This prevents speedups from pipelining on specialized hardware.

**Requirement 4 — Dataset regeneration.** The dataset is regenerated periodically based on block height. The regeneration interval is DATASET_REGEN_INTERVAL, a genesis parameter measured in blocks. A hardware implementation that caches the dataset must update it at each regeneration event. This raises the cost of hardware optimization by preventing indefinite reuse of a fixed dataset.

**Requirement 5 — Deterministic and verifiable.** Given a block header and a nonce, any node can verify a PoW solution in substantially less time than it takes to find one. Verification must not require the full dataset.

**Input to the algorithm.** The miner runs the algorithm over:

MINING_INPUT = HASH(block_header || nonce)

where block_header is the serialized candidate block header as defined in Section 2, nonce is a 64-bit value the miner varies to search for a valid solution, and HASH is the protocol hash function defined in Section 9.

**Valid solution.** A nonce value produces a valid PoW solution if:

MINING_OUTPUT(MINING_INPUT) < CURRENT_DIFFICULTY

where MINING_OUTPUT is the output of the memory-hard algorithm applied to MINING_INPUT and CURRENT_DIFFICULTY is the difficulty target for that block height as computed by Section 5.5.

---

## 5.4 Block Header for Mining

The block header fields committed to by the miner are defined in Section 2. For PoW purposes the relevant fields are:

- Previous block hash
- Merkle root of transactions
- Timestamp
- Difficulty target
- Nonce

The miner varies the nonce field while all other header fields remain fixed. If the full 64-bit nonce space is exhausted without finding a valid solution the miner may update the timestamp or reconstruct the transaction set to produce a new Merkle root and continue searching.

---

## 5.5 Difficulty Adjustment

Difficulty adjusts after every block using a Linearly Weighted Moving Average over the most recent LWMA_WINDOW blocks.

**Purpose of LWMA.** Simple moving averages weight all recent blocks equally, making them slow to respond when hashrate changes suddenly. Linear weighting gives more recent blocks proportionally higher weight, allowing the algorithm to track genuine hashrate changes quickly while smoothing over statistical noise in individual solve times.

**Timewarp resistance.** LWMA prevents the timewarp attack by capping the influence any single block timestamp can have on the difficulty calculation. Miners cannot manipulate difficulty by artificially backdating block timestamps because the algorithm's linear weighting and timestamp clamping limit the achievable distortion.

**Algorithm.**

Let B(i) denote the block at height i, with timestamp TS(i) in Unix seconds.

Let N = LWMA_WINDOW.

For the block at height H, collect the N most recently finalized blocks: B(H-N) through B(H-1).

Compute the solve time for each block, clamped to prevent manipulation:

ST(i) = clamp(TS(i) - TS(i-1), 1, MAX_SOLVE_TIME)

where MAX_SOLVE_TIME = 6 * TARGET_BLOCK_TIME and clamp(x, lo, hi) returns lo if x < lo, hi if x > hi, and x otherwise. A minimum solve time of 1 second prevents division-by-zero and timestamp collisions. A maximum solve time of 6 * TARGET_BLOCK_TIME limits the influence of blocks with abnormally long gaps caused by hashrate drops or timestamp manipulation.

Compute the linearly weighted sum of solve times:

WEIGHTED_SUM = sum over i from 1 to N of (i * ST(H-N+i-1))

Compute the sum of weights:

WEIGHT_TOTAL = N * (N + 1) / 2

Compute the weighted average solve time:

WEIGHTED_AVG = WEIGHTED_SUM / WEIGHT_TOTAL

Compute the new difficulty target:

NEW_TARGET = PREV_TARGET * TARGET_BLOCK_TIME / WEIGHTED_AVG

where PREV_TARGET is the difficulty target of block B(H-1).

Clamp the new target to prevent runaway difficulty changes in a single adjustment:

NEW_TARGET = clamp(NEW_TARGET, PREV_TARGET / MAX_ADJUSTMENT, PREV_TARGET * MAX_ADJUSTMENT)

where MAX_ADJUSTMENT is a genesis parameter limiting how much difficulty can change from one block to the next. A value of 2 means difficulty can at most double or halve per block.

Clamp the new target to the valid difficulty range:

NEW_TARGET = clamp(NEW_TARGET, MIN_DIFFICULTY, MAX_TARGET)

where MIN_DIFFICULTY is the minimum allowed difficulty (preventing trivially easy mining) and MAX_TARGET is 2^256 - 1 (the maximum possible target, representing minimum difficulty). Both are genesis parameters.

The resulting NEW_TARGET is the difficulty target that the miner of block H must satisfy.

---

## 5.6 Timestamp Rules

Block timestamps must satisfy the following rules for a block to be valid:

1. The timestamp must be greater than the median timestamp of the previous MEDIAN_WINDOW blocks. MEDIAN_WINDOW is a genesis parameter. This prevents miners from backdating blocks to manipulate difficulty.

2. The timestamp must not be more than MAX_FUTURE_TIME seconds ahead of the validating node's local clock. MAX_FUTURE_TIME is a genesis parameter. This prevents miners from far-future-dating blocks to manipulate difficulty.

A block with a timestamp that violates either rule is invalid and rejected. Nodes do not relay invalid blocks.

---

## 5.7 Genesis Block

The genesis block is mined with an initial difficulty of GENESIS_DIFFICULTY, a genesis parameter chosen to produce an expected solve time of approximately TARGET_BLOCK_TIME on modest commodity hardware available at launch. The genesis block difficulty is not computed by the LWMA algorithm. All subsequent blocks use LWMA.

The genesis block timestamp is set at the moment of mining and is not adjustable after the fact.

---

## 5.8 Relationship to PoS Layer

A valid PoW solution is necessary but not sufficient for a block to be added to the chain. The block must also receive supermajority staker approval as defined in Section 6. A miner who finds a valid PoW solution and broadcasts a candidate block that is subsequently rejected by stakers receives no reward. The PoW solution is discarded.

Difficulty adjusts based on finalized blocks only. Rejected candidate blocks do not contribute to the LWMA window and do not affect difficulty.

---

## 5.9 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| TARGET_BLOCK_TIME | Target average seconds between finalized blocks |
| LWMA_WINDOW | Number of recent blocks used in difficulty adjustment |
| MAX_ADJUSTMENT | Maximum factor by which difficulty can change per block |
| MAX_FUTURE_TIME | Maximum seconds a block timestamp may exceed node local time |
| MEDIAN_WINDOW | Number of recent blocks used for median timestamp check |
| MIN_DIFFICULTY | Minimum allowed difficulty target |
| MAX_TARGET | Maximum allowed difficulty target (2^256 - 1) |
| GENESIS_DIFFICULTY | Initial difficulty target for the genesis block |
| DATASET_SIZE | Minimum memory dataset size for the mining algorithm |
| DATASET_REGEN_INTERVAL | Blocks between mining dataset regenerations |
