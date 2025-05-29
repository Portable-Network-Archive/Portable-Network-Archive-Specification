## 13. Glossary

**AES (Rijndael)**  
A symmetric-key block cipher algorithm standardized as Advanced Encryption Standard (AES). Rijndael was selected as the AES algorithm and supports key sizes of 128, 192, and 256 bits. In PNA, AES with a 256-bit key and 128-bit block size is used for encryption.

**Ancillary chunk**  
A chunk that provides additional information. A decoder can still produce a meaningful archive, though not necessarily the best possible archive, without processing the chunk.

**Archive number**  
A 4-byte field in the AHED chunk indicating the sequence number of a split archive part. The first part is numbered 0, and subsequent parts increment by 1.

**Argon2**  
A modern memory-hard key derivation function and password hashing algorithm, designed to resist GPU and side-channel attacks. Argon2 is the winner of the Password Hashing Competition (PHC) and is recommended for secure password storage and key derivation.

**Byte**  
Eight bits; also called an octet.

**Camellia**  
A symmetric-key block cipher algorithm developed by NTT and Mitsubishi Electric. Camellia is designed to offer high security and performance comparable to AES, and supports key sizes of 128, 192, and 256 bits. In PNA, Camellia with a 256-bit key and 128-bit block size is used for encryption.

**Cipher mode**  
A method of using a block cipher to encrypt data, defining how blocks of plaintext are transformed into ciphertext. Common modes include CBC (Cipher Block Chaining) and CTR (Counter mode), each providing different security properties and operational characteristics.

**Chunk**  
A section of a PNA file. Each chunk has a type indicated by its chunk type name. Most types of chunks also include some data. The format and meaning of the data within the chunk are determined by the type name.

**CRC (Cyclic Redundancy Check)**  
A CRC is a type of check value designed to catch most transmission errors. A decoder calculates the CRC for the received data and compares it to the CRC that the encoder calculated, which is appended to the data. A mismatch indicates that the data was corrupted in transit.

**Critical chunk**  
A chunk that must be understood and processed by the decoder in order to produce a meaningful archive from a PNA file.

**Datastream**  
A sequence of bytes. This term is used rather than "file" to describe a byte sequence that is only a portion of a file. We also use it to emphasize that a PNA archive might be generated and consumed "on-the-fly", never appearing in a stored file at all.

**Deflate**  
The name of the compression algorithm used in standard PNG files, as well as in zip, gzip, pkzip, and other compression programs. Deflate is a member of the LZ77 family of compression methods.

**Entry**  
A logical unit within a PNA archive, representing a file, directory, symbolic link, hard link, or a reference to a previously stored file. Each entry is described by a header, optional ancillary information, data, and a tailer.

**File signature**  
A fixed 8-byte sequence at the beginning of every PNA file, used to identify the file as a PNA archive and detect file corruption or misidentification.

**Initialization Vector (IV)**  
A random or unique value used as the initial input for certain encryption modes (such as CBC and CTR) to ensure that identical plaintexts encrypt to different ciphertexts. The IV must never be reused with the same key.

**Key derivation function (KDF)**  
A cryptographic algorithm that derives one or more secret keys from a password or passphrase, often using a salt and multiple iterations to increase security against brute-force attacks.

**Lossless compression**  
Any method of data compression that guarantees the original data can be reconstructed exactly, bit-for-bit.

**Lossy compression**  
Any method of data compression that reconstructs the original data approximately, rather than exactly.

**LSB (Least Significant Byte)**  
Least Significant Byte of a multi-byte value.

**LZMA (Lempel-Ziv-Markov chain algorithm)**  
A lossless data compression algorithm that uses a dictionary compression scheme and range encoding. LZMA is known for its high compression ratio and is used in formats such as xz and 7z.

**MSB (Most Significant Byte)**  
Most Significant Byte of a multi-byte value.

**Password hash**  
A cryptographic hash of a password, typically generated using a key derivation function (such as PBKDF2 or Argon2) and a salt, to securely store or verify passwords without saving the original password.

**PBKDF2 (Password-Based Key Derivation Function 2)**  
A widely used key derivation function that applies a pseudorandom function (such as HMAC) to the input password along with a salt and repeats the process many times to produce a derived key. PBKDF2 is specified in RFC 2898.

**PHC string format**  
A standardized string format for encoding password hash parameters and results, as defined by the Password Hashing Competition (PHC). It encodes the algorithm, parameters, salt, and hash in a single string.

**PNA editor**  
A program that modifies a PNA file and preserves ancillary information, including chunks that it does not recognize. Such a program must obey the rules given in Chunk Ordering Rules.

**Salt**  
A random value added to a password before hashing, to ensure that identical passwords result in different hashes and to protect against precomputed attacks such as rainbow tables.

**Solid mode**  
A storage mode in which multiple entries' data are concatenated and compressed/encrypted as a single continuous stream, improving compression ratio for similar files. Solid mode entries use dedicated chunk types (SHED, SDAT, SEND).

**Split archive**  
A PNA archive that is divided into multiple files for easier storage or transfer. Each part contains a portion of the archive, and the order is managed by the Archive number and file naming conventions (e.g., .part1.pna, .part2.pna).

**UTF-8**  
A variable-width character encoding used for electronic communication, capable of encoding all possible Unicode code points. UTF-8 is the required encoding for all textual data in PNA files, including file paths.

**xz**  
A file format and software for lossless data compression, utilizing the LZMA2 algorithm. The xz format is commonly used for compressing software packages and archives in Unix-like systems.

**Zstandard (Zstd)**  
A fast lossless compression algorithm developed by Facebook, providing high compression ratios and very fast decompression. Zstandard is widely used in modern systems and supports adjustable compression levels.

**zlib**  
A particular format for data that has been compressed using deflate-style compression. Also the name of a library implementing this method. PNA implementations need not use the zlib library, but they must conform to its format for compressed data.
