# Formal Protocol Specification

This directory contains the formal specification of the Community protocol.

The specification is written before any code exists. It defines every rule, every parameter, every cryptographic construction precisely enough that an independent team could implement Community from scratch using only this document. No trust required. Read it and decide for yourself.

All ten sections are drafted. Seven are complete. Three (sections 3, 4, and 9) contain open cryptographic design questions that require independent expert review before they can be finalized. Those open problems are documented explicitly in the sections themselves and summarized below.

---

## Specification Index

- [x] 1. Network and peer discovery
- [x] 2. Block structure and validation
- [x] 3. Transaction structure and validation (draft, depends on section 9 resolution)
- [x] 4. Privacy model — ring signatures, stealth addresses, confidential transactions (draft, open problems marked)
- [x] 5. Consensus — PoW algorithm, difficulty adjustment
- [x] 6. Consensus — PoS staker selection, voting, finality
- [x] 7. Emission schedule and block reward split
- [x] 8. Faucet system
- [x] 9. Cryptographic primitives — FALCON, CRYSTALS-Kyber (draft, open problems marked)
- [x] 10. Fork choice rules

---

## Open Problems

Three cryptographic design questions are unresolved. These are genuinely hard problems, not gaps from carelessness. Anyone who has worked in post-quantum cryptography will recognize them immediately.

The first is post-quantum ring signatures. Standard constructions like MLSAG and CLSAG are built on elliptic curve discrete log assumptions, which Shor's algorithm breaks. A lattice-based replacement is needed that preserves ring membership proofs, key image binding, and signer anonymity. See sections 4 and 9.

The second is the key image construction. The standard approach derives key images using the discrete log relationship between a private key and its public key. That relationship does not exist in a lattice setting. The replacement needs to be uniquely deterministic per output, unlinkable across different spends from the same wallet, and unforgeable without the private key. See section 9.6.

The third is the commitment scheme for confidential transactions. Pedersen commitments are not quantum resistant. Their hiding property depends on discrete log hardness. A replacement needs to be hiding, binding, homomorphic so transaction balances can be verified without revealing amounts, and must support efficient range proofs. This is the hardest of the three. See sections 4.4 and 9.7.

---

## How to Contribute

Read the whitepaper first: [../docs/community-whitepaper.md](../docs/community-whitepaper.md)

If you are a cryptographer, the three open problems above are where outside expertise is most needed. Read the relevant sections, open an issue with proposed constructions, counterarguments, or papers worth reading. There is no bad contribution here. A finding that proves the current approach is wrong is as valuable as one that proves it right.

Everything is public. Every finding gets published in full. That is not a policy. It is the only way this gets built correctly.
