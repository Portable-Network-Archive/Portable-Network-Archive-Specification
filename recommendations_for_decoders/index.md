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

### 12.3. Owner information chunks

When both `fPRM` and the owner information chunks (§4.2.6) are present, a decoder should prefer the §4.2.6 chunks. When only `fPRM` is present, a decoder may read it as a fallback for archives that predate §4.2.6. A malformed individual facet chunk should be treated as that facet being absent, without failing the entry or affecting the other facets. The absence of a facet means the value was not recorded; a decoder should not substitute a default such as user ID 0, root, or a fixed permission mode.

The owner facets are alternative representations of one identity, and this specification does not define a resolution order among them. A decoder should select a facet according to the target platform and policy. For example, on Windows a decoder may prefer `fOSi`, then `fONm`, then `fUId` only under an explicit numeric-owner option; on POSIX systems it may prefer `fONm`, then `fUId`. The `fONm` and `fGNm` values are opaque strings that may be qualified (for example `user@domain`); resolving any such qualification is left to the decoder's name resolver.

### 12.4. AEAD-specific decoder behavior

A decoder processes an encrypted datastream with `Cipher mode = 2` (GCM, [§7.4](../cipher_modes/index.md#74-galoiscounter-mode-gcm)) according to [§7.5](../cipher_modes/index.md#75-aead-datastream-layout).

1. Read the first 43 bytes of the datastream as the stream header. If `segment_size` is 0 or exceeds 67,108,864 (64 MiB), the datastream must be treated as **malformed**.
2. Derive `K_stream` ([§8.3](../key_derivation_algorithms/index.md#83-hkdf-aead-stream-key-derivation)) and read segments in order. Until a segment's tag verification succeeds, its plaintext must not be passed to any upper layer, including decompression.
3. Whether a segment is final must be determined before verification, because it selects the final flag of the nonce. The decoder reads one segment ahead and treats a segment as final only when no datastream bytes follow it.
4. A tag verification failure is an **authentication failure**. It should be reported distinctly from format errors and I/O errors. No further segments may be processed. Because GCM cannot in principle distinguish a wrong password from tampering, an API that claims to distinguish the two should not be provided.
5. If bytes remain in the datastream after successful verification of a segment with final flag `0x01`, the datastream must be treated as **malformed**.
6. If the datastream is exhausted before a final segment verifies successfully, this is a **truncation error**. The entry as a whole must be reported as incomplete, even if previously verified plaintext has already been released to upper layers. If the datastream ends with fewer than 16 bytes remaining and no segment has been verified yet, the datastream contains no complete segment and decoders should report it as **malformed** (equivalent to a total length below the 59-byte minimum). If at least one segment has already been verified, decoders should report **truncation**.

#### 12.4.1 Unauthenticated metadata

In per-entry mode, ancillary chunks are neither encrypted nor authenticated. A decoder should not, by default, apply setuid/setgid bits, capabilities, or ACLs taken from unauthenticated ancillary chunks.

For use cases that require confidentiality and authenticity of metadata, solid mode is appropriate: in solid mode the whole entry is serialized inside the SDAT stream and is therefore encrypted and authenticated.
