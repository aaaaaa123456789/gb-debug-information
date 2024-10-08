# Game Boy ROM debug information format specification — version history

This document is a companion for the [Game Boy ROM debug information format specification](debuginfo.md), listing all
changes since its first published version.
Each version also lists the repository tag where it can be found.

#### Copyright

This document, being a companion to the main specification, is also released to the public domain under the
[Creative Commons Zero][cc0] dedication.
No copyright is claimed on this document; attribution is appreciated.

[cc0]: https://creativecommons.org/publicdomain/zero/1.0/legalcode

* * *

### Version 0.2.1 (8 October 2024)

Repository tag: [v0.2.1](https://github.com/aaaaaa123456789/gb-debug-information/blob/v0.2.1/debuginfo.md)

* Section "2. Introduction": improved the wording to clarify intent.
* Section "3.4. Header": noted that the declared header size cannot be larger than the size of the file.
* Section "4. Standard block types":
    * Updated the minimum element sizes in accordance with changes to the corresponding sections.
    * Added new block type to the table.
* Section "4.5. Source file table block type":
    * Noted that paths should be relative only if possible.
    * Added an exception for source files that are not written by the user.
* Section "4.6. ROM image information block type":
    * Defined the correct handling for ROM images whose checksum (in the ROM header) doesn't match the data.
    * Updated the definitions of the ROM checksum and ROM checksum invalid flag fields to match this change.
    * Clarified the endianness of the ROM checksum field.
    * Required the build timestamp field to ignore leap seconds.
    * Noted that the invalid value is not a real build timestamp.
    * Reinforced the requirement for the game title field to point to a valid string (i.e., no invalid UTF-8
      sequences), noting the correct handling for ROM images that contain such sequences (as well as invalid or
      control characters) in the game title in the ROM header.
* Section "4.8. Section table block type": reworded introduction to account for unnamed sections.
* Section "4.9. Symbol table block type": added a new field to indicate the parent of a symbol.
* Section "4.13. Checksum table block type": added a new field to support multiple checksums for a file.
* Section "4.15. Memory map block type": created new section for new block type.
* Section "A.5. Checksum samples": created new section.
* Minor editorial changes.

### Version 0.2.0 (8 October 2023)

Repository tag: [v0.2.0](https://github.com/aaaaaa123456789/gb-debug-information/blob/v0.2.0/debuginfo.md)

* Section "2.1. Conventions":
    * Defined the numbering of bit positions for bit-packed fields that span multiple bytes.
    * Noted that names of components are English phrases, not valid identifiers in any programming language.
* Section "3.2. Definitions":
    * Clarified the constraint requiring valid UTF-8.
    * Added an explicit reference to the UTF-8 specification.
    * Required strings to end in a null terminator ($00 byte).
    * Explicitly required strings using different character sequences to represent the same glyphs to be considered as
      different.
    * Added definitions for ROM image, program image and overlay file.
* Section "3.4. Header":
    * Removed the references to a fixed 32-byte size in the section's introduction, given that the header size field
      can contain a value larger than 32.
    * Capitalized the keyword MUST in the requirement referring to the block type of the extension block descriptor
      block.
* Section "3.5. Master block table": noted that the table parameters (location, entry size, entry count) are defined
  in the header.
* Section "3.6. Extension blocks": strengthened the recommendation for consumers to ignore extension blocks they don't
  process from a MAY to a SHOULD.
* Section "4.2. String table block type":
    * Reworded the explanation regarding other blocks sharing a string table.
    * Required strings to end in a null terminator ($00 byte).
* Section "4.6. ROM image information block type":
    * Required the file to only have one block of this type.
    * Noted that the filename cannot contain forward slashes.
    * Allowed the file size field to contain the size before padding.
    * Required the other checksum field to be set to the invalid index if the size field is set to zero.
    * Explained the relationship between the value in the manufacturer code field and the actual manufacturer code
      rendered as ASCII characters.
    * Allowed the game title field to strip trailing spaces from the title in the ROM image header.
    * Added the overlay ROM image filename and overlay ROM image checksum fields.
* Section "4.8. Section table block type":
    * Noted that overlaid sections can share a name.
    * Required sections without a type to have their section type string field set to the invalid string index.
    * Lessened the requirement for unbanked sections to have their bank set to zero from a MUST to a SHOULD.
    * Expanded the alignment requirement subfield to 5 bits.
    * Added the additional constraints flag and bank placement requirement subfields.
    * Rearranged the flags field, extending it to two bytes (taking up the formerly reserved field next to it), for a
      more optimal placement of the expanded and new subfields.
* Section "4.9. Symbol table block type":
    * Clarified the interaction between the address flag and the banked symbol flag.
    * Noted that the value and bank fields can form a single 4-byte field.
    * Lessened the requirement for unbanked symbols to have their bank set to zero from a MUST to a SHOULD, removing
      the remark about the bank field being meaningless in that case.
* Section "4.10. Line-to-address mapping table block type": lessened the requirement for entries referencing unbanked
  memory to have their bank set to zero from a MUST to a SHOULD.
* Section "4.11. Address-to-line mapping table block type":
    * Required blocks to not be empty.
    * Lessened the requirement for blocks mapping an unbanked region of memory to have the upper 16 bits of their
      reference field (representing the bank) set to zero from a MUST to a SHOULD.
    * Required blocks mapping unbanked regions of memory to map only one such region.
    * Added starting column and ending column fields.
* Section "4.12. Preferred address-to-line mappings block type": lessened the requirement for blocks mapping an
  unbanked region of memory to have the upper 16 bits of their reference field (representing the bank) set to zero
  from a MUST to a SHOULD.
* Section "4.13. Checksum table block type": added provisions for unused table entries.
* Section "4.14. Unused memory area table block type": created the section.
* Minor editorial changes.

### Version 0.1.0 (24 September 2023)

Repository tag: [v0.1.0](https://github.com/aaaaaa123456789/gb-debug-information/blob/v0.1.0/debuginfo.md)

* Initial draft version.
