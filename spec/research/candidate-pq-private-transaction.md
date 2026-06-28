# Candidate construction: post-quantum private transactions with full-chain anonymity

Status: strawman for review. This is not part of the normative specification. Nothing here has a security proof, an implementation, or chosen parameters. It is a proposal stitched together from existing, reviewed components and written down so that people who know more than its authors can take it apart. If you find a flaw, the document did its job.

## What it is trying to do

The goal is one private spend that hides three things at once and survives a quantum computer. It hides how much moved, it hides who sent it among every output that has ever existed on the chain rather than a small ring of decoys, and it stops the same coin from being spent twice. Everything rests on lattice assumptions, so there is no elliptic curve anywhere in the privacy path for Shor's algorithm to break.

The model to keep in mind is Monero's current FCMP++ design, which gets full-chain anonymity from curve trees over a cycle of elliptic curves. That construction is not post-quantum, because the curve work falls to Shor. The question this candidate is built around is whether the same shape can be rebuilt on lattices.

## The statement a spender proves

When you spend, you publish a single zero-knowledge proof of the following claim, and nothing else:

There is an output somewhere in the set of all outputs ever created such that I know the secret key that controls it, the nullifier attached to this transaction is correctly derived from that output and my secret, and the amounts committed in my inputs equal the amounts committed in my outputs.

A verifier learns that the claim is true. It does not learn which output you spent, who you are, or any amount.

## The components, and how settled each one is

The amount layer is the most settled. It can use the lattice commitment and range proof from MatRiCT and MatRiCT+, which are additively homomorphic so the network can check that inputs and outputs balance without opening anything, and which already have a working implementation. This is the lowest-risk part of the design.

The nullifier, the tag that prevents double spends, can be a deterministic hash of the spender's secret and the output being spent. It is the same every time that specific coin would be spent, and it reveals nothing about which coin produced it. Being hash based, it is quantum safe. The open piece is proving that two nullifiers from the same wallet cannot be linked to each other, which needs formal analysis rather than a hand wave.

The accumulator is the structure that compresses every output on the chain into one short root, so a spender can later prove membership against it. The candidate uses a Merkle tree built with a hash chosen to be cheap to reason about inside the lattice proof system.

The membership proof is the hard part and the reason this document exists. The spender has to prove, in zero knowledge, that they know a path from their hidden leaf up to the public root. Done naively over a lattice accumulator, that proof runs to hundreds of kilobytes for a set of a hundred million leaves, which is far too large to put on a chain millions of times a day. The bet here is to compress it using recent recursive lattice arguments, LaBRADOR and the Greyhound polynomial commitment built on top of it, in roughly the role that curve-cycle recursion plays for curve trees.

## Where the risk actually lives

Three joints are where this could fall apart, and they are the things reviewers should attack first.

The first is whether the membership proof can actually be made small. The whole design only matters if a LaBRADOR or Greyhound based proof of a Merkle path over about a hundred million leaves lands in the low kilobytes with a prover cost a normal machine can bear. If it cannot, the full-chain target fails and the design falls back to a ring.

The second is composition. The amount machinery from MatRiCT and the membership argument have to live in compatible settings and join into one proof without leaking anything at the seam. A construction can be sound in two halves and broken where they meet.

The third is the nullifier. A plain hash of secret and output may be enough for unlinkability and binding, or it may need a more structured pseudorandom function. This needs to be argued, not assumed.

## Questions for reviewers

1. Can a LaBRADOR or Greyhound based membership proof over roughly 10^8 leaves realistically reach the low-kilobyte range, and what prover time would that take in practice?
2. Which lattice-friendly hash for the accumulator minimizes the cost of proving a path inside the proof system?
3. Do the MatRiCT or MatRiCT+ commitment setting and the membership argument compose cleanly in one statement, or is there an impedance mismatch between their underlying rings or assumptions?
4. Is a deterministic hash of secret and output sufficient for an unlinkable, binding nullifier in this setting, or is a dedicated PRF required?
5. Is this even the right overall shape? Would a Seraphis-style transaction protocol paired with a post-quantum proof system dominate this approach?

## What this does not claim

It does not claim to be secure. There is no security proof, no parameter set, and no implementation. It is a starting point for people to break, and a finding that it is broken is as useful as a finding that some part of it holds.

## References

- MatRiCT: Efficient, Scalable and Post-Quantum Blockchain Confidential Transactions Protocol. Esgin, Zhao, Steinfeld, Liu, Liu. CCS 2019. eprint 2019/1287.
- MatRiCT+: More Efficient Post-Quantum Private Blockchain Payments. IEEE S&P 2022. eprint 2021/545.
- LaBRADOR: Compact Proofs for R1CS from Module-SIS. Beullens, Seiler. CRYPTO 2023.
- Greyhound: Fast Polynomial Commitments from Lattices. CRYPTO 2024. eprint 2024/1293.
- Zero-Knowledge Arguments for Lattice-Based Accumulators. Libert, Ling, Nguyen, Wang.
- Curve Trees, for the elliptic-curve construction this is trying to replace. eprint 2022/756.
