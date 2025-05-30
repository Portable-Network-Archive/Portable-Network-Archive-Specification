## 7. Cipher modes

PNA employs multiple cipher modes of operation. This specification is designed to ensure the secure use of PNA by enabling the selection of alternative cipher modes of operation in the event of a critical flaw discovered in a specific cipher mode of operation.

### 7.1. Cipher Block Chaining Mode (CBC)
PNA cipher mode of operation method 0 specifies CBC with a block length of 128 bits.

CBC mode is a block cipher mode of operation that provides confidentiality and integrity through chaining encrypted blocks. In CBC mode, each plaintext block is XORed with the previous ciphertext block before encryption. It introduces a dependency between blocks, making it suitable for scenarios where each block's integrity relies on the preceding block.

This document specifies the use of the Rijndael or Camellia cipher in CBC mode within PNA.

This mode requires an Initialization Vector (IV) that is the same size as the block size. Use of a randomly generated IV prevents generation of identical ciphertext from packets which have identical data that spans the first block of the cipher algorithm's block size.

The IV is XOR'd with the first plaintext block before it is encrypted. Then for successive blocks, the previous ciphertext block is XOR'd with the current plaintext, before it is encrypted.

More information on CBC mode can be obtained in [MODES](../references/index.md#modes), [CRYPTO-S](../references/index.md#crypto-s)

### 7.2. Counter Mode (CTR)
PNA cipher mode of operation method 1 specifies CTR with a block length of 128 bits.

CTR mode is a stream cipher mode of operation, where each plaintext block is encrypted by XORing it with the corresponding block of a keystream. CTR offers excellent parallelizability and allows random access to individual blocks, making it suitable for scenarios where seeking and parallel processing are required.

This document specifies the use of the Rijndael or Camellia cipher in CTR mode within PNA.

CTR requires the encryptor to generate a unique per-packet value and communicate this value to the decryptor. This specification calls this per-packet value an initialization vector (IV).  The same IV and key combination MUST NOT be used more than once. The encryptor can generate the IV in any manner that ensures uniqueness. Common approaches to IV generation include incrementing a counter for each packet.

More information on CTR mode can be obtained in [MODES](../references/index.md#modes)


### 7.3. Initialization Vector (IV) Placement in Encrypted Streams

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
