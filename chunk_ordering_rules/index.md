## 9. Chunk Ordering Rules

To allow new chunk types to be added to PNA, it is necessary to establish rules about the ordering requirements for all chunk types. Otherwise, a PNA editing program cannot know what to do when it encounters an unknown chunk.

We define a "PNA editor" as a program that modifies a PNA file and wishes to preserve as much as possible of the ancillary information in the file. Two examples of PNA editors are a program that adds or modifies text chunks, and a program that adds or modifies an entry to a PNA file. (Note: we strongly encourage programs handling PNA files to preserve ancillary information whenever possible.)

As an example of possible problems, consider a hypothetical new ancillary chunk type that is safe-to-copy and is required to appear after PHSF if PHSF is present. If our program to add a suggested PHSF does not recognize this new chunk, it may insert PHSF in the wrong place, namely after the new chunk. We could prevent such problems by requiring PNA editors to discard all unknown chunks, but that is a very unattractive solution. Instead, PNA requires ancillary chunks not to have ordering restrictions like this.

To prevent this type of problem while allowing for future extension, we put some constraints on both the behavior of PNA editors and the allowed ordering requirements for chunks.

### 9.1. Behavior of PNA editors

The rules for PNA editors are:

- When copying an unknown unsafe-to-copy ancillary chunk, a PNA editor must not move the chunk relative to any critical chunk. It can relocate the chunk freely relative to other ancillary chunks that occur between the same pair of critical chunks. (This is well defined since the editor must not add, delete, modify, or reorder critical chunks if it is preserving unknown unsafe-to-copy chunks.)
- When copying an unknown safe-to-copy ancillary chunk, a PNA editor must not move the chunk from before FDAT to after FDAT or vice versa. (This is well defined because FDAT is always present.) This is the same goes for SDAT. Any other reordering is permitted.
- When copying a known ancillary chunk type, an editor need only honor the specific chunk ordering rules that exist for that chunk type. However, it can always choose to apply the above general rules instead.
- PNA editors must give up on encountering an unknown critical chunk type, because there is no way to be certain that a valid file will result from modifying a file containing such a chunk. (Note that simply discarding the chunk is not good enough, because it might have unknown implications for the interpretation of other chunks.)
- These rules are expressed in terms of copying chunks from an input file to an output file, but they apply in the obvious way if a PNA file is modified in place.

See also [Chunk naming conventions](../file_structure/index.md#36-chunk-naming-conventions).

### 9.2. Ordering of ancillary chunks

The ordering rules for an ancillary chunk type cannot be any stricter than this:

- Unsafe-to-copy chunks can have ordering requirements relative to critical chunks.
- Safe-to-copy chunks can have ordering requirements relative to FDAT or SDAT.

The actual ordering rules for any particular ancillary chunk type may be weaker. See for example the ordering rules for the standard ancillary chunk types ([Summary of standard chunks](../chunk_specifications/index.md#43-summary-of-standard-chunks)).

**Decoders must not assume more about the positioning of any ancillary chunk than is specified by the chunk ordering rules.** In particular, it is never valid to assume that a specific ancillary chunk type occurs with any particular positioning relative to other ancillary chunks. (For example, it is unsafe to assume that your private ancillary chunk occurs immediately before AEND, FEND or SEND. Even if your application always writes it there, a PNA editor might have inserted some other ancillary chunk after it. But you can safely assume that your chunk will remain somewhere between AHED and AEND or FDAT and FEND or SDAT and SEND.)

### 9.3. Ordering of critical chunks

Critical chunks can have arbitrary ordering requirements, because PNA editors are required to give up if they encounter unknown critical chunks. For example, AHED has the special ordering rule that it must always appear first. A PNA editor, or indeed any PNA-writing program, must know and follow the ordering rules for any critical chunk type that it can emit.
