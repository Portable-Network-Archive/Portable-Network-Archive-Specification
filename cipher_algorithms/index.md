## 6. Cipher algorithms

PNA employs multiple cipher algorithms. This specification is designed to ensure the secure use of PNA by enabling the selection of alternative cipher algorithms in the event of a critical flaw discovered in a specific algorithm. In this chapter, we will define the cipher algorithms available for use in PNA. For information on cipher mode, please refer to the following chapter.

### 6.1 Rijndael

PNA cipher method 0 specifies Rijndael encryption with a key length of 256 bits and a block length of 128 bits. Rijndael, generally known as AES(Advanced Encryption Standard), is a widely accepted symmetric encryption algorithm renowned for its security and efficiency. It operates on blocks of data and supports key sizes of 128, 192, and 256 bits. In the PNA, Rijndael with a key length of 256 bits is utilized. This provides a high level of encryption strength, ensuring that the archived data remains secure against unauthorized access.

Rijndael-encrypted datastreams within PNA are stored in a format depending on the cipher mode.

For PNA cipher method 0, the Rijndael encryption method/flags code must specify method code 1 ("Rijndael" encryption). Note that the Rijndael encryption method number is not the same as the PNA cipher method number. A PNA decoder must be able to decrypt any valid Rijndael-encrypted datastream that satisfies these additional constraints.

In an entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHAD and FEND makes up a Rijndael-encrypted datastream as specified above. This datastream decrypts to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the Rijndael-encrypted datastream. There is not necessarily any correlation between FDAT chunk boundaries and block boundaries or any other feature of the Rijndael-encrypted data. 

In the same vein, there is no required correlation between the structure of the file data and deflate block boundaries or FDAT chunk boundaries. The complete entry data is represented by a single Rijndael-encrypted datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

PNA also uses Rijndael-encrypted datastreams in SDAT chunks, where the remainder of the chunk following the compression method byte is a Rijndael-encrypted datastream as specified above.

Additional documentation is available from the Cryptographic Standards and Guidelines archives at [http://www.nist.gov/aes](http://www.nist.gov/aes) and the FIPS Publication 197 [FIPS-197](https://csrc.nist.gov/publications/detail/fips/197/final).

### 6.2 Camellia

PNA cipher method 1 specifies Camellia encryption with a key length of 256 bits and a block length of 128 bits. Camellia is a symmetric encryption algorithm developed jointly by Nippon Telegraph and Telephone Corporation (NTT) and Mitsubishi Electric Corporation. It is designed to offer high security and performance comparable to AES. Camellia supports key sizes of 128, 192, and 256 bits. In the PNA, Camellia with a key length of 256 bits is employed, enhancing the overall security of the archived data.

Camellia-encrypted datastream within PNA is stored in a format depending on the cipher use mode.

For PNA cipher method 1, the Camellia encryption method/flags code must specify method code 2 ("Camellia" encryption). Note that the Camellia encryption method number is not the same as the PNA cipher method number. A PNA decoder must be able to decrypt any valid Camellia-encrypted datastream that satisfies these additional constraints.

In an entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHAD and FEND makes up a Camellia-encrypted datastream as specified above. This datastream decrypts to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the Camellia-encrypted datastream. There is not necessarily any correlation between FDAT chunk boundaries and block boundaries or any other feature of the Camellia-encrypted data. 

In the same vein, there is no required correlation between the structure of the file data and deflate block boundaries or FDAT chunk boundaries. The complete entry data is represented by a single Camellia-encrypted datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

PNA also uses Camellia-encrypted datastreams in SDAT chunks, where the remainder of the chunk following the compression method byte is a Camellia-encrypted datastream as specified above.

Additional documentation is available from the Camellia Specification at [RFC-3713](https://datatracker.ietf.org/doc/html/rfc3713).
