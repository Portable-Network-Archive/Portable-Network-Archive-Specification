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
