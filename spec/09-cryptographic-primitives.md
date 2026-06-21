# Section 9: Cryptographic Primitives

> **DRAFT — Requires independent cryptographic review before finalization.**
> Sections marked [OPEN PROBLEM] identify unresolved design questions requiring expert input.
> This draft describes the intended constructions and their security requirements.
> No implementation should proceed from this section until it has been reviewed and approved
> by independent cryptographers.

---

## 9.1 Overview

Community's cryptographic foundation uses exclusively post-quantum primitives standardized by NIST in 2024. No construction in this protocol relies on the hardness of the discrete logarithm problem or integer factorization — the two problem families that Shor's algorithm solves efficiently on a quantum computer.

The security assumptions underlying this protocol are:

- The hardness of the NTRU lattice shortest vector problem (basis for FALCON security)
- The hardness of the Module Learning With Errors problem (basis for CRYSTALS-Kyber security)
- The collision resistance and preimage resistance of the chosen hash function

All three assumptions are believed to remain hard against quantum adversaries. The best known quantum algorithms provide only polynomial speedup against lattice problems, not the exponential speedup that Shor's algorithm provides against discrete log and factorization.

---

## 9.2 Hash Function

Community uses BLAKE3 as its protocol hash function throughout, denoted HASH() in all sections of this specification.

BLAKE3 properties relevant to this protocol:

- Output size: 256 bits (32 bytes) for standard use; variable length via its extendable output mode
- Construction: a Merkle tree of compression functions based on the ChaCha permutation
- Security: 128-bit collision resistance, 256-bit preimage resistance
- Performance: significantly faster than SHA-256 and SHA-3 on modern CPUs
- Quantum resistance: hash functions with 256-bit output retain approximately 128-bit security against Grover's algorithm on quantum computers, which is acceptable

The double-hash convention HASH(HASH(x)) is used for block header hashing and transaction hashing to maintain consistency with established blockchain conventions and to prevent length extension attacks in contexts where BLAKE3's streaming properties could otherwise be relevant.

The extendable output mode of BLAKE3 is used where variable-length pseudorandom output is required, denoted EXPAND(seed, length) in this specification. EXPAND(seed, length) returns `length` bytes of pseudorandom output derived deterministically from `seed`. This function is used in staker panel selection as defined in Section 6.5.

---

## 9.3 Digital Signatures — FALCON

Community uses FALCON-1024 for all digital signatures.

FALCON (Fast Fourier Lattice-based Compact Signatures over NTRU) was standardized by NIST in 2024 as FIPS 206. FALCON-1024 provides 256-bit post-quantum security, meaning the best known quantum attack requires approximately 2^256 operations. FALCON-512 provides 128-bit post-quantum security.

FALCON-1024 is selected over FALCON-512 for this protocol because Community is designed to operate indefinitely without protocol upgrades. The stronger security margin of FALCON-1024 provides headroom against future improvements in lattice attack algorithms.

**Key sizes for FALCON-1024:**

| Object | Size |
|---|---|
| Private key | 2305 bytes |
| Public key | 1793 bytes |
| Signature | 1330 bytes (average), 1756 bytes (maximum) |

FALCON signatures are larger than ECDSA signatures (64 bytes). This is an accepted tradeoff. Block and transaction size parameters are set to accommodate FALCON-1024 signature sizes throughout this protocol.

**Signing.** To sign a message M with private key sk:

SIGNATURE = FALCON_SIGN(sk, M)

**Verification.** To verify a signature against a public key pk:

VALID = FALCON_VERIFY(pk, M, SIGNATURE)

Returns true if the signature is valid, false otherwise.

All wallet transaction signatures use FALCON-1024. Node identity keypairs as defined in Section 1.3 also use FALCON-1024. Ring signature construction is defined separately in Section 4 and uses FALCON as its underlying signature primitive.

---

## 9.4 Key Encapsulation — CRYSTALS-Kyber

Community uses CRYSTALS-Kyber-1024 (ML-KEM-1024 in NIST FIPS 203 terminology) for key encapsulation.

Kyber is a key encapsulation mechanism (KEM) based on the Module Learning With Errors (MLWE) problem. It is used in this protocol for stealth address construction, specifically for the sender to derive a one-time address that only the recipient can identify. The full stealth address scheme is defined in Section 4.

**Key sizes for Kyber-1024:**

| Object | Size |
|---|---|
| Private key | 3168 bytes |
| Public key | 1568 bytes |
| Ciphertext (encapsulation) | 1568 bytes |
| Shared secret | 32 bytes |

**Key generation.** Each wallet generates a Kyber-1024 keypair:

(KYBER_PK, KYBER_SK) = KYBER_KEYGEN()

**Encapsulation.** A sender encapsulates a shared secret to a recipient's public key:

(CIPHERTEXT, SHARED_SECRET) = KYBER_ENCAPS(KYBER_PK)

**Decapsulation.** The recipient recovers the shared secret:

SHARED_SECRET = KYBER_DECAPS(KYBER_SK, CIPHERTEXT)

The shared secret is 32 bytes of pseudorandom data that both parties derive identically without it being observable by any third party.

---

## 9.5 Wallet Key Structure

Each Community wallet contains the following keys:

**Spend keypair (FALCON-1024):**
- SPEND_SK: private spend key. Used to authorize spending transactions. Never leaves the wallet.
- SPEND_PK: public spend key. Used in address construction.

**View keypair (Kyber-1024):**
- VIEW_SK: private view key. Used to identify incoming transactions. Can be shared with a third party to enable read-only wallet scanning without spending authority.
- VIEW_PK: public view key. Published as part of the wallet address.

A wallet address encodes both public keys:

ADDRESS = ENCODE(HASH(SPEND_PK) || VIEW_PK)

The address encoding is a Base58Check encoding of the concatenated fields with a Community-specific version byte prefix. The SPEND_PK is hashed in the address so the public key is never directly exposed in an address.

When funds arrive at an address, they are sent to a one-time stealth address derived from VIEW_PK as defined in Section 4. The wallet scans the blockchain using VIEW_SK to identify which stealth addresses belong to it. Spending requires SPEND_SK.

---

## 9.6 Key Images

Key images enforce double-spend prevention for private transactions. Each unspent output, when spent, produces a unique key image that is recorded on the blockchain. The network rejects any transaction whose key image has already appeared.

In traditional ring signature schemes (such as Monero's LSAG/CLSAG), key images are constructed using elliptic curve operations: I = x * H_p(P), where x is the private key and H_p is a hash-to-curve function. This construction is quantum vulnerable because it relies on the discrete log relationship between x and P.

**[OPEN PROBLEM — Requires cryptographer review]**

The post-quantum key image construction used in Community must satisfy:

1. **Uniqueness.** Each unspent output, when spent by its rightful owner, produces exactly one possible key image. Two honest spends of the same output always produce the same key image.

2. **Unlinkability.** The key image for output O spent with private key x must not be linkable to the key image for a different output O' spent with the same key x. An observer who sees both key images cannot determine they were spent by the same wallet.

3. **Unforgeability.** A party without the private key x cannot produce a valid key image for an output controlled by x.

4. **Quantum resistance.** The construction must not rely on the hardness of discrete logarithm or factorization.

The candidate construction under consideration is:

KEY_IMAGE = HASH(SPEND_SK || OUTPUT_COMMITMENT)

where SPEND_SK is the private spend key and OUTPUT_COMMITMENT uniquely identifies the specific output being spent.

This construction satisfies requirements 1, 3, and 4. Whether it satisfies requirement 2 (unlinkability between key images from the same wallet spending different outputs) requires formal analysis. If HASH is modeled as a random oracle, the construction is unlinkable, but the hash function's concrete security properties must be verified against the specific unlinkability requirement.

**[END OPEN PROBLEM]**

Until this construction is formally reviewed and approved, this specification uses the notation KEY_IMAGE(SPEND_SK, OUTPUT) to refer to the key image of a specific output, deferring the exact construction to the review process.

---

## 9.7 Commitment Scheme

[OPEN PROBLEM — Requires cryptographer review]

Confidential transactions as described in Section 4 require a commitment scheme with the following properties:

1. **Hiding.** A commitment COM(v, r) to value v with randomness r reveals nothing about v to any observer.

2. **Binding.** A committed party cannot produce two different (v, r) pairs that open the same commitment.

3. **Homomorphic.** COM(v1, r1) + COM(v2, r2) = COM(v1 + v2, r1 + r2). This property allows the network to verify that transaction inputs equal outputs without knowing any individual amounts.

4. **Range proofs.** There must exist an efficient proof system that demonstrates a committed value lies within a valid range [0, MAX_COIN_VALUE] without revealing the value.

5. **Quantum resistance.** The hiding and binding properties must hold against quantum adversaries.

The standard construction, Pedersen commitments, satisfies properties 1 through 4 but fails property 5. Pedersen commitments are defined as COM(v, r) = v*G + r*H over an elliptic curve group, where G and H are group generators. The hiding property relies on the hardness of the discrete logarithm problem. Shor's algorithm breaks this.

Candidate post-quantum commitment schemes include:

- **Hash-based commitments.** COM(v, r) = HASH(v || r). Satisfies properties 1, 2, and 5. Does not satisfy property 3 (not homomorphic), which means standard confidential transaction construction does not apply. An alternative transaction validity proof system would be required.

- **Lattice-based commitments.** Constructions based on the Learning With Errors (LWE) or Short Integer Solution (SIS) problems. Some constructions achieve homomorphism but with larger commitment sizes and more complex proving systems. Production-ready range proof systems over lattice commitments are an active area of research.

- **Hybrid approach.** Use Pedersen commitments for the current protocol with a planned migration path, accepting the quantum vulnerability of historical transactions. This contradicts Community's design goal of quantum resistance from day one and is listed only for completeness. It is not the preferred approach.

This is the primary open cryptographic design problem in the Community protocol. The resolution of this section determines the transaction format defined in Section 3 and the privacy model defined in Section 4. This section will be finalized after independent cryptographic review identifies a construction satisfying all five properties.

[END OPEN PROBLEM]

---

## 9.8 Pseudorandom Number Generation

All cryptographic randomness used in key generation, transaction construction, and staker operations must be sourced from a cryptographically secure pseudorandom number generator (CSPRNG) seeded from system entropy.

Implementations must not use deterministic pseudorandom number generators, language default random functions, or any randomness source that cannot provide cryptographic security guarantees.

Failure to use cryptographically secure randomness in FALCON key generation or signature generation can catastrophically compromise private key security. FALCON's signing algorithm in particular requires high-quality randomness. Implementations must follow the reference implementation guidance for FALCON randomness requirements exactly.

---

## 9.9 Summary of Primitives

| Primitive | Algorithm | Standard | Status |
|---|---|---|---|
| Hash function | BLAKE3 | — | Finalized |
| Digital signatures | FALCON-1024 | NIST FIPS 206 | Finalized |
| Key encapsulation | Kyber-1024 (ML-KEM-1024) | NIST FIPS 203 | Finalized |
| Key images | TBD | — | Open problem |
| Commitment scheme | TBD | — | Open problem |
| Range proofs | TBD | — | Depends on commitment scheme |
