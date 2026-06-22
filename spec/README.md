# Formal Protocol Specification

This directory contains the formal specification of the Community protocol.

The specification is written before any code exists. It defines every rule, every parameter, every cryptographic construction precisely enough that an independent team could implement Community from scratch using only this document. No trust required. Read it and decide for yourself.

All ten sections are drafted. Seven are complete. Three (sections 3, 4, and 9) contain open cryptographic design questions that require independent expert review before they can be finalized. Those open problems are documented explicitly in the sections themselves and summarized below.

---

## Specification Index

- [x] 1. Network and peer discovery
- [x] 2. Block structure and validation
- [x] 3. Transaction structure and validation (draft, depends on section 9 resolution)
- [x] 4. Privacy model — membership proofs, stealth addresses, confidential transactions (draft, open problems marked)
- [x] 5. Consensus — PoW algorithm, difficulty adjustment
- [x] 6. Consensus — PoS staker selection, voting, finality
- [x] 7. Emission schedule and block reward split
- [x] 8. Faucet system
- [x] 9. Cryptographic primitives — FALCON, CRYSTALS-Kyber (draft, open problems marked)
- [x] 10. Fork choice rules

---

## Open Problems

Three cryptographic design questions are unresolved. These are genuinely hard problems, not gaps from carelessness. Anyone who has worked in post-quantum cryptography will recognize them immediately.

The first is post-quantum sender privacy. Ring signatures (MLSAG, CLSAG) are obsolete. Monero replaced them in January 2026 with Full-Chain Membership Proofs (FCMP++), which hide a spend among the entire output history instead of a small ring of decoys. But FCMP++ is built on curve trees over an elliptic curve cycle, so it is still discrete-log based and not post-quantum for funds security. The open question is whether a post-quantum analog of a full-chain membership proof exists: lattice or hash based, proving membership in a set of millions of outputs with proof sizes in the low kilobytes. A lattice ring signature is the weaker fallback. This may be the hardest of the three problems, since curve trees get their efficiency from recursion over a curve cycle and there is no known practical lattice equivalent. See sections 4.2 and 9.

The second is the key image construction. The standard approach derives key images using the discrete log relationship between a private key and its public key. That relationship does not exist in a lattice setting. The replacement needs to be uniquely deterministic per output, unlinkable across different spends from the same wallet, and unforgeable without the private key. See section 9.6.

The third is the commitment scheme for confidential transactions. Pedersen commitments are not quantum resistant. Their hiding property depends on discrete log hardness. A replacement needs to be hiding, binding, homomorphic so transaction balances can be verified without revealing amounts, and must support efficient range proofs. This is the hardest of the three. See sections 4.4 and 9.7.

---

## How to Contribute

Read the whitepaper first: [../docs/community-whitepaper.md](../docs/community-whitepaper.md)

If you are a cryptographer, the three open problems above are where outside expertise is most needed. Read the relevant sections, open an issue with proposed constructions, counterarguments, or papers worth reading. There is no bad contribution here. A finding that proves the current approach is wrong is as valuable as one that proves it right.

Everything is public. Every finding gets published in full. That is not a policy. It is the only way this gets built correctly.
