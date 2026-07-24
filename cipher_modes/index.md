## 7. Cipher modes

PNA employs multiple cipher modes of operation. This specification is designed to ensure the secure use of PNA by enabling the selection of alternative cipher modes of operation in the event of a critical flaw discovered in a specific cipher mode of operation.

### 7.1. Cipher Block Chaining Mode (CBC)
PNA cipher mode of operation method 0 specifies CBC with a block length of 128 bits.

CBC mode is a block cipher mode of operation that provides confidentiality through chaining encrypted blocks. In CBC mode, each plaintext block is XORed with the previous ciphertext block before encryption. It introduces a dependency between blocks.

This document specifies the use of the Rijndael or Camellia cipher in CBC mode within PNA.

This mode requires an Initialization Vector (IV) that is the same size as the block size. Use of a randomly generated IV prevents generation of identical ciphertext from packets which have identical data that spans the first block of the cipher algorithm's block size.

The IV is XOR'd with the first plaintext block before it is encrypted. Then for successive blocks, the previous ciphertext block is XOR'd with the current plaintext, before it is encrypted.

More information on CBC mode can be obtained in [MODES](../references/index.md#modes), [CRYPTO-S](../references/index.md#crypto-s)

### 7.2. Counter Mode (CTR)
PNA cipher mode of operation method 1 specifies CTR with a block length of 128 bits.

CTR mode is a stream cipher mode of operation, where each plaintext block is encrypted by XORing it with the corresponding block of a keystream. CTR offers excellent parallelizability and allows random access to individual blocks, making it suitable for scenarios where seeking and parallel processing are required.

This document specifies the use of the Rijndael or Camellia cipher in CTR mode within PNA.

CTR requires the encryptor to generate a unique per-packet value and communicate this value to the decryptor. This specification calls this per-packet value an initialization vector (IV). The same IV and key combination MUST NOT be used more than once. The encryptor can generate the IV in any manner that ensures uniqueness. Common approaches to IV generation include incrementing a counter for each packet.

More information on CTR mode can be obtained in [MODES](../references/index.md#modes)


### 7.3. Initialization Vector (IV) Placement in Encrypted Streams

This section applies to CBC and CTR modes. AEAD modes use the layout defined in [§7.5](#75-aead-datastream-layout).

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

GCM is an AEAD mode of operation built on a 128-bit block cipher. It combines counter-mode confidentiality (CTR) with a message authentication code based on Galois field arithmetic (GHASH), producing one fixed-length authentication tag per invocation.

This document specifies the use of the Rijndael or Camellia cipher in GCM mode within PNA.

GCM is defined by [NIST SP 800-38D](../references/index.md#nist-sp-800-38d). This section describes only the parameters PNA uses when invoking GCM. The encrypted datastream layout is defined in [§7.5](#75-aead-datastream-layout), and key derivation is defined in [§8.3](../key_derivation_algorithms/index.md#83-hkdf-aead-stream-key-derivation).

#### 7.4.1 Parameters

| Parameter | Value | Note |
|---|---|---|
| Block cipher | Rijndael (AES-256) or Camellia-256 | Per FHED or SHED `Encryption method` field |
| Key | 256-bit `K_stream` | Derived per datastream ([§8.3](../key_derivation_algorithms/index.md#83-hkdf-aead-stream-key-derivation)) |
| Nonce size | 96 bits (12 bytes) | Derived deterministically ([§7.5.3](#753-nonce-derivation)) |
| Authentication tag size | 128 bits (16 bytes) | |
| AAD | Empty (zero-length) | Binding to the entry is provided by key derivation ([§8.3.1](../key_derivation_algorithms/index.md#831-entry_context)) |
| Maximum plaintext per invocation | 64 MiB (67,108,864 bytes) | Upper bound of `segment_size` ([§7.5.1](#751-stream-header)) |

The combination of Camellia with GCM is specified in [RFC 6367](../references/index.md#rfc-6367).

### 7.5. AEAD Datastream Layout

When an AEAD mode (GCM, [§7.4](#74-galoiscounter-mode-gcm)) is used, the encrypted datastream (in per-entry mode, the concatenation of all FDAT chunk data fields of the entry in archive order; in solid mode, the concatenation of all SDAT chunk data fields from SHED to SEND in archive order) has the following layout:

```
[stream header (43 bytes)][segment 0][segment 1]...[segment N-1]
```

Segments may span FDAT/SDAT chunk data field boundaries.

#### 7.5.1 Stream Header

The stream header occupies the first 43 bytes of the datastream. It is not encrypted.

| Bytes | Field | Description |
|---|---|---|
| 32 | stream_salt | Generated with a CSPRNG. The HKDF salt of [§8.3](../key_derivation_algorithms/index.md#83-hkdf-aead-stream-key-derivation) |
| 7 | nonce_prefix | Generated with a CSPRNG. The fixed part of every segment nonce ([§7.5.3](#753-nonce-derivation)) |
| 4 | segment_size | Unsigned 32-bit big-endian. Plaintext bytes per segment |

`segment_size` must be at least 1 and at most 67,108,864 (64 MiB). Values outside this range are malformed.

#### 7.5.2 Segments

Each segment is one GCM invocation and consists of `ciphertext || 16-byte authentication tag`.

- Every segment except the last carries exactly `segment_size` bytes of plaintext.
- The last segment (final segment) carries between 0 and `segment_size` bytes of plaintext.
- Canonical form: when the plaintext length is a non-zero multiple of `segment_size`, an empty trailing segment must not be emitted. An empty plaintext datastream is represented as a single empty final segment (16-byte tag only).
- The minimum datastream length is 59 bytes (43-byte stream header plus a 16-byte empty final segment).
- Segment i (zero-based) starts at datastream offset `43 + i × (segment_size + 16)`.

#### 7.5.3 Nonce Derivation

Nonces are not stored in the archive. A decoder recomputes the nonce from the segment's position. The nonce (12 bytes) of segment i is the following byte concatenation:

```
nonce = nonce_prefix (7 bytes) || segment counter (4 bytes, unsigned big-endian, = i) || final flag (1 byte)
```

The final flag is `0x01` for the final segment and `0x00` otherwise. The segment counter must not wrap; therefore one datastream holds at most 2^32 segments.

This follows the STREAM construction ([HRRV15](../references/index.md#hrrv15)). Reordering and duplication of segments are detected through counter mismatches, truncation through the missing final flag, and extension beyond the final segment through the detection of trailing bytes. Splicing between streams is detected because each datastream has a distinct `K_stream` ([§8.3](../key_derivation_algorithms/index.md#83-hkdf-aead-stream-key-derivation)).
