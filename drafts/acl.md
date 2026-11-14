# Access control lists (`aCLS`) — Draft

## Status

Work in progress for the PNA 0.0 specification. The chunk type and format IDs below are proposed allocations, not registered assignments. The common structure, POSIX.1e, extended NFSv4 family, and Darwin layouts record the current design decisions. The Windows NT payload is incomplete; the Darwin layout and other draft formats require further validation and must not be treated as finalized wire formats. Open questions and validation work are listed separately at the end.

This draft specifies the on-wire representation and its meaning, not how an implementation obtains ACLs, resolves principals, converts security models, or restores metadata on a filesystem.

## 1. `aCLS` Access control list

The `aCLS` chunk records access control metadata associated with an entry. It is public, ancillary, and unsafe to copy (see [Chunk naming conventions](../file_structure/index.md#36-chunk-naming-conventions)). Its data is not required to decode the entry contents.

| Field | Size | Description |
|:------|-----:|:------------|
| Format ID | 2 bytes | Unsigned ACL format identifier |
| Payload | n bytes | Representation defined by the Format ID |

The enclosing PNA chunk length determines the payload length; there is no additional shared length, version, or flags field. The Format ID uses network byte order. A model-specific payload follows its own specified byte order; in particular, a referenced external binary representation is not implicitly byte-swapped to network byte order.

### 1.1. Occurrence and placement

- `aCLS` is optional and may appear at most once in an entry.
- It may appear anywhere between `FHED` and `FEND`, with no ordering dependency on other ancillary chunks. Existing critical-chunk ordering constraints, including consecutive `FDAT` chunks, still apply.
- The same rules apply to the entries embedded in a solid stream (`SDAT`). `aCLS` is not metadata of the enclosing solid entry itself.
- The common format does not restrict `FHED.Entry kind`. Individual ACL formats may have semantic limitations. For a symbolic link, the ACL describes the link itself, not its target. How to reconcile ACLs for entries sharing one underlying filesystem object is outside the common binary layout; see open questions.
- Absence of `aCLS` means no ACL is recorded by this chunk, not that the source object's ACL was empty or nonexistent.

Proposed addition to the summary of standard chunks (section 4.3):

| Name | Multiple in Archive | Multiple in Entry | Optional | Ordering constraints |
|:----:|:-------------------:|:-----------------:|:--------:|:---------------------|
| aCLS | Yes | No | Yes | Between `FHED` and `FEND` |

### 1.2. Format identifiers

A Format ID identifies both an ACL model and an encoding of that model. An incompatible encoding revision requires a different public Format ID; an existing public ID must not be redefined.

| Value | Format | Draft status |
|:------|:-------|:-------------|
| `0x0000` | Reserved (invalid Format ID) | Agreed |
| `0x0001` | POSIX.1e ACL v1 | Layout agreed |
| `0x0002` | PNA Extended NFSv4 ACL v1 | Layout agreed; mapping requires validation |
| `0x0003` | Windows NT ACL v1 | Proposed allocation; payload incomplete |
| `0x0004` | Darwin ACL v1 | Layout agreed; field details and mapping require validation |
| `0x0005` | PNA Extended NFSv4.1 ACL v1 | Layout agreed; mapping requires validation |
| `0x0006`–`0x7FFF` | Reserved for future public formats | Proposed range |
| `0x8000`–`0xFFFF` | Private formats | Agreed approach |

Private format meanings are defined by agreement among participating implementations. Private values are not globally unique; the specification does not establish a collision-avoidance mechanism. An implementation must not infer a private format's meaning merely from an unfamiliar identifier.

### 1.3. Unknown and malformed data

An `aCLS` chunk shorter than two bytes cannot contain a Format ID. A reserved invalid identifier, a duplicate `aCLS`, or an invalid payload of a supported format is an ACL metadata error. Whether to continue decoding an entry follows the general ancillary-chunk error handling policy; such an error need not be fatal to extraction of entry contents.

An unknown nonzero Format ID does not by itself make an archive invalid. Its payload is opaque. Implementations may ignore it or preserve it without interpretation. Editors handling an unsupported format should treat it as an unrecognized unsafe-to-copy chunk under [Chunk Ordering Rules](../chunk_ordering_rules/index.md), including the restrictions on changes to critical chunks. Unknown ACL information must not be mistaken for an empty ACL.

A known format's payload is bounded by the enclosing chunk; the payload must not be parsed into another chunk. Counts, lengths, and arithmetic must be checked against available data. Implementation resource limits and error-reporting policy are distinct from the wire format.

### 1.4. Relationship to ownership and mode

The ACL describes rules for an entry; it does not redefine the entry's owner or owning group. Owner facets (`fUId`, `fGId`, `fONm`, `fGNm`, `fOSi`, `fGSi`) remain responsible for ownership metadata. Special principals that denote an object's owner or owning group are relative references to that object. Owner facets need not be present for the ACL payload to be structurally valid. Named principals within an ACL are independent of the entry's ownership facets.

Where an ACL format describes ordinary access permissions, the ACL is authoritative for those permissions when it is interpreted. The special mode bits (set-user-ID, set-group-ID and sticky) in `fMOd` remain independent information. An audit-only component does not replace ordinary access permissions. Model-specific details are given below.

An inconsistency between independent ancillary chunks does not make the ACL encoding malformed. An editor that does not understand an ACL may legally modify another ancillary chunk without updating the ACL; the format must remain interpretable afterward. Diagnostics, strict restoration, and best-effort behavior are implementation policy.

## 2. POSIX.1e ACL v1 (`0x0001`)

The payload represents the POSIX.1e access ACL and optional default ACL. All integers are unsigned and encoded in network byte order. Fields are packed without alignment padding.

| Field | Size | Description |
|:------|-----:|:------------|
| Access entry count | 4 bytes | Number of ACEs in the access ACL |
| Access entries | variable | That number of POSIX ACEs |
| Default entry count | 4 bytes | Number of ACEs in the default ACL |
| Default entries | variable | That number of POSIX ACEs |

The access ACL contains at least three ACEs. A zero default entry count denotes an absent default ACL; a nonzero count denotes a complete default ACL. Default ACLs have directory-specific semantics. No count is a marker for failed acquisition. The payload contains no trailing fields or bytes.

### 2.1. POSIX ACE

| Field | Size | Description |
|:------|-----:|:------------|
| Tag | 1 byte | Entry type |
| Permissions | 1 byte | POSIX read/write/execute bits |
| Qualifier | 8 bytes, conditional | UID or GID |
| Name length | 2 bytes, conditional | Length of Name in bytes |
| Name | n bytes, conditional | UTF-8 name |

The Qualifier, Name length, and Name fields are present only for `USER` and `GROUP` tags. No placeholder qualifier or name fields occur for other tags.

| Tag value | Meaning | Additional fields |
|:---------:|:--------|:------------------|
| 0 | `USER_OBJ` | None |
| 1 | `USER` | UID, name length, name |
| 2 | `GROUP_OBJ` | None |
| 3 | `GROUP` | GID, name length, name |
| 4 | `MASK` | None |
| 5 | `OTHER` | None |

The Permissions byte uses bit 2 for read, bit 1 for write, and bit 0 for execute/search. Bits 3–7 are reserved and must be zero in this format. Tags other than 0–5 and nonzero reserved bits do not have defined POSIX.1e v1 representations.

The qualifier is a numeric UID for `USER` or numeric GID for `GROUP`. The name, when present, is an alternative representation of the same principal. Its length counts UTF-8 bytes, not characters; a length of zero denotes no name. No Unicode normalization form is required. Invalid UTF-8 is not a valid Name field. No name-to-ID lookup, priority, or identity mapping method is specified. The numeric qualifier is mandatory for these tags; zero is an ordinary numeric value, not a missing-value sentinel.

`USER_OBJ` and `GROUP_OBJ` refer to the object's owner and owning group, respectively, without embedding their identifiers.

### 2.2. ACL validity and interpretation

Each access ACL and each nonempty default ACL is interpreted independently under POSIX.1e ACL semantics. Each contains exactly one `USER_OBJ`, `GROUP_OBJ`, and `OTHER` ACE; at most one `MASK`; and no duplicate numeric qualifiers within the same named tag. A `MASK` is required if a `USER` or `GROUP` ACE is present. A mask without named ACEs is permitted. Numeric UID and GID are distinct identifier classes. ACE order is not significant; decoders must not require a canonical order.

A default ACL describes inheritance for newly created objects, not ordinary access to the containing directory. A nonempty default ACL on a non-directory has no valid POSIX default ACL semantics; the exact treatment by an archive editor or extractor is a separate concern (see open questions).

### 2.3. Relationship to `fMOd`

For this format, the access ACL is authoritative for the nine ordinary POSIX permission bits if both representations are interpreted. Owner bits correspond to `USER_OBJ`; group-class bits correspond to `MASK` when present, otherwise `GROUP_OBJ`; other bits correspond to `OTHER`. The special mode bits are not encoded in the ACL. Disagreement with the `fMOd` ordinary bits does not invalidate the wire representation of either chunk.

### 2.4. Encoder recommendations (non-normative)

An encoder may omit a redundant ACL where the basic three-entry access ACL is represented entirely by `fMOd` and no default ACL exists. This is not a condition of file-format validity. The encoder's selection of which metadata to record is outside the binary contract. To preserve additional ACL semantics, it is recommended not to discard explicit masks, named entries, or a default ACL when recording ACL metadata.

When producing both representations together, it is recommended to emit consistent ordinary mode bits. A deterministic ACE order may be used: `USER_OBJ`, named `USER` by UID, `GROUP_OBJ`, named `GROUP` by GID, `MASK` if present, `OTHER`. This is not a decoder constraint. The method of acquiring names and IDs, and the restoration policy, are not specified.

## 3. PNA Extended NFSv4 ACL v1 (`0x0002`)

This format is based on the ACL model described in [RFC 8881, section 6](https://datatracker.ietf.org/doc/html/rfc8881#section-6) and its NFSv4 predecessor [RFC 7530](https://datatracker.ietf.org/doc/html/rfc7530). It uses [RFC 4506](https://www.rfc-editor.org/rfc/rfc4506.html) XDR, with the PNA-specific principal representation and inherited-ACE extension defined below. It is not byte-compatible with the standard NFS ACL attribute.

The payload consists of a variable-length array of PNA NFS ACEs. The unsigned 32-bit count precedes the array. XDR alignment is relative to the beginning of the payload immediately after the two-byte Format ID, with no padding between the ID and the payload. XDR lengths are measured in bytes for variable-length opaque UTF-8 fields; XDR padding bytes are zero. All XDR integers use network byte order. The payload is fully consumed by the defined representation.

### 3.1. Common ACE and principal representation

The following XDR-style declarations define the PNA-specific wire structure. Type, flag and access-mask values refer to the corresponding NFSv4 ACE fields in RFC 8881 §6.2.1. The UTF-8 field below is an XDR variable-length opaque byte sequence interpreted as UTF-8 (corresponding to `utf8str_mixed`).

```c
typedef opaque pna_utf8<>;

struct pna_nfsprincipal4 {
    unsigned hyper *numeric_id;
    pna_utf8 name;
};

union pna_nfswho4 switch (unsigned int kind) {
case 0:
    pna_utf8 special;
case 1:
    pna_nfsprincipal4 principal;
};

struct pna_nfsace4 {
    unsigned int type;
    unsigned int flag;
    unsigned int access_mask;
    pna_nfswho4 who;
};

typedef pna_nfsace4 pna_nfsacl4<>;
```

`numeric_id` uses XDR optional-data encoding (a 32-bit boolean discriminator, followed by a 64-bit unsigned integer only if present). The name field is always encoded; a zero-length name means no name. A principal must have at least one of numeric ID and nonempty name. A zero numeric ID is not a missing-value sentinel. No domain qualification or resolution order is required; when both facets are present they describe one principal. `ACE4_IDENTIFIER_GROUP` distinguishes named users from named groups. The two union arms distinguish reserved special principals from ordinary principal names.

The special arm represents special identifiers from RFC 8881 §6.2.1.5, including `OWNER@`, `GROUP@`, `EVERYONE@`, `INTERACTIVE@`, `NETWORK@`, `DIALUP@`, `BATCH@`, `ANONYMOUS@`, `AUTHENTICATED@`, and `SERVICE@`. `ACE4_IDENTIFIER_GROUP` has no meaning for special principals. Owner/group special principals do not duplicate the entry's owner facets. `EVERYONE@` is not the POSIX `OTHER` principal.

### 3.2. ACE semantics

| Type | Value |
|:-----|:------|
| ALLOW | 0 |
| DENY | 1 |
| AUDIT | 2 |
| ALARM | 3 |

The access mask and flag assignments are those of RFC 8881 §6.2.1.3 and §6.2.1.4. Unlike the RFC's legacy `acl` attribute, PNA permits `ACE4_INHERITED_ACE` (`0x00000080`) in this format, retaining its meaning as an inherited ACE. This exception permits recording local NFSv4-style ACLs that expose that information. It does not change the NFS protocol.

ACE order is significant and is part of the represented ACL. Repeated principals and ACEs are allowed. An array of zero ACEs denotes an explicitly recorded empty ACL, not an absent ACL.

Unknown mask and flag bits remain representable at the binary level; this draft assigns no meaning to them. They must not be silently interpreted as defined permissions. The handling of unknown ACE types and unrecognized special identifiers is an open semantic question; no additional wire fields are implied. Whether a particular operating system or server accepts any ACL is not a binary validity condition.

## 4. PNA Extended NFSv4.1 ACL v1 (`0x0005`)

This format represents the NFSv4.1 DACL and/or SACL with their independent ACL-wide flags, using the same ACE extension and XDR rules as section 3. It is based on RFC 8881 §6.2.2, §6.2.3 and §6.4.3.2; it is not byte-compatible with their standard attribute encodings.

| Field | Size | Description |
|:------|-----:|:------------|
| Presence | 4 bytes | Component presence bitmask |
| DACL | variable, conditional | Encoded `pna_nfsacl41` if bit 0 is set |
| SACL | variable, conditional | Encoded `pna_nfsacl41` if bit 1 is set |

When both components are present, DACL precedes SACL. No additional per-component boolean or length is stored. A component that is not present occupies zero bytes.

| Presence bit | Meaning |
|:-------------|:--------|
| 0 | DACL recorded |
| 1 | SACL recorded |
| 2–31 | Reserved |

The proposed validity rule is that reserved bits are zero and at least one component is present (final confirmation pending). A component that is not recorded is distinct from a recorded component with zero ACEs.

```c
struct pna_nfsacl41 {
    unsigned int flags;
    pna_nfsace4 aces<>;
};
```

The XDR `aces` array has a 32-bit count followed by that number of ACEs. The ACL-wide flag values are defined by RFC 8881 §6.4.3.2:

| Flag | Value |
|:-----|:------|
| `ACL4_AUTO_INHERIT` | `0x00000001` |
| `ACL4_PROTECTED` | `0x00000002` |
| `ACL4_DEFAULTED` | `0x00000004` |

DACL contains ALLOW/DENY ACEs; SACL contains AUDIT/ALARM ACEs. Their sequence and flags are independent. As in section 3, unknown bits have no assigned interpretation in this draft. The entire payload is consumed by the declared components.

### 4.1. Relationship to `fMOd` (both NFSv4 formats)

Ordinary access permissions are described by the combined NFSv4 ACL or by the NFSv4.1 DACL when present. A SACL alone describes auditing/alarm rules and does not supersede ordinary permissions. The special POSIX mode bits remain separate `fMOd` information. No POSIX.1e-specific one-to-one relationship between ACEs and ordinary mode bits is imposed on either NFSv4 format. Inconsistency is not a structural violation of the ACL payload. See RFC 8881 §6.3–6.4 for the model's mode interaction.

## 5. Windows NT ACL v1 (`0x0003`) — incomplete

The agreed direction is to use the Microsoft self-relative security descriptor as the basis for the payload, with owner/group identities stored separately in existing PNA owner facets. Owner and Group SID offsets can be zero in the external representation; the Windows API permits selecting security information components independently. See [MS-DTYP Security Descriptor](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/7d4dac05-9cef-4563-a058-f108abecce1d) and [GetSecurityInfo](https://learn.microsoft.com/en-us/windows/win32/api/aclapi/nf-aclapi-getsecurityinfo).

The design must distinguish unrecorded DACL/SACL information from recorded absent, NULL, or empty ACL states, and retain relevant descriptor control flags and ordered ACE data. Retaining the descriptor's native little-endian encoding is the proposed approach. The current candidate layout is a component-capture indicator followed by the descriptor bytes; the indicator's width and bit allocations are not finalized. SACL subcomponents, capture scope, descriptor normalization, and control-flag coverage remain open. Owner/group fields are planned to be omitted from the stored descriptor; this is not a claim to preserve an entire original security descriptor byte for byte. The format must not yet be emitted as an interoperable public encoding.

## 6. Darwin ACL v1 (`0x0004`)

This format extends the Darwin `kauth_acl` and `kauth_ace` representation with additional principal identity facets. It is not the complete native `kauth_filesec` blob. In particular, it does not include the native magic number or the file's owner and group UUIDs. Owner information remains separate; dedicated owner/group UUID chunks are a separate design topic.

### 6.1. Payload

All multibyte integers use network byte order. Fields are serialized consecutively without implicit alignment or padding.

| Field | Type | Description |
|:------|:-----|:------------|
| ace_count | u32 | Number of ACEs, or `0xFFFFFFFF` (no ACL) |
| acl_flags | u32 | ACL-wide flags |
| aces | DarwinAce[] | Ordered ACE records; omitted when ace_count is `0xFFFFFFFF` |

The representation distinguishes three cases:

- Absence of the `aCLS` chunk: ACL information was not recorded.
- `ace_count = 0xFFFFFFFF`: absence of an ACL is explicitly recorded.
- Any other count: an ACL is recorded, including an empty ACL when the count is zero.

The sentinel is based on the Darwin `KAUTH_FILESEC_NOACL` constant. For the draft, the fixed header is retained even when the sentinel is used; the ACL flags carry no defined ACL semantics in that case. The exact sentinel/flags validity rule remains to be confirmed. The sentinel is never interpreted as an array length. A known payload must contain exactly the records indicated by its count, with no trailing data.

### 6.2. ACE representation

| Field | Type | Description |
|:------|:-----|:------------|
| uuid | 16 bytes | Principal UUID in native external byte order |
| flags | u32 | ACE kind and flags |
| access_mask | u32 | Darwin vnode access rights |
| principal_kind | u8 | Principal kind |
| id_present | u8 | Numeric ID presence indicator |
| numeric_id | u64 (conditional) | Numeric UID or GID, present only when indicated |
| name_length | u16 | Length in bytes of the optional name |
| name | name_length bytes | UTF-8 name; zero length denotes no name |

The leading UUID, flags, and access mask correspond to the native external ACE representation. The remaining fields are PNA extensions. ACE order is meaningful and is retained.

Principal kinds:

| Value | Meaning |
|:------|:--------|
| 0 | Unknown |
| 1 | User |
| 2 | Group |

The proposed encoding for `id_present` is 0 (absent) or 1 (present). The UUID is required; the numeric ID and name are optional. A numeric ID for a known user or group is a UID or GID, respectively. The validity and interpretation of a numeric ID with unknown principal kind remain to be decided. Other principal kinds and presence values are not assigned in this draft.

The identity facets describe the same principal. Identity resolution, lookup order, and the choice of facets used when restoring the ACL are not specified. Nonempty names must be valid UTF-8; no normalization is prescribed.

### 6.3. Flags and access rights

The draft uses the bit assignments of Darwin's `kauth_ace` and `kauth_acl` rather than translating them to NFSv4 or POSIX.1e values. The following assignments are documented by XNU.

| ACE flags bits | Meaning |
|:---------------|:--------|
| 0–3 | ACE kind: 1 permit, 2 deny; 3 audit and 4 alarm are defined but not implemented by the referenced XNU interface |
| 4 | Inherited |
| 5 | File inheritance |
| 6 | Directory inheritance |
| 7 | Limited inheritance |
| 8 | Inherit only |
| 9–10 | Audit success/failure (defined but not implemented by the referenced XNU interface) |

| ACL flags bits | Meaning |
|:---------------|:--------|
| 0–15 | Filesystem-private flags |
| 16 | Deferred inheritance |
| 17 | No inheritance |

The access mask follows the Darwin vnode ACL rights assignments. All three flag/mask fields are represented at their full 32-bit width. Filesystem-private bits can be retained without assigning portable meanings to them. The exact version-fixed rights table, reserved-bit semantics, and validity of unknown ACE kinds are pending; the reference to XNU does not automatically assign meaning to future XNU additions.

No platform-specific maximum ACE count is imposed by the draft wire layout beyond the enclosing chunk size and sentinel reservation.

### 6.4. Relation to mode and owner information

Darwin ACLs and POSIX mode bits can participate in the same authorization decision. The ACL does not replace the nine permission bits recorded by `fMOd` in the manner of a POSIX.1e access ACL. Both are independently meaningful. No requirement to derive or validate one from the other is introduced.

File owner and group UUIDs are distinct from ACE principal UUIDs and are outside this payload. Existing owner information chunks remain independent of `aCLS`.
## 7. General implementation recommendations (non-normative)

These recommendations are not additional wire-format validity constraints.

- Preserve the recorded model and meaningful ACL information when practicable. Do not mistake representation, acquisition completeness, and successful application for the same guarantee. Equivalent effective permissions in a different environment cannot be guaranteed merely by copying the representation.
- Identity lookup (including numeric versus name-based lookup), cross-model conversion, ACL acquisition, and filesystem application are implementation concerns. A model-specific storage structure should not force a shared principal-resolution policy.
- Avoid silently treating unrecorded, unsupported, or incomplete ACL information as an empty ACL. Account for inherited permissions, ownership changes, and permission-setting operations that can change an ACL. Report loss of semantics according to the application's chosen policy.
- Existing security considerations for unauthenticated ancillary metadata apply, including [§12.4.1](../recommendations_for_decoders/index.md#1241-unauthenticated-metadata). In per-entry mode, ACL metadata is not encrypted or authenticated merely because file contents are encrypted; authenticated encryption of the enclosing solid stream can protect embedded metadata.
- Legacy CLI private chunks `faCl` and `faCe` are not standardized by this draft. Migration or read compatibility is an implementation matter; information already lost in an old encoding cannot be reconstructed by conversion.

## 8. Open issues and validation checklist (not normative)

The following work remains before freezing public format assignments or merging this specification:

- **Common/registry:** confirm the proposed public chunk name, Format ID allocation, precise treatment of duplicate chunks and unknown supported-format values, and editor semantics for unknown private IDs. Verify final terminology against the existing chunk rules.
- **POSIX.1e:** confirm directory-specific default-ACL validity versus contextual handling; verify byte-level test vectors, qualifier widths and all structural invariants against native APIs. Verify mode/ACL round-trip without promoting encoder recommendations into validation requirements.
- **NFSv4 family:** validate that the local FreeBSD/UFS/OpenZFS ACL APIs map without semantic loss to the extended representation, including inherited flags, identity facets, and permissions. Confirm behavior of unknown ACE types/special identifiers and reserved presence bits. Recheck all referenced RFC clauses and extensions; the PNA representation intentionally differs from standard NFS wire attributes. Validate empty ACL and all component-presence combinations. Do not infer that a particular native API captures DACL/SACL just because the model can represent them.
- **Windows NT:** independently verify owner/group-free self-relative descriptors with the documented APIs, descriptor validation, round-trip and applicable control flags; specify the capture indicator, SACL/LABEL and other security-information coverage, partial acquisition, absent vs NULL vs empty, and restoration of inherited information. Owner/group defaulted flags and the limits of the existing owner facets need explicit decisions. Do not claim a successful native round-trip until tested.
- **Darwin:** finalize the proposed presence-indicator values, numeric IDs with unknown principal kind, and flags in the no-ACL sentinel case. Freeze the native rights/flag values and unknown-kind/bit rules; validate UUID facets, empty vs absent vs unavailable ACLs, filesystem-private flags, native ACL APIs, inheritance metadata, and round-trip fixtures. Dedicated owner/group UUID chunks are deliberately deferred to a separate design.
- **Entry kinds and overlapping metadata:** clarify conflicting ACLs on hard-link entries, directory-target links, semantics of `FHED.Entry kind = 4`, symlink restoration, and conflicts with ACL information in `xATR`, AppleDouble or legacy private chunks. Do not add an implicit dependency on other ancillary chunks.
- **Validation and integration:** add binary fixtures, malformed/truncated input and oversized count tests, unknown-format preservation and editor tests, native round-trip tests, and integration with the specification's section 4.3 table, table of contents, encoder/decoder recommendations, and references. The Windows format remains incomplete and the Darwin layout requires further validation; no code or tests have been executed by this draft.

## References

- [PNA §3.5–3.6, File structure](../file_structure/index.md)
- [PNA §4, Chunk specifications](../chunk_specifications/index.md)
- [PNA §9, Chunk ordering rules](../chunk_ordering_rules/index.md)
- [PNA §11–12, Encoder and decoder recommendations](../recommendations_for_encoders/index.md)
- [Linux acl(5), POSIX access control lists](https://man7.org/linux/man-pages/man5/acl.5.html) (informative)
- [RFC 4506, XDR: External Data Representation Standard](https://www.rfc-editor.org/rfc/rfc4506.html)
- [RFC 7530, Network File System (NFS) Version 4 Protocol](https://www.rfc-editor.org/rfc/rfc7530.html)
- [RFC 8881, Network File System (NFS) Version 4 Minor Version 1 Protocol](https://datatracker.ietf.org/doc/html/rfc8881)
- [Microsoft MS-DTYP, SECURITY_DESCRIPTOR](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/7d4dac05-9cef-4563-a058-f108abecce1d)
- [Microsoft GetSecurityInfo](https://learn.microsoft.com/en-us/windows/win32/api/aclapi/nf-aclapi-getsecurityinfo), [SetSecurityInfo](https://learn.microsoft.com/en-us/windows/win32/api/aclapi/nf-aclapi-setsecurityinfo)

- [Apple XNU, kauth.h](https://github.com/apple-oss-distributions/xnu/blob/main/bsd/sys/kauth.h) (informative; version-fixed mapping pending)
- [Apple Libc, acl_translate.c](https://github.com/apple-oss-distributions/Libc/blob/main/posix1e/acl_translate.c) (informative)
