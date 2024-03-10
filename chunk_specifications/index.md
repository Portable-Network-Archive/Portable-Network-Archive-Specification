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
1 is AES
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

#### 4.1.5. PHSF Password hash

The information about the key derivation function when encrypting a file.  
This chunk appeared after `FHAD` chunk and before `FDAT` chunk.  
If the value of encryption method field of `FHAD` chunk is not 0, this chunk is required.

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

### 4.1.10. SEND Solid mode tailer

This signals the end of the solid data stream.  
The chunk data area is empty.

### 4.2. Ancillary chunks

All Auxiliary Chunks must appear before the `AEND` chunk.

#### 4.2.1 Timestamp information

##### 4.2.1.1 cTIM Created timestamp

The creation datetime is recorded in Unix time.
When this chunk appears after the `FHAD` chunk and before the `FEND` chunk, it indicates the creation datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

##### 4.2.1.2 mTIM Modified timestamp

The last modified datetime is recorded in Unix time.
When this chunk appears after the `FHAD` chunk and before the `FEND` chunk, it indicates the last modified datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

##### 4.2.1.3 aTIM Accessed timestamp

The last accessed datetime is recorded in Unix time.
When this chunk appears after the `FHAD` chunk and before the `FEND` chunk, it indicates the last accessed datetime of the entry.

|  size  | description    |
|:------:|:---------------|
| 8byte  | Unix timestamp |

#### 4.2.2 permission information

##### 4.2.2.1 fPRM File permission

File permissions are recorded.
This chunk appeared after `FHAD` chunk and before `FEND` chunk.

| significance |  size  | description           |
|:-------------|:------:|:----------------------|
| uid          | 8-byte | user ID               |
| uname length | 1-byte | length of uname       |
| uname        | n-byte | Unix user name        |
| gid          | 8-byte | group ID              |
| gname length | 1-byte | length of gname       |
| gname        | n-byte | Unix group name       |
| permissions  | 2-byte | file permission bytes |

### 4.3. Summary of standard chunks

This table summarizes some properties of the standard chunk types.

Critical chunks

| Name | Multiple | Optional | Ordering constraints               |
|:----:|:--------:|:--------:|:-----------------------------------|
| AHED |    No    |    No    | Must be first                      |
| FHED |   Yes    |   Yes    |                                    |
| PHSF |   Yes    |   Yes    | Before FDAT or SDAT                |
| FDAT |   Yes    |   Yes    | Multiple FDATs must be consecutive |
| FEND |   Yes    |   Yes    |                                    |
| SHED |   Yes    |   Yes    |                                    |
| SDAT |   Yes    |   Yes    | Multiple SDATs must be consecutive |
| SEND |   Yes    |   Yes    |                                    |
| ANXT |    No    |   Yes    |                                    |
| AEND |    No    |    No    | Must be last                       |

### 4.4. Additional chunk types
Additional public PNG chunk types are defined in the document "Extensions to the PNG 1.2 Specification, Version 1.2.0" [PNG-EXTENSIONS]. Chunks described there are expected to be less widely supported than those defined in this specification. However, application authors are encouraged to use those chunk types whenever appropriate for their applications. Additional chunk types can be proposed for inclusion in that list by contacting the PNG specification maintainers at png-info@uunet.uu.net or at png-group@w3.org.

New public chunks will be registered only if they are of use to others and do not violate the design philosophy of PNG. Chunk registration is not automatic, although it is the intent of the authors that it be straightforward when a new chunk of potentially wide application is needed. Note that the creation of new critical chunk types is discouraged unless absolutely necessary.

Applications can also use private chunk types to carry data that is not of interest to other applications. See Recommendations for Encoders: Use of private chunks.

Decoders must be prepared to encounter unrecognized public or private chunk-type codes. Unrecognized chunk types must be handled as described in Chunk naming conventions.
