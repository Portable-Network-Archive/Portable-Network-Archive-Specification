## 5. Compression algorithms

### 5.1. Deflate

PNA compression method (the only compression method presently defined for PNA) specifies deflate/inflate compression with a sliding window of at most 32768 bytes. Deflate compression is an LZ77 derivative used in zip, gzip, pkzip, and related programs. Extensive research has been done supporting its patent-free status. Portable C implementations are freely available.

Deflate-compressed datastreams within PNA are stored in the "zlib" format, which has the structure:

   Compression method/flags code: 1 byte
   Additional flags/check bits:   1 byte
   Compressed data blocks:        n bytes
   Check value:                   4 bytes
Further details on this format are given in the zlib specification [RFC-1950](../references/index.md#rfc-1950).

For PNA compression method deflate, the zlib compression method/flags code must specify method code 0 ("deflate" compression) and an LZ77 window size of not more than 32768 bytes. Note that the zlib compression method number is not the same as the PNA compression method number. The additional flags must not specify a preset dictionary. A PNA decoder must be able to decompress any valid zlib datastream that satisfies these additional constraints.

If the data to be compressed contains 16384 bytes or fewer, the encoder can set the window size by rounding up to a power of 2 (256 minimum). This decreases the memory required not only for encoding but also for decoding, without adversely affecting the compression ratio.

The compressed data within the zlib datastream is stored as a series of blocks, each of which can represent raw (uncompressed) data, LZ77-compressed data encoded with fixed Huffman codes, or LZ77-compressed data encoded with custom Huffman codes. A marker bit in the final block identifies it as the last block, allowing the decoder to recognize the end of the compressed datastream. Further details on the compression algorithm and the encoding are given in the deflate specification [RFC-1951](../references/index.md#rfc-1951).

The check value stored at the end of the zlib datastream is calculated on the uncompressed data represented by the datastream. Note that the algorithm used is not the same as the CRC calculation used for PNA chunk check values. The zlib check value is useful mainly as a cross-check that the deflate and inflate algorithms are implemented correctly. Verifying the chunk CRCs provides adequate confidence that the PNA file has been transmitted undamaged.

In a entry of PNA file, the concatenation of the contents of all the FDAT chunks between FHAD and FEND makes up a zlib datastream as specified above. This datastream decompresses to file data as described elsewhere in this document.

It is important to emphasize that the boundaries between FDAT chunks are arbitrary and can fall anywhere in the zlib datastream. There is not necessarily any correlation between FDAT chunk boundaries and deflate block boundaries or any other feature of the zlib data. For example, it is entirely possible for the terminating zlib check value to be split across FDAT chunks.

In the same vein, there is no required correlation between the structure of the file data (i.e., scanline boundaries) and deflate block boundaries or FDAT chunk boundaries. The complete image data is represented by a single zlib datastream that is stored in some number of FDAT chunks; a decoder that assumes any more than this is incorrect. (Of course, some encoder implementations may emit files in which some of these structures are indeed related. But decoders cannot rely on this.)

PNA also uses zlib datastreams in SDAT chunks, where the remainder of the chunk following the compression method byte is a zlib datastream as specified above.

Additional documentation and portable C code for deflate and inflate are available from the Info-ZIP archives at [ftp://ftp.info-zip.org/pub/infozip/](ftp://ftp.info-zip.org/pub/infozip/).
