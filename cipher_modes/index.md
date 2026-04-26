## 7. Cipher modes

PNA employs multiple cipher modes of operation. This specification is designed to ensure the secure use of PNA by enabling the selection of alternative cipher modes of operation in the event of a critical flaw discovered in a specific cipher mode of operation.

> **Note on terminology.** Two categories of modes coexist in this chapter:
> - **Confidentiality-only modes** (CBC, CTR): provide encryption without authenticity. Available in all PNA Minor versions.
> - **Authenticated Encryption with Associated Data (AEAD) modes** (GCM): provide encryption AND authenticity in one construction. Available in PNA Minor version `1` and higher (under Major version 0).
>
> Both categories share the same `Cipher mode` field in the FHED chunk; the field value selects which mode applies.

### 7.1. Cipher Block Chaining Mode (CBC)
PNA cipher mode of operation method 0 specifies CBC with a block length of 128 bits.

> **Status: legacy (read-only recommended).** CBC provides confidentiality but no authenticity. Encrypted CBC archives are vulnerable to padding-oracle attacks (Vaudenay 2002) and to silent ciphertext tampering. New PNA archives SHOULD use [§7.4 GCM](#74-galoiscounter-mode-gcm) instead. Decoders MUST continue to support CBC for reading archives produced by older encoders.

CBC mode is a block cipher mode of operation that provides confidentiality and integrity through chaining encrypted blocks. In CBC mode, each plaintext block is XORed with the previous ciphertext block before encryption. It introduces a dependency between blocks, making it suitable for scenarios where each block's integrity relies on the preceding block.

This document specifies the use of the Rijndael or Camellia cipher in CBC mode within PNA.

This mode requires an Initialization Vector (IV) that is the same size as the block size. Use of a randomly generated IV prevents generation of identical ciphertext from packets which have identical data that spans the first block of the cipher algorithm's block size.

The IV is XOR'd with the first plaintext block before it is encrypted. Then for successive blocks, the previous ciphertext block is XOR'd with the current plaintext, before it is encrypted.

More information on CBC mode can be obtained in [MODES](../references/index.md#modes), [CRYPTO-S](../references/index.md#crypto-s)

### 7.2. Counter Mode (CTR)
PNA cipher mode of operation method 1 specifies CTR with a block length of 128 bits.

> **Status: legacy (read-only recommended).** CTR provides confidentiality but no authenticity. CTR ciphertexts are trivially malleable: flipping any ciphertext bit flips the corresponding plaintext bit, and the modification is undetectable without an external integrity mechanism. New PNA archives SHOULD use [§7.4 GCM](#74-galoiscounter-mode-gcm) instead. Decoders MUST continue to support CTR for reading archives produced by older encoders.

CTR mode is a stream cipher mode of operation, where each plaintext block is encrypted by XORing it with the corresponding block of a keystream. CTR offers excellent parallelizability and allows random access to individual blocks, making it suitable for scenarios where seeking and parallel processing are required.

This document specifies the use of the Rijndael or Camellia cipher in CTR mode within PNA.

CTR requires the encryptor to generate a unique per-packet value and communicate this value to the decryptor. This specification calls this per-packet value an initialization vector (IV).  The same IV and key combination MUST NOT be used more than once. The encryptor can generate the IV in any manner that ensures uniqueness. Common approaches to IV generation include incrementing a counter for each packet.

More information on CTR mode can be obtained in [MODES](../references/index.md#modes)


### 7.3. Initialization Vector (IV) Placement in Encrypted Streams

> **Applies to: CBC and CTR modes only.** This section defines IV placement for confidentiality-only modes. The placement of nonces and authentication tags for AEAD modes (GCM) is defined in [§7.5](#75-nonce-and-tag-placement-for-aead-modes).

When using an encryption mode that requires an Initialization Vector (IV), such as CBC (Cipher Block Chaining) or CTR (Counter Mode), PNA requires that the IV be prepended to the beginning of the encrypted datastream.

#### 7.3.1 Placement

For both per-entry (`FDAT`) and solid mode (`SDAT`) encrypted datastreams, the IV appears at the start of the stream. The IV is always 16 bytes in length, corresponding to the 128-bit block size used by both Rijndael (AES) and Camellia.

The structure of an encrypted stream is:

```
[16-byte IV] + [ciphertext data...]
```

#### 7.3.2 Decoder Behavior

A decoder must extract the first 16 bytes of the encrypted datastream as the Initialization Vector. The remainder of the datastream represents the ciphertext, which must be decrypted using the specified cipher algorithm and mode.

#### 7.3.3 Encoder Behavior

An encoder must generate a new, cryptographically secure random IV for each encrypted stream. The IV must be prepended to the encrypted datastream.

Reuse of an IV with the same encryption key is prohibited and renders the archive cryptographically insecure.

### 7.4. Galois/Counter Mode (GCM)

PNA cipher mode of operation method 2 specifies GCM with a block length of 128 bits.

> **Status: recommended for new archives (requires PNA Minor version 1 or higher).** GCM provides both confidentiality and authenticity in a single construction (Authenticated Encryption with Associated Data, AEAD). It is the only mode in this specification that detects and rejects ciphertext tampering. New encoders SHOULD use GCM as the default cipher mode.

GCM is an AEAD mode of operation built on top of a 128-bit block cipher (Rijndael or Camellia in this specification). It combines counter-mode confidentiality (CTR) with a Galois-field-based message authentication code (GHASH) to produce a single fixed-length authentication tag for each invocation.

This document specifies the use of the Rijndael or Camellia cipher in GCM mode within PNA.

GCM is defined in [NIST SP 800-38D](../references/index.md#nist-sp-800-38d). The full algorithm specification (including pseudocode for `GCM-AE` and `GCM-AD`) is normative; this section only describes PNA-specific parameter choices and integration constraints.

#### 7.4.1 Parameters

| Parameter | Value | Note |
|---|---|---|
| Block cipher | Rijndael (AES-256) or Camellia-256 | Per FHED `Encryption method` field |
| Key size | 256 bits (32 bytes) | Derived from the entry's PHSF chunk (Argon2id or PBKDF2 output, used directly as GCM key); see [§4.1.5 PHSF](../chunk_specifications/index.md#415-phsf-password-hash) |
| Nonce size | 96 bits (12 bytes) | Required by NIST SP 800-38D §5.2.1.1; other sizes are NOT permitted in PNA |
| Authentication tag size | 128 bits (16 bytes) | Full-length tags only; truncated tags are NOT permitted in PNA |
| Maximum plaintext per nonce | 2^39 - 256 bits ≈ 64 GiB | NIST SP 800-38D §5.2.1.1; encoders MUST chunk longer streams |
| Maximum AAD per nonce | 2^61 bytes | NIST SP 800-38D §5.2.1.1 |
| Maximum invocations per key | 2^32 (when nonces are randomly generated) | NIST SP 800-38D §8.3; PNA mitigates this by per-entry key derivation (see [§7.4.4](#744-key-and-nonce-uniqueness)) |

> **Implementation note (Camellia-GCM).** [RFC 6367](../references/index.md#rfc-6367) defines Camellia in GCM mode for use in TLS and is the normative reference for PNA's use of Camellia-GCM. The construction is identical to AES-GCM with the underlying 128-bit block cipher substituted.

#### 7.4.2 Associated Data (AAD)

GCM requires an Associated Data (AAD) input that is authenticated but not encrypted. PNA constructs the AAD deterministically per chunk so that any tampering with cipher parameters, entry position, or chunk ordering is detected at tag verification.

The AAD for each FDAT or SDAT chunk MUST be the byte concatenation of the following fields, in order:

| Bytes | Field | Source |
|---|---|---|
| 1 | Encryption method | FHED `Encryption method` byte (1 = AES, 2 = Camellia) |
| 1 | Cipher mode       | FHED `Cipher mode` byte (= 2 for GCM) |
| 4 | Entry index       | u32 big-endian; 0-indexed position of this entry within the archive |
| 4 | Chunk index       | u32 big-endian; 0-indexed position of this FDAT/SDAT chunk within the entry's data stream |
| 1 | Final-chunk flag  | `0x01` if this is the final FDAT (or SDAT) chunk before FEND (or SEND); `0x00` otherwise |

Total AAD length per chunk: **11 bytes** (constant for `Cipher mode = 2`).

> **Rationale.** Each field defends against a specific tampering class:
> - The cipher parameter bytes (Encryption method, Cipher mode) prevent algorithm/mode downgrade attacks.
> - The Entry index prevents intra-archive entry reorder attacks and cross-entry chunk swap attacks (relevant when encoders share a PHSF salt across AEAD entries, as recommended).
> - The Chunk index prevents intra-entry chunk reorder attacks.
> - The Final-chunk flag prevents truncation attacks (a removed last chunk causes the new "last" chunk to authenticate with `Final-chunk flag = 0x00` instead of `0x01`).

> **Future-version domain separation.** Each new PNA AEAD framework version MUST allocate a new FHED `Cipher mode` value (e.g., a hypothetical PNA AEAD v2 would use `Cipher mode = 3`, not reuse `2`). The 1-byte `Cipher mode` field in the AAD then provides natural inter-version domain separation without a magic prefix.

#### 7.4.3 Nonce Generation

The 96-bit nonce for each FDAT or SDAT chunk MUST be generated by a cryptographically secure pseudorandom number generator (CSPRNG) per NIST SP 800-38D §8.2.2 (RBG-based construction). The encoder MUST generate a fresh 12-byte nonce for every chunk and store it inline as the first 12 bytes of the chunk's data field (see [§7.5](#75-nonce-and-tag-placement-for-aead-modes)).

PNA does NOT use deterministic nonce construction (NIST SP 800-38D §8.2.1) in `Cipher mode = 2`; nonce uniqueness is provided by CSPRNG randomness, not by counter state.

#### 7.4.4 Key and Nonce Uniqueness

GCM is catastrophically insecure under nonce reuse with the same key (Joux 2006: a single nonce repetition reveals the GHASH key, enabling unlimited forgeries). PNA enforces nonce uniqueness via:

1. **CSPRNG-generated 96-bit nonces (§7.4.3).** The probability of nonce collision under one key after `N` invocations is approximately `N² / 2^97`. NIST SP 800-38D §8.3 limits this to `N ≤ 2^32` (collision probability ≤ 2^-33).
2. **Bounded invocations per key.** A single PHSF chunk derives one GCM key (from password + salt). All entries sharing the same PHSF chunk share the same key; the cumulative chunk count under that key MUST NOT exceed 2^32. Encoders writing more than 2^32 chunks under one password MUST emit at least one entry with a fresh PHSF salt to start a new key.

Encoders MUST NOT reuse a nonce under the same key. Detecting CSPRNG output collision is generally infeasible for the encoder; the 2^32 invocations cap is the operational mitigation.

#### 7.4.5 Random Number Generation

The 12-byte nonce of each FDAT/SDAT chunk MUST be generated using a cryptographically secure pseudorandom number generator (CSPRNG). On systems where the CSPRNG can fail or block (e.g., early-boot Linux without sufficient entropy), encoders MUST surface the failure and abort archive creation rather than silently producing weak random values.

#### 7.4.6 Authentication Failure Handling

A decoder MUST verify the authentication tag of each FDAT or SDAT chunk before releasing any plaintext from that chunk to upper layers (the "Releasing Unverified Plaintext" prohibition; see [Bellare-Namprempre 2000](../references/index.md#bellare-namprempre-2000)).

If tag verification fails for any chunk, the decoder MUST:

1. Discard all decrypted plaintext from that chunk.
2. Report an error indicating an authentication failure (distinguishable from "I/O error" or "format error").
3. Cease processing the affected entry.

A decoder MAY continue processing subsequent entries in the archive after an authentication failure on one entry, but the failed entry MUST NOT be partially extracted or returned as success.

### 7.5. Nonce and Tag Placement for AEAD Modes

When using an AEAD mode (currently GCM, [§7.4](#74-galoiscounter-mode-gcm)), PNA does NOT use the IV-prepend layout described in [§7.3](#73-initialization-vector-iv-placement-in-encrypted-streams). Instead, each chunk carries its own random nonce inline (§7.4.3 / §7.4.5) and the authentication tag appended at the end.

#### 7.5.1 FDAT and SDAT Layout under AEAD

Each FDAT (per-entry) or SDAT (solid mode) chunk's data field, when the entry's `Cipher mode` field is 2 (GCM), MUST be laid out as:

```
[12-byte random nonce] [ciphertext data ...] [16-byte authentication tag]
```

The chunk's standard length field counts the nonce, ciphertext, and trailing tag (`12 + len(ciphertext) + 16`). The minimum data field length is therefore 28 bytes (12 nonce + 0 ciphertext + 16 tag); a chunk shorter than 28 bytes is malformed and MUST be rejected by the decoder.

> **Note.** The chunk-level CRC32 (already required for every chunk) covers the entire `(nonce || ciphertext || tag)` byte sequence. CRC32 detects transit errors but provides no cryptographic authenticity; the GCM tag is the authoritative integrity check.

#### 7.5.2 Decoder Behavior

For each FDAT or SDAT chunk in an entry whose `Cipher mode` field is 2 (GCM), a decoder MUST:

1. Read the chunk's data field (length `N` bytes; required `N ≥ 28`).
2. Treat the first 12 bytes as the nonce.
3. Treat the last 16 bytes as the authentication tag.
4. Treat bytes `[12 .. N-16]` as the ciphertext.
5. Construct the AAD as specified in [§7.4.2](#742-associated-data-aad).
6. Verify the tag using the cipher algorithm (Rijndael or Camellia) selected by the FHED `Encryption method` field, with the per-entry GCM key derived from PHSF (see [§4.1.5 PHSF](../chunk_specifications/index.md#415-phsf-password-hash)).
7. If tag verification succeeds, decrypt the ciphertext to recover plaintext.
8. If tag verification fails, follow the procedure in [§7.4.6](#746-authentication-failure-handling).

Decoders MUST reject any FDAT or SDAT chunk whose data field is shorter than 28 bytes when the entry uses GCM mode.

#### 7.5.3 Encoder Behavior

For each FDAT or SDAT chunk emitted under GCM mode, an encoder MUST:

1. Determine the chunk's plaintext payload (typically a fixed-size segment such as 64 KiB; see [Recommendations for Encoders](../recommendations_for_encoders/index.md)).
2. Compute the chunk's AAD as specified in [§7.4.2](#742-associated-data-aad).
3. Generate a fresh 12-byte nonce from a CSPRNG (§7.4.5).
4. Encrypt the plaintext under the per-entry GCM key with the generated nonce and AAD, producing a ciphertext of the same length as the plaintext and a 16-byte tag.
5. Emit a single FDAT (or SDAT) chunk whose data field is the concatenation `(nonce || ciphertext || tag)`.
6. For the chunk preceding FEND (or SEND), set the final-chunk flag in the AAD to `0x01`. For all other chunks, the flag MUST be `0x00`.

Encoders SHOULD choose a uniform chunk plaintext size (e.g., 64 KiB or 1 MiB) for an entire entry to simplify decoder buffering. The final chunk of an entry MAY be shorter than the chosen size but MUST NOT be empty unless the entry's plaintext is itself empty.
