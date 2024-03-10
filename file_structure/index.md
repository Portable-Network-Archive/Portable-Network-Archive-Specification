## 3. File Structure

A PNA file consists of a PNA signature followed by a series of chunks.
This chapter defines the signature and the basic properties of chunks. Individual chunk types are discussed in the next chapter.

### 3.1. PNA file signature
The first eight bytes of a PNA file always contain the following values:

|  hex  |    ASCII    |
|:-----:|:-----------:|
| 0x89  |    ¥x89     |
| 0x50  |      P      |
| 0x4E  |      N      |
| 0x41  |      A      |
| 0x0D  | CR(Ctrl-M)  |
| 0x0A  | LF(Ctrl-J)  |
| 0x1A  |   Ctrl-Z    |
| 0x0A  | LF(Ctrl-J)  |

This signature indicates that the remainder of the file contains a single PNA archive, consisting of a series of chunks beginning with an AHED chunk and ending with an AEND chunk.

### 3.2. Entry layout

Each entry consists of three parts:

**Header:**

The basic information of the entry, the type of entry, the compression algorithm, the encryption algorithm and the encryption use mode are recorded.
Decoders are required to select the appropriate algorithm based on this information and extract the file.

**Ancillary:**

Ancillary information such as the creation and update datetime, ownership, permissions, and others.
These are not required for entry.

**Data:**

Actual data of the entry.
The data is allowed to be split into multiple chunks.
By splitting the data into smaller chunks can help to detect file corruption more quickly.
Depending on the type of entry, this data is also allowed to be empty.
About the types of entry are in the next chapter.

**Tailer:**

End of entry marker.
The point where this chunk appears is the end point of a single entry.
It is permitted to start the next entry following this entry or to continue with a chunk indicating the end of the archive.

### 3.3 Entry kind

The following types of entry are currently supported.

**Regular file**

**Directory**

**Symbolic link**

**Hard link**

### 3.4 Special entry

A special entry is supported, a solid-mode-only entry that holds data that concatenated the above entries.
This entry is similar to a normal entries, but the data part records the data of concatenated the normal entry instead of the actual data of the entry.
The chunks that make up this special entry have chunks dedicated to solid mode.

### 3.5 Chunk layout

Each chunk consists of four parts:

**Length**

A 4-byte unsigned integer giving the number of bytes in the chunk's data field. The length counts only the data field, not itself, the chunk type code, or the CRC. Zero is a valid length.

**Chunk Type**

A 4-byte chunk type code. For convenience in description and in examining PNA files, type codes are restricted to consist of uppercase and lowercase ASCII letters (A-Z and a-z, or 65-90 and 97-122 decimal). However, encoders and decoders must treat the codes as fixed binary values, not character strings. Additional naming conventions for chunk types are discussed in the next section.

**Chunk Data**

The data bytes appropriate to the chunk type, if any. This field can be of zero length.

**CRC**

A 4-byte CRC (Cyclic Redundancy Check) calculated on the preceding bytes in the chunk, including the chunk type code and chunk data fields, but not including the length field. The CRC is always present, even for chunks containing no data. See [CRC algorithm](#37-crc-algorithm).

The chunk data length can be any number of bytes up to the maximum; therefore, implementors cannot assume that chunks are aligned on any boundaries larger than bytes.

Chunks can appear in any order, subject to the restrictions placed on each chunk type. (One notable restriction is that FHED must appear first and FEND must appear last; thus the FEND chunk serves as an end-of-file marker.) Multiple chunks of the same type can appear, but only if specifically permitted for that type.

### 3.6. Chunk naming conventions

Chunk type codes are assigned so that a decoder can determine some properties of a chunk even when it does not recognize the type code. These rules are intended to allow safe, flexible extension of PNA format, by allowing a decoder to decide what to do when it encounters an unknown chunk. The naming rules are not normally of interest when the decoder does recognize the chunk's type.

Four bits of the type code, namely bit 5 (value 32) of each byte, are used to convey chunk properties. This choice means that a human can read off the assigned properties according to whether each letter of the type code is uppercase (bit 5 is 0) or lowercase (bit 5 is 1). However, decoders should test the properties of an unknown chunk by numerically testing the specified bits; testing whether a character is uppercase or lowercase is inefficient, and even incorrect if a locale-specific case definition is used.

It is worth noting that the property bits are an inherent part of the chunk name, and hence are fixed for any chunk type. Thus, BLOB and bLOb would be unrelated chunk type codes, not the same chunk with different properties. Decoders must recognize type codes by a simple four-byte literal comparison; it is incorrect to perform case conversion on type codes.

The semantics of the property bits are:

**Ancillary bit: bit 5 of first byte**

0 (uppercase) = critical, 1 (lowercase) = ancillary.
Chunks that are not strictly necessary in order to meaningfully extract the contents of the file are known as "ancillary" chunks. A decoder encountering an unknown chunk in which the ancillary bit is 1 can safely ignore the chunk and proceed to extract the file. The created time chunk (cTIM) is an example of an ancillary chunk.

Chunks that are necessary for successful extract of the file's contents are called "critical" chunks. A decoder encountering an unknown chunk in which the ancillary bit is 0 must indicate to the user that the file contains information it cannot safely interpret. The file header chunk (FHED) is an example of a critical chunk.

**Private bit: bit 5 of second byte**

0 (uppercase) = public, 1 (lowercase) = private.
A public chunk is one that is part of PNA specification or is registered in the list of PNA special-purpose public chunk types. Applications can also define private (unregistered) chunks for their own purposes. The names of private chunks must have a lowercase second letter, while public chunks will always be assigned names with uppercase second letters. Note that decoders do not need to test the private-chunk property bit, since it has no functional significance; it is simply an administrative convenience to ensure that public and private chunk names will not conflict. See [Additional chunk types](../chunk_specifications/index.md#44-additional-chunk-types), and Recommendations for Encoders: [Use of private chunks](TODO: link to ../recommendations_for_encoders/index.md).

**Reserved bit: bit 5 of third byte**

Must be 0 (uppercase) in files conforming to this version of PNA.
The significance of the case of the third letter of the chunk name is reserved for possible future expansion. At the present time all chunk names must have uppercase third letters. (Decoders should not complain about a lowercase third letter, however, as some future version of PNA specification could define a meaning for this bit. It is sufficient to treat a chunk with a lowercase third letter in the same way as any other unknown chunk type.)

**Safe-to-copy bit: bit 5 of fourth byte**

0 (uppercase) = unsafe to copy, 1 (lowercase) = safe to copy.
This property bit is not of interest to pure decoders, but it is needed by PNA editors (programs that modify PNA files). This bit defines the proper handling of unrecognized chunks in a file that is being modified.

If a chunk's safe-to-copy bit is 1, the chunk may be copied to a modified PNA file whether or not the software recognizes the chunk type, and regardless of the extent of the file modifications.

If a chunk's safe-to-copy bit is 0, it indicates that the chunk depends on the file data. If the program has made any changes to critical chunks, including addition, modification, deletion, or reordering of critical chunks, then unrecognized unsafe chunks must not be copied to the output PNA file. (Of course, if the program does recognize the chunk, it can choose to output an appropriately modified version.)

A PNA editor is always allowed to copy all unrecognized chunks if it has only added, deleted, modified, or reordered ancillary chunks. This implies that it is not permissible for ancillary chunks to depend on other ancillary chunks.

PNA editors that do not recognize a critical chunk must report an error and refuse to process that PNA file at all. The safe/unsafe mechanism is intended for use with ancillary chunks. The safe-to-copy bit will always be 0 for critical chunks.

Rules for PNA editors are discussed further in Chunk Ordering Rules.

For example, the hypothetical chunk type name bLOb has the property bits:

bLOb  <-- 32-bit chunk type code represented in text form
||||
|||+- Safe-to-copy bit is 1 (lowercase letter; bit 5 is 1)
||+-- Reserved bit is 0     (uppercase letter; bit 5 is 0)
|+--- Private bit is 0      (uppercase letter; bit 5 is 0)
+---- Ancillary bit is 1    (lowercase letter; bit 5 is 1)
Therefore, this name represents an ancillary, public, safe-to-copy chunk.

### 3.7. CRC algorithm

Chunk CRCs are calculated using standard CRC methods with pre and post conditioning, as defined by ISO 3309 [ISO-3309](../references/index.md#iso-3309) or ITU-T V.42 [ITU-T-V42](../references/index.md#itu-t-v42). The CRC polynomial employed is

x^32+x^26+x^23+x^22+x^16+x^12+x^11+x^10+x^8+x^7+x^5+x^4+x^2+x+1

The 32-bit CRC register is initialized to all 1's, and then the data from each byte is processed from the least significant bit (1) to the most significant bit (128). After all the data bytes are processed, the CRC register is inverted (its ones complement is taken). This value is transmitted (stored in the file) MSB first. For the purpose of separating into bytes and ordering, the least significant bit of the 32-bit CRC is defined to be the coefficient of the x31 term.

Practical calculation of the CRC always employs a precalculated table to greatly accelerate the computation.
