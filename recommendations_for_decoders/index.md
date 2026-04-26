## 12. Recommendations for Decoders

This chapter gives some recommendations for decoder behavior. The only absolute requirement on a PNA decoder is that it successfully reads any file conforming to the format specified in the preceding chapters. However, best results will usually be achieved by following these recommendations.

### 12.1. Error checking

To ensure early detection of common file-transfer problems, decoders should verify that all eight bytes of the PNA file signature are correct. (See Rationale: [PNA file signature](../file_structure/index.md#31-pna-file-signature).) A decoder can have additional confidence in the file's integrity if the next eight bytes are an IHDR chunk header with the correct chunk length.

Unknown chunk types must be handled as described in [Chunk naming conventions](../file_structure/index.md#36-chunk-naming-conventions). An unknown chunk type is not to be treated as an error unless it is a critical chunk.

It is strongly recommended that decoders should verify the CRC on each chunk.

In some situations, it is desirable to check chunk headers (length and type code) before reading the chunk data and CRC. The chunk type can be checked for plausibility by seeing whether all four bytes are ASCII letters (codes 65-90 and 97-122); note that this needs to be done only for unrecognized type codes. If the total file size is known (from file system information, HTTP protocol, etc), the chunk length can be checked for plausibility as well.

For known-length chunks such as AHED, decoders should treat an unexpected chunk length as an error. Future extensions to this specification will not add new fields to existing chunks; instead, new chunk types will be added to carry new information.

Unexpected values in fields of known chunks (for example, an unexpected compression method in the FHED chunk) must be checked for and treated as errors. However, it is recommended that unexpected field values be treated as fatal errors only in critical chunks. An unexpected value in an ancillary chunk can be handled by ignoring the whole chunk as though it were an unknown chunk type. (This recommendation assumes that the chunk's CRC has been verified. In decoders that do not check CRCs, it is safer to treat any unexpected value as indicating a corrupted file.)

### 12.2. Text processing

If practical, decoders should have a way to display to the user all text found in the file. Even if the decoder does not recognize a particular text keyword, the user might be able to understand it.

If there are no specific instructions, text is assumed to be represented in UTF-8 without a BOM, but decoders should not rely on this. Decoders should always verify whether the text is UTF-8 encoded, considering the possibility that it might not be.

### 12.3. AEAD-specific decoder behavior

This section applies when a decoder encounters an entry with `Cipher mode = 2` (GCM, [§7.4](../cipher_modes/index.md#74-galoiscounter-mode-gcm)) or any other AEAD cipher mode introduced by future revisions of this specification.

#### 12.3.1 Version handling

Decoders MUST inspect the AHED `Major version` and `Minor version` fields BEFORE attempting to decode any entry:

| AHED Major | AHED Minor | Decoder behavior |
|---|---|---|
| 0 | 0 | Process as legacy archive (CBC/CTR only). |
| 0 | 1 | Process; entries MAY use CBC/CTR or GCM. Requires AEAD-capable decoder for any AEAD entry. |
| 0 | ≥ 2 (future) | Process best-effort; unknown critical chunk types added by higher minor versions will trigger fatal errors per [§12.1](#121-error-checking). |
| ≥ 1 (future) | any | MUST reject if the decoder does not implement that Major version. |

A decoder that does not support AEAD (e.g., a legacy decoder built only against PNA spec Minor version 0) will encounter `Cipher mode = 2` in an FHED chunk when reading a Minor=1 archive that uses AEAD entries. Per [§4.1.4 FHED](../chunk_specifications/index.md#414-fhed-file-header), an unrecognized `Cipher mode` value in a critical chunk is a fatal error; the decoder MUST report it and SHOULD include the offending value (`2`) in the error message for diagnostics.

#### 12.3.2 Authentication tag verification (mandatory)

For every FDAT or SDAT chunk in an entry whose `Cipher mode` is 2 (GCM), decoders MUST:

1. Extract the leading 12 bytes of the chunk's data field as the nonce.
2. Extract the trailing 16 bytes of the chunk's data field as the authentication tag.
3. Treat bytes `[12 .. N-16]` as the ciphertext (where `N` is the chunk data field length).
4. Construct the AAD per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad).
5. Verify the tag using the per-entry GCM key derived directly from the entry's PHSF chunk (see [§4.1.5 PHSF](../chunk_specifications/index.md#415-phsf-password-hash); the AEAD design applies no additional key derivation step).
6. **NOT release any decrypted plaintext to upper layers (the application, the file system, or the user) until tag verification has succeeded for that chunk.** This is the "Releasing Unverified Plaintext" prohibition (Bellare-Namprempre 2000).
7. On tag verification failure, treat the entire entry as corrupted: discard any plaintext already buffered for that entry, report an authentication failure (distinguishable from format errors and I/O errors), and cease processing the entry. The decoder MAY continue with subsequent entries in the archive.

Decoders MUST use a constant-time comparison (e.g., subtle XOR-and-OR reduction) when comparing the computed tag with the chunk's stored tag, to avoid leaking information through timing side channels.

#### 12.3.3 Truncation detection

The final chunk of each entry's encrypted data stream carries an AAD `Final-chunk flag` value of `0x01` (per [§7.4.2](../cipher_modes/index.md#742-associated-data-aad)). Decoders MUST refuse to complete extraction of an entry if no chunk with `Final-chunk flag = 0x01` has been authenticated before reaching FEND or SEND. A truncated archive in which the last data chunk was removed will fail authentication on the new "last" chunk (whose AAD was computed with `Final-chunk flag = 0x00`) — this is the intended detection mechanism and decoders MUST report it as an authentication failure, NOT as a successful early termination.

#### 12.3.4 Multipart consistency

In this AEAD design, AEAD context is per-entry (no archive-wide AEAD state chunks). Multipart archives therefore need no AEAD-specific consistency check beyond the standard part-linking rules ([§3.3 Multipart](../file_structure/index.md)).

Cross-archive part substitution defense is provided indirectly by the per-entry PHSF salt: a substituted entry from a different archive (with a different PHSF salt) will derive a different GCM key, causing tag verification to fail on the very first FDAT/SDAT chunk of the substituted entry. Decoders MUST treat this failure as an authentication failure (per [§12.3.5](#1235-error-reporting)), not as a generic format error, so that adversarial substitution can be distinguished from transit corruption.

#### 12.3.5 Error reporting

To aid forensic analysis and user debugging, decoders SHOULD distinguish the following error categories in user-visible diagnostics:

- **Format error**: chunk length, type code, or CRC32 invalid.
- **Version error**: AHED Major version unsupported, or AHED Minor version requires features the decoder does not implement.
- **Cryptographic configuration error**: missing `PHSF` chunk for an AEAD entry, unknown `Encryption method` or `Cipher mode` value.
- **Authentication failure**: GCM tag verification failed for one or more chunks in an entry.
- **Truncation**: archive ended before AEND, or before the entry's final-flagged chunk.

The cryptographic categories (Authentication failure, Truncation) carry the strongest implication of adversarial activity and SHOULD be surfaced clearly in CLI output and programmatic APIs, not merged into a generic "I/O error" or "format error" category.
