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

#### 8.1.5 Recommendations

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

#### 8.2.5 Recommendations

As per current best practices, it is advised to allocate as much memory as is practical for the application and at least two iterations. The parallelism should be set according to the number of available processor cores. The salt should be a unique, cryptographically secure random value for each password.

For new PNA archives (AHED Minor version 1 with AEAD entries), encoders SHOULD use Argon2id with the [RFC 9106](../references/index.md#rfc-9106) "first recommended" profile:

| Parameter | Value | Rationale |
|---|---|---|
| Memory cost (m_cost) | 65536 KiB (64 MiB) | RFC 9106 §4 first-recommended profile |
| Time cost (t_cost) | 3 | RFC 9106 §4 first-recommended profile |
| Parallelism (p_cost) | `available_parallelism()` (typically 4) | Adapts to host CPU |
| Salt length | 16 bytes from CSPRNG | RFC 9106 §3.1 minimum |
| Output (master key) length | 32 bytes (256 bits) | Sized for AES-256 / Camellia-256 |

The 19 MiB / t=2 / p=1 "second recommended" profile (RFC 9106 §4) is acceptable only on memory-constrained devices.

### 8.3. HKDF (HMAC-based Key Derivation Function)

This section specifies the use of HKDF (HMAC-based Extract-and-Expand Key Derivation Function), as defined in [RFC 5869](../references/index.md#rfc-5869), for deriving per-entry cryptographic keys from a previously established archive master key. HKDF is REQUIRED when an entry uses an AEAD cipher mode such as GCM ([§7.4](../cipher_modes/index.md#74-galoiscounter-mode-gcm)).

> **Scope.** HKDF is NOT a password hash function and MUST NOT be used to derive keys directly from low-entropy inputs such as passwords. It is a key derivation function that takes a uniformly random input keying material (IKM) and produces an arbitrary number of independent output keys. In PNA, the IKM is the archive master key, which is itself derived from the user-supplied password by Argon2id ([§8.2](#82-argon2)) or PBKDF2 ([§8.1](#81-password-based-key-derivation-function-2pbkdf2)).

#### 8.3.1 Algorithm Specification

PNA uses HKDF instantiated with HMAC-SHA-256 (HKDF-SHA-256). The function operates in two phases:

- **Extract**: a salt and the IKM are combined via HMAC to produce a fixed-length pseudorandom key (PRK).
- **Expand**: the PRK and a context-specific `info` string are combined to produce the desired output key.

For per-entry key derivation in PNA, both phases are invoked together (the conventional `HKDF` function).

#### 8.3.2 Parameters

| Parameter | Value (PNA per-entry key derivation) |
|---|---|
| Hash function | SHA-256 |
| IKM (input keying material) | The 32-byte archive master key derived from the user password via Argon2id (§8.2) or PBKDF2 (§8.1) |
| Salt | Byte concatenation `archive_identifier \|\| entry_random` (32 bytes total): `archive_identifier` from the `AENC` chunk for the entry's cipher_mode ([§4.1.x AENC](../chunk_specifications/index.md)); `entry_random` from this entry's `FENC` chunk ([§4.1.x FENC](../chunk_specifications/index.md)) |
| Info | The byte concatenation `"PNA-AEAD-v1-" \|\| algorithm_name \|\| "-256-GCM-" \|\| entry_index_be4`, where `algorithm_name` is `"AES"` (for `Encryption method = 1`) or `"Camellia"` (for `Encryption method = 2`), and `entry_index_be4` is the 4-byte big-endian encoding of the 0-indexed position of this entry within the archive |
| Output length (L) | 32 bytes (256 bits) |

The resulting 32-byte output is the per-entry GCM key.

#### 8.3.3 Process

The full HKDF-SHA-256 procedure is normatively specified in [RFC 5869](../references/index.md#rfc-5869) §2 and is reproduced here only for convenience:

```
salt = AENC[cipher_mode_id].archive_identifier (16B) || FENC.entry_random (16B)
       (32 bytes total)
info = "PNA-AEAD-v1-" || algorithm_name || "-256-GCM-" || entry_index_be4

PRK  = HMAC-SHA-256(salt, master_key)
T(1) = HMAC-SHA-256(PRK, info || 0x01)
output = T(1)[0..32]   (since L = 32 ≤ 32, only one block of T is needed)
```

#### 8.3.4 Security Considerations

The `info` string binds the per-entry key derivation to:
- the chosen cipher algorithm (AES vs Camellia, via `algorithm_name`)
- the AEAD construction version (`"PNA-AEAD-v1-..."`)
- the entry's archive position (via `entry_index_be4`)

Any attempt to tamper with the `Encryption method` byte in FHED to switch the cipher algorithm (downgrade attack), or to reorder entries, will cause the decoder to derive a different per-entry key, leading to authentication tag verification failure (see [§7.4.2](../cipher_modes/index.md#742-associated-data-aad)).

The `salt` input combines two independent random sources:
- `archive_identifier` (from `AENC`) ensures cross-archive isolation: two archives with the same password derive different per-entry keys because their `archive_identifier` values differ.
- `entry_random` (from `FENC`) provides defense-in-depth against encoder bugs that might otherwise produce duplicate per-entry contexts (e.g., reusing an entry_index across entries).

A collision of `entry_random` between two entries within the same archive would not by itself break security if `entry_index_be4` is correctly assigned, but combined with an `entry_index` collision it would derive identical per-entry GCM keys, causing catastrophic GCM nonce reuse (see [Joux 2006](../references/index.md#joux-2006)). Encoders MUST regenerate `entry_random` from a CSPRNG for every entry.

The use of HKDF is what makes PNA's GCM construction safe under the [NIST SP 800-38D](../references/index.md#nist-sp-800-38d) §8.3 limit of 2^32 invocations per key: each entry gets its own GCM key, and the chunk counter resets per entry, so the per-key invocation count is bounded by the chunk count of a single entry (typically far below 2^32).

#### 8.3.5 Recommendations

Encoders SHOULD use a separate `entry_random` from a CSPRNG for every entry, even when the same plaintext is being archived multiple times in the same archive. There is no scenario in which two entries should share an `entry_random`.

Decoders MUST NOT cache or reuse per-entry derived keys across archives or sessions. Each archive opening session SHOULD re-derive keys from the password through Argon2id/PBKDF2, so that interrupting and resuming archive processing does not retain key material in memory longer than necessary.
