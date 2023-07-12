# PNA (Portable Network Archive) Specification

## Status of this Document

This is a PNA 0.0 specification and is subject to change without notice as it is still under development.

## Abstract

This document describes PNA (Portable Network Archive), an extensible file format for the lossless, portable, well-compressed archive of files. PNA can also replace many common uses of zip and tar. Multiple types of compression, including Zstanderd, multiple types of encryption, including AES, and their solid modes, plus optional archive splitting are supported.

PNA is robust, providing both full file integrity checking and simple detection of common transmission errors.

This specification defines the Internet Media Type "application/pna".

## Table of Contents

- [1. Introduction](./introduction/index.md)
- [2. Data Representation](./data_representation/index.md#2-data-representation)
  - [2.1 Archive layout](./data_representation/index.md#21-archive-layout)
  - [2.2. Integers and byte order](./data_representation/index.md#22-integers-and-byte-order)
  - [2.3 Text encodings](./data_representation/index.md#23-text-encodings)
- [3. File Structure](./file_structure/index.md#3-file-structure)
  - [3.1. PNA file signature](./file_structure/index.md#31-pna-file-signature)
  - [3.2. Entry layout](./file_structure/index.md#32-entry-layout)
  - [3.3. Entry kind](./file_structure/index.md#33-entry-kind)
  - [3.4. Special entry](./file_structure/index.md#34-special-entry)
  - [3.5. Chunk layout](./file_structure/index.md#35-chunk-layout)
  - [3.6. Chunk naming conventions](./file_structure/index.md#36-chunk-naming-conventions)
  - [3.7. CRC algorithm](./file_structure/index.md#37-crc-algorithm)
- [4. Chunk Specifications](./chunk_specifications/index.md)
  - [4.1. Critical chunks](./chunk_specifications/index.md#41-critical-chunks)
    - [4.1.1. AHED Archive header](./chunk_specifications/index.md#411-ahed-archive-header)
    - [4.1.2. AEND Archive tailer](./chunk_specifications/index.md#412-aend-archive-tailer)
    - [4.1.3. ANXT Archive continue marker](./chunk_specifications/index.md#413-anxt-archive-continue-marker)
    - [4.1.4. FHED File header](./chunk_specifications/index.md#414-fhed-file-header)
    - [4.1.5. PHSF Password hash](./chunk_specifications/index.md#415-phsf-password-hash)
    - [4.1.6. FDAT File data](./chunk_specifications/index.md#416-fdat-file-data)
    - [4.1.7. FEND File tailer](./chunk_specifications/index.md#417-fend-file-tailer)
    - [4.1.8. SHED Solid mode header](./chunk_specifications/index.md#418-shed-solid-mode-header)
    - [4.1.9. SDAT Solid mode data](./chunk_specifications/index.md#419-sdat-solid-mode-data)
    - [4.1.10. SEND Solid mode tailer](./chunk_specifications/index.md#4110-send-solid-mode-tailer)
  - [4.2. Ancillary chunks](./chunk_specifications/index.md#42-ancillary-chunks)
    - [4.2.1 Timestamp information](./chunk_specifications/index.md#421-timestamp-information)
      - [4.2.1.1 cTIM Created timestamp](./chunk_specifications/index.md#4211-ctim-created-timestamp)
      - [4.2.1.2 mTIM Modified timestamp](./chunk_specifications/index.md#4212-mtim-modified-timestamp)
    - [4.2.2 permission information](./chunk_specifications/index.md#422-permission-information)
      - [4.2.2.1 fPRM File permission](./chunk_specifications/index.md#4221-fprm-file-permission)
  - [4.3. Summary of standard chunks](./chunk_specifications/index.md#43-summary-of-standard-chunks)
  - [4.4. Additional chunk types](./chunk_specifications/index.md#44-additional-chunk-types)
- [5. Compression algorithms](./compression_algorithms/index.md)
  - [5.1. Deflate](./compression_algorithms/index.md#51-deflate)
  - [5.2. ZStandard](./compression_algorithms/index.md#52-zstandard)
  - [5.3. LZMA](./compression_algorithms/index.md#53-lzma)
- [6. Cipher algorithms](./cipher_algorithms/index.md)
  - [6.1. Rijndael](./cipher_algorithms/index.md#61-rijndael)
  - [6.2. Camellia](./cipher_algorithms/index.md#62-camellia)
- [7. Cipher modes](./cipher_modes/index.md)
