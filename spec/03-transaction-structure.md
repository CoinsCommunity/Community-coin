# Section 3: Transaction Structure and Validation

> **DRAFT — Sections marked [OPEN PROBLEM] depend on unresolved cryptographic design
> questions in Section 4 and Section 9. Transfer transaction structure in particular
> cannot be finalized until the commitment scheme is resolved.**

---

## 3.1 Overview

A transaction is the fundamental unit of state change in Community. Transactions transfer coin between wallets, register and unregister staking commitments, record staker votes, claim faucet drops, and report protocol violations. Every transaction that modifies state must be included in a finalized block.

Transaction types are defined in Section 2.4. This section defines the structure and validity rules for each type.

---

## 3.2 Common Transaction Fields

Every transaction regardless of type begins with the following fields:

| Field | Size | Description |
|---|---|---|
| VERSION | 4 bytes | Transaction format version. Current value: 1 |
| TYPE | 1 byte | Transaction type identifier as defined in Section 2.4 |
| LOCK_HEIGHT | 8 bytes | Block height before which this transaction is invalid. 0 means no lock. |

LOCK_HEIGHT allows a sender to create a transaction that cannot be included in a block until a specific height. A transaction with LOCK_HEIGHT > current block height is not yet valid and is held in the mempool until the lock height is reached.

The transaction hash is:

TX_HASH = HASH(HASH(serialized_transaction))

TX_HASH uniquely identifies the transaction and is used in Merkle tree construction and in inventory announcements.

---

## 3.3 Transfer Transaction (TYPE 0x02)

The transfer transaction moves coin privately from one or more inputs to one or more outputs. It is the primary transaction type for sending Community between wallets.

**[OPEN PROBLEM — Structure depends on commitment scheme resolution in Section 9.7 and sender-privacy resolution in Section 4.2]**

The transfer transaction structure as described below is conditional on two unresolved design questions: the post-quantum commitment scheme (Section 9.7) and the post-quantum sender-privacy construction (Section 4.2). Field sizes and the exact membership-proof representation will change depending on the chosen constructions. The structure below assumes the full-chain membership proof model that Section 4.2 targets.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT |
| INPUT_COUNT | Number of inputs (1 or more) |
| INPUTS | Array of INPUT_COUNT input records |
| OUTPUT_COUNT | Number of outputs (1 or more) |
| OUTPUTS | Array of OUTPUT_COUNT output records |
| FEE_COMMITMENT | Commitment to the transaction fee amount |
| FEE_RANGE_PROOF | Range proof for the fee commitment |
| MEMBERSHIP_PROOFS | Array of INPUT_COUNT membership proofs, one per input |

**Input record structure:**

| Field | Description |
|---|---|
| ANONYMITY_SET_REF | Reference to the output-set commitment the membership proof is computed against |
| KEY_IMAGE | The key image (linking tag) for the output being spent |
| INPUT_COMMITMENT | Commitment to the input amount |

Under the full-chain membership proof model, ANONYMITY_SET_REF identifies the committed set of all eligible outputs the proof is made against, for example a tree root at a referenced block height, rather than an enumerated list of decoys. The true input being spent is one of the outputs in that set. Which one is concealed by the membership proof. If the lattice ring signature fallback in Section 4.2 is used instead, ANONYMITY_SET_REF enumerates the ring members and a MIN_RING_SIZE minimum applies.

**Output record structure:**

| Field | Description |
|---|---|
| ONE_TIME_PK | The stealth one-time public key for the recipient |
| KYBER_CIPHERTEXT | Kyber encapsulation ciphertext for the recipient's view key |
| OUTPUT_COMMITMENT | Commitment to the output amount |
| RANGE_PROOF | Range proof demonstrating the output amount is in [0, MAX_COIN_VALUE] |

**Validity rules for transfer transactions:**

1. INPUT_COUNT >= 1 and OUTPUT_COUNT >= 1.
2. Each input's ANONYMITY_SET_REF resolves to a valid output-set commitment on the canonical chain. Under the ring signature fallback, the enumerated set contains at least MIN_RING_SIZE entries.
3. Every output the membership proof depends on exists in finalized blocks on the canonical chain.
4. No KEY_IMAGE appears in the canonical chain's key image set or more than once in this transaction.
5. Every MEMBERSHIP_PROOF is valid for its input against the referenced anonymity set.
6. The balance equation holds: sum(INPUT_COMMITMENT) = sum(OUTPUT_COMMITMENT) + FEE_COMMITMENT.
7. Every RANGE_PROOF is valid for its corresponding commitment.
8. FEE_RANGE_PROOF is valid for FEE_COMMITMENT.
9. The fee amount committed to in FEE_COMMITMENT is at least MIN_RELAY_FEE.
10. LOCK_HEIGHT is either 0 or a future block height.

A transfer transaction that fails any of these rules is invalid and rejected. Invalid transactions are not relayed and not included in blocks.

**[END OPEN PROBLEM]**

---

## 3.4 Coinbase Transaction (TYPE 0x01)

The coinbase transaction distributes the block reward. One appears in every block as the first transaction. It has no inputs. It creates new coins.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT (must be 0) |
| BLOCK_HEIGHT | The height of the block this coinbase belongs to |
| MINER_OUTPUT | Output paying the miner reward |
| STAKER_OUTPUT_COUNT | Number of staker reward outputs |
| STAKER_OUTPUTS | Array of STAKER_OUTPUT_COUNT outputs, one per approving staker |
| FAUCET_OUTPUT | Output paying the faucet pool allocation |

Each output in the coinbase transaction is a plaintext output, not a confidential output. Coinbase outputs are not private. Block rewards are publicly visible. The coinbase outputs use the structure:

| Field | Description |
|---|---|
| RECIPIENT_ADDRESS | The wallet address receiving this output |
| AMOUNT | The plaintext coin amount |

A coinbase output cannot be spent or included in any anonymity set until it has matured. Coinbase outputs mature after COINBASE_MATURITY blocks. COINBASE_MATURITY is a genesis parameter.

**Validity rules for coinbase transactions:**

1. LOCK_HEIGHT is 0.
2. BLOCK_HEIGHT matches the height of the block containing this transaction.
3. MINER_OUTPUT.AMOUNT equals the miner reward computed by Section 7 for BLOCK_HEIGHT.
4. STAKER_OUTPUT_COUNT equals the number of stakers who submitted valid APPROVE votes that contributed to supermajority as defined in Section 6.
5. Each STAKER_OUTPUT.RECIPIENT_ADDRESS matches the STAKER_ADDRESS of a staker whose APPROVE vote is included in this block.
6. Each STAKER_OUTPUT.AMOUNT equals the individual staker reward computed by Section 7 for BLOCK_HEIGHT.
7. FAUCET_OUTPUT.AMOUNT equals the faucet pool allocation computed by Section 7 for BLOCK_HEIGHT.
8. The sum of all outputs does not exceed R(BLOCK_HEIGHT) as computed in Section 7.

---

## 3.5 Stake Transaction (TYPE 0x03)

The stake transaction locks coins into a staking commitment.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT |
| INPUT | A transfer-style input consuming the coins being staked |
| STAKE_AMOUNT | Plaintext amount being staked |
| STAKER_ADDRESS | The address receiving staking rewards |
| SIGNATURE | FALCON-1024 signature over the transaction by the spending key |

**Validity rules:**

1. STAKE_AMOUNT >= MIN_STAKE as defined in Section 6.
2. The INPUT is a valid unspent output controlled by the signing key.
3. SIGNATURE is a valid FALCON-1024 signature over the transaction by the key corresponding to the INPUT.
4. The coins being staked are not coinbase outputs that have not yet matured.

A confirmed stake transaction removes STAKE_AMOUNT coins from the sender's spendable balance and creates an active staking commitment under STAKER_ADDRESS for STAKE_LOCK_PERIOD blocks.

---

## 3.6 Unstake Transaction (TYPE 0x04)

The unstake transaction returns matured staking commitment coins to spendable balance.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT |
| COMMITMENT_TX_HASH | TX_HASH of the stake transaction being matured |
| RECIPIENT_ADDRESS | Address receiving the unstaked coins |
| SIGNATURE | FALCON-1024 signature over the transaction |

**Validity rules:**

1. COMMITMENT_TX_HASH references a confirmed stake transaction.
2. The stake transaction's lock period has expired: current block height >= stake confirmation height + STAKE_LOCK_PERIOD.
3. The stake transaction has not already been unstaked.
4. SIGNATURE is valid over the transaction by the key controlling STAKER_ADDRESS of the referenced stake transaction.

A confirmed unstake transaction transfers the staked coins to RECIPIENT_ADDRESS as a spendable balance.

---

## 3.7 Vote Transaction (TYPE 0x05)

The vote transaction records a staker's approval or rejection of a candidate block.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT (must be 0) |
| CANDIDATE_BLOCK_HASH | Hash of the candidate block being voted on |
| VOTE | 1 byte: 0x01 for APPROVE, 0x02 for REJECT |
| VOTER_ADDRESS | The staker address casting this vote |
| SIGNATURE | FALCON-1024 signature over the transaction |

**Validity rules:**

1. LOCK_HEIGHT is 0.
2. CANDIDATE_BLOCK_HASH references a pending candidate block.
3. VOTER_ADDRESS is a member of the voting panel for CANDIDATE_BLOCK_HASH as computed in Section 6.5.
4. No prior vote transaction from VOTER_ADDRESS for CANDIDATE_BLOCK_HASH has been confirmed.
5. The current block height is within the voting window for CANDIDATE_BLOCK_HASH.
6. SIGNATURE is a valid FALCON-1024 signature over the transaction by the key corresponding to VOTER_ADDRESS.
7. VOTE is either 0x01 or 0x02.

---

## 3.8 Claim Transaction (TYPE 0x06)

The claim transaction collects a faucet drop allocation.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT (must be 0) |
| DROP_ID | Identifier of the faucet drop being claimed |
| CLAIMING_ADDRESS | Wallet address receiving the drop payout |
| SIGNATURE | FALCON-1024 signature over the transaction |

**Validity rules:**

1. LOCK_HEIGHT is 0.
2. DROP_ID references a valid drop event as defined in Section 8.
3. The claiming window for DROP_ID is still open.
4. CLAIMING_ADDRESS has not previously submitted a valid claim for DROP_ID.
5. The faucet pool balance is sufficient for the drop tier payout.
6. SIGNATURE is a valid FALCON-1024 signature by a key controlling CLAIMING_ADDRESS.

Claim transactions carry no fee. They are exempt from MIN_RELAY_FEE.

---

## 3.9 Slash Transaction (TYPE 0x07)

The slash transaction reports a staker who submitted conflicting votes and claims the slash bounty.

**Structure:**

| Field | Description |
|---|---|
| Common fields | VERSION, TYPE, LOCK_HEIGHT |
| VOTE_TX_1 | Full serialized first vote transaction |
| VOTE_TX_2 | Full serialized second conflicting vote transaction |
| REPORTER_ADDRESS | Address receiving the slash bounty |
| SIGNATURE | FALCON-1024 signature over the transaction |

**Validity rules:**

1. VOTE_TX_1 and VOTE_TX_2 are both valid vote transactions.
2. Both vote transactions reference the same block height (conflicting votes at the same height).
3. Both vote transactions are signed by the same VOTER_ADDRESS.
4. VOTE_TX_1 and VOTE_TX_2 reference different CANDIDATE_BLOCK_HASH values (confirming they are conflicting votes, not duplicates).
5. The referenced VOTER_ADDRESS has an active staking commitment that has not already been slashed.
6. SIGNATURE is a valid FALCON-1024 signature by a key controlling REPORTER_ADDRESS.

A confirmed slash transaction destroys the staking commitment of the offending VOTER_ADDRESS and pays SLASH_BOUNTY_RATE of the slashed amount to REPORTER_ADDRESS.

---

## 3.10 Mempool

Unconfirmed transactions are held in each node's mempool pending inclusion in a block. The mempool has a maximum size of MAX_MEMPOOL_SIZE bytes. When the mempool is full, incoming transactions with fees below the current minimum are dropped.

Transactions are evicted from the mempool after MEMPOOL_EXPIRY blocks if not confirmed. Expired transactions are not rebroadcast and must be resubmitted by the originating wallet.

Miners select transactions from the mempool to include in candidate blocks, prioritizing by fee per byte. Claim transactions are included without fee priority and are placed after fee-paying transactions in block ordering.

MAX_MEMPOOL_SIZE and MEMPOOL_EXPIRY are configurable local node parameters.

---

## 3.11 Minimum Relay Fee

MIN_RELAY_FEE is the minimum fee per byte required for a node to relay a transfer transaction. Transactions below this threshold are not relayed and not included in blocks by conforming miners.

MIN_RELAY_FEE is a genesis parameter for the floor value. Nodes may set a higher local minimum but not lower than the genesis floor.

---

## 3.12 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| MIN_RELAY_FEE | Minimum fee per byte for transfer transaction relay |
| COINBASE_MATURITY | Blocks before a coinbase output may be spent or used as a decoy |
