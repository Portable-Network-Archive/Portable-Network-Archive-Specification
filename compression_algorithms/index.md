## 5. Compression algorithms

PNA allows the use of multiple compression algorithms. This is designed to achieve high performance by switching compression algorithms depending on the intended use and environment.

### 5.1. Deflate

PNA compression method 0 specifies deflate/inflate compression with a sliding window of at most 32768 bytes. Deflate compression is an LZ77 derivative used in zip, gzip, pkzip, and related programs. Extensive research has been done supporting its patent-free status. Portable C implementations are freely available.

Deflate-compressed datastreams within PNA are stored in the "zlib" format, which has the structure:

   Compression method/flags code: 1 byte
   Additional flags/check bits:   1 byte
   Compressed data blocks:        n bytes
   Check value:                   4 bytes
Further details on this format are given in the zlib specification [RFC-1950](../references/index.md#rfc-1950).

For PNA compression method 0, the zlib compression method/flags code must specify method code 1 ("deflate" compression) and an LZ77 window size of not more than 32768 bytes. Note that the zlib compression method number is not the same as PNA compression method number. The additional flags must not specify a preset dictionary. A PNA decoder must be able to decompress any valid zlib datastream that satisfies these additional constraints.

If the data to be compressed contains 16384 bytes or fewer, the encoder can set the window size by rounding up to a power of 2 (256 minimum). This decreases the memory required not only for encoding but also for decoding, without adversely affecting the compression ratio.

The compressed data within the zlib datastream is stored as a series of blocks, each of which can represent raw (uncompressed) data, LZ77-compressed data encoded with fixed Huffman codes, or LZ77-compressed data encoded with custom Huffman codes. A marker bit in the final block identifies it as the last block, allowing the decoder to recognize the end of the compressed datastream. Further details on the compression algorithm and the encoding are given in the deflate specification [RFC-1951](../references/index.md#rfc-1951).

The check value stored at the end of the zlib datastream is calculated on the uncompressed data represented by the datastream. Note that the algorithm used is not the same as the CRC calculation used for PNA chunk check values. The zlib check value is useful mainly as a cross-check that the deflate and inflate algorithms are implemented correctly. Verifying the chunk CRCs provides adequate confidence that PNA file has been transmitted undamaged.

In an entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHED and FEND makes up a zlib datastream as specified above. This datastream decompresses to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the zlib datastream. There is not necessarily any correlation between FDAT chunk boundaries and deflate block boundaries or any other feature of the zlib data. For example, it is entirely possible for the terminating zlib check value to be split across FDAT chunks.

In the same vein, there is no required correlation between the structure of the file data and deflate block boundaries or FDAT chunk boundaries. The complete entry data is represented by a single zlib datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

Treat all SDAT chunks between SHAD and SEND in the same way.

Additional documentation and portable C code for deflate and inflate are available from the Info-ZIP archives at [ftp://ftp.info-zip.org/pub/infozip/](ftp://ftp.info-zip.org/pub/infozip/).

### 5.2. ZStandard

PNA compression method 1 specifies ZStandard compression with a [RFC-8878](../references/index.md#rfc-8878). ZStandard compression is an LZ77 derivative used in linux kernel, btrfs, squashfs, and related programs. Reference implementations are BSD license and freely available.

For PNA compression method 1, the ZStandard compression method/flags code must specify method code 2 ("ZStandard" compression). Note that the ZStandard compression method number is not the same as PNA compression method number. The additional flags must not specify a preset dictionary. A PNA decoder must be able to decompress any valid ZStandard datastream that satisfies these additional constraints.

In an entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHED and FEND makes up a ZStandard datastream as specified above. This datastream decompresses to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the ZStandard datastream. There is not necessarily any correlation between FDAT chunk boundaries or any other feature of the ZStandard data. For example, it is entirely possible for the terminating ZStandard check value to be split across FDAT chunks.

In the same vein, there is no required correlation between the structure of the file data or FDAT chunk boundaries. The complete entry data is represented by a single ZStandard datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

Treat all SDAT chunks between SHAD and SEND in the same way.

Additional documentation and Reference implementations of ZStandard are available from GitHub at [https://facebook.github.io/zstd/](https://facebook.github.io/zstd/) and [https://github.com/facebook/zstd](https://github.com/facebook/zstd)

### 5.3. LZMA

PNA compression method 2 specifies LZMA compression with a [](). LZMA compression is an LZ77 derivative used in linux kernel, 7-zip, xz-utils, and related programs. Reference implementations are BSD license and freely available.

LZMA-compressed datastreams within PNA are stored in the "xz" format.
Further details on this format are given in the xz specification [https://tukaani.org/xz/xz-file-format.txt](https://tukaani.org/xz/xz-file-format.txt).

For PNA compression method 2, the LZMA compression method/flags code must specify method code 4 ("LZMA" compression). Note that the LZMA compression method number is not the same as PNA compression method number. The additional flags must not specify a preset dictionary. A PNA decoder must be able to decompress any valid LZMA datastream that satisfies these additional constraints.

In an entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHED and FEND makes up a LZMA datastream as specified above. This datastream decompresses to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the LZMA datastream. There is not necessarily any correlation between FDAT chunk boundaries or any other feature of the LZMA data. For example, it is entirely possible for the terminating LZMA check value to be split across FDAT chunks.

In the same vein, there is no required correlation between the structure of the file data or FDAT chunk boundaries. The complete entry data is represented by a single LZMA datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

Treat all SDAT chunks between SHAD and SEND in the same way.

Additional documentation and Reference implementations of LZMA are available from GitHub at [https://github.com/tukaani-project/xz](https://github.com/tukaani-project/xz).
