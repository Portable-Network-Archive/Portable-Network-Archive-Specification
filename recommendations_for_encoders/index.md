## 11. Recommendations for Encoders

This chapter gives some recommendations for encoder behavior. The only absolute requirement on a PNA encoder is that it produce files that conform to the format specified in the preceding chapters. However, best results will usually be achieved by following these recommendations.

### 11.1. Use of private chunks

Applications can use PNA private chunks to carry information that need not be understood by other applications. Such chunks must be given names with lowercase second letters, to ensure that they can never conflict with any future public chunk definition. Note, however, that there is no guarantee that some other application will not use the same private chunk name. If you use a private chunk type, it is prudent to store additional identifying information at the beginning of the chunk data.

Use an ancillary chunk type (lowercase first letter), not a critical chunk type, for all private chunks that store information that is not absolutely essential to read the archive. Creation of private critical chunks is discouraged because they render PNA files unportable. Such chunks should not be used in publicly available software or files. If private critical chunks are essential for your application, it is recommended that one appear near the start of the file, so that a standard decoder need not read very far before discovering that it cannot handle the file.

If you want others outside your organization to understand a chunk type that you invent, contact the maintainers of the PNA specification to submit a proposed chunk name and definition for addition to the list of special-purpose public chunks (see [Additional chunk types](../chunk_specifications/index.md#44-additional-chunk-types)). Note that a proposed public chunk name (with uppercase second letter) must not be used in publicly available software or files until registration has been approved.

If an ancillary chunk contains textual information that might be of interest to a human user, you should **not** create a special chunk type for it. Instead use a text chunk and define a suitable keyword. That way, the information will be available to users not using your software.

Keywords in text chunks should be reasonably self-explanatory, since the idea is to let other users figure out what the chunk contains. If of general usefulness, new keywords can be registered with the maintainers of the PNA specification. But it is permissible to use keywords without registering them first.

### 11.2. Private type and method codes

This specification defines the meaning of only some of the possible values of some fields. For example, only compression method 0 through 4, encryption method 0 through 3 and cipher mode 0 through 1 are defined. Numbers greater than 63 must be used when inventing experimental or private definitions of values for any of these fields. Numbers below 64 are reserved for possible future public extensions of this specification. Note that use of private type codes may render a file unreadable by standard decoders. Such codes are strongly discouraged except for experimental purposes, and should not appear in publicly available software or files.
