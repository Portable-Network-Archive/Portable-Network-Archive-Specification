## 8. Key derivation algorithms

### 8.1. Password-Based Key Derivation Function 2(PBKDF2)

This section specifies the method for deriving a cryptographic key from a password using the Password-Based Key Derivation Function 2 (PBKDF2). The PBKDF2 algorithm is designed to produce keys that are computationally intensive to derive, thereby providing a defense against attacks such as dictionary attacks and brute force.

#### 8.1.1 Algorithm Specification

PBKDF2 applies a pseudorandom function (PRF) to the input password along with a salt value and iterates this process a specified number of times to produce a derived key. The iteration count is a critical security parameter and should be chosen with consideration to the desired level of security and the performance constraints of the system.

#### 8.1.2 Parameters

- Password: The secret input value.
- Salt: A sequence of bits, known as a cryptographic salt.
- Iteration Count: A positive integer specifying the number of iterations.
- Derived Key Length: The desired length of the derived key.
- Pseudorandom Function: Typically, a HMAC (Hash-Based Message Authentication Code) is used as the PRF, with a secure hash function such as SHA-256.

#### 8.1.3 Process

1. Initialize a counter to one.
2. Concatenate the password and salt.
3. Apply the PRF to the combined password and salt.
4. Repeat the PRF process for the number of iterations specified.
5. Output the final block of data as the derived key.

Further details on the key derivation algorithm are given in the PBKDF2 specification [RFC-2898](../references/index.md#rfc-2898) and PBKDF2 Test Vectors [RFC-6070](../references/index.md#rfc-6070)

#### 8.1.4 Security Considerations

The security of PBKDF2 is directly related to the number of iterations, the strength of the PRF, and the length and randomness of the salt. It is recommended to use a salt that is unique to each derivation process to prevent the use of precomputed tables for deriving keys.

#### 8.1.5 AEAD Key Length

When PBKDF2 is used as the PHSF algorithm for an AEAD encrypted datastream (FHED or SHED `Cipher mode = 2`), the derived key length is 32 bytes (256 bits), and the output is used as `K_master` in [§8.3](#83-hkdf-aead-stream-key-derivation).

#### 8.1.6 Recommendations

As per [NIST SP 800-132](../references/index.md#nist-sp-800-132), it is recommended to use at least 10,000 iterations for PBKDF2 when deriving keys for non-interactive applications. However, this value should be increased as computational power advances to ensure the security of the derived keys.

### 8.2. Argon2

This section delineates the methodology for deriving cryptographic keys via the memory-hard key derivation function, Argon2. Recognized as the winner of the Password Hashing Competition in 2015, Argon2 is engineered to resist attacks from both specialized hardware and parallel computing, making it a robust choice for password hashing and key derivation.

#### 8.2.1 Algorithm Specification

Argon2 is a high-level key derivation function that operates with three distinct variants: Argon2d, Argon2i, and Argon2id, each tailored for different security applications. The function utilizes a large memory size, parallelism, and a variable number of iterations to thwart off-line brute-force attacks.

#### 8.2.2 Parameters

Memory Size: The amount of memory used by the algorithm.
Iterations: The number of iterations the function is to perform.
Parallelism: The number of threads and lanes that the algorithm utilizes.
Salt: A unique sequence of bytes used as an input to the hash function.
Tag Length: The desired length of the output key.
Secret Value: An optional secret value that can be used as a key for HMAC when generating the hash.

#### 8.2.3 Process

Assign the memory to a matrix of blocks.
Fill the matrix with hashes derived from the password, salt, and optional secret.
Perform the specified number of iterations, mixing the blocks both within and between threads.
Extract the tag of the requested length as the output of the function.

Further details on the key derivation algorithm are given in the [argon2 specification](../references/index.md#argon2) and [RFC-9106](../references/index.md#rfc-9106)

#### 8.2.4 Security Considerations

The selection between Argon2d, Argon2i, and Argon2id should be made according to the threat model:

Argon2d maximizes resistance to GPU cracking attacks and is suitable for cryptocurrencies and applications without a threat from side-channel attacks.
Argon2i is optimized to resist side-channel attacks and is preferable for password hashing and key derivation where the input is not secret.
Argon2id is a hybrid that combines the resistance to side-channel attacks of Argon2i with the GPU cracking resistance of Argon2d, suitable for applications that require a balance of both.

#### 8.2.5 AEAD Key Length

When Argon2 is used as the PHSF algorithm for an AEAD encrypted datastream (FHED or SHED `Cipher mode = 2`), the derived key length is 32 bytes (256 bits), and the output is used as `K_master` in [§8.3](#83-hkdf-aead-stream-key-derivation).

#### 8.2.6 Recommendations

As per current best practices, it is advised to allocate as much memory as is practical for the application and at least two iterations. The parallelism should be set according to the number of available processor cores. The salt should be a unique, cryptographically secure random value for each password.

### 8.3. HKDF (AEAD stream key derivation)

Keys for AEAD encrypted datastreams (`Cipher mode = 2`) are derived in two stages.

1. The 32-byte output of the PHSF KDF ([§8.1 PBKDF2](#81-password-based-key-derivation-function-2pbkdf2) or [§8.2 Argon2](#82-argon2)) is **K_master**. K_master must not be used directly for encryption.
2. For each encrypted datastream, **K_stream** is derived with HKDF-SHA-256 ([RFC 5869](../references/index.md#rfc-5869)):

```
K_stream = HKDF-SHA-256(IKM = K_master, salt = stream_salt, info = entry_context)
output length = 32 bytes
```

`stream_salt` is the 32-byte value stored in the stream header ([§7.5.1](../cipher_modes/index.md#751-stream-header)). Because a random salt separates the key of each stream, GCM nonce management is confined to a single stream and per-key usage limits are not a concern.

CBC and CTR (`Cipher mode = 0, 1`) use the KDF output directly as the encryption key.

#### 8.3.1 entry_context

`entry_context` is the byte concatenation of the following fields, 77 bytes in total:

| Bytes | Field | Value |
|---|---|---|
| 13 | Domain tag | ASCII `"PNA-STREAM-v1"` |
| 32 | Header hash | SHA-256(FHED chunk type \|\| data) or SHA-256(SHED chunk type \|\| data) |
| 32 | KDF params hash | SHA-256(PHSF chunk data) |

The input to the Header hash is the byte concatenation of the target header chunk's 4-byte Type field and its raw Data field, excluding Length and CRC. In per-entry mode this is the ASCII bytes `FHED` followed by the FHED data of the entry; in solid mode it is the ASCII bytes `SHED` followed by the SHED data of the solid stream.

The input to the KDF params hash is the raw Data field of the PHSF chunk, excluding Length, Type, and CRC. The PHSF is the one placed before the FDAT or SDAT chunks of the datastream.

This construction binds the header chunk's contents and the KDF parameters into the key. Tampering with either, or swapping encrypted datastreams between entries, results in a key mismatch and fails tag verification on the first segment.

When an archive contains multiple entries whose FHED chunk data is byte-for-byte identical, swapping their encrypted datastreams cannot be detected. Likewise, an entry copied in its entirety into another archive protected by the same password decrypts successfully: the key derivation binds an encrypted datastream to its entry, not to the containing archive.
