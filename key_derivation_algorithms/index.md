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
