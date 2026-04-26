# AEAD Spec Revision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the AEAD minimal redesign (`docs/superpowers/specs/2026-04-26-aead-minimal-redesign.md`) to the 5 PNA spec markdown files, replacing the AENC/FENC/HKDF framework drafted in commit `a31aa01` with the per-entry AEAD design (zero new chunk types, 11-byte AAD, random nonce inline, no HKDF).

**Architecture:** Edit-and-verify-per-file, then one atomic commit. Cross-references between files (e.g., `cipher_modes/index.md` linking to `[§4.1.x AENC]` in `chunk_specifications/index.md`) would break if commits were per-file, so all edits land in a single commit with a final cross-reference grep gate.

**Tech Stack:** Markdown (CommonMark with internal anchor links). No code, no tests beyond grep-based reference verification.

**Branch:** Continue on `spec/aead-gcm-introduction` (already contains `a31aa01` and the design doc commit `e25e5d4`). New commits on this branch supersede `a31aa01`'s additions.

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `references/index.md` | Bibliography | Remove RFC 5869 entry (HKDF no longer used). |
| `key_derivation_algorithms/index.md` | KDF specs | Remove §8.3 HKDF. Update §8.2.5 Argon2 note (output used directly as GCM key). |
| `cipher_modes/index.md` | Cipher mode specs | Rewrite §7 intro Minor 1 note. Rewrite §7.4.2 AAD (86B → 11B). Replace §7.4.3 nonce derivation with random generation. Update §7.4.4 key uniqueness. Update §7.4.5 RNG. Rewrite §7.5 to add 12B inline nonce to layout. |
| `chunk_specifications/index.md` | Chunk type specs | Remove §4.1.x AENC and §4.1.x FENC sections. Update AHED Minor 1 description. Update FHED Cipher mode 2 note (no AENC/FENC requirement). Simplify PHSF AEAD note. |
| `recommendations_for_decoders/index.md` | Decoder guidance | Remove AENC/FENC mentions in §12.3. Simplify §12.3.4 Multipart consistency. Update §12.3.5 error categories. |

---

## Task 1: Update `references/index.md` (remove RFC 5869)

**Files:**
- Modify: `references/index.md:59-61`

- [ ] **Step 1: Remove the `[RFC-5869]` entry**

Use Edit on `references/index.md`:

```
old_string:
### **[RFC-5869]**
H. Krawczyk and P. Eronen, "HMAC-based Extract-and-Expand Key Derivation Function (HKDF)", RFC 5869, IETF, May 2010.
[https://datatracker.ietf.org/doc/html/rfc5869](https://datatracker.ietf.org/doc/html/rfc5869)

### **[RFC-6367]**

new_string:
### **[RFC-6367]**
```

- [ ] **Step 2: Verify the entry is gone**

Run: `grep -n "RFC 5869\|RFC-5869\|rfc5869" references/index.md`
Expected: no output

---

## Task 2: Update `key_derivation_algorithms/index.md`

**Files:**
- Modify: `key_derivation_algorithms/index.md:35` (PBKDF2 §8.1.5 add AEAD note)
- Modify: `key_derivation_algorithms/index.md:75` (Argon2 §8.2.5 update)
- Modify: `key_derivation_algorithms/index.md:87-149` (remove §8.3 HKDF entirely)

- [ ] **Step 0: Add AEAD note to §8.1.5 PBKDF2 Recommendations**

PBKDF2 is also a valid PHSF algorithm. The same "output used directly as GCM key" principle must be stated.

Use Edit on `key_derivation_algorithms/index.md`:

```
old_string:
#### 8.1.5 Recommendations

As per [NIST SP 800-132](../references/index.md#nist-sp-800-132), it is recommended to use at least 10,000 iterations for PBKDF2 when deriving keys for non-interactive applications. However, this value should be increased as computational power advances to ensure the security of the derived keys.

### 8.2. Argon2

new_string:
#### 8.1.5 Recommendations

As per [NIST SP 800-132](../references/index.md#nist-sp-800-132), it is recommended to use at least 10,000 iterations for PBKDF2 when deriving keys for non-interactive applications. However, this value should be increased as computational power advances to ensure the security of the derived keys.

> **Note for AEAD entries.** When PBKDF2 is used as the PHSF algorithm for an AEAD entry (FHED `Cipher mode = 2`), the derived key length MUST be 32 bytes (256 bits) and the output is used directly as the per-entry GCM key. No further key derivation is applied. Argon2id (§8.2) is recommended over PBKDF2 for new AEAD archives.

### 8.2. Argon2

```

- [ ] **Step 1: Update §8.2.5 Argon2 recommendation paragraph**

The current paragraph mentions "master key" because HKDF derives per-entry keys from it. With HKDF removed, Argon2id output IS the per-entry GCM key directly.

Use Edit on `key_derivation_algorithms/index.md`:

```
old_string:
For new PNA archives (AHED Minor version 1 with AEAD entries), encoders SHOULD use Argon2id with the [RFC 9106](../references/index.md#rfc-9106) "first recommended" profile:

| Parameter | Value | Rationale |
|---|---|---|
| Memory cost (m_cost) | 65536 KiB (64 MiB) | RFC 9106 §4 first-recommended profile |
| Time cost (t_cost) | 3 | RFC 9106 §4 first-recommended profile |
| Parallelism (p_cost) | `available_parallelism()` (typically 4) | Adapts to host CPU |
| Salt length | 16 bytes from CSPRNG | RFC 9106 §3.1 minimum |
| Output (master key) length | 32 bytes (256 bits) | Sized for AES-256 / Camellia-256 |

new_string:
For new PNA archives (AHED Minor version 1 with AEAD entries), encoders SHOULD use Argon2id with the [RFC 9106](../references/index.md#rfc-9106) "first recommended" profile:

| Parameter | Value | Rationale |
|---|---|---|
| Memory cost (m_cost) | 65536 KiB (64 MiB) | RFC 9106 §4 first-recommended profile |
| Time cost (t_cost) | 3 | RFC 9106 §4 first-recommended profile |
| Parallelism (p_cost) | `available_parallelism()` (typically 4) | Adapts to host CPU |
| Salt length | 16 bytes from CSPRNG | RFC 9106 §3.1 minimum |
| Output length | 32 bytes (256 bits) | Sized for AES-256 / Camellia-256; used directly as the per-entry GCM key |

> **Note for AEAD entries.** When an entry uses an AEAD `Cipher mode` (e.g., 2 = GCM), the 32-byte Argon2id output is used directly as the GCM key for that entry's FDAT/SDAT chunks. Encoders SHOULD emit byte-identical PHSF chunks for all AEAD entries within a single archive that share a password, so that the password hash is computed only once per archive.
```

- [ ] **Step 2: Remove §8.3 HKDF section entirely**

Use Edit on `key_derivation_algorithms/index.md`. The entire §8.3 spans from `### 8.3. HKDF (HMAC-based Key Derivation Function)` (line 87) through line 149 (end of §8.3.5 Recommendations). Verify the file currently ends after §8.3.5 — if so, delete from the start of the §8.3 heading to the end of file.

```
old_string:
### 8.3. HKDF (HMAC-based Key Derivation Function)
new_string: (delete everything from this heading to end of file)
```

The simplest way: read the file, capture lines 1–86, write back with only those lines.

Run:

```bash
head -86 key_derivation_algorithms/index.md > key_derivation_algorithms/index.md.tmp && mv key_derivation_algorithms/index.md.tmp key_derivation_algorithms/index.md
```

- [ ] **Step 3: Verify HKDF and stale references are gone**

Run: `grep -n "HKDF\|archive_identifier\|entry_random\|AENC\|FENC" key_derivation_algorithms/index.md`
Expected: no output (HKDF section removed; the new note added in Step 1 references neither AENC nor FENC nor HKDF).

Run: `wc -l key_derivation_algorithms/index.md`
Expected: ~86 lines (down from 149).

---

## Task 3: Update `cipher_modes/index.md`

**Files:**
- Modify: `cipher_modes/index.md:28` (§7 intro Minor 1 description)
- Modify: `cipher_modes/index.md:92-118` (§7.4.2 AAD complete rewrite)
- Modify: `cipher_modes/index.md:120-131` (§7.4.3 nonce derivation → random generation)
- Modify: `cipher_modes/index.md:133-140` (§7.4.4 key and nonce uniqueness — remove HKDF/AENC/FENC refs)
- Modify: `cipher_modes/index.md:142-144` (§7.4.5 RNG — only nonce now, no archive_identifier/entry_random)
- Modify: `cipher_modes/index.md:158-200` (§7.5 add 12B nonce to chunk layout)

- [ ] **Step 1: (no §7 intro Minor row update needed)**

The Minor version table is only in `chunk_specifications/index.md`; `cipher_modes/index.md` only has prose mentions of "Minor version 1 or higher" that do not name AENC/FENC. Verify with: `grep -n "AENC\|FENC" cipher_modes/index.md` (must return no output before the rest of Task 3 is applied).

If `grep -n "AENC\|FENC" cipher_modes/index.md` returns matches, those are the AAD/nonce/uniqueness sections rewritten in Steps 2–8 below; proceeding with those steps will eliminate the matches.

- [ ] **Step 2: Replace §7.4.2 AAD section**

Use Edit on `cipher_modes/index.md`:

```
old_string:
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

new_string:
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
```

- [ ] **Step 3: Replace §7.4.3 Nonce Derivation with Nonce Generation**

Use Edit on `cipher_modes/index.md`:

```
old_string:
#### 7.4.3 Nonce Derivation

The 96-bit nonce for each FDAT or SDAT chunk MUST be constructed deterministically as follows:

```
nonce[0..8]   = AENC[cipher_mode_id=2].archive_identifier[0..8]   (per-archive, 8 bytes)
nonce[8..12]  = chunk_index                                       (u32 big-endian, per-entry counter)
```

The `chunk_index` starts at 0 for the first FDAT (or SDAT) chunk of each entry and increments by 1 for each subsequent chunk in the same entry. Indices are NOT shared across entries.

> **Note.** The high-order 8 bytes of `archive_identifier` are reused as the nonce prefix; the low-order 8 bytes serve as additional HKDF input (see [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function)). Splitting the 16-byte `archive_identifier` this way is safe because the per-entry HKDF derivation produces an independent encryption key per entry, so nonce uniqueness only needs to hold within a single (key, nonce) scope, which is guaranteed by `chunk_index`.

new_string:
#### 7.4.3 Nonce Generation

The 96-bit nonce for each FDAT or SDAT chunk MUST be generated by a cryptographically secure pseudorandom number generator (CSPRNG) per NIST SP 800-38D §8.2.2 (RBG-based construction). The encoder MUST generate a fresh 12-byte nonce for every chunk and store it inline as the first 12 bytes of the chunk's data field (see [§7.5](#75-nonce-and-tag-placement-for-aead-modes)).

PNA does NOT use deterministic nonce construction (NIST SP 800-38D §8.2.1) in `Cipher mode = 2`; nonce uniqueness is provided by CSPRNG randomness, not by counter state.
```

- [ ] **Step 4: Update §7.4.4 Key and Nonce Uniqueness**

Use Edit on `cipher_modes/index.md`:

```
old_string:
#### 7.4.4 Key and Nonce Uniqueness

GCM is catastrophically insecure under nonce reuse with the same key (Joux 2006: a single nonce repetition reveals the GHASH key, enabling unlimited forgeries). PNA enforces nonce uniqueness via two mechanisms:

1. **Per-entry key derivation.** Each entry uses a key derived from the archive's master key via HKDF-SHA-256, as specified in [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function). The HKDF inputs include both the per-archive `archive_identifier` (from `AENC`) and the per-entry `entry_random` (from `FENC`), so different entries derive different GCM keys even if some other input collides. Cross-entry GCM nonce collisions are therefore neutralized at the key level.
2. **Monotonic chunk counter within an entry.** Within a single entry, the `chunk_index` is strictly monotonically increasing, guaranteeing nonce uniqueness within (key, nonce) pairs.

Encoders MUST NOT reuse a nonce across distinct invocations under the same per-entry key. Encoders MUST regenerate `AENC.archive_identifier` with a cryptographically secure random number generator for each new archive. Encoders MUST regenerate `FENC.entry_random` for each new entry within an archive.

new_string:
#### 7.4.4 Key and Nonce Uniqueness

GCM is catastrophically insecure under nonce reuse with the same key (Joux 2006: a single nonce repetition reveals the GHASH key, enabling unlimited forgeries). PNA enforces nonce uniqueness via:

1. **CSPRNG-generated 96-bit nonces (§7.4.3).** The probability of nonce collision under one key after `N` invocations is approximately `N² / 2^97`. NIST SP 800-38D §8.3 limits this to `N ≤ 2^32` (collision probability ≤ 2^-33).
2. **Bounded invocations per key.** A single PHSF chunk derives one GCM key (from password + salt). All entries sharing the same PHSF chunk share the same key; the cumulative chunk count under that key MUST NOT exceed 2^32. Encoders writing more than 2^32 chunks under one password MUST emit at least one entry with a fresh PHSF salt to start a new key.

Encoders MUST NOT reuse a nonce under the same key. Detecting CSPRNG output collision is generally infeasible for the encoder; the 2^32 invocations cap is the operational mitigation.
```

- [ ] **Step 5: Update §7.4.5 Random Number Generation**

Use Edit on `cipher_modes/index.md`:

```
old_string:
#### 7.4.5 Random Number Generation

The cryptographic random fields (`AENC.archive_identifier`, `FENC.entry_random`) MUST be generated using a cryptographically secure pseudorandom number generator (CSPRNG). On systems where the CSPRNG can fail or block (e.g., early-boot Linux without sufficient entropy), encoders MUST surface the failure and abort archive creation rather than silently producing weak random values.

new_string:
#### 7.4.5 Random Number Generation

The 12-byte nonce of each FDAT/SDAT chunk MUST be generated using a cryptographically secure pseudorandom number generator (CSPRNG). On systems where the CSPRNG can fail or block (e.g., early-boot Linux without sufficient entropy), encoders MUST surface the failure and abort archive creation rather than silently producing weak random values.
```

- [ ] **Step 6: Replace §7.5 to add 12B nonce to chunk layout**

Use Edit on `cipher_modes/index.md`:

```
old_string:
### 7.5. Nonce and Tag Placement for AEAD Modes

When using an AEAD mode (currently GCM, [§7.4](#74-galoiscounter-mode-gcm)), PNA does NOT use the IV-prepend layout described in [§7.3](#73-initialization-vector-iv-placement-in-encrypted-streams). Instead, the nonce is derived deterministically per chunk (§7.4.3) and is NOT stored in the chunk data. The authentication tag is appended to each chunk's ciphertext.

#### 7.5.1 FDAT and SDAT Layout under AEAD

Each FDAT (per-entry) or SDAT (solid mode) chunk's data field, when the entry's `Cipher mode` field is 2 (GCM), MUST be laid out as:

```
[ciphertext data ...] [16-byte authentication tag]
```

The chunk's standard length field counts both the ciphertext and the trailing tag. The tag occupies the final 16 bytes of the chunk's data field, regardless of the chunk's data length.

> **Note.** The chunk-level CRC32 (already required for every chunk) covers the entire `(ciphertext || tag)` byte sequence. CRC32 detects transit errors but provides no cryptographic authenticity; the GCM tag is the authoritative integrity check.

new_string:
### 7.5. Nonce and Tag Placement for AEAD Modes

When using an AEAD mode (currently GCM, [§7.4](#74-galoiscounter-mode-gcm)), PNA does NOT use the IV-prepend layout described in [§7.3](#73-initialization-vector-iv-placement-in-encrypted-streams). Instead, each chunk carries its own random nonce inline (§7.4.3 / §7.4.5) and the authentication tag appended at the end.

#### 7.5.1 FDAT and SDAT Layout under AEAD

Each FDAT (per-entry) or SDAT (solid mode) chunk's data field, when the entry's `Cipher mode` field is 2 (GCM), MUST be laid out as:

```
[12-byte random nonce] [ciphertext data ...] [16-byte authentication tag]
```

The chunk's standard length field counts the nonce, ciphertext, and trailing tag (`12 + len(ciphertext) + 16`). The minimum data field length is therefore 28 bytes (12 nonce + 0 ciphertext + 16 tag); a chunk shorter than 28 bytes is malformed and MUST be rejected by the decoder.

> **Note.** The chunk-level CRC32 (already required for every chunk) covers the entire `(nonce || ciphertext || tag)` byte sequence. CRC32 detects transit errors but provides no cryptographic authenticity; the GCM tag is the authoritative integrity check.
```

- [ ] **Step 7: Update §7.5.2 Decoder Behavior**

The current §7.5.2 instructs the decoder to derive the nonce. With inline nonce, the decoder reads it.

Use Edit on `cipher_modes/index.md`:

```
old_string:
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

new_string:
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
```

- [ ] **Step 8: Update §7.5.3 Encoder Behavior**

Use Edit on `cipher_modes/index.md`:

```
old_string:
For each FDAT or SDAT chunk emitted under GCM mode, an encoder MUST:

1. Determine the chunk's plaintext payload (typically a fixed-size segment such as 64 KiB; see [Recommendations for Encoders](../recommendations_for_encoders/index.md)).
2. Compute the chunk's AAD as specified in [§7.4.2](#742-associated-data-aad).
3. Compute the chunk's nonce as specified in [§7.4.3](#743-nonce-derivation).
4. Encrypt the plaintext under the per-entry GCM key with the computed nonce and AAD, producing a ciphertext of the same length as the plaintext and a 16-byte tag.
5. Emit a single FDAT (or SDAT) chunk whose data field is the concatenation `(ciphertext || tag)`.
6. For the chunk preceding FEND (or SEND), set the final-chunk flag in the AAD to `0x01`. For all other chunks, the flag MUST be `0x00`.

new_string:
For each FDAT or SDAT chunk emitted under GCM mode, an encoder MUST:

1. Determine the chunk's plaintext payload (typically a fixed-size segment such as 64 KiB; see [Recommendations for Encoders](../recommendations_for_encoders/index.md)).
2. Compute the chunk's AAD as specified in [§7.4.2](#742-associated-data-aad).
3. Generate a fresh 12-byte nonce from a CSPRNG (§7.4.5).
4. Encrypt the plaintext under the per-entry GCM key with the generated nonce and AAD, producing a ciphertext of the same length as the plaintext and a 16-byte tag.
5. Emit a single FDAT (or SDAT) chunk whose data field is the concatenation `(nonce || ciphertext || tag)`.
6. For the chunk preceding FEND (or SEND), set the final-chunk flag in the AAD to `0x01`. For all other chunks, the flag MUST be `0x00`.
```

- [ ] **Step 9: Verify cipher_modes/index.md is consistent**

Run: `grep -n "AENC\|FENC\|HKDF\|archive_identifier\|entry_random\|PNA-AEAD-v1\|metadata_hash\|Metadata hash" cipher_modes/index.md`
Expected: no output.

Run: `grep -n "12-byte\|11 bytes" cipher_modes/index.md`
Expected: at least one match for "12-byte" (in §7.5 layout) and one for "11 bytes" (in §7.4.2 AAD total).

---

## Task 4: Update `chunk_specifications/index.md`

**Files:**
- Modify: `chunk_specifications/index.md:28` (AHED Minor 1 description)
- Modify: `chunk_specifications/index.md:106` (FHED Cipher mode 2 note — remove AENC/FENC requirement)
- Modify: `chunk_specifications/index.md:152` (PHSF AEAD note — simplify, remove HKDF/AENC/FENC)
- Modify: `chunk_specifications/index.md:154-206` (remove §4.1.x AENC and §4.1.x FENC sections entirely)

- [ ] **Step 1: Update AHED Minor 1 description**

Use Edit on `chunk_specifications/index.md`:

```
old_string:
| `1` | Authenticated Encryption with Associated Data (AEAD) is added (in addition to CBC/CTR). Introduces the `AENC` (per-archive) and `FENC` (per-entry) chunk types and reserves `Cipher mode` value 2 for GCM. See [§7.4](../cipher_modes/index.md#74-galoiscounter-mode-gcm). |

new_string:
| `1` | Authenticated Encryption with Associated Data (AEAD) is added (in addition to CBC/CTR). Reserves `Cipher mode` value 2 for GCM. No new chunk types are introduced. See [§7.4](../cipher_modes/index.md#74-galoiscounter-mode-gcm). |
```

- [ ] **Step 2: Update FHED Cipher mode 2 explanatory paragraph**

Use Edit on `chunk_specifications/index.md`:

```
old_string:
When `Cipher mode = 2` (GCM), the entry's data layout follows the AEAD rules defined in [§7.4 GCM](../cipher_modes/index.md#74-galoiscounter-mode-gcm) and [§7.5 Nonce and Tag Placement for AEAD Modes](../cipher_modes/index.md#75-nonce-and-tag-placement-for-aead-modes). The archive MUST contain an `AENC` chunk for `cipher_mode_id = 2` after AHED ([§4.1.x](#41x-aenc-archive-encryption-context)), and the entry MUST contain a `FENC` chunk after FHED ([§4.1.x](#41x-fenc-per-entry-encryption-context)).

new_string:
When `Cipher mode = 2` (GCM), the entry's data layout follows the AEAD rules defined in [§7.4 GCM](../cipher_modes/index.md#74-galoiscounter-mode-gcm) and [§7.5 Nonce and Tag Placement for AEAD Modes](../cipher_modes/index.md#75-nonce-and-tag-placement-for-aead-modes). The entry's PHSF chunk derives the per-entry GCM key directly (no HKDF; see [§4.1.5 PHSF](#415-phsf-password-hash)).
```

- [ ] **Step 3: Replace PHSF AEAD note**

Use Edit on `chunk_specifications/index.md`:

```
old_string:
> **Note for AEAD entries.** When the entry uses an AEAD `Cipher mode` (e.g., 2 = GCM), the password hash described by PHSF derives the **archive master key**, not the per-entry encryption key. The per-entry encryption key is derived from the master key via HKDF (see [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function)) using two inputs: the per-archive `archive_identifier` carried in the `AENC` chunk for the entry's cipher_mode, and the per-entry `entry_random` carried in the entry's `FENC` chunk.

new_string:
> **Note for AEAD entries.** When the entry uses an AEAD `Cipher mode` (e.g., 2 = GCM), the password hash described by PHSF (Argon2id or PBKDF2 output, 32 bytes) is used directly as the per-entry GCM key. No further key derivation is applied. Encoders SHOULD emit byte-identical PHSF chunks for all AEAD entries within a single archive that share a password, so that the password hash is computed only once per archive (see [§8.2.5](../key_derivation_algorithms/index.md#825-recommendations)).
```

- [ ] **Step 4: Remove §4.1.x AENC and §4.1.x FENC sections entirely**

Use Edit on `chunk_specifications/index.md`. The sections span from `#### 4.1.x. AENC Archive ENcryption Context` (line 154) through the end of the FENC section (line 206), immediately before `### 4.1.6. FDAT File data` (line 208).

```
old_string:
#### 4.1.x. AENC Archive ENcryption Context

The `AENC` chunk carries per-archive AEAD context data for one cipher mode. It uses an inline-tag schema so that the same chunk type can serve any AEAD cipher mode, and it carries an explicit framework version field for future schema evolution.

new_string:
(delete from this point until the line immediately before "### 4.1.6. FDAT File data")
```

The simplest approach: locate the start (line 154) and end (line 207, blank line before §4.1.6), and replace that entire range with nothing. Confirm via grep first:

Run: `grep -n "^#### 4.1.x\|^### 4.1.6" chunk_specifications/index.md`
Expected: lines 154 (AENC), 185 (FENC), 208 (4.1.6 FDAT).

Then use a single Edit covering the full span. The `old_string` is the entire block from `#### 4.1.x. AENC` through the blank line before `### 4.1.6. FDAT File data`. The `new_string` is empty.

Easiest concrete approach: read lines 154–207 verbatim, paste as `old_string`, set `new_string` to empty.

- [ ] **Step 5: Verify chunk_specifications/index.md is clean**

Run: `grep -n "AENC\|FENC\|archive_identifier\|entry_random\|HKDF" chunk_specifications/index.md`
Expected: no output.

Run: `grep -n "^### 4.1.6. FDAT" chunk_specifications/index.md`
Expected: line 154 (was 208, shifted up by ~54 deleted lines).

---

## Task 5: Update `recommendations_for_decoders/index.md`

**Files:**
- Modify: `recommendations_for_decoders/index.md:40` (§12.3.1 unknown-chunk paragraph)
- Modify: `recommendations_for_decoders/index.md:42-53` (§12.3.2 Authentication tag verification — fix nonce reading and remove HKDF/§8.3 ref)
- Modify: `recommendations_for_decoders/index.md:55-57` (§12.3.3 Truncation detection — rename `is_final_chunk` for consistency)
- Modify: `recommendations_for_decoders/index.md:59-66` (§12.3.4 Multipart consistency)
- Modify: `recommendations_for_decoders/index.md:74` (§12.3.5 error categories)

- [ ] **Step 1: Replace §12.3.1 Version handling note about unknown chunks**

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
A decoder that does not support AEAD (e.g., a legacy decoder built only against PNA spec Minor version 0) will encounter the AEAD-specific critical chunks `AENC` (in the archive) and `FENC` (in the entry) when reading a Minor=1 archive that uses AEAD entries, and MUST report a fatal error per the standard "unknown critical chunk" rule. Decoders SHOULD include the unrecognized chunk type code in the error message for diagnostics.

new_string:
A decoder that does not support AEAD (e.g., a legacy decoder built only against PNA spec Minor version 0) will encounter `Cipher mode = 2` in an FHED chunk when reading a Minor=1 archive that uses AEAD entries. Per [§4.1.4 FHED](../chunk_specifications/index.md#414-fhed-file-header), an unrecognized `Cipher mode` value in a critical chunk is a fatal error; the decoder MUST report it and SHOULD include the offending value (`2`) in the error message for diagnostics.
```

- [ ] **Step 2: Update §12.3.2 Authentication tag verification (fix nonce reading and HKDF reference)**

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
For every FDAT or SDAT chunk in an entry whose `Cipher mode` is 2 (GCM), decoders MUST:

1. Extract the trailing 16 bytes of the chunk's data field as the authentication tag.
2. Reconstruct the AAD per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad).
3. Reconstruct the nonce per [§7.4.3](../cipher_modes/index.md#743-nonce-derivation).
4. Verify the tag using the per-entry GCM key (derived per [§8.3](../key_derivation_algorithms/index.md#83-hkdf-hmac-based-key-derivation-function)).

new_string:
For every FDAT or SDAT chunk in an entry whose `Cipher mode` is 2 (GCM), decoders MUST:

1. Extract the leading 12 bytes of the chunk's data field as the nonce.
2. Extract the trailing 16 bytes of the chunk's data field as the authentication tag.
3. Treat bytes `[12 .. N-16]` as the ciphertext (where `N` is the chunk data field length).
4. Construct the AAD per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad).
5. Verify the tag using the per-entry GCM key derived directly from the entry's PHSF chunk (see [§4.1.5 PHSF](../chunk_specifications/index.md#415-phsf-password-hash); no HKDF is applied in this AEAD design).
```

(Note: this changes the numbered steps from 1–4 to 1–5; subsequent step numbering in the same subsection — items 5/6 in the original "NOT release plaintext" / "On tag verification failure" — must also be renumbered. Apply the following Edit immediately after to renumber.)

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
5. **NOT release any decrypted plaintext to upper layers (the application, the file system, or the user) until tag verification has succeeded for that chunk.** This is the "Releasing Unverified Plaintext" prohibition (Bellare-Namprempre 2000).
6. On tag verification failure, treat the entire entry as corrupted: discard any plaintext already buffered for that entry, report an authentication failure (distinguishable from format errors and I/O errors), and cease processing the entry. The decoder MAY continue with subsequent entries in the archive.

new_string:
6. **NOT release any decrypted plaintext to upper layers (the application, the file system, or the user) until tag verification has succeeded for that chunk.** This is the "Releasing Unverified Plaintext" prohibition (Bellare-Namprempre 2000).
7. On tag verification failure, treat the entire entry as corrupted: discard any plaintext already buffered for that entry, report an authentication failure (distinguishable from format errors and I/O errors), and cease processing the entry. The decoder MAY continue with subsequent entries in the archive.
```

- [ ] **Step 3: Update §12.3.3 Truncation detection (rename flag for consistency)**

The new §7.4.2 AAD spec calls this field "Final-chunk flag", not "is_final_chunk". Rename for consistency.

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
The final chunk of each entry's encrypted data stream carries an AAD `is_final_chunk` flag value of `0x01` (per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad)). Decoders MUST refuse to complete extraction of an entry if no chunk with `is_final_chunk = 0x01` has been authenticated before reaching FEND or SEND. A truncated archive in which the last data chunk was removed will fail authentication on the new "last" chunk (whose AAD was computed with `is_final_chunk = 0x00`) — this is the intended detection mechanism and decoders MUST report it as an authentication failure, NOT as a successful early termination.

new_string:
The final chunk of each entry's encrypted data stream carries an AAD `Final-chunk flag` value of `0x01` (per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad)). Decoders MUST refuse to complete extraction of an entry if no chunk with `Final-chunk flag = 0x01` has been authenticated before reaching FEND or SEND. A truncated archive in which the last data chunk was removed will fail authentication on the new "last" chunk (whose AAD was computed with `Final-chunk flag = 0x00`) — this is the intended detection mechanism and decoders MUST report it as an authentication failure, NOT as a successful early termination.
```

- [ ] **Step 4: Replace §12.3.4 Multipart consistency**

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
#### 12.3.4 Multipart consistency

For multipart archives, decoders MUST verify that every `AENC` chunk in every part contains byte-identical values across all parts (compared per `cipher_mode_id`). Any mismatch, or missing/extra `AENC` chunks compared to other parts, indicates one of:

- A part has been substituted with a part from a different archive (cross-archive substitution attack).
- A part has been corrupted or tampered with.

In either case, the decoder MUST report a fatal error and refuse to extract any entry whose data spans the inconsistent boundary.

new_string:
#### 12.3.4 Multipart consistency

In this AEAD design, AEAD context is per-entry (no archive-wide AEAD state chunks). Multipart archives therefore need no AEAD-specific consistency check beyond the standard part-linking rules ([§3.3 Multipart](../file_structure/index.md)).

Cross-archive part substitution defense is provided indirectly by the per-entry PHSF salt: a substituted entry from a different archive (with a different PHSF salt) will derive a different GCM key, causing tag verification to fail on the very first FDAT/SDAT chunk of the substituted entry. Decoders MUST treat this failure as an authentication failure (per [§12.3.5](#1235-error-reporting)), not as a generic format error, so that adversarial substitution can be distinguished from transit corruption.
```

- [ ] **Step 5: Update §12.3.5 Error reporting categories**

Use Edit on `recommendations_for_decoders/index.md`:

```
old_string:
- **Cryptographic configuration error**: missing required `AENC` (for any AEAD `cipher_mode` used in the archive) or `FENC` chunk, missing `PHSF` chunk, unknown `Encryption method` or `Cipher mode` value, AENC `framework_version` unsupported by the decoder.

new_string:
- **Cryptographic configuration error**: missing `PHSF` chunk for an AEAD entry, unknown `Encryption method` or `Cipher mode` value.
```

- [ ] **Step 6: Verify recommendations_for_decoders/index.md is clean**

Run: `grep -n "AENC\|FENC\|archive_identifier\|entry_random\|HKDF\|cipher_mode_id\|is_final_chunk" recommendations_for_decoders/index.md`
Expected: no output.

---

## Task 6: Cross-reference verification (whole-spec grep)

- [ ] **Step 1: Whole-spec grep for stale terms**

Run from the spec repo root:

```bash
grep -rn "AENC\|FENC\|HKDF\|archive_identifier\|entry_random\|PNA-AEAD-v1\|RFC 5869\|RFC-5869\|metadata_hash\|Metadata hash\|cipher_mode_id\|framework_version" --include="*.md" . | grep -v "docs/superpowers/"
```

Expected: no output (the design doc and this plan in `docs/superpowers/` are intentionally excluded).

If any output appears, stop and fix the matched line. Common cases:
- A link target text like `[§4.1.x AENC](...)` was missed in an Edit.
- A paragraph in `references/index.md` references HKDF in a context unrelated to RFC 5869.

- [ ] **Step 2: Verify all internal anchor links resolve**

Run from the spec repo root (lists internal anchor links for manual scan):

```bash
grep -rEho '\(\.\./[^)]+\.md#[a-z0-9-]+\)' --include="*.md" . | sort -u | head -40
```

Compare against actual section headings:

```bash
grep -rEh '^####? ' --include="*.md" . | head -60
```

Spot-check that no link points to a removed section (e.g., `#83-hkdf-hmac-based-key-derivation-function`, `#41x-aenc-archive-encryption-context`, `#41x-fenc-per-entry-encryption-context`).

Run specifically for these removed anchors:

```bash
grep -rn "#83-hkdf\|#41x-aenc\|#41x-fenc\|#743-nonce-derivation" --include="*.md" . | grep -v "docs/superpowers/"
```

Expected: no output. (Note: §7.4.3 was renamed from "Nonce Derivation" to "Nonce Generation"; old `#743-nonce-derivation` anchor is dead. Replace any matches with `#743-nonce-generation`.)

- [ ] **Step 3: Verify line counts moved as expected**

Run:

```bash
wc -l chunk_specifications/index.md cipher_modes/index.md key_derivation_algorithms/index.md recommendations_for_decoders/index.md references/index.md
```

Expected (approximate):
- `chunk_specifications/index.md`: ~478 lines (was 532; -54 from AENC/FENC removal)
- `cipher_modes/index.md`: ~190 lines (was 201; -11 net from AAD shrink and §7.5 expansion)
- `key_derivation_algorithms/index.md`: ~88 lines (was 149; -61 from §8.3 removal, +2 from §8.2.5 note)
- `recommendations_for_decoders/index.md`: ~75 lines (was 79; -4 net from rewrites)
- `references/index.md`: 73 lines (was 76; -3 from RFC 5869 removal)

If counts are wildly off (>20% deviation), re-inspect the affected file before committing.

---

## Task 7: Final commit

- [ ] **Step 1: Pre-commit hygiene**

Run:

```bash
pwd
git remote -v
git status
git branch --show-current
```

Expected:
- `pwd`: `/Users/tsunekawa/Documents/GitHub/Portable-Network-Archive-Specification`
- `git remote`: `Portable-Network-Archive-Specification.git`
- Branch: `spec/aead-gcm-introduction`
- Modified: 5 files (chunk_specifications, cipher_modes, key_derivation_algorithms, recommendations_for_decoders, references)

- [ ] **Step 2: Stage all 5 files**

Run:

```bash
git add chunk_specifications/index.md cipher_modes/index.md key_derivation_algorithms/index.md recommendations_for_decoders/index.md references/index.md
git diff --cached --stat
```

Expected output: 5 files changed, with insertions/deletions summing to roughly: ~150 deletions, ~50 insertions (net negative since this is a simplification).

- [ ] **Step 3: Commit**

Run:

```bash
git commit -m ":memo: Apply AEAD minimal redesign: per-entry, no new chunks" -m "Replaces the AENC/FENC framework drafted in a31aa01 with a minimal
design that requires zero new chunk types. Per the redesign spec
(docs/superpowers/specs/2026-04-26-aead-minimal-redesign.md):

- Remove AENC and FENC chunk specifications.
- Remove HKDF section (§8.3) from key derivation algorithms.
- Remove RFC 5869 reference.
- Rewrite cipher_modes §7.4.2 AAD: 86 bytes -> 11 bytes.
- Rewrite cipher_modes §7.4.3 nonce: deterministic derivation -> CSPRNG.
- Rewrite cipher_modes §7.5 layout: add 12-byte inline nonce per chunk.
- Update PHSF note: Argon2id output used directly as GCM key.
- Update Decoder recommendations §12.3 to drop AENC/FENC consistency
  checks and document multipart cross-substitution defense via PHSF
  salt natural divergence.

This commit supersedes the chunk additions in a31aa01 but preserves
the AHED Minor=1 + FHED Cipher mode=2 reservations introduced there."
```

- [ ] **Step 4: Verify commit landed cleanly**

Run:

```bash
git log --oneline -3
git show --stat HEAD
```

Expected: latest commit modifies 5 files; previous commits are `e25e5d4` (design doc) and `a31aa01` (initial AENC/FENC draft).

- [ ] **Step 5: Do NOT push**

Per `feedback_stop_at_push.md`: push only with explicit user instruction. Stop here and report the local commit hash to the user.

---

## Self-Review (executed by plan author after writing this plan)

**1. Spec coverage:** Each migration step listed in the design doc's "Migration from Prior Draft" section maps to a specific Task in this plan:
- chunk_specifications/index.md changes → Task 4 (Steps 1–4)
- cipher_modes/index.md changes → Task 3 (Steps 1–8)
- key_derivation_algorithms/index.md changes → Task 2 (Steps 1–2)
- recommendations_for_decoders/index.md changes → Task 5 (Steps 1–3)
- references/index.md changes → Task 1 (Step 1)
✅ No gaps.

**2. Placeholder scan:** No "TBD", "TODO", "implement later", "fill in details", or "similar to Task N" patterns. Each Edit shows the exact `old_string` and `new_string`.

**3. Type/anchor consistency:** §7.4.3 is renamed from "Nonce Derivation" to "Nonce Generation"; Task 6 Step 2 explicitly grep-checks for the dead `#743-nonce-derivation` anchor. AAD width is consistently stated as 11 bytes. Per-FDAT layout is consistently `[12B nonce][ciphertext][16B tag]` across cipher_modes (§7.5.1 and §7.5.3) and the decoder/encoder behavior subsections.

**4. Commit boundary:** All 5 file edits land in one atomic commit (Task 7) so cross-references between files never break in intermediate states.

---

## Out-of-scope

The following are deferred to separate plans (per `feedback_independent_entities.md`):

- **libpna implementation revision** (different repo: `Portable-Network-Archive`). The libpna `lib/aead-prototype` branch contains an `AeadContext` skeleton that assumes the prior 86-byte AAD and HKDF derivation; revising it to match this minimal design is its own plan.
- **Spec PR creation / push to `origin`**. This plan stops at local commit; pushing requires explicit user instruction.
