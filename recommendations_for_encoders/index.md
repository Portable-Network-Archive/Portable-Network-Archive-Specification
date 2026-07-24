## 11. Recommendations for Encoders

This chapter gives some recommendations for encoder behavior. The only absolute requirement on a PNA encoder is that it produce files that conform to the format specified in the preceding chapters. However, best results will usually be achieved by following these recommendations.

### 11.1. Use of private chunks

Applications can use PNA private chunks to carry information that need not be understood by other applications. Such chunks must be given names with lowercase second letters, to ensure that they can never conflict with any future public chunk definition. Note, however, that there is no guarantee that some other application will not use the same private chunk name. If you use a private chunk type, it is prudent to store additional identifying information at the beginning of the chunk data.

Use an ancillary chunk type (lowercase first letter), not a critical chunk type, for all private chunks that store information that is not absolutely essential to read the archive. Creation of private critical chunks is discouraged because they render PNA files unportable. Such chunks should not be used in publicly available software or files. If private critical chunks are essential for your application, it is recommended that one appear near the start of the file, so that a standard decoder need not read very far before discovering that it cannot handle the file.

If you want others outside your organization to understand a chunk type that you invent, contact the maintainers of the PNA specification to submit a proposed chunk name and definition for addition to the list of special-purpose public chunks (see [Additional chunk types](../chunk_specifications/index.md#44-additional-chunk-types)). Note that a proposed public chunk name (with uppercase second letter) must not be used in publicly available software or files until registration has been approved.

If an ancillary chunk contains textual information that might be of interest to a human user, you should **not** create a special chunk type for it. Instead use a text chunk and define a suitable keyword. That way, the information will be available to users not using your software.

Keywords in text chunks should be reasonably self-explanatory, since the idea is to let other users figure out what the chunk contains. If of general usefulness, new keywords can be registered with the maintainers of the PNA specification. But it is permissible to use keywords without registering them first.

### 11.2. Private type and method codes

This specification defines the meaning of only some of the possible values of some fields. For example, only compression method 0 through 4, encryption method 0 through 2 and cipher mode 0 through 2 are defined. Numbers greater than 63 must be used when inventing experimental or private definitions of values for any of these fields. Numbers below 64 are reserved for possible future public extensions of this specification. Note that use of private type codes may render a file unreadable by standard decoders. Such codes are strongly discouraged except for experimental purposes, and should not appear in publicly available software or files.

### 11.3. Owner information chunks

An encoder that supports the owner information chunks (§4.2.6) should write those chunks and should not write the deprecated `fPRM` chunk. An encoder should write only the facets it has captured; a facet that was not captured is represented by the absence of its chunk, never by a placeholder value. An encoder should not write `fMOd` for an entry whose POSIX permission mode cannot be faithfully determined.

### 11.4. Encryption modes

New encrypted archives should use GCM (`Cipher mode = 2`) rather than CBC or CTR. CBC and CTR provide confidentiality without authenticity; they are primarily useful for compatibility with older archives.

For CBC and CTR, an encoder must follow the IV requirements of [§7.3.3 Encoder Behavior](../cipher_modes/index.md#733-encoder-behavior).

For GCM, an encoder:

1. must generate a fresh KDF salt for each archive. With a reused salt, an encrypted datastream can be transplanted from another archive that shares the same password when the header chunk data and PHSF data are byte-for-byte identical; a fresh salt confines this substitution to within a single archive.
2. should use Argon2id for new password-protected archives ([§8.2 Argon2](../key_derivation_algorithms/index.md#82-argon2)).
3. should emit byte-identical PHSF chunks for all AEAD encrypted datastreams in a single archive that share the same password. This allows `K_master` to be computed only once per archive.
4. must generate a fresh 32-byte `stream_salt` and 7-byte `nonce_prefix` with a CSPRNG for each encrypted datastream, choose a `segment_size` (recommended: 1 MiB), and write the 43-byte stream header ([§7.5.1](../cipher_modes/index.md#751-stream-header)) at the start of the datastream.
5. must finalize the FHED (or SHED) and PHSF chunk data before encryption begins, construct `entry_context` ([§8.3.1](../key_derivation_algorithms/index.md#831-entry_context)), and derive `K_stream`. These are known before any ciphertext is written, so this does not conflict with one-pass streaming.
6. must split the plaintext datastream into segments as defined in [§7.5.2](../cipher_modes/index.md#752-segments), buffering one segment ahead so that the final flag is set only on the final segment.
7. may split the resulting encrypted datastream into FDAT (or SDAT) chunks at arbitrary byte boundaries.
