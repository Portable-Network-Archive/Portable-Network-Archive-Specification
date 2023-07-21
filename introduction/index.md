## 1. Introduction

The Portable Network Archive (PNA) format provides a portable, compressible, encryptable, splittable, and well-specified standard for archive files.

TAR features retained in PNA include:

Streamability: files can be read and written serially, thus allowing the file format to be used as a communications protocol for on-the-fly generation.
Ancillary information: textual comments and other data can be stored within the archive file.
Complete hardware and platform independence.

ZIP features retained in PNA include:

Per-file basis: an archive can contain multiple files and each file is stored separately. As a result, each file in a PNA archive can be extracted individually.

Important new features of PNA, not available in TAR or ZIP, include:

Support for both per-file compression and archive-wide compression.
In addition, both can be mixed within a single archive, allowing files to be added to the archive efficiently without having to decompress the entire archive.

PNA is designed to be:

Simple and portable: Developers should be able to implement PNA easily.
Compressible: Provides flexible compression by supporting multiple compression algorithms. Of course, it also supports uncompressed.
Encryptable: Supports 256-bit AES and 256-bit Camellia with the same encryption strength as 256-bit AES.
Splittable: By using the same data unit as the PNG file, the archive can be easily split.
Interchangeable: any standard-conforming PNA decoder must read all conforming PNA files.
Flexible: The format allows for future extensions and private add-ons, without compromising the interchangeability of basic PNA.
Robust: the design supports full file integrity checking as well as simple, quick detection of common transmission errors.

The main part of this specification gives the definition of the file format and recommendations for encoder and decoder behavior. An appendix gives the rationale for many design decisions. Although the rationale is not part of the formal specification, reading it can help implementors understand the design. Cross-references in the main text point to relevant parts of the rationale.

The words "must", "required", "should", "recommended", "may", and "optional" in this document are to be interpreted as described in [RFC-2119](../references/index.md#rfc-2119), which is consistent with their plain English meanings. The word "can" carries the same force as "may".
