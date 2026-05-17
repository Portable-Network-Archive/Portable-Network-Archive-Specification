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
