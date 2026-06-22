# Section 4: Privacy Model

> **DRAFT — Requires independent cryptographic review before finalization.**
> Sections marked [OPEN PROBLEM] identify unresolved design questions requiring expert input.
> The sender-privacy and stealth address constructions described here are directionally correct
> but require formal security proofs and expert review before implementation.
>
> **Note (2026-06):** This section originally specified ring signatures for sender privacy.
> Ring signatures have been superseded in current privacy-coin design. Monero deprecated them
> in January 2026 in favor of Full-Chain Membership Proofs (FCMP++). Section 4.2 has been
> revised to reflect this and to reframe the open problem around a post-quantum membership
> proof rather than a post-quantum ring signature.

---

## 4.1 Overview

Every transaction on the Community network is private. Sender, receiver, and amount are all concealed by default. There is no transparency mode, no audit mode, and no mechanism by which any party can make a transaction visible to observers on the blockchain. Privacy is a property of the protocol, not a feature a user enables.

Three cryptographic constructions work together to achieve this. Each closes a distinct information channel.

A membership proof conceals the sender. Stealth addresses conceal the receiver. Confidential transactions conceal the amount. Together they leave a transaction as a black box on the blockchain: provably valid, provably not double-spent, provably not creating coins from nothing, while revealing nothing about who sent it, who received it, or how much moved.

The cryptographic primitives underlying these constructions are defined in Section 9.

---

## 4.2 Membership Proofs — Concealing the Sender

Sender privacy works by proving that the output being spent belongs to some set of outputs on the blockchain, without revealing which one. The larger that set, the stronger the anonymity. There are two families of construction for this, and the difference between them drives Community's design choice.

**Ring signatures (the older approach).** A ring signature combines the spender's output with a small set of decoy outputs pulled from the blockchain, and proves that one member of the ring authorized the spend without revealing which. The anonymity set is the ring size, typically a small fixed number such as 16. This is the approach Monero used through 2025. Its weakness is the small anonymity set, which leaves the construction exposed to statistical decoy-analysis over time.

**Full-chain membership proofs (the current approach).** Rather than proving membership in a small ring of decoys, the spender proves in zero knowledge that their output is one of every output that has ever existed on the chain. The anonymity set becomes the entire output history, on the order of hundreds of millions, instead of a handful of decoys. Monero deprecated ring signatures in January 2026 and replaced them with this approach, called FCMP++ (Full-Chain Membership Proofs). The construction is based on Curve Trees (eprint 2022/756) combined with Eagen's elliptic-curve divisor techniques (eprint 2022/596). It also provides forward secrecy: because the membership proof reveals nothing about which output was spent, an adversary who later gains the ability to solve the discrete logarithm problem still cannot retroactively deanonymize past spends.

Community targets the full-chain membership proof model rather than ring signatures, because a chain-wide anonymity set is strictly stronger than a fixed decoy ring and removes the decoy-selection problem entirely.

**[OPEN PROBLEM — Requires cryptographic review — sender-privacy layer]**

The current full-chain membership proof construction (FCMP++) is built on Curve Trees over a cycle of elliptic curves. Its security therefore rests on the discrete logarithm problem. The forward-secrecy property protects the privacy of past transactions against a future quantum adversary, which is valuable, but it does not make the construction post-quantum for funds security: the spend authorization built on top of it is still elliptic-curve based, so a quantum adversary could forge spends. For a protocol that must be post-quantum on every layer from genesis, FCMP++ cannot be adopted as-is.

The open problem is whether a post-quantum analog of a full-chain membership proof exists. The construction Community needs must satisfy:

1. Quantum resistance. Security based on lattice or hash hardness assumptions, not discrete log.
2. Large anonymity set. Ideally chain-wide membership rather than a small decoy ring.
3. Practical proof size. Proof size in the low kilobytes even when the membership set is in the millions, comparable to the few-kilobyte proofs FCMP++ achieves.
4. Spend authorization. A linking mechanism (see key image below) that prevents the same output from being spent twice, and is itself post-quantum.
5. Compatibility with MAX_BLOCK_SIZE as defined in Section 2.

Curve Trees achieve their small proof size partly through recursive proofs over an elliptic curve cycle. There is no known lattice equivalent of that recursion that is practical at chain scale. This means the sender-privacy layer may be a harder open problem than the amount-hiding layer in Section 4.4.

A lattice-based ring signature, with its smaller but still meaningful anonymity set, is the fallback if no post-quantum full-chain construction is practical. Lattice ring signatures exist in the literature but have large signature sizes that scale with the anonymity set, which limits how large the set can be at acceptable transaction sizes.

The specific construction is deferred to cryptographic review. This specification uses the notation MEMBERSHIP_PROOF(output_set, spent_output, SPEND_SK) to denote a valid proof that spent_output is in output_set, authorized by SPEND_SK, without revealing which member spent_output is.

[END OPEN PROBLEM]

**Key image.** Every spend includes a key image as defined in Section 9.6, also called a linking tag or nullifier. The key image is unique to the specific output being spent and is derived deterministically from the spend key and the output, so the same output always produces the same key image no matter what anonymity set it is proven against. The network maintains a global set of all key images that have appeared in finalized blocks. A transaction whose key image already appears in this set is a double-spend attempt and is rejected. The key image reveals nothing about which output in the anonymity set was spent.

---

## 4.3 Stealth Addresses — Concealing the Receiver

Stealth addresses ensure that the recipient of a transaction is never identifiable from blockchain data, even if the recipient's public address is publicly known.

Each Community wallet has a public address encoding two public keys: SPEND_PK and VIEW_PK, as defined in Section 9.5.

**Sending to a stealth address.** When a sender wants to send funds to a recipient with address ADDRESS:

1. The sender decodes ADDRESS to recover SPEND_PK and VIEW_PK.

2. The sender performs a Kyber-1024 encapsulation to the recipient's VIEW_PK:

   (CIPHERTEXT, SHARED_SECRET) = KYBER_ENCAPS(VIEW_PK)

3. The sender derives a one-time public key for this transaction:

   ONE_TIME_PK = HASH(SHARED_SECRET || tx_index) * G + SPEND_PK

   [OPEN PROBLEM: This derivation uses G, an elliptic curve generator point, which is not post-quantum. The one-time public key derivation must be replaced with a lattice-based construction. The requirement is that ONE_TIME_PK is uniquely derivable by the recipient but unlinkable to SPEND_PK or VIEW_PK by any observer. See Section 9 for resolution.]

4. The sender includes CIPHERTEXT in the transaction so the recipient can recover the shared secret.

5. Funds are sent to the ONE_TIME_PK address. The transaction output records ONE_TIME_PK and CIPHERTEXT. The published recipient address ADDRESS does not appear anywhere in the transaction.

**Receiving with a stealth address.** The recipient's wallet scans the blockchain:

1. For each transaction output, the wallet attempts to decapsulate the included CIPHERTEXT:

   SHARED_SECRET = KYBER_DECAPS(VIEW_SK, CIPHERTEXT)

2. The wallet derives the expected one-time key:

   EXPECTED_ONE_TIME_PK = HASH(SHARED_SECRET || tx_index) * G + SPEND_PK

3. If EXPECTED_ONE_TIME_PK matches the output's ONE_TIME_PK, this output belongs to the wallet.

4. The wallet records the output as received. The corresponding one-time private key for spending is derived using SPEND_SK.

This scanning process requires VIEW_SK but not SPEND_SK. A wallet owner may share VIEW_SK with a third party (an auditor, an exchange, a tax service) to allow them to identify incoming transactions without granting them spending authority. SPEND_SK is never required for scanning and never shared.

**Privacy property.** An observer watching the blockchain sees ONE_TIME_PK values and CIPHERTEXT values. Without VIEW_SK, the observer cannot determine which outputs belong to any given wallet. Without SPEND_SK, the observer cannot spend outputs even if they identify them. The recipient's published address ADDRESS never appears on the blockchain under normal operation.

---

## 4.4 Confidential Transactions — Concealing the Amount

Confidential transactions hide all transaction amounts while allowing the network to verify that no coins are created or destroyed.

**[OPEN PROBLEM — Requires cryptographic review — Primary open problem]**

The standard implementation of confidential transactions uses Pedersen commitments: COM(v, r) = v*G + r*H over an elliptic curve group. The homomorphic property of Pedersen commitments allows the network to verify sum(inputs) = sum(outputs) without knowing any individual amount. Range proofs (Bulletproofs) then prove each committed value is non-negative without revealing it.

Pedersen commitments are not quantum resistant. Their hiding property relies on the discrete logarithm problem. A quantum computer running Shor's algorithm can recover the committed value from the commitment.

This is the primary unresolved cryptographic design problem in Community. The amount-hiding layer must be rebuilt on quantum-resistant foundations. Requirements:

1. **Hiding.** The commitment reveals nothing about the committed amount to any observer, including quantum adversaries.
2. **Binding.** The committer cannot open a commitment to two different values.
3. **Homomorphic.** The network can verify that committed inputs equal committed outputs without knowing any individual amount.
4. **Range proofs.** There exists an efficient proof that a committed amount is within [0, MAX_COIN_VALUE] without revealing the amount. Efficient means proof size is logarithmic or better in MAX_COIN_VALUE.
5. **Quantum resistant.** All security properties hold against polynomial-time quantum adversaries.

Known post-quantum commitment schemes and their current limitations:

Hash-based commitments satisfy requirements 1, 2, and 5 but not 3 (not homomorphic). Without homomorphism, an entirely different transaction validity proof system is required, and current alternatives are larger and more complex.

Lattice-based commitments based on LWE/SIS can achieve homomorphism. Range proofs over lattice commitments are an active research area. Several academic constructions exist but they are newer, have larger proof sizes, and have received less scrutiny than Bulletproofs.

This section will be completed after independent cryptographic review. The chosen construction determines the transaction wire format defined in Section 3 and the coinbase transaction structure defined in Section 2.

Pending resolution, this specification uses the notation COMMIT(v, r) to denote a commitment to value v with randomness r, and RANGE_PROOF(COMMIT(v, r), v, r) to denote a range proof that v is in [0, MAX_COIN_VALUE].

[END OPEN PROBLEM]

**Transaction balance rule.** For every transfer transaction, the network verifies:

sum(COMMIT(input_i, r_i)) = sum(COMMIT(output_j, r_j)) + COMMIT(fee, r_fee)

This equality is verifiable without knowing any individual amount due to the homomorphic property. A transaction that does not satisfy this equality is creating or destroying coins and is invalid.

**Range proof requirement.** Every output commitment in a transfer transaction must be accompanied by a range proof demonstrating the committed value is in [0, MAX_COIN_VALUE]. A transaction with a missing or invalid range proof is invalid. Range proofs prevent a sender from committing to a negative value, which would allow coins to be created from nothing through the balance rule.

---

## 4.5 Combined Privacy Properties

The three mechanisms address three independent information channels:

The membership proof ensures that for any transaction on the blockchain, no observer can determine which output in the anonymity set was actually spent. Under the full-chain model this anonymity set is every eligible output that has ever existed on the blockchain, rather than a small ring of decoys.

Stealth addresses ensure that no observer can link any output to a recipient's published address. Every output goes to a unique one-time address. No two outputs sent to the same wallet share any identifying information on the blockchain.

Confidential transactions ensure that no observer can determine the amount of any output. The only information available is that all amounts in a valid transaction sum correctly.

Together, a Community transaction leaks only: that a transaction occurred, at what time, and what fee was paid. Everything else — who sent it, who received it, how much — is concealed by mathematics that no observer, no court order, and no quantum computer can reverse.

---

## 4.6 View Key Disclosure

The view key mechanism provides an optional, selective disclosure path for situations where a wallet owner needs to prove their transaction history.

A wallet owner who shares VIEW_SK allows the recipient to scan the blockchain and identify all incoming transactions to that wallet. This does not allow the recipient to spend funds. It does not reveal outgoing transactions, since membership proof spends are not traceable even with VIEW_SK.

This disclosure is voluntary and irreversible for the period of disclosure. A wallet owner who shares VIEW_SK cannot un-share it for historical transactions already on the blockchain.

The protocol does not enforce or enable any mandatory disclosure mechanism. There is no master key, no backdoor, and no regulator key. The only disclosure that can occur is voluntary disclosure by the wallet owner.

---

## 4.7 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| MAX_COIN_VALUE | Maximum value representable in a transaction output |

The sender-privacy layer adds further parameters once its construction is chosen. If a full-chain membership proof is used, the anonymity set is the whole chain and no ring-size parameter is needed. If a lattice-based ring signature fallback is used instead, a MIN_RING_SIZE parameter sets the minimum anonymity set per transfer. This is left open pending the resolution of the sender-privacy open problem in Section 4.2.
