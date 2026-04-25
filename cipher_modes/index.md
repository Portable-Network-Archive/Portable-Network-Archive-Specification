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
| Key size | 256 bits (32 bytes) | Derived per-entry, see [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function) |
| Nonce size | 96 bits (12 bytes) | Required by NIST SP 800-38D §5.2.1.1; other sizes are NOT permitted in PNA |
| Authentication tag size | 128 bits (16 bytes) | Full-length tags only; truncated tags are NOT permitted in PNA |
| Maximum plaintext per nonce | 2^39 - 256 bits ≈ 64 GiB | NIST SP 800-38D §5.2.1.1; encoders MUST chunk longer streams |
| Maximum AAD per nonce | 2^61 bytes | NIST SP 800-38D §5.2.1.1 |
| Maximum invocations per key | 2^32 (when nonces are randomly generated) | NIST SP 800-38D §8.3; PNA mitigates this by per-entry key derivation (see [§7.4.4](#744-key-and-nonce-uniqueness)) |

> **Implementation note (Camellia-GCM).** [RFC 6367](../references/index.md#rfc-6367) defines Camellia in GCM mode for use in TLS and is the normative reference for PNA's use of Camellia-GCM. The construction is identical to AES-GCM with the underlying 128-bit block cipher substituted.

#### 7.4.2 Associated Data (AAD)

GCM requires an Associated Data (AAD) input that is authenticated but not encrypted. PNA constructs the AAD deterministically per chunk so that any tampering with cipher parameters, chunk ordering, archive identity, or entry metadata is detected at tag verification.

The AAD for each FDAT or SDAT chunk MUST be the byte concatenation of the following fields, in order:

| Bytes | Field | Source |
|---|---|---|
| 11 | ASCII string `"PNA-AEAD-v1"` (no NUL) | Magic constant binding the framework version |
| 1  | Encryption method | FHED `Encryption method` byte (1 = AES, 2 = Camellia) |
| 1  | Cipher mode       | FHED `Cipher mode` byte (= 2 for GCM) |
| 16 | Archive identifier | `archive_identifier` field of the archive's `AENC` chunk for this `cipher_mode` (see [§4.1.x AENC](../chunk_specifications/index.md)) |
| 16 | Entry random       | `entry_random` field of this entry's `FENC` chunk (see [§4.1.x FENC](../chunk_specifications/index.md)) |
| 4  | Entry index       | u32 big-endian; 0-indexed position of this entry within the archive |
| 4  | Chunk index       | u32 big-endian; 0-indexed position of this FDAT/SDAT chunk within the entry's data stream |
| 1  | Final-chunk flag  | `0x01` if this is the final FDAT (or SDAT) chunk before FEND (or SEND); `0x00` otherwise |
| 32 | Metadata hash     | SHA-256 of the byte concatenation of FHED chunk data and all ancillary chunks appearing between FHED and the first FDAT, in their on-disk order |

Total AAD length per chunk: **86 bytes** (constant for `framework_version = 1`).

> **Rationale.** Each field defends against a specific tampering class:
> - The magic and the cipher parameter bytes prevent algorithm/mode downgrade attacks.
> - The archive identifier prevents cross-archive ciphertext substitution.
> - The entry random provides defense-in-depth against encoder bugs that might otherwise produce duplicate per-entry contexts.
> - The entry and chunk indices prevent reordering attacks.
> - The final-chunk flag prevents truncation attacks.
> - The metadata hash binds the entry's plaintext-readable metadata to the ciphertext, so that filename, permissions, timestamps, and extended attributes cannot be silently modified.

#### 7.4.3 Nonce Derivation

The 96-bit nonce for each FDAT or SDAT chunk MUST be constructed deterministically as follows:

```
nonce[0..8]   = AENC[cipher_mode_id=2].archive_identifier[0..8]   (per-archive, 8 bytes)
nonce[8..12]  = chunk_index                                       (u32 big-endian, per-entry counter)
```

The `chunk_index` starts at 0 for the first FDAT (or SDAT) chunk of each entry and increments by 1 for each subsequent chunk in the same entry. Indices are NOT shared across entries.

> **Note.** The high-order 8 bytes of `archive_identifier` are reused as the nonce prefix; the low-order 8 bytes serve as additional HKDF input (see [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function)). Splitting the 16-byte `archive_identifier` this way is safe because the per-entry HKDF derivation produces an independent encryption key per entry, so nonce uniqueness only needs to hold within a single (key, nonce) scope, which is guaranteed by `chunk_index`.

#### 7.4.4 Key and Nonce Uniqueness

GCM is catastrophically insecure under nonce reuse with the same key (Joux 2006: a single nonce repetition reveals the GHASH key, enabling unlimited forgeries). PNA enforces nonce uniqueness via two mechanisms:

1. **Per-entry key derivation.** Each entry uses a key derived from the archive's master key via HKDF-SHA-256, as specified in [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function). The HKDF inputs include both the per-archive `archive_identifier` (from `AENC`) and the per-entry `entry_random` (from `FENC`), so different entries derive different GCM keys even if some other input collides. Cross-entry GCM nonce collisions are therefore neutralized at the key level.
2. **Monotonic chunk counter within an entry.** Within a single entry, the `chunk_index` is strictly monotonically increasing, guaranteeing nonce uniqueness within (key, nonce) pairs.

Encoders MUST NOT reuse a nonce across distinct invocations under the same per-entry key. Encoders MUST regenerate `AENC.archive_identifier` with a cryptographically secure random number generator for each new archive. Encoders MUST regenerate `FENC.entry_random` for each new entry within an archive.

#### 7.4.5 Random Number Generation

The cryptographic random fields (`AENC.archive_identifier`, `FENC.entry_random`) MUST be generated using a cryptographically secure pseudorandom number generator (CSPRNG). On systems where the CSPRNG can fail or block (e.g., early-boot Linux without sufficient entropy), encoders MUST surface the failure and abort archive creation rather than silently producing weak random values.

#### 7.4.6 Authentication Failure Handling

A decoder MUST verify the authentication tag of each FDAT or SDAT chunk before releasing any plaintext from that chunk to upper layers (the "Releasing Unverified Plaintext" prohibition; see [Bellare-Namprempre 2000](../references/index.md#bellare-namprempre-2000)).

If tag verification fails for any chunk, the decoder MUST:

1. Discard all decrypted plaintext from that chunk.
2. Report an error indicating an authentication failure (distinguishable from "I/O error" or "format error").
3. Cease processing the affected entry.

A decoder MAY continue processing subsequent entries in the archive after an authentication failure on one entry, but the failed entry MUST NOT be partially extracted or returned as success.

### 7.5. Nonce and Tag Placement for AEAD Modes

When using an AEAD mode (currently GCM, [§7.4](#74-galoiscounter-mode-gcm)), PNA does NOT use the IV-prepend layout described in [§7.3](#73-initialization-vector-iv-placement-in-encrypted-streams). Instead, the nonce is derived deterministically per chunk (§7.4.3) and is NOT stored in the chunk data. The authentication tag is appended to each chunk's ciphertext.

#### 7.5.1 FDAT and SDAT Layout under AEAD

Each FDAT (per-entry) or SDAT (solid mode) chunk's data field, when the entry's `Cipher mode` field is 2 (GCM), MUST be laid out as:

```
[ciphertext data ...] [16-byte authentication tag]
```

The chunk's standard length field counts both the ciphertext and the trailing tag. The tag occupies the final 16 bytes of the chunk's data field, regardless of the chunk's data length.

> **Note.** The chunk-level CRC32 (already required for every chunk) covers the entire `(ciphertext || tag)` byte sequence. CRC32 detects transit errors but provides no cryptographic authenticity; the GCM tag is the authoritative integrity check.

#### 7.5.2 Decoder Behavior

For each FDAT or SDAT chunk in an entry whose `Cipher mode` field is 2 (GCM), a decoder MUST:

1. Read the chunk's data field (length `N` bytes; required `N ≥ 16`).
2. Treat the last 16 bytes as the authentication tag.
3. Treat the first `N - 16` bytes as ciphertext.
4. Reconstruct the AAD as specified in [§7.4.2](#742-associated-data-aad).
5. Reconstruct the nonce as specified in [§7.4.3](#743-nonce-derivation).
6. Verify the tag using the cipher algorithm (Rijndael or Camellia) selected by the FHED `Encryption method` field.
7. If tag verification succeeds, decrypt the ciphertext to recover plaintext.
8. If tag verification fails, follow the procedure in [§7.4.6](#746-authentication-failure-handling).

Decoders MUST reject any FDAT or SDAT chunk whose data field is shorter than 16 bytes when the entry uses GCM mode.

#### 7.5.3 Encoder Behavior

For each FDAT or SDAT chunk emitted under GCM mode, an encoder MUST:

1. Determine the chunk's plaintext payload (typically a fixed-size segment such as 64 KiB; see [Recommendations for Encoders](../recommendations_for_encoders/index.md)).
2. Compute the chunk's AAD as specified in [§7.4.2](#742-associated-data-aad).
3. Compute the chunk's nonce as specified in [§7.4.3](#743-nonce-derivation).
4. Encrypt the plaintext under the per-entry GCM key with the computed nonce and AAD, producing a ciphertext of the same length as the plaintext and a 16-byte tag.
5. Emit a single FDAT (or SDAT) chunk whose data field is the concatenation `(ciphertext || tag)`.
6. For the chunk preceding FEND (or SEND), set the final-chunk flag in the AAD to `0x01`. For all other chunks, the flag MUST be `0x00`.

Encoders SHOULD choose a uniform chunk plaintext size (e.g., 64 KiB or 1 MiB) for an entire entry to simplify decoder buffering. The final chunk of an entry MAY be shorter than the chosen size but MUST NOT be empty unless the entry's plaintext is itself empty.
