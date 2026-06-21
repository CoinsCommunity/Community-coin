# Section 4: Privacy Model

> **DRAFT — Requires independent cryptographic review before finalization.**
> Sections marked [OPEN PROBLEM] identify unresolved design questions requiring expert input.
> The ring signature and stealth address constructions described here are directionally correct
> but require formal security proofs and expert review before implementation.

---

## 4.1 Overview

Every transaction on the Community network is private. Sender, receiver, and amount are all concealed by default. There is no transparency mode, no audit mode, and no mechanism by which any party can make a transaction visible to observers on the blockchain. Privacy is a property of the protocol, not a feature a user enables.

Three cryptographic constructions work together to achieve this. Each closes a distinct information channel.

Ring signatures conceal the sender. Stealth addresses conceal the receiver. Confidential transactions conceal the amount. Together they leave a transaction as a black box on the blockchain: provably valid, provably not double-spent, provably not creating coins from nothing, while revealing nothing about who sent it, who received it, or how much moved.

The cryptographic primitives underlying these constructions are defined in Section 9.

---

## 4.2 Ring Signatures — Concealing the Sender

A ring signature allows a member of a defined group to sign a message in a way that proves one member of the group authorized it without revealing which member. Every member of the group is an equally valid candidate to any observer.

In Community, when a sender spends an output, their transaction is not signed alone. Instead, the sender's output is combined with a set of decoy outputs pulled from the blockchain. The sender signs the transaction using a ring signature over this combined set. An observer can verify that one of the ring members authorized the transaction but cannot determine which one.

**Ring size.** The minimum number of ring members is enforced by the protocol as MIN_RING_SIZE, a genesis parameter. This minimum applies to every transfer transaction on the network. A transaction with fewer ring members than MIN_RING_SIZE is invalid and rejected. Senders may use larger rings for additional anonymity but not smaller ones.

**Decoy selection.** Decoys are real past outputs pulled from the blockchain. The sender's wallet selects decoys according to a probability distribution that weights recently created outputs more heavily, matching the expected distribution of real spending behavior. Selecting decoys that stand out from the statistical distribution of real outputs degrades anonymity by making the true output statistically identifiable. Wallets must follow the defined decoy selection algorithm.

The decoy selection algorithm is: select outputs from the blockchain with probability proportional to a gamma distribution fitted to the empirical age distribution of outputs at time of spending. The specific parameters of the gamma distribution are defined as genesis parameters and may be revisited based on observed network behavior before launch.

**[OPEN PROBLEM — Requires cryptographic review]**

Standard Monero-style ring signatures (MLSAG, CLSAG) are constructed over elliptic curve groups. They rely on the discrete logarithm problem for their security. A post-quantum ring signature scheme must be constructed over a different mathematical structure.

Lattice-based ring signatures exist in the academic literature. They provide quantum resistance but have significantly larger signature sizes than curve-based constructions, which directly affects transaction size and block throughput. The construction chosen for Community must satisfy:

1. Quantum resistance — security based on lattice hardness assumptions.
2. Ring membership proof — provably demonstrates the signer is one of the ring members.
3. Key image binding — each output, when spent, produces a unique and consistent key image regardless of which ring it appears in, preventing double-spending.
4. Unconditional signer anonymity — no observer can determine the actual signer given any polynomial amount of computation, including on a quantum computer.
5. Practical signature size — the ring signature size at the minimum ring size must be compatible with MAX_BLOCK_SIZE as defined in Section 2.

The specific post-quantum ring signature construction is deferred to cryptographic review. This specification uses the notation RING_SIGN(outputs, index, SPEND_SK) to denote a valid ring signature where outputs is the set of ring members, index is the position of the true signer's output in that set, and SPEND_SK is the signer's private spend key.

[END OPEN PROBLEM]

**Key image.** Every ring signature includes a key image as defined in Section 9.6. The key image is unique to the specific output being spent. The network maintains a global set of all key images that have appeared in finalized blocks. A transaction whose key image already appears in this set is a double-spend attempt and is rejected.

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

Ring signatures ensure that for any transaction on the blockchain, no observer can determine which of the ring members was the true sender. The anonymity set is every wallet that has ever created an output of a matching type on the blockchain.

Stealth addresses ensure that no observer can link any output to a recipient's published address. Every output goes to a unique one-time address. No two outputs sent to the same wallet share any identifying information on the blockchain.

Confidential transactions ensure that no observer can determine the amount of any output. The only information available is that all amounts in a valid transaction sum correctly.

Together, a Community transaction leaks only: that a transaction occurred, at what time, and what fee was paid. Everything else — who sent it, who received it, how much — is concealed by mathematics that no observer, no court order, and no quantum computer can reverse.

---

## 4.6 View Key Disclosure

The view key mechanism provides an optional, selective disclosure path for situations where a wallet owner needs to prove their transaction history.

A wallet owner who shares VIEW_SK allows the recipient to scan the blockchain and identify all incoming transactions to that wallet. This does not allow the recipient to spend funds. It does not reveal outgoing transactions (ring signature membership is not traceable even with VIEW_SK).

This disclosure is voluntary and irreversible for the period of disclosure. A wallet owner who shares VIEW_SK cannot un-share it for historical transactions already on the blockchain.

The protocol does not enforce or enable any mandatory disclosure mechanism. There is no master key, no backdoor, and no regulator key. The only disclosure that can occur is voluntary disclosure by the wallet owner.

---

## 4.7 Genesis Parameters Summary

| Parameter | Description |
|---|---|
| MIN_RING_SIZE | Minimum number of ring members in a transfer transaction |
| MAX_COIN_VALUE | Maximum value representable in a transaction output |
