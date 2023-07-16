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
