## 2. Data Representation

This chapter discusses basic data representations used in PNA files, as well as the expected representation of the archive data.

### 2.1 Archive layout

Conceptually, PNA archive is represented by a series of multiple entries between the start and end headers.
There are multiple types of entries, including files and directories.

Three types of archive are supported:
- An archive in which one entry consists of one file or directory and corresponding entries
- An archive consisting of an entry that is a collection of multiple entries.
- Archives that mix the above two

Optionally, PNA archive can be split.
Splitting is allowed across entries.

### 2.2. Integers and byte order

All integers that require more than one byte must be in network byte order: the most significant byte comes first, then the less significant bytes in descending order of significance (MSB LSB for two-byte integers, B3 B2 B1 B0 for four-byte integers). The highest bit (value 128) of a byte is numbered bit 7; the lowest bit (value 1) is numbered bit 0. Values are unsigned unless otherwise noted. Values explicitly noted as signed are represented in two's complement notation.

### 2.3 Text encodings

All character information must be encoded in UTF-8: this also includes the path for each entry.
