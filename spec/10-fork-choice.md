# Section 10: Fork Choice Rules

## 10.1 Overview

Fork choice rules define which chain a node considers canonical when two or more competing chains exist. In Community, forks can arise from network partitions, near-simultaneous block discoveries, or deliberate attacks. The fork choice rules are deterministic: every honest node applying them to the same chain data reaches the same conclusion about which chain is canonical.

---

## 10.2 Definitions

**Canonical chain** — the chain a node considers authoritative. Transactions and blocks not on the canonical chain are ignored for balance and state purposes.

**Chain weight** — the cumulative proof-of-work difficulty of all finalized blocks in a chain. Only finalized blocks contribute to chain weight. Candidate blocks awaiting staker approval do not contribute.

**Finalized block** — a block that has received supermajority staker approval as defined in Section 6. A block with a valid PoW solution that has not yet received supermajority approval is a candidate block, not a finalized block.

**Tip** — the most recent finalized block in a chain.

**Orphan** — a finalized block that is not part of the canonical chain because a competing chain of greater weight exists.

**Fork point** — the most recent block that appears in both competing chains. All blocks before the fork point are shared history.

---

## 10.3 Primary Fork Choice Rule

When a node observes two or more competing chains, it selects the canonical chain by the following rule:

The canonical chain is the chain with the greatest cumulative proof-of-work difficulty among all finalized blocks from the genesis block through the tip.

Formally, for a chain C, define:

CHAIN_WEIGHT(C) = sum of BLOCK_DIFFICULTY(B) for every finalized block B in C

where BLOCK_DIFFICULTY(B) is the inverse of the difficulty target T of block B, defined as:

BLOCK_DIFFICULTY(B) = 2^256 / T(B)

A lower difficulty target T represents more work required, so a lower T produces a higher BLOCK_DIFFICULTY value. This ensures that blocks mined under stricter difficulty targets contribute more weight to the chain.

The node adopts the chain C* where:

CHAIN_WEIGHT(C*) >= CHAIN_WEIGHT(C) for all competing chains C

If two chains have identical cumulative weight, the node retains its current canonical chain without switching. Ties do not cause a chain reorganization.

---

## 10.4 Finalization Requirement

Only finalized blocks count toward chain weight. A chain consisting of blocks with valid PoW solutions but no staker approval does not accumulate chain weight regardless of how much computation was expended mining them.

This rule prevents a class of attacks where an attacker mines a long chain in private and then broadcasts it to trigger a reorganization. In a pure PoW system a privately mined chain accumulates weight silently. In Community a privately mined chain has no staker approvals and therefore zero chain weight. An attacker cannot pre-accumulate a competing chain without engaging the staker population at each block.

---

## 10.5 Chain Reorganization

When a node receives a competing chain with greater weight than its current canonical chain, it reorganizes:

1. Identify the fork point, the most recent block shared by both chains.
2. Roll back all blocks from the current tip back to the fork point. Transactions in rolled-back blocks that are not present in the new canonical chain are returned to the mempool as unconfirmed.
3. Apply all blocks from the fork point through the tip of the new canonical chain in order.
4. Update all balances, staking state, faucet pool state, and difficulty to reflect the new canonical chain.

A reorganization is valid only if every block in the new canonical chain is a finalized block. A reorganization that would install a chain containing any unfinalized block is rejected regardless of claimed chain weight.

---

## 10.6 Depth Limit on Reorganizations

Reorganizations are limited to a maximum depth of REORG_LIMIT blocks from the current tip. REORG_LIMIT is a genesis parameter.

A competing chain that would require rolling back more than REORG_LIMIT blocks from the current tip is rejected regardless of its chain weight. The node continues treating its existing canonical chain as authoritative.

This limit provides a practical guarantee of settlement finality. A transaction confirmed more than REORG_LIMIT blocks deep is treated as settled. Merchants and users can rely on this depth for high-value transactions.

The REORG_LIMIT does not protect against an attacker who controls both sufficient hashrate and sufficient stake to build a competing finalized chain of greater weight. It protects against probabilistic reorganizations from network partitions and natural forks. Defense against a majority attacker requires the economic security properties described in Section 5 and Section 6.

---

## 10.7 Handling Candidate Blocks During Forks

When two candidate blocks are broadcast at the same height before either is finalized, nodes hold both in memory and wait for staker votes.

The first candidate block to achieve supermajority staker approval at that height becomes the finalized block. The other candidate block is discarded. Votes submitted for the losing candidate block after the winning block is finalized are ignored.

If staker votes are split across two candidate blocks at the same height and neither reaches supermajority approval within the voting window, both are rejected. The height remains unfinalized. Miners compete to produce a new candidate block at that height. This process repeats until a candidate block achieves supermajority approval.

---

## 10.8 Genesis Block

The genesis block is the root of the only valid chain. Its hash is hardcoded in all compliant implementations. Any chain that does not descend from the genesis block is not a valid Community chain regardless of its chain weight or staker approvals.

---

## 10.9 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| REORG_LIMIT | Maximum blocks deep a reorganization may reach from the current tip |

The genesis block hash is not a tunable parameter. It is the hash of the actual genesis block and is fixed at launch.
