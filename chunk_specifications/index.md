## 4. Chunk Specifications

### 4.1. Critical chunks

#### 4.1.1. AHED Archive header

The AHED chunk must appear FIRST. It contains:

| Significance             |  Size  | Description          |
|:-------------------------|:------:|:---------------------|
| Major version            | 1-byte | Major version of PNA |
| Minor version            | 1-byte | Minor version of PNA |
| General purpose bit flag | 2-byte | Bit flags            |
| Archive number           | 4-byte | Archive number       |

##### Major version

Currently, only 0 is defined.
It may be changed if there is a change in the structure of each chunk that makes up PNA.

##### Minor version

Currently, only 0 is defined.
It may be changed when there is a change in the type of chunks that make up PNA.

##### General purpose bit flag

__Bit0__ ~ __Bit15__ currently does not used. reserve for the future.

##### Archive number

Contains the number of the archive when the archive is split.  
Archive number is start with 0.

### 4.1.2. AEND Archive tailer

This chunk must appear last.
This signals the end of PNA data stream.
The chunk data area is empty.
Decoders should not load more than this chunk.

### 4.1.3. ANXT Archive continues marker

Indicates that the archive is split and the following file exists.
The Archive number field of the `AHED` chunk of the next file will be the value of the Archive number field of the `AHED` chunk of the current file incremented by 1.
The chunk data area is empty.

### 4.1.4. FHED File header

Basic information about each entry is stored.

| Significance       |  Size  | Description        |
|:-------------------|:------:|:-------------------|
| Major version      | 1-byte | Major version      |
| Minor version      | 1-byte | Minor version      |
| Entry kind         | 1-byte | Entry kind         |
| Compression method | 1-byte | Compression method |
| Encryption method  | 1-byte | Encryption method  |
| Cipher mode        | 1-byte | Cipher mode        |
| Path               | n-byte | File path          |

##### Entry kind

The entry kind is recorded.
0 is regular file
1 is directory
2 is symbolic link
3 is hard link
4 is a file that has previously appeared in the archive

##### Compression method

The compression method is recorded.
0 is not compression
1 is deflate
2 is zstandard
4 is lzma

##### Encryption method

The encryption method is recorded.
0 is not encryption
1 is AES (Rijndael)
2 is Camellia

When this field value is 0, `PHSF` chunk is not required.

##### Cipher mode

Cipher mode of encryption.
0 is cbc mode
1 is ctr mode

Not interested in the value of this field, if Encryption method filed value is 0.

##### File path

The path of entry is encoded by UTF-8.
The / is used as a path separator.
Paths should not contain / at the leading or trailing.
Decoders should ignore them even if they contain a leading or trailing /.

#### FHED field values (PNA method code)

The `Compression method`, `Encryption method`, and `Cipher mode` fields in the FHED chunk must be set to the following PNA method codes (integer values):

| Field               | Value | Algorithm/Mode        | Note                       | Defined in                                                         |
|:--------------------|------:|:----------------------|:---------------------------|:-------------------------------------------------------------------|
| Compression method  | 0     | No compression        |                            |                                                                    |
|                     | 1     | Deflate               | zlib compatible            | [§5.1](../compression_algorithms/index.md#51-deflate)              |
|                     | 2     | Zstandard             |                            | [§5.2](../compression_algorithms/index.md#52-zstandard)            |
|                     | 4     | LZMA                  | xz                         | [§5.3](../compression_algorithms/index.md#53-lzma)                 |
| Encryption method   | 0     | No encryption         |                            |                                                                    |
|                     | 1     | AES (Rijndael)        | 256-bit key                | [§6.1](../cipher_algorithms/index.md#61-rijndael)                  |
|                     | 2     | Camellia              | 256-bit key                | [§6.2](../cipher_algorithms/index.md#62-camellia)                  |
| Cipher mode         | 0     | CBC                   | Cipher Block Chaining      | [§7.1](../cipher_modes/index.md#71-cipher-block-chaining-mode-cbc) |
|                     | 1     | CTR                   | Counter Mode               | [§7.2](../cipher_modes/index.md#72-counter-mode-ctr)               |

**Note:**
- Do not use algorithm-internal method codes (such as zlib's method/flags code) in these fields. Only the PNA method code (integer value) must be stored in the FHED chunk fields.
- For extensions or private algorithms, use values 64 or greater as recommended in the encoder guidelines.

#### 4.1.5. PHSF Password hash

The PHSF chunk provides information about the password hashing algorithm and its parameters used in key derivation for encrypted entries.
This chunk must appeared after `FHED` chunk and before `FDAT` chunk.  
If the value of the Encryption method field of `FHED` chunk is not 0, this chunk is required.

|  size   | description       |
|:-------:|:------------------|
| n-byte  | PHC string format |

About [PHC string format](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md)

### 4.1.6. FDAT File data

The actual data of the file is recorded.
Multiple of these chunks are permitted for a single entry.

### 4.1.7. FEND File tailer

This signals the end of the entry data stream.
The chunk data area is empty.

### 4.1.8. SHED Solid mode header

Basic information of Solid mode archive is stored.

| significance       |  size   | description        |
|:-------------------|:-------:|:-------------------|
| Major version      | 1-byte  | Major version      |
| Minor version      | 1-byte  | Minor version      |
| Compression method | 1-byte  | Compression method |
| Encryption method  | 1-byte  | Encryption method  |
| Cipher mode        | 1-byte  | Cipher mode        |


### 4.1.9. SDAT Solid mode data

Solid mode archive data.
Contains chunks representing entries.

#### 4.1.9.1 Structure of Entries in SDAT

In solid mode archives, the `SDAT` chunk contains a continuous stream of data representing multiple entries.
Unlike per-entry mode where each entry is composed of discrete `FHED` through `FEND` chunks within the archive, the `SDAT` chunk serializes these chunks together within its own data stream.

The following structural rules apply to the contents of an `SDAT` chunk:

- The `SDAT` chunk MUST contain a sequence of standard entry structures, each beginning with a `FHED` chunk and ending with a corresponding `FEND` chunk.
- All chunks appearing within an `SDAT` chunk must conform to the same structure, layout, and constraints as defined for individual entries in per-entry mode.
- Ancillary chunks (e.g., `cTIM`, `fPRM`, `xATR`) may appear between `FHED` and `FEND`, following the same rules as in regular entry mode.
- Encryption and compression, if applied to the solid stream, are performed **after** the entire SDAT datastream has been composed.

Each entry in solid mode is not stored as an independent top-level chunk sequence in the file but is instead **embedded** within the data field of the `SDAT` chunk.
Decoders must parse the contents of `SDAT` as a logical concatenation of standard entry structures.

The use of solid mode is intended to improve compression efficiency by processing similar files together as a single compression/encryption unit.

**Example structure of SDAT content (conceptual):**

```
[ FHED | Ancillary* | FDAT* | FEND ]
[ FHED | Ancillary* | FDAT* | FEND ]
...
```

The outer `SDAT` chunk may be followed by additional `SDAT` chunks if the solid data stream is split. These must be interpreted as a single contiguous logical stream, terminated by the `SEND` chunk.

### 4.1.10. SEND Solid mode tailer

This signals the end of the solid data stream.  
The chunk data area is empty.

### 4.2. Ancillary chunks

All Auxiliary Chunks must appear before the `AEND` chunk.

#### 4.2.1 Timestamp information

##### 4.2.1.1 cTIM Created timestamp

The creation datetime is recorded in Unix time.
When this chunk appears after the `FHED` chunk and before the `FEND` chunk, it indicates the creation datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

##### 4.2.1.2 mTIM Modified timestamp

The last modified datetime is recorded in Unix time.
When this chunk appears after the `FHED` chunk and before the `FEND` chunk, it indicates the last modified datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

##### 4.2.1.3 aTIM Accessed timestamp

The last accessed datetime is recorded in Unix time.
When this chunk appears after the `FHED` chunk and before the `FEND` chunk, it indicates the last accessed datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

##### 4.2.1.4 cTNS Created timestamp (Nanoseconds)

Provides the nanosecond portion of the file creation time, to be used in conjunction with the `cTIM` chunk.

- **Context**: Must appear after the `FHED` chunk and before the `FEND` chunk.
- **Dependency**: Ignored unless `cTIM` is also present.
- **Interpretation**: `cTIM` seconds + (`cTNS` / 1_000_000_000.0)

| Size   | Description                              |
|--------|------------------------------------------|
| 4-byte | Unsigned integer (`u32`), nanoseconds    |

Valid values are in the range `0 <= value < 1,000,000,000`.

##### 4.2.1.5 mTNS Modified timestamp (Nanoseconds)

Provides the nanosecond portion of the last modified time, to be used in conjunction with the `mTIM` chunk.

- **Context**: Must appear after the `FHED` chunk and before the `FEND` chunk.
- **Dependency**: Ignored unless `mTIM` is also present.
- **Interpretation**: `mTIM` seconds + (`mTNS` / 1_000_000_000.0)

| Size   | Description                              |
|--------|------------------------------------------|
| 4-byte | Unsigned integer (`u32`), nanoseconds    |

Valid values are in the range `0 <= value < 1,000,000,000`.

##### 4.2.1.6 aTNS Accessed timestamp (Nanoseconds)

Provides the nanosecond portion of the last accessed time, to be used in conjunction with the `aTIM` chunk.

- **Context**: Must appear after the `FHED` chunk and before the `FEND` chunk.
- **Dependency**: Ignored unless `aTIM` is also present.
- **Interpretation**: `aTIM` seconds + (`aTNS` / 1_000_000_000.0)

| Size   | Description                              |
|--------|------------------------------------------|
| 4-byte | Unsigned integer (`u32`), nanoseconds    |

Valid values are in the range `0 <= value < 1,000,000,000`.

**Decoder Notes**:
- If any `*TNS` chunk is present without its corresponding `*TIM` chunk, it must be ignored.
- Values outside the range `[0, 999_999_999]` are invalid and must cause a decoder error.
- Encoders should omit `*TNS` chunks unless sub-second precision is explicitly needed.

#### 4.2.2 permission information

##### 4.2.2.1 fPRM File permission

File permissions are recorded.
This chunk appeared after `FHED` chunk and before `FEND` chunk.

| significance |  size  | description           |
|:-------------|:------:|:----------------------|
| uid          | 8-byte | user ID               |
| uname length | 1-byte | length of uname       |
| uname        | n-byte | Unix user name        |
| gid          | 8-byte | group ID              |
| gname length | 1-byte | length of gname       |
| gname        | n-byte | Unix group name       |
| permissions  | 2-byte | file permission bytes |

#### 4.2.3 Extended attribute

##### 4.2.3.1 xATR Extended attribute

An extended attribute are recorded.
this chunk appeared after `FHED` chunk and before `FEND` chunk. this chunk can appear many times.

| significance |  size  | description           |
|:-------------|:------:|:----------------------|
| name length  | 4-byte | length of name        |
| name         | n-byte | attribute name        |
| body length  | 4-byte | length of body        |
| body         | n-byte | attribute value       |

#### 4.2.4 Link target type

##### 4.2.4.1 fLTP Link target type

The link target type for link entries is recorded.
This chunk appeared after `FHED` chunk and before `FEND` chunk.
This chunk is meaningful only when the `Entry kind` field of the `FHED` chunk is `2` (symbolic link) or `3` (hard link).

| significance     |  size  | description           |
|:-----------------|:------:|:----------------------|
| link target type | 1-byte | link target type code |

##### Link target type code

The link target type is recorded.
0 is unknown (reserved for explicit unknown)
1 is file
2 is directory

Values 3 through 63 are reserved for future public extensions.
Values 64 through 255 are reserved for private extensions.

##### Constraints

- Encoders MAY write `fLTP` on entries whose `FHED.Entry kind` is not `2` (symbolic link) or `3` (hard link); decoders MUST ignore `fLTP` in such cases.

##### Directory hard link

When `FHED.Entry kind` is `3` (hard link) and the link target type is `2` (directory), the entry represents a **directory hard link** (see Glossary): a hard link whose target is a directory. This is the canonical PNA encoding for directory-target link semantics, and corresponds to platform-native constructs such as Windows NTFS junctions.

On systems that cannot create hard links to directories (e.g., POSIX-compliant file systems that prohibit hardlink-to-directory), implementations MAY fall back to creating a symbolic link to the target directory when extracting such entries. This fallback is a local substitution at extraction time and does not modify the on-wire `FHED.Entry kind` or `fLTP` values.

#### 4.2.5 Raw file size hint

##### 4.2.5.1 fSIZ Raw file size hint

A hint of the byte count of the source file, prior to any compression or encryption, is recorded.
This chunk appeared after `FHED` chunk and before `FEND` chunk.

| significance |  size   | description                 |
|:-------------|:-------:|:----------------------------|
| size         | n-byte  | Big-endian unsigned integer |

The chunk data consists of zero or more bytes representing the size as a big-endian unsigned integer. Decoders MUST interpret a zero-length chunk data as a size of zero.

##### Constraints

- Encoders MAY write `fSIZ` for any entry kind; for entries whose `FHED.Entry kind` is not `0` (regular file), the value has no defined semantics and decoders MUST ignore it.
- `fSIZ` is informational. The value is a hint only. Decoders MUST NOT rely on it for buffer allocation, memory reservation, or security decisions, and MUST NOT require the reported size to match the actual size of the decompressed and decrypted entry data.

#### 4.2.6 fACL Access Control List

Access Control List (ACL) information is recorded.
An `fACL` chunk consists of an 8-byte header followed by a sequence of Access Control Entries (ACEs). Values in the `Ace type`, `Permissions`, `Ace flag`, and `Bit flags` fields are interpreted according to the `Platform` field; decoders MUST read `Platform` before interpreting those fields.
This chunk appeared after `FHED` chunk and before `FEND` chunk.

| significance |  size       | description                           |
|:-------------|:-----------:|:--------------------------------------|
| Version      | 1-byte      | fACL schema version                   |
| Reserved     | 1-byte      | reserved, MUST be 0                   |
| Platform     | 1-byte      | acl platform type                     |
| Acl type     | 1-byte      | type of acl                           |
| Bit flags    | 2-byte      | platform-dependent ACL-level flags    |
| Entry count  | 2-byte      | number of access control entries      |
| Ace ...      | variable    | repeat access control entries         |

##### Version

Encoders conforming to this specification MUST set `Version` to `0`.
Decoders MUST reject `fACL` chunks with an unknown `Version` value, because the interpretation of subsequent fields may change in future schema versions.

##### Platform

0 is POSIX.1e (classic)
1 is Darwin extended
2 is Windows DACL
3 is NFSv4 DACL

Values 4 through 63 are reserved for future public extensions.
Values 64 through 255 are reserved for private extensions.

##### Acl type

0 is primary ACL (Windows DACL / POSIX.1e primary / Darwin / NFSv4)
1 is SACL (Windows audit ACL)
2 is POSIX.1e default ACL

Values 3 through 63 are reserved for future public extensions.
Values 64 through 255 are reserved for private extensions.

Multiple `fACL` chunks MAY appear in the same entry to represent different ACL objects (for example, a Windows DACL and SACL pair, or a POSIX.1e primary and default ACL pair). Each `fACL` chunk represents exactly one ACL object.

##### Bit flags

The `Bit flags` field holds ACL-level flags whose interpretation depends on `Platform`. Undefined bits are reserved; encoders MUST set undefined bits to 0, and decoders MUST ignore them.

###### POSIX.1e (classic)

All bits reserved. Encoders MUST set the field to 0.

###### Darwin extended

All bits reserved. Encoders MUST set the field to 0.

###### Windows DACL

For `Acl type` 0 (DACL) or 1 (SACL), the `Bit flags` field encodes the ACL-applicable subset of `SECURITY_DESCRIPTOR.Control` per MS-DTYP §2.4.6:

const WINDOWS_ACL_PROTECTED      = 0x0001;  
const WINDOWS_ACL_AUTO_INHERITED = 0x0002;  
const WINDOWS_ACL_DEFAULTED      = 0x0004;  

###### NFSv4 DACL

The `Bit flags` field encodes `aclflag4` per RFC 5661 §6.4.2.1:

const NFSv4_ACL_AUTO_INHERIT = 0x0001;  
const NFSv4_ACL_PROTECTED    = 0x0002;  
const NFSv4_ACL_DEFAULTED    = 0x0004;  

##### Entry count

`Entry count` is a big-endian unsigned 16-bit integer. A value of 0 indicates an empty ACL.

##### Ace

The Ace (Access Control Entry) structure is repeated `Entry count` times.

| significance      |  size    | description                           |
|:------------------|:--------:|:--------------------------------------|
| Ace type          | 1-byte   | platform-dependent ace type code      |
| Reserved          | 5-byte   | reserved, MUST be 0                   |
| Permissions       | 4-byte   | platform-dependent permission bits    |
| Ace flag          | 4-byte   | platform-dependent ACE flag bits      |
| Name length       | 2-byte   | byte length of Name (0 if absent)     |
| Name              | n-byte   | UTF-8 portable trustee identifier     |
| Native length     | 2-byte   | byte length of Native (0 if absent)   |
| Native            | m-byte   | platform-native trustee identifier    |
| Extension length  | 4-byte   | byte length of Extension (0 if none)  |
| Extension data    | k-byte   | ace-type-specific trailer             |

The `Name` field holds a UTF-8 trustee identifier portable across platforms (for example, a user or group name). The `Native` field holds the platform-authoritative trustee representation (see Identifier per Platform below). At least one of `Name` or `Native` MUST be populated unless the Ace type context implies no trustee (for example, POSIX.1e `USER_OBJ`, `GROUP_OBJ`, `MASK`, or `OTHER`).

Encoders SHOULD populate both `Name` and `Native` when both are resolvable. If a decoder cannot simultaneously honor both on the target platform, it SHOULD resolve the trustee from `Name`.

##### Ace type

The interpretation of the `Ace type` field depends on `Platform`.

###### POSIX.1e (classic)

// POSIX.1e classic ACL semantic tags.
// Numeric values are defined by this specification rather than
// transcribed from a specific implementation, because Linux and
// FreeBSD use non-uniform acl_tag_t encodings.
const POSIX_CLASSIC_USER_OBJ  = 0x00;  
const POSIX_CLASSIC_USER      = 0x01;  
const POSIX_CLASSIC_GROUP_OBJ = 0x02;  
const POSIX_CLASSIC_GROUP     = 0x03;  
const POSIX_CLASSIC_MASK      = 0x04;  
const POSIX_CLASSIC_OTHER     = 0x05;  

For `USER_OBJ`, `GROUP_OBJ`, `MASK`, and `OTHER`, `Name length` and `Native length` MUST both be 0.

###### Darwin extended

// Darwin ACL: sys/acl.h (macOS SDK, MacOSX15.4.sdk)
const DARWIN_ACL_EXTENDED_ALLOW = 0x01;  
const DARWIN_ACL_EXTENDED_DENY  = 0x02;  

###### Windows DACL

// Windows ACE type: MS-DTYP §2.4.4.1 ACE_HEADER
const WINDOWS_ACCESS_ALLOWED_ACE_TYPE                 = 0x00;  
const WINDOWS_ACCESS_DENIED_ACE_TYPE                  = 0x01;  
const WINDOWS_SYSTEM_AUDIT_ACE_TYPE                   = 0x02;  
const WINDOWS_SYSTEM_ALARM_ACE_TYPE                   = 0x03;  
const WINDOWS_ACCESS_ALLOWED_COMPOUND_ACE_TYPE        = 0x04;  
const WINDOWS_ACCESS_ALLOWED_OBJECT_ACE_TYPE          = 0x05;  
const WINDOWS_ACCESS_DENIED_OBJECT_ACE_TYPE           = 0x06;  
const WINDOWS_SYSTEM_AUDIT_OBJECT_ACE_TYPE            = 0x07;  
const WINDOWS_SYSTEM_ALARM_OBJECT_ACE_TYPE            = 0x08;  
const WINDOWS_ACCESS_ALLOWED_CALLBACK_ACE_TYPE        = 0x09;  
const WINDOWS_ACCESS_DENIED_CALLBACK_ACE_TYPE         = 0x0A;  
const WINDOWS_ACCESS_ALLOWED_CALLBACK_OBJECT_ACE_TYPE = 0x0B;  
const WINDOWS_ACCESS_DENIED_CALLBACK_OBJECT_ACE_TYPE  = 0x0C;  
const WINDOWS_SYSTEM_AUDIT_CALLBACK_ACE_TYPE          = 0x0D;  
const WINDOWS_SYSTEM_ALARM_CALLBACK_ACE_TYPE          = 0x0E;  
const WINDOWS_SYSTEM_AUDIT_CALLBACK_OBJECT_ACE_TYPE   = 0x0F;  
const WINDOWS_SYSTEM_ALARM_CALLBACK_OBJECT_ACE_TYPE   = 0x10;  
const WINDOWS_SYSTEM_MANDATORY_LABEL_ACE_TYPE         = 0x11;  
const WINDOWS_SYSTEM_RESOURCE_ATTRIBUTE_ACE_TYPE      = 0x12;  
const WINDOWS_SYSTEM_SCOPED_POLICY_ID_ACE_TYPE        = 0x13;  

###### NFSv4 DACL

// NFSv4 ACE type: RFC 5661 §6.2.1.1.
// NFSv4 natively encodes acetype as a 32-bit unsigned integer.
// fACL stores the least significant byte. A future revision
// defining values beyond 0xFF will require a Version bump.
const NFSv4_ACCESS_ALLOWED_ACE_TYPE = 0x00;  
const NFSv4_ACCESS_DENIED_ACE_TYPE  = 0x01;  
const NFSv4_SYSTEM_AUDIT_ACE_TYPE   = 0x02;  
const NFSv4_SYSTEM_ALARM_ACE_TYPE   = 0x03;  

##### Permissions

The interpretation of the `Permissions` field depends on `Platform`.

###### POSIX.1e (classic)

const POSIX_READ    = 0x00000004;  
const POSIX_WRITE   = 0x00000002;  
const POSIX_EXECUTE = 0x00000001;  

###### Darwin extended

// Darwin ACL: sys/acl.h (macOS SDK, MacOSX15.4.sdk)
const DARWIN_READ_DATA           = 0x00000002;  
const DARWIN_LIST_DIRECTORY      = 0x00000002;  
const DARWIN_WRITE_DATA          = 0x00000004;  
const DARWIN_ADD_FILE            = 0x00000004;  
const DARWIN_EXECUTE             = 0x00000008;  
const DARWIN_SEARCH              = 0x00000008;  
const DARWIN_DELETE              = 0x00000010;  
const DARWIN_APPEND_DATA         = 0x00000020;  
const DARWIN_ADD_SUBDIRECTORY    = 0x00000020;  
const DARWIN_DELETE_CHILD        = 0x00000040;  
const DARWIN_READ_ATTRIBUTES     = 0x00000080;  
const DARWIN_WRITE_ATTRIBUTES    = 0x00000100;  
const DARWIN_READ_EXTATTRIBUTES  = 0x00000200;  
const DARWIN_WRITE_EXTATTRIBUTES = 0x00000400;  
const DARWIN_READ_SECURITY       = 0x00000800;  
const DARWIN_WRITE_SECURITY      = 0x00001000;  
const DARWIN_CHANGE_OWNER        = 0x00002000;  
const DARWIN_SYNCHRONIZE         = 0x00100000;  

###### Windows DACL

// Windows access mask: MS-DTYP §2.4.3 ACCESS_MASK
const WINDOWS_READ_DATA            = 0x00000001;  
const WINDOWS_LIST_DIRECTORY       = 0x00000001;  
const WINDOWS_WRITE_DATA           = 0x00000002;  
const WINDOWS_ADD_FILE             = 0x00000002;  
const WINDOWS_APPEND_DATA          = 0x00000004;  
const WINDOWS_ADD_SUBDIRECTORY     = 0x00000004;  
const WINDOWS_READ_NAMED_ATTRS     = 0x00000008;  
const WINDOWS_WRITE_NAMED_ATTRS    = 0x00000010;  
const WINDOWS_EXECUTE              = 0x00000020;  
const WINDOWS_DELETE_CHILD         = 0x00000040;  
const WINDOWS_READ_ATTRIBUTES      = 0x00000080;  
const WINDOWS_WRITE_ATTRIBUTES     = 0x00000100;  
const WINDOWS_DELETE               = 0x00010000;  
const WINDOWS_READ_ACL             = 0x00020000;  
const WINDOWS_WRITE_ACL            = 0x00040000;  
const WINDOWS_WRITE_OWNER          = 0x00080000;  
const WINDOWS_SYNCHRONIZE          = 0x00100000;  

###### NFSv4 DACL

// NFSv4 access mask: RFC 7530 / RFC 5661
const NFSv4_READ_DATA            = 0x00000001;  
const NFSv4_LIST_DIRECTORY       = 0x00000001;  
const NFSv4_WRITE_DATA           = 0x00000002;  
const NFSv4_ADD_FILE             = 0x00000002;  
const NFSv4_APPEND_DATA          = 0x00000004;  
const NFSv4_ADD_SUBDIRECTORY     = 0x00000004;  
const NFSv4_READ_NAMED_ATTRS     = 0x00000008;  
const NFSv4_WRITE_NAMED_ATTRS    = 0x00000010;  
const NFSv4_EXECUTE              = 0x00000020;  
const NFSv4_DELETE_CHILD         = 0x00000040;  
const NFSv4_READ_ATTRIBUTES      = 0x00000080;  
const NFSv4_WRITE_ATTRIBUTES     = 0x00000100;  
const NFSv4_WRITE_RETENTION      = 0x00000200;  
const NFSv4_WRITE_RETENTION_HOLD = 0x00000400;  
const NFSv4_DELETE               = 0x00010000;  
const NFSv4_READ_ACL             = 0x00020000;  
const NFSv4_WRITE_ACL            = 0x00040000;  
const NFSv4_WRITE_OWNER          = 0x00080000;  
const NFSv4_SYNCHRONIZE          = 0x00100000;  

##### Ace flag

The interpretation of the `Ace flag` field depends on `Platform`.

###### POSIX.1e (classic)

All bits reserved. Encoders MUST set the field to 0.

###### Darwin extended

// Darwin ACL: sys/acl.h (macOS SDK, MacOSX15.4.sdk)
const DARWIN_ACL_FLAG_DEFER_INHERIT      = 0x00000001;  
const DARWIN_ACL_FLAG_NO_INHERIT         = 0x00020000;  
const DARWIN_ACL_ENTRY_INHERITED         = 0x00000010;  
const DARWIN_ACL_ENTRY_FILE_INHERIT      = 0x00000020;  
const DARWIN_ACL_ENTRY_DIRECTORY_INHERIT = 0x00000040;  
const DARWIN_ACL_ENTRY_LIMIT_INHERIT     = 0x00000080;  
const DARWIN_ACL_ENTRY_ONLY_INHERIT      = 0x00000100;  

###### Windows DACL

// Windows ACE flags: MS-DTYP §2.4.4.1 ACE_HEADER
const WINDOWS_OBJECT_INHERIT_ACE         = 0x01;  
const WINDOWS_CONTAINER_INHERIT_ACE      = 0x02;  
const WINDOWS_NO_PROPAGATE_INHERIT_ACE   = 0x04;  
const WINDOWS_INHERIT_ONLY_ACE           = 0x08;  
const WINDOWS_INHERITED_ACE              = 0x10;  
const WINDOWS_SUCCESSFUL_ACCESS_ACE_FLAG = 0x40;  
const WINDOWS_FAILED_ACCESS_ACE_FLAG     = 0x80;  

###### NFSv4 DACL

// NFSv4 ACE flag: RFC 5661 §6.2.1.4
const NFSv4_FILE_INHERIT_ACE           = 0x00000001;  
const NFSv4_DIRECTORY_INHERIT_ACE      = 0x00000002;  
const NFSv4_NO_PROPAGATE_INHERIT_ACE   = 0x00000004;  
const NFSv4_INHERIT_ONLY_ACE           = 0x00000008;  
const NFSv4_SUCCESSFUL_ACCESS_ACE_FLAG = 0x00000010;  
const NFSv4_FAILED_ACCESS_ACE_FLAG     = 0x00000020;  
const NFSv4_IDENTIFIER_GROUP           = 0x00000040;  
const NFSv4_INHERITED_ACE              = 0x00000080;  

##### Identifier per Platform

For ACEs whose `Ace type` requires a trustee, the `Name` and `Native` fields encode the trustee per `Platform`:

| Platform              | Name (UTF-8 portable)                       | Native (platform verbatim)                                         |
|:----------------------|:--------------------------------------------|:-------------------------------------------------------------------|
| 0 POSIX.1e (classic)  | user or group name                          | `uid` or `gid` as 4-byte big-endian unsigned integer               |
| 1 Darwin extended     | user or group name                          | GUID (16-byte binary)                                              |
| 2 Windows DACL        | `DOMAIN\User` or UPN (`user@domain`)        | SID per MS-DTYP §2.4.2.2 serialized binary                         |
| 3 NFSv4 DACL          | `user@domain` or well-known principal (e.g., `OWNER@`, `GROUP@`, `EVERYONE@`) per RFC 5661 §6.2.1.5 | `Native length` MUST be 0; the who-string appears in `Name` only   |

##### Extension data

The `Extension data` trailer carries ace-type-specific information that does not fit the fixed fields above. Its format depends on both `Platform` and `Ace type`.

###### Windows Object ACE

For Windows ACE types `ACCESS_ALLOWED_OBJECT` (`0x05`), `ACCESS_DENIED_OBJECT` (`0x06`), `SYSTEM_AUDIT_OBJECT` (`0x07`), `SYSTEM_ALARM_OBJECT` (`0x08`), `ACCESS_ALLOWED_CALLBACK_OBJECT` (`0x0B`), `ACCESS_DENIED_CALLBACK_OBJECT` (`0x0C`), `SYSTEM_AUDIT_CALLBACK_OBJECT` (`0x0F`), and `SYSTEM_ALARM_CALLBACK_OBJECT` (`0x10`), the `Extension data` layout mirrors the object ACE body defined in MS-DTYP §2.4.4.3:

| significance           |  size    | description                                                        |
|:-----------------------|:--------:|:-------------------------------------------------------------------|
| Flags                  | 4-byte   | ObjectType presence flags (bit 0 = ObjectType, bit 1 = InheritedObjectType) |
| ObjectType             | 16-byte  | present when Flags bit 0 is set                                    |
| InheritedObjectType    | 16-byte  | present when Flags bit 1 is set                                    |
| ApplicationData        | variable | callback body (present for CALLBACK variants, empty otherwise)     |

###### Windows Callback ACE

For Windows ACE types `ACCESS_ALLOWED_CALLBACK` (`0x09`), `ACCESS_DENIED_CALLBACK` (`0x0A`), `SYSTEM_AUDIT_CALLBACK` (`0x0D`), and `SYSTEM_ALARM_CALLBACK` (`0x0E`), the `Extension data` is the raw callback body.

###### Windows Resource Attribute ACE

For Windows ACE type `SYSTEM_RESOURCE_ATTRIBUTE` (`0x12`), the `Extension data` is a `CLAIM_SECURITY_ATTRIBUTE_RELATIVE_V1` structure per MS-DTYP §2.4.10.1.

###### Other combinations

For any other `Platform` + `Ace type` combination, `Extension length` MUST be 0.

##### Constraints

- An `fACL` chunk MAY appear multiple times within a single entry, each representing a distinct `Platform` + `Acl type` combination. When multiple `fACL` chunks share the same `Platform` + `Acl type` pair, decoders MUST treat the last occurrence as authoritative.
- Encoders MAY write `fACL` on entries of any `FHED.Entry kind`. Decoders that cannot interpret a chunk's `Platform` value MUST skip the chunk; extraction MAY proceed using `fPRM` as a fallback source of basic permission information.
- Decoders SHOULD prefer `Native` for same-platform restoration and `Name` for cross-platform restoration.

### 4.3. Summary of standard chunks

This table summarizes some properties of the standard chunk types.

Critical chunks

| Name | Multiple in Archive | Multiple in Entry | Optional | Ordering constraints               |
|:----:|:-------------------:|:-----------------:|:--------:|:-----------------------------------|
| AHED |         No          |        N/A        |    No    | Must be first                      |
| FHED |        Yes          |        No         |   Yes    | Must start an entry                |
| PHSF |        Yes          |        No         |   Yes    | Before FDAT or SDAT if used        |
| FDAT |        Yes          |       Yes         |   Yes    | Multiple FDATs must be consecutive |
| FEND |        Yes          |        No         |   Yes    | Must end an entry                  |
| SHED |        Yes          |        No         |   Yes    | Must start an Solid mode data      |
| SDAT |        Yes          |       Yes         |   Yes    | Multiple SDATs must be consecutive |
| SEND |        Yes          |        No         |   Yes    | Must end an Solid mode data        |
| ANXT |         No          |        N/A        |   Yes    |                                    |
| AEND |         No          |        N/A        |    No    | Must be last                       |

Ancillary chunks

| Name  | Multiple in Archive | Multiple in Entry | Optional | Ordering constraints                          |
|:-----:|:-------------------:|:-----------------:|:--------:|:----------------------------------------------|
| cTIM  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| mTIM  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| aTIM  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| cTNS  |        Yes          |        No         |   Yes    | Must accompany `cTIM`, between `FHED`–`FEND` |
| mTNS  |        Yes          |        No         |   Yes    | Must accompany `mTIM`, between `FHED`–`FEND` |
| aTNS  |        Yes          |        No         |   Yes    | Must accompany `aTIM`, between `FHED`–`FEND` |
| fPRM  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| xATR  |        Yes          |       Yes         |   Yes    | Between `FHED` and `FEND`                    |
| fLTP  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| fSIZ  |        Yes          |        No         |   Yes    | Between `FHED` and `FEND`                    |
| fACL  |        Yes          |       Yes         |   Yes    | Between `FHED` and `FEND`                    |

### 4.4. Additional chunk types

Additional public PNA chunk types are defined in the document "Extensions to the PNA 0.0 Specification, Version 0.0.0" [PNA-EXTENSIONS]. Chunks described there are expected to be less widely supported than those defined in this specification. However, application authors are encouraged to use those chunk types whenever appropriate for their applications. Additional chunk types can be proposed for inclusion in that list by contacting the PNA specification maintainers at @Portable-Network-Archive on GitHub.

New public chunks will be registered only if they are of use to others and do not violate the design philosophy of PNA. Chunk registration is not automatic, although it is the intent of the authors that it be straightforward when a new chunk of potentially wide application is needed. Note that the creation of new critical chunk types is discouraged unless absolutely necessary.

Applications can also use private chunk types to carry data that is not of interest to other applications. See Recommendations for Encoders: Use of private chunks.

Decoders must be prepared to encounter unrecognized public or private chunk-type codes. Unrecognized chunk types must be handled as described in Chunk naming conventions.
