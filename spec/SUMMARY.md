# Community (CMTY) — Technical Summary

A two-minute overview for cryptographers and protocol designers. The full specification is in this directory. The whitepaper is in `../docs/`.

## What this is

A specification for a privacy-focused cryptocurrency built entirely on post-quantum primitives. It is design-first: the complete protocol is written before any implementation, so the cryptography can be reviewed before code exists. There is no company, no funding, no token, and no premine. The protocol is intended to be immutable at deployment, with no upgrade mechanism, which sets the correctness bar high and is the reason the specification is being scrutinized this early.

## Design constraints

- Post-quantum on every layer from genesis. No primitive may rest on the hardness of discrete log or integer factorization.
- Mandatory privacy. Sender, receiver, and amount concealed by default, no transparent mode.
- Hybrid PoW/PoS consensus. An attacker must control both majority hashrate and majority stake simultaneously.
- No governance. Immutable on deploy.

## Cryptographic stack

| Layer | Choice | Status |
|---|---|---|
| Hash | BLAKE3 | settled |
| Signatures | FALCON-1024 (FIPS 206) | settled |
| Key encapsulation | ML-KEM-1024 / Kyber (FIPS 203) | settled |
| Sender privacy | post-quantum membership proof | open |
| Amount hiding | post-quantum homomorphic commitment + range proof | open |
| Key image / linking tag | post-quantum, deterministic, unlinkable | open |

The signature and KEM layers are standard. The privacy layer is where review is most needed.

## The three open problems

**1. Sender privacy at full-chain scale (hardest).** Ring signatures are obsolete; Monero moved to Full-Chain Membership Proofs (FCMP++) in January 2026, built on curve trees over an elliptic curve cycle, which is not post-quantum for funds security. The question is whether a post-quantum analog exists: a lattice or hash based proof of membership in a set of millions of outputs with proof size in the low kilobytes. Lattice accumulators with ZK membership exist (Libert, Ling, Nguyen, Wang) but run to hundreds of KB at chain scale. Recursive lattice arguments, LaBRADOR (CRYPTO 2023) and Greyhound (CRYPTO 2024), are the most promising direction as a possible analog of curve-cycle recursion, but a practical full-chain construction is unsolved.

**2. Key image / linking tag.** The standard key image uses the discrete-log relation between private and public key, which does not exist in a lattice setting. A replacement must be deterministic per output, unlinkable across spends from the same wallet, and unforgeable without the private key. Candidate under consideration: a hash-based linking tag, with unlinkability requiring formal analysis.

**3. Amount hiding (most tractable).** Pedersen commitments are not post-quantum. A replacement must be hiding, binding, additively homomorphic (for input/output balance checks without opening), and admit an efficient range proof. Strongest prior art: MatRiCT (Esgin et al., CCS 2019) and MatRiCT+ (IEEE S&P 2022), implemented post-quantum lattice RingCT protocols with a lattice commitment and range proofs. Requires independent review before adoption.

## Where to engage

The open problems above are where outside expertise is most valuable. The full specification is in this directory, section by section. Objections, proposed constructions, and references to relevant work are all welcome. Findings are published in full.
