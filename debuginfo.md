# Game Boy ROM debug information format specification

Version 0.1.0 — 24 September 2023

#### Copyright

This document is released to the public domain under the [Creative Commons Zero][cc0] dedication.
No copyright is claimed on this document; attribution is appreciated.
The latest version of this document lives in [the `gb-debug-information` repository][repo].

[cc0]: https://creativecommons.org/publicdomain/zero/1.0/legalcode
[repo]: https://github.com/aaaaaa123456789/gb-debug-information

* * *

## 1. Objective and scope

The present document describes a file format that can contain debugging information for a Game Boy or Game Boy Color
program, such as file and line number information, intended for disassemblers, interactive debuggers and similar
developer tools to aid in analyzing or displaying the contents of the ROM image or the executing program image.
The debug information file described by this document is intended to accompany the ROM image file, not to contain it
or supplant it; it is not designed to be meaningful on its own, but rather to augment the files it references, such as
the ROM image file in question or the source code files that were used to build it.

## 2. Introduction

Analyzing a ROM image is not a trivial process, even for a simple system like the Game Boy.
Analysis is needed for many purposes: for instance, an interactive debugger needs to analyze the program to know how
to display some part of it (e.g., as code or as data).
But this analysis is inherently inaccurate and lossy: there is no way to know exactly how the developer entered a
specific line of source code, and thus any attempt at rendering that line is a best guess that is often far apart from
the actual source text.
This issue is compounded when the source code is not in assembly, but in a higher-level language such as C:
recovering the original source code would require automated decompilation, and this is infeasible at the level of
reconstruction generally desired by developers (for example, comments and variable names cannot be reconstructed at
all).

Therefore, a debug information file can contain references to source files, symbol tables, and other useful
information that an analyzer (such as an interactive debugger) may need to properly process, analyze and display the
program code and data.
By far and large, the goal of such a file is not to contain, but to reference: a debug information file doesn't
actually hold the program's source code or ROM image, but it links one to the other, establishing the necessary cross
references so that a tool with access to both can properly connect them.

At the time of writing, the landscape is dominated by two languages (assembly and C), a handful of toolchains, and
about as many emulators with advanced enough debugging capabilities as to make use of debugging information of the
kind described by this document.
However, it is not the goal of this document to dictate a choice of language, toolchain or emulator.
Rather, the purpose of this document is to describe a format that can integrate into both existing and new tools, so
as to improve the debugging experience of developers across the board.
Therefore, the format described here is deliberately extensible and as agnostic about the choice of development and
debugging environment as realistically possible; this document thus avoids using specific programming languages for
the description of the format's contents, opting instead for structure and data tables that should be implementable in
most languages used by toolchains and emulators.

The format is designed for ease of consumption over ease of generation.
A consequence of this is that some data in the file may be redundant or harder to generate; the purpose is to speed up
access to the data in the file, perhaps at the cost of some extra build time.
It is the responsibility of the generator of the file (typically the build toolchain) to ensure that all data in the
file is consistent; inconsistencies in the data can result in an incorrect analysis by the consumer of the debugging
information.
However, this means that, in turn, consumers must expect debugging information that may not be fully consistent: for
example, an address-to-line mapping might not exactly match the corresponding line-to-address mapping.
Requiring this data to be fully consistent would be prohibitive in certain edge cases, and checking that consistency
would make tools unacceptably slow (due to the non-polynomial worst-case performance of the checking algorithms
involved); instead, tools (such as debuggers) consuming the data should be prepared to handle this kind of mismatch.

### 2.1. Conventions

These conventions are followed throughout the document for consistency:

* Numbers prefixed with a $ sign represent hexadecimal values, and those without are decimal.
  For example, $100 and 256 represent the same number.
* Numbers, even hexadecimal ones, are non-negative unless explicitly preceded by a minus sign.
  For example, $FFFF (even as a 2-byte integer) represents the value 65,535, not -1.
* Numerical comparisons like "less than", "smaller than" or "greater than" are always strict: if two values are equal,
  neither of them is considered to be less than (or greater than) the other.
  On the other hand, the opposites of those comparisons ("not less than", "not greater than", etc.), being boolean
  negations of the ones listed above, do accept equal values.
* Structures never contain any implicit padding.
  If a structure needs any padding (for example, for alignment), that padding is shown explicitly as reserved fields.
* Fields in a structure are shown with a running offset for clarity.
  The offset of the first field is always zero, and the offset of subsequent fields is the offset of the previous
  field plus its size.
  Bit-packed fields are shown in a similar layout, indicating the starting bit position for each subfield and the
  number of bits that make up the subfield; in this case, bit 0 is always the least significant bit.
* Single-bit subfields in bit-packed fields are often referred to as "flags"; in this case, it is often stated that
  the flag is "cleared" or "set".
  A flag (i.e., a single-bit value) is set if its value is 1, and it is cleared if its value is 0.
* The document does not establish what kind of tool can generate or process a debug information file.
  In the interest of keeping the specification as generic as possible, the process or program that generates a debug
  information file is called the "generator", and the process or program that takes such a file and processes it in
  any manner is called the "consumer".
* The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT
  RECOMMENDED", "MAY" and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119][rfc2119] when,
  and only when, they appear in all capitals, as shown here.

[rfc2119]: https://www.rfc-editor.org/rfc/rfc2119

## 3. Definitions and components

This section describes the overall structure of the debug information file format and the basic components it uses.

### 3.1. Format overview

A debug information file is made up of three components: a [header][sect3.4], a [master block table][sect3.5], and a
number of [blocks][sect3.3] containing the actual debugging information.
Blocks can be standard blocks or extension blocks; [extension blocks][sect3.6] are implementation-defined blocks that
are identified by a pair of strings and can contain any information the implementation in question wants or needs.
This document only defines the contents of standard blocks; implementations that want to define extension blocks MUST
document the purpose and contents of said blocks, as well as their [identification strings][sect3.6].

Each block has a type, which defines what data the block in question can contain.
The type of each block is given in the [master block table][sect3.5].
Some blocks contain raw data, but some others contain tables.
(A table is an array of elements, all the same size, laid out one after the other in the file.)
All tables are indexed starting from zero: the first entry in a table is entry number 0, the next one is entry number
1, and so on.
(This also applies to the master block table itself: the first block in the table is block number 0.)
If some data element in the file needs to refer to a special or null table entry, the [invalid index][def-invalid] is
used for that purpose, not zero.

### 3.2. Definitions

This section establishes definitions for some terms and concepts used throughout the document, as well as the
conditions and constraints to which those elements are subject.

#### Numbers

All numbers used in this file format are fixed-size integers, and they are always stored in little-endian form (that
is, least significant byte first).
Unless otherwise specified, all numeric fields are unsigned, and all values are treated as unsigned; if a numeric
field is signed, negative values are always stored in two's complement form.

Numbers are usually 1, 2, 4 or 8 bytes in size, and they are always aligned to their natural alignment (i.e., the
alignment that matches their size).
Some numbers can also appear in bit-packed fields; those numbers will always be smaller than 64 bits, and they will be
subject to the alignment constraints for their containing bit-packed field.

#### Invalid value

Whenever a numeric field (typically an index into some table) needs to be able to contain a special value designating
an invalid or null value, the maximal value for that numeric field is used as this invalid value.
This will be referred to throughout the document as the invalid index, invalid value, or any similar terms.
This special value has all bits set, and its numeric value is the maximum that can be contained by a field of its
size: for example, since block indexes are 4-byte values, the invalid block index is $FFFFFFFF.

Invalid values are also used wherever a null value would be needed.
(Therefore, **zero is not used as the null value**, allowing its use as the starting index of tables.)
For example, the [master block table][sect3.5] contains a linked block field for each block; if a block doesn't have
any linked block to reference, the linked block for that block is set to the invalid block index.
In order to avoid confusion, this value is never called a null value; the term "invalid" is used instead in all cases,
even when the true meaning of the value is to represent a null value.

#### Reserved fields

Reserved fields are fields of structures with no designated meaning in this specification, reserved as padding or for
future versions of the specification.
Reserved fields MUST be filled with $FF bytes when the debug information file is generated; reserved subfields in
bit-packed fields MUST be set to an all-bits-set value (i.e., the maximum value the subfield could have if it was
interpreted as a number).
This ensures that, if these reserved fields are eventually interpreted as numeric fields, they will have been set to
the [invalid value][def-invalid].

Consumers MUST treat reserved fields not set this way as an error, as those values may indicate that the value is
being used for some feature not known to the consumer.

Tables whose element size is declared to be greater than the size required by this specification MUST be treated as if
the elements were padded to size with reserved fields.
(For example, if this specification requires a table to have a minimum element size of 32, and the file declares an
element size of 40, the table elements MUST be handled as if an 8-byte reserved field was appended at the end of each
element.)
These reserved padding bytes MUST be handled as specified above.

#### Strings

A string is a sequence of characters of known length.
Strings MUST be encoded in [UTF-8][rfc3629] without any embedded null characters ($00 bytes) and without any
surrogates (codepoints $D800 through $DFFF).
The maximum length of a string is 65,535 bytes.

Strings are referenced through [string table blocks][sect4.2], which contain information about the length and location
of each string used by the file.
As the length of each string is explicitly encoded in the file, null terminators are not necessary; generators MAY
emit null terminator ($00) bytes after each string, but consumers MUST NOT expect to find these null terminator bytes.

Whenever strings are compared, two strings MUST be considered different if they don't have the exact same byte
representation (or equivalently, the same character codepoints).
In particular, strings that differ only in case MUST NOT be considered equal; all string comparisons are
case-sensitive.

[rfc3629]: https://www.rfc-editor.org/rfc/rfc3629

### 3.3. Blocks

Blocks are the main data unit of debug information files.
Blocks are listed in the [master block table][sect3.5]; consumers can determine which information is available to them
in a debug information file by scanning said table.
Unless a specific block requires another block to be present (for example, a [string table block][sect4.2] requires a
[data block][sect4.1] to contain the actual string data), there are no mandatory blocks: all of the information a
debug information file can contain is OPTIONAL.
A debug information file MAY contain any number of blocks (up to a maximum of $FFFFFFFF) or even no blocks at all,
although this latter case is not particularly useful.
Each block is identified by its block index, which is its index into the [master block table][sect3.5]; the first
block in the table has index 0.
The kind of information that a block contains is described by its [type][sect4], which is a 2-byte number that
determines the layout and semantics of the block.

Outside the [header][sect3.4] and the [master block table][sect3.5], all data in the debug information file MUST be
contained in blocks.
It is possible for blocks to not cover the entire file: there MAY be gaps between blocks, or at the end of the file.
However, these gaps MUST NOT contain any debug information; they MUST be ignored by any consumers.
Likewise, blocks, the header and the master block table SHOULD NOT overlap.

Blocks can contain raw data or a table.
A table is an array of elements, typically structures, which are all the same size; if a block contains a table, the
table will be the entire block, with no other data before or after it.
(However, other blocks can reference that block, and thus its table, augmenting it with ancillary data.)

Blocks MAY be of any size, subject to any constraints in their definitions as given in this document.
If a block contains raw data, the [master block table][sect3.5] will declare its size, up to $FFFFFFFF bytes.
On the other hand, if a block contains a table, the [master block table][sect3.5] will declare its element count (also
up to $FFFFFFFF elements); the size of the block is given by the product between that count and the element size.
In this case, the [master block table][sect3.5] will also declare the element size of each element in the table; this
element size MUST NOT be smaller than the value specified for that block type in this document, but it MAY be larger
(this is allowed for compatibility with future versions of the specification).
These additional bytes in each entry MUST be treated as [reserved fields][def-reserved].

Blocks MAY appear anywhere in the file; this allows generators to generate those blocks with any layout that is
convenient for them.
However, each block type has an alignment constraint that MUST be obeyed for blocks of that type.
The location and size of each block are declared in the [master block table][sect3.5]; both of them are subject to the
block's alignment constraint.
If the block contains a table, the element size MUST also follow the same alignment constraint; in that case, the
element count is unconstrained.
Alignment constraints are always 1, 2, 4 or 8 bytes.
(The master block table itself is also subject to these rules, and it has an alignment constraint of 8 bytes; the
location and size of the master block table are defined in the [header][sect3.4].)

### 3.4. Header

The header of a debug information file is a 32-byte structure that is located at the beginning of the file, which
allows consumers to locate and process the rest of the information contained in the file.
The fixed size of this header implies that a debug information file MUST be at least 32 bytes long.

The layout of the header is the following:

|Offset|Size|Description                                      |
|-----:|---:|:------------------------------------------------|
|     0|   8|Magic number                                     |
|     8|   4|Version information                              |
|    12|   2|Header size                                      |
|    14|   2|[Master block table][sect3.5] element size       |
|    16|   8|[Master block table][sect3.5] location           |
|    24|   4|[Master block table][sect3.5] size               |
|    28|   4|[Extension block descriptor][sect4.3] block index|

* **Magic number**: this field MUST contain the fixed value $49444247DB61000A.
  (As [all numbers in the file][def-numbers], this value is stored in little-endian byte order.)
  This value identifies the file as a GB/GBC debug information file.
* **Version information**: version number for the version of this specification to which the file conforms to.
  For more information about the layout and semantics of this field, see [the corresponding section][sect6.2].
* **Header size**: size of this header, in bytes.
  This field MUST be set to a multiple of 8 no less than 32; using a value greater than 32 for this field is NOT
  RECOMMENDED.
  If this field is set to a value greater than 32, additional bytes in the header MUST be treated as reserved fields
  and set to the [invalid value][def-invalid].
* **[Master block table][sect3.5] element size**: size of each element in the master block table; this value MUST be a
  multiple of 8 no less than than 24.
  Using a value greater than 24 for this field is NOT RECOMMENDED.
* **[Master block table][sect3.5] location**: byte offset of the master block table in the debug information file;
  this value MUST be a multiple of 8, and the end of the table as determined implicitly by this value (i.e., this
  value plus the product between the element size and the block count) MUST NOT go beyond the end of the file.
* **[Master block table][sect3.5] size**: number of entries in the [master block table][sect3.5].
  Unless the [master block table][sect3.5] contains unused entries (i.e., entries with their block type field set to
  the [invalid value][def-invalid]), this value will also be the number of blocks in the file.
* **[Extension block descriptor][sect4.3] block index**: block index for the block that defines the identification
  strings that fully identify the block type for each [extension block type][sect3.6] index used by the file.
  This block must have the [extension block descriptor][sect4.3] type.
  If the file doesn't use any extension blocks, this field MAY be set to the [invalid block index][def-invalid].

### 3.5. Master block table

The master block table describes every block in the debug information file.
Applications that intend to consume a debug information file MUST process this table to discover which information is
available to them; blocks MUST NOT be assumed to be in any particular order or layout in the file beyond what this
table describes.

The master block table has an alignment constraint of 8 and a minimum element size of 24.
This means that, in the [header][sect3.4], both the location and the element size for the master block table MUST be
set to multiples of 8, and the element size MUST be at least 24.
Using a value greater than 24 for the element size is NOT RECOMMENDED; if a greater value is used, additional bytes in
each entry MUST be treated as reserved fields and set to the [invalid value][def-invalid].

Master block table entries have the following layout:

|Offset|Size|Description        |
|-----:|---:|:------------------|
|     0|   2|[Block type][sect4]|
|     2|   2|Element size       |
|     4|   4|Block size         |
|     8|   8|Block location     |
|    16|   4|Linked block       |
|    20|   4|Reference          |

* **[Block type][sect4]**: number that identifies the kind of content the block has.
  If this number is smaller than $4000, the block is an [extension block][sect3.6]; in that case, the block type is an
  index into the [extension block descriptor][sect4.3], and the actual block type depends on the data in that block.
  Otherwise, the value MUST be one of the values listed in [the corresponding section][sect4] unless the entry is
  unused (i.e., it is a dummy entry that does not represent a block).
  If an entry in the master block table is unused (for example, because the block in question was deleted from the
  file, but the implementation doesn't want to renumber all the blocks in the file), the block type for that entry
  MUST be set to the [invalid value][def-invalid].
  In that case, the element size, size and location fields MUST be set to zero, and the remaining fields MUST be set
  to the [invalid value][def-invalid].
* **Element size**: size of each element in a block that contains a table.
  If the block in question doesn't contain a table, this value MUST be set to zero.
  On the other hand, if the block does contain a table, this value MUST be set to a size no smaller than the minimum
  element size indicated for the block type in question, and it MUST be a multiple of the block type's alignment
  constraint.
  (This field therefore can also be used to determine if a block contains a table or not.)
* **Block size**: size of the block, in bytes or in elements.
  If the block in question doesn't contain a table (i.e., the element size field is zero), this field specifies the
  size in bytes of the block, and it MUST be a multiple of the block type's alignment constraint.
  On the other hand, if the block does contain a table (i.e., the element size field is not zero), this field
  specifies the number of entries in the table (which MAY be zero); the actual size (in bytes) of the block in that
  case is the product of this field and the element size.
  (The block size for blocks that contain a table is unconstrained by alignment because the element size is already
  aligned.)
* **Block location**: byte offset of the block in the debug information file.
  This value indicates the position of the beginning of the block; the end of the block (as indicated by the value of
  this field and the size determined by the element size and size fields) MUST NOT go beyond the end of the file.
  The value of this field MUST be a multiple of the block type's alignment constraint.
* **Linked block**: index of a block that this block is linked to.
  Some blocks need to reference data from another block.
  For example, a [string table block][sect4.2] contains a table of string offsets and lengths, but it needs to
  reference the actual string contents from a [data block][sect4.1].
  This field contains the index of the block that the current block links to for this purpose; the meaning of this
  field is defined for each block type.
  If a block doesn't use the linked block field (either because it is meaningless for the its block type or because it
  chooses not to link to any other block), this field MUST be set to the [invalid block index][def-invalid].
  This field MUST NOT contain a nonexistent block index (other than the [invalid block index][def-invalid]).
* **Reference**: indication of which object or location the block is referring to.
  The effective meaning and semantics of this field are defined for each block type.
  Most blocks don't use this field at all; in that case, this field MUST be set to the [invalid value][def-invalid].
  (For example, an [address-to-line mapping table block][sect4.11] can use this field to indicate which addresses it
  maps.)
  Some blocks use this field to link another block; in that case, the behavior for this field is the same as for the
  linked block field.

### 3.6. Extension blocks

Extension blocks are blocks with implementation-defined semantics; they represent the specification's main extension
point.
By their nature, their semantics cannot be described by this document.
Therefore, this section aims to describe the framework under which these blocks can be defined, detected and used.

As with all other blocks, consumers MAY ignore any extension blocks they don't intend to process.
Therefore, extension blocks MUST NOT redefine the meaning of standard blocks in a way that is incompatible with their
standard definition; implementations that ignore unknown extension blocks should be able to process the blocks they do
know correctly.
Likewise, consumers SHOULD NOT discard a file entirely simply due to the presence of unknown extensions; it is instead
RECOMMENDED to process as many blocks as they can and ignore the rest.

Standard block types can be identified by a simple and small numeric identifier because they come out of a small list
that is fully defined by this document.
Extension block types, on the other hand, come from a potentially endless list that has no central registry.
Therefore, to reduce the risk of collisions, instead of using small integers as their identifiers, extension block
types use two strings: the originator string and the type string.
Together, these strings uniquely identify an extension block type; implementations SHOULD strive to choose these
strings (particularly the originator string) in a unique enough way so as to minimize the risk of collisions.
These strings MUST NOT be empty.

The originator string identifies the source of the extension block type.
This isn't necessarily the generator of the debug information file, as implementations MAY support each other's
extension block types.
Rather, it is a string that uniquely identifies the source that defines and documents the block type.
Originators SHOULD attempt to choose a string that will identify them in a way that is unlikely to be used by anyone
else; for example, any system of generating unique identifiers (such as URNs, URLs, GUIDs or domain names) can be used
for this purpose.

The type string is a string that identifies the extension block type itself amongst any other types with the same
originator string.
Since the originator string acts as a namespace, the type string is fully under the control of the originator, and
therefore it can be a short, human-readable name for the block type.

Since the [master block table][sect3.5] only contains numeric identifiers for the block types, extension block types
need to be mapped to those numeric identifiers in some way.
Therefore, any file using extension blocks MUST contain an [extension block descriptor block][sect4.3], which is a
table that defines the identification strings for each extension block type used in the file.
This block is also referenced in the [header][sect3.4] for ease of processing.
The numeric block types used by extension blocks are therefore indexes into this table; the numbers are otherwise
meaningless by themselves, and implementations MUST NOT rely on the numeric index assigned to a particular extension
block type in a particular file.
Numeric types that are smaller than $4000 are considered extension block types, and therefore indexes into this table;
the maximum valid index is therefore defined by the size of the [extension block descriptor block][sect4.3].
Implementations MUST NOT use numeric extension block types (i.e., numeric types below $4000) that do not correspond to
valid indexes into this table.
The table MAY have any size, even larger than $4000 entries; however, entries beyond the first $4000 are meaningless.

If the [extension block descriptor block][sect4.3] contains two or more entries with the same identification strings,
the corresponding indexes MUST be considered to be equivalent.
However, implementations SHOULD avoid duplicate entries in the table.

## 4. Standard block types

Block types with numeric indexes not smaller than $4000 are standard block types.
Only the type identifiers defined in this section can be used; generators MUST NOT use a standard block type other
than the ones defined here.
(Implementations that want to define their own types for testing MUST use an [extension block type][sect3.6] instead.
They can also apply for standardization and inclusion in this list.)
Block types below $4000 are reserved as [extension block types][sect3.6] and used as described in that section; also,
the [invalid block type][def-invalid] (that is, $FFFF) can be used to indicate an unused block.

The following table defines all numeric identifiers for standard block types.
The columns have the following meaning:

* **ID**: numeric identifier for the block type, given in hexadecimal.
* **Name**: name of the block type, as defined by this document.
  The name consists of human-readable text, not an identifier suitable for use in code; implementations are encouraged
  to define constants in their code with names as similar as possible to the ones listed in this document, suitably
  abbreviated if needed.
  All the names in the table link to the corresponding section in this document defining that block type.
* **Element**: minimum element size for block types that contain a table.
  If the corresponding block type doesn't contain a table, this column contains a long dash (—) instead.
  While generators MAY use an element size larger than the size listed here (as long as the alignment constraint is
  followed), using the exact size listed here is RECOMMENDED.
* **Align**: alignment constraint for the block type.
  This is always 1, 2, 4 or 8 bytes.
* **Unique**: indicates whether the file can contain multiple blocks of that block type.
  (All blocks are OPTIONAL, so it's always valid for a file to contain no blocks at all of a certain type, unless they
  are required by another block in the file.)
  Possible values are:
    * **Yes**: the block type requires uniqueness.
      A file MUST NOT contain more than one block of this type.
    * **Rec.**: uniqueness is RECOMMENDED for the block type.
      A file SHOULD NOT contain more than one block of this type.
      While including multiple blocks of this type is not considered a violation of this specification, and consumers
      MUST accept files with multiple blocks of this type, generators are strongly advised to not include them.
    * **No**: the block type does not represent a unique block.
      Files MAY contain any number of blocks of this type, and there are often legitimate reasons for that to happen.

|   ID  |Name                                          |Element|Align|Unique|
|:-----:|:---------------------------------------------|------:|----:|:----:|
|`$4000`|[Data][sect4.1]                               |      —|    1|  No  |
|`$4001`|[String table][sect4.2]                       |      8|    4|  No  |
|`$4002`|[Extension block descriptor][sect4.3]         |      8|    4| Yes  |
|`$4003`|[Comment][sect4.4]                            |      —|    1|  No  |
|`$4004`|[Source file table][sect4.5]                  |     12|    4| Rec. |
|`$4005`|[ROM image information][sect4.6]              |      —|    8| Rec. |
|`$4006`|[File stack][sect4.7]                         |     12|    4|  No  |
|`$4007`|[Section table][sect4.8]                      |     32|    4| Rec. |
|`$4008`|[Symbol table][sect4.9]                       |     16|    4| Rec. |
|`$4009`|[Line-to-address mapping table][sect4.10]     |     20|    4|  No  |
|`$400A`|[Address-to-line mapping table][sect4.11]     |     20|    4|  No  |
|`$400B`|[Preferred address-to-line mappings][sect4.12]|     12|    4|  No  |
|`$400C`|[Checksum table][sect4.13]                    |      8|    4|  No  |

### 4.1. Data block type

* **Numeric type identifier**: $4000
* **Element size**: not a table (set to zero)
* **Alignment constraint**: 1 byte
* **Linked block field**: not used (set to [invalid][def-invalid])
* **Reference field**: not used (set to [invalid][def-invalid])

This block exists to hold raw data that other blocks may need to reference.
It is meaningless by itself; its only purpose is to act as a container for extra data in other blocks, particularly
for blocks that contain tables.
For example, a [string table block][sect4.2] will link to a data block to contain the actual string contents, since
the string contents are variable size and therefore don't fit inside a fixed-size table element.

As a result of the above, this block will mostly be used as a linked block in some other block.
By itself, it has no semantics or constraints; however, generators SHOULD NOT generate data blocks that aren't used by
any other blocks in the file.

### 4.2. String table block type

* **Numeric type identifier**: $4001
* **Element size**: 8 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [data block][sect4.1] containing the string contents
* **Reference field**: not used (set to [invalid][def-invalid])

String table blocks are used to assign numeric identifiers to [strings][def-strings].
Every component in a debug information file that needs to reference a string (for instance, a filename) will do so
through a string table.
A file can contain any number of string table blocks; if multiple blocks need to reference a string table block, they
MAY share the same string table block across as many of those blocks as desired.
The numeric identifier for each string is its index into the string table block.
(As usual, the first string in the table has index 0.)

The table contains location and size information for each string, not the actual string contents.
The string contents are contained in a [data block][sect4.1] that the string table block links to.
The contents of each string MUST be valid UTF-8, without any embedded null characters ($00 bytes); consumers MUST NOT
expect strings to end in a null terminator character.

The table entries have the following format:

|Offset|Size|Description       |
|-----:|---:|:-----------------|
|     0|   4|Location          |
|     4|   2|Size              |
|     6|   2|Maximum characters|

* **Location**: offset into the linked [data block][sect4.1] where the [string][def-strings] begins; this value MUST
  be smaller than the linked block's size.
  Consumers MUST NOT expect any particular alignment for the location specified by this value.
  If an entry is unused, this field MUST be set to [the invalid value][def-invalid] and the other fields set to zero;
  unused entries MUST NOT be referenced at all.
  (An unused entry isn't the same as the empty string.)
  Empty strings MAY use any value (no larger than the size of the linked [data block][sect4.1]) for this field; if no
  value is particularly meaningful, zero is recommended.
* **Size**: size, in bytes, of the string contents.
  The size MUST NOT cause the string to extend past the end of the linked [data block][sect4.1], and it MUST NOT
  result in a partial UTF-8 encoding (e.g., a stray $C0 byte) at the end of the string.
* **Maximum characters**: maximum number of codepoints encoded by the string.
  This value is meant as a hint for consumers that use some other representation (such as UTF-32) instead of UTF-8
  internally for their strings.
  The actual number of codepoints encoded by the string MAY be smaller than this value, but it MUST NOT be larger.
  If the exact number of encoded codepoints is not known and would be difficult or slow to compute, a known upper
  bound may be stored in this field instead; since each codepoint requires at least one byte, in the absence of any
  better information, the value of the size field MAY be stored in this field as well.

### 4.3. Extension block descriptor block type

* **Numeric type identifier**: $4002
* **Element size**: 8 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [string table block][sect4.2]
* **Reference field**: not used (set to [invalid][def-invalid])

An extension block descriptor block is used to define the actual block types used for [extension blocks][sect3.6].
A file MUST NOT contain more than one extension block descriptor block, and if it does contain one, it MUST be
referenced in the [header][sect3.4].
The block defines the identification strings for each extension block type that the file can use; excess entries (that
is, entries that are not actually in use) are allowed, but generators SHOULD NOT include them.
Each entry defines the identification strings for the block type with a numeric identifier equal to that entry's index
number, up to $3FFF (inclusive); if the table contains more than $4000 entries, further entries are meaningless.
Numeric block type identifiers below $4000 that do not match an entry in the table MUST NOT be used; if the file does
not contain an extension block descriptor block, numeric block type identifiers below $4000 MUST NOT be used at all.

The table entries have the following format:

|Offset|Size|Description      |
|-----:|---:|:----------------|
|     0|   4|Originator string|
|     4|   4|Type string      |

These two fields are index numbers into the [string table][sect4.2] linked by the linked block field; the strings
referenced this way uniquely identify an extension block type.
Duplicate entries (that is, entries that have the same pair of strings, either because they have the same values or
because they reference identical strings) SHOULD NOT be used.
If an entry is unused, both values MAY be set to the [invalid index][def-invalid]; in this case, the corresponding
numeric block type identifier MUST NOT be used in the file.
The generator MUST NOT set only one of the fields to the [invalid index][def-invalid].

### 4.4. Comment block type

* **Numeric type identifier**: $4003
* **Element size**: not a table (set to zero)
* **Alignment constraint**: 1 byte
* **Linked block field**: not used (set to [invalid][def-invalid])
* **Reference field**: not used (set to [invalid][def-invalid])

This block contains human-readable comments.
It is meant for users to add comments to a debug information file; it is **not** meant for applications to exchange
machine-parsable information.
Applications MUST NOT use this block type to extend the format; [extension blocks][sect3.6] MUST be used for that
purpose instead.
Generators MAY allow users to insert comments into a debug information file when the file is generated; likewise,
consumers MAY show comments in such a file to the user.

The contents of this block are any arbitrary text.
This block MUST contain valid UTF-8 text without any embedded null ($00) characters; these are the same restrictions
enforced for [strings][def-strings].
However, the text is not limited to a maximum of 65,535 bytes; the block can be as large as necessary, up to the
maximum size of a block (i.e., $FFFFFFFF bytes).
The size of the block is the size of the text: no padding or other extraneous data is contained in the block.
Likewise, the block MUST NOT contain a null terminator (i.e., a $00 byte) at the end of the text.

Subject to these restrictions, the block MAY contain any text whatsoever; the text is intended for human consumption,
not for application usage.
(For example, the text can contain newlines.)

### 4.5. Source file table block type

* **Numeric type identifier**: $4004
* **Element size**: 12 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [string table block][sect4.2]
* **Reference field**: optional [checksum table block][sect4.13]

This block contains information about the source code files used to build the program referenced by the debug
information file.
This block is intended for actual source code files, not binary assets included in the final ROM image: as a general
rule, files referenced by this block SHOULD have meaningful line numbers.
The file SHOULD NOT contain more than one block of this type.

This block contains links to two other blocks through the linked block and reference fields.
The linked block is a [string table block][sect4.2] used to define the filenames; this block is REQUIRED.
On the other hand, the reference field contains a link to a [checksum table block][sect4.13] used to specify the
checksums for any or all of the files in this block.
If all entries in the table contain the [invalid index][def-invalid] in the checksum field, the reference field MAY be
set to the [invalid block index][def-invalid].
Otherwise, the field MUST point to a [checksum table block][sect4.13].

The table entries have the following format:

|Offset|Size|Description|
|-----:|---:|:----------|
|     0|   4|Filename   |
|     4|   4|Line count |
|     8|   4|Checksum   |

* **Filename**: index into the [string table][sect4.2] linked by the linked block field which refers to the string
  containing the name of the file in question.
  This filename MUST use forward slashes (`/`) as path separators, regardless of the operating system in which the
  debug information file is generated or consumed; implementations MUST convert to and from forward-slash path
  separators into whatever separator their host operating system uses when needed.
  Filenames SHOULD be relative to the location of the built ROM image.
* **Line count**: number of lines in the file; MAY be used to validate that the file is correct.
  References to line numbers in this file MUST NOT be larger than this field's value.
  If the line count is not known, this field is set to the [invalid value][def-invalid].
* **Checksum**: index into the linked [checksum table block][sect4.13], linked through this block's reference field,
  referencing the entry in that table that contains the checksum for this file.
  This field MAY be used to validate that the file hasn't been changed since it was used to build the ROM image.
  The checksum MUST be calculated using LF line-ending characters (codepoint $0A), not CRLF line endings (codepoint
  sequence $0D, $0A); implementations MUST convert as needed to calculate this checksum.
  If the checksum is not known, or if calculating it would be prohibitively inefficient, this field MAY be set to the
  [invalid index][def-invalid] instead.
  If the block's reference field is set to the [invalid block index][def-invalid], this field MUST be set to the
  [invalid index][def-invalid] as well.

### 4.6. ROM image information block type

* **Numeric type identifier**: $4005
* **Element size**: not a table (set to zero)
* **Alignment constraint**: 8 bytes
* **Linked block field**: optional [string table block][sect4.2]
* **Reference field**: optional [checksum table block][sect4.13]

This block contains information about the actual ROM image file that the debug information file is built for.
The file SHOULD NOT contain more than one block of this type.
The information in this block can be used by consumers to verify that the debug information file they are handling
matches the ROM image it is supposed to be for.

The data in this block is a structure, but the block is not a table because it only contains one instance of that
structure.
The block size MUST be large enough to contain the entire structure; the block can be larger (as long as the alignment
constraint is followed), but this is NOT RECOMMENDED.
If the block is larger than the structure, additional bytes are considered reserved and MUST be filled with $FF.

Several fields in the structure can contain [strings][def-strings].
These strings are referenced through a [string table][sect4.2], linked through the linked block field.
If none of those string fields is used (i.e., if all string fields are set to the [invalid index value][def-invalid]),
then the linked block field is OPTIONAL, and it MAY be set to the [invalid block index][def-invalid] as well.
Otherwise, the linked block field MUST point to a [string table block][sect4.2].

Likewise, the reference field MAY contain a reference to a [checksum table block][sect4.13] used to specify the
checksum for the ROM image (using an algorithm other than the one used for the ROM header).
If the structure's other checksum field contains the [invalid index][def-invalid], the block's reference field is
OPTIONAL, and it MAY be set to the [invalid block index][def-invalid] as well.
Otherwise, the reference field MUST point to a [checksum table block][sect4.13].

The structure has the following fields:

|Offset|Size|Description        |
|-----:|---:|:------------------|
|     0|   4|Filename           |
|     4|   4|File size          |
|     8|   4|Other checksum     |
|    12|   2|ROM checksum       |
|    14|   1|Feature flags      |
|    15|   1|Mapper code        |
|    16|   4|Mapper string      |
|    20|   4|Manufacturer code  |
|    24|   8|Build timestamp    |
|    32|   4|Game title         |
|    36|   4|Toolchain ID string|

The feature flags are a bit-packed field with the following subfields:

|Start|Bits|Description                   |
|----:|---:|:-----------------------------|
|    0|   1|ROM checksum invalid flag     |
|    1|   1|Mapper code unknown flag      |
|    2|   1|Manufacturer code invalid flag|
|    3|   1|External title flag           |
|    4|   2|Preferred platform            |
|    6|   2|Reserved                      |

* **Filename**: [string table][sect4.2] index referencing a the string containing the name of the ROM image filename.
  This MUST be a bare filename, without a path.
  If the filename is unknown or not available, this field MUST be set to the [invalid string index][def-invalid].
* **File size**: size of the ROM image file, in bytes.
  This is the exact size as output by the toolchain; if the file doesn't contain padding, this field will contain a
  value not a multiple of the ROM bank size.
  If the file size is not available, this field MAY be set to zero.
  However, generators SHOULD set this field (to a non-zero value) whenever possible.
* **Other checksum**: this field contains an index into the [checksum table block][sect4.13] linked via this block's
  reference field, referencing an entry that contains the checksum for the ROM image file.
  (The field is named "other checksum" to tell it apart from the checksum contained in the ROM header, which requires
  the use of a specific simple algorithm.)
  This checksum is calculated over a file with the file size specified in the file size field; if the ROM image has
  been expanded from its original size, these excess bytes MUST be ignored when calculating or verifying the checksum.
  If the checksum is not known, or if calculating it would be prohibitively inefficient, this field is set to the
  [invalid index][def-invalid].
  If the block's reference field is set to the [invalid block index][def-invalid], indicating that there is no
  [checksum table block][sect4.13] linked by this block, this field MUST be set to the [invalid index][def-invalid].
* **ROM checksum**: this is the checksum contained in the ROM header, at offset $14E.
  This value is stored with its endianness flipped compared to the value stored in the ROM image, because the value in
  the ROM image is stored in big-endian format.
  If the ROM checksum invalid flag (in the feature flags field) is set, this field is meaningless and it SHOULD be set
  to the [invalid value][def-invalid]; consumers MUST ignore this field if that flag is set.
* **Feature flags**: bit-packed field containing the following values:
    * **ROM checksum invalid flag**: if set, it indicates that the value stored in the ROM checksum field is
      meaningless and MUST be ignored by consumers.
    * **Mapper code unknown flag**: if set, it indicates that the single-byte mapper code stored in the mapper code
      field doesn't necessarily represent one of the standard well-known mappers.
      (The mapper string field, if available, can be used to determine the actual mapper in use if needed.)
      This flag can be set because the mapper is unknown or because the mapper doesn't have a known single-byte code.
    * **Manufacturer code invalid flag**: if set, it indicates that the value stored in the manufacturer code field is
      meaningless and MUST be ignored by consumers.
    * **External title flag**: if set, it indicates that the game title field contains a game title set externally
      (i.e., not encoded in the ROM image header).
      This flag can be set if the user explicitly gives the ROM image a title, but the title cannot fit as is in the
      ROM image; it can also be set if the game title specified in this block doesn't match the one in the ROM image
      header for any reason.
    * **Preferred platform**: indicates the major variant of the platform under which the game should be debugged, if
      possible.
      Possible values are 0 for DMG, 1 for SGB, 2 for CGB, or 3 for unknown.
    * The reserved bits have no meaning assigned to them by this version of the specification.
      Like any reserved field, it MUST be set to the [invalid value][def-invalid], which is an all-bits-set value.
* **Mapper code**: single-byte code that identifies the mapper in use by the program.
  If the mapper is known and matches one of the well-known single-byte mapper codes, that code MUST be set as the
  value of this field, even if it doesn't match the value given in the ROM image header (at offset $147).
  In this case, the mapper code unknown flag (in the feature flags field) MUST be cleared.
  Otherwise, the mapper code unknown flag MUST be set, and this value SHOULD be set to the single-byte mapper code
  found at offset $147 in the ROM image header if possible.
* **Mapper string**: [string table][sect4.2] index referencing a string that describes the mapper in use by the
  program.
  This SHOULD be a machine-parsable, human-readable string that describes the components of the mapper in question.
  (For example, this string could be set to a value like `MBC3+TIMER+BATTERY`.)
  If the mapper information is not available, this field MUST be set to the [invalid string index][def-invalid].
* **Manufacturer code**: four-character code that identifies the program in question, typically used for licensed
  games.
  This value is found in the ROM image header at offset $13F; note that the value as a 4-byte integer will be parsed
  in little-endian format (so, for instance, a manufacturer code of `ABCD` would be read as $44434241).
  If the manufacturer code invalid flag (in the feature flags field) is set, this field is meaningless and SHOULD be
  set to the [invalid value][def-invalid]; consumers MUST ignore this field if that flag is set.
* **Build timestamp**: timestamp when the ROM image was last built or updated.
  This value is stored as the number of microseconds since 1 January 1970, midnight UTC (i.e., it is a Unix timestamp
  scaled up by 1,000,000).
  If the build timestamp is unknown, this field MUST be set to the [invalid value][def-invalid].
* **Game title**: [string table][sect4.2] index referencing a string that identifies the title of the game or program.
  Whenever possible, this SHOULD be a copy of the string stored in the ROM image header, at offset $134.
  When this is the case, the external title flag (in the feature flags field) MUST be cleared.
  Otherwise, if that flag is set, the game title can be any user-supplied string that identifies the game or program.
  If the game title is not available, this field MUST be set to the [invalid string index][def-invalid].
  (In that case, the external title flag is meaningless.)
* **Toolchain ID string**: [string table][sect4.2] index referencing a string that identifies the toolchain that
  produced the ROM image.
  This can be any string that the toolchain wants to use to identify itself; those strings SHOULD be reasonably unique
  (i.e., different from other toolchains' ID strings), and the same version of the same toolchain SHOULD always
  generate the same value for this field.
  If the toolchain is unknown, or if it is configured not to identify itself, this field MUST be set to the
  [invalid string index][def-invalid].

### 4.7. File stack block type

* **Numeric type identifier**: $4006
* **Element size**: 12 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [source file table block][sect4.5]
* **Reference field**: not used (set to [invalid][def-invalid])

This block is used by other blocks which need to assign some part of the program image (e.g., a specific address) to
one or more lines of code.
By itself, it is generally meaningless, although it MAY still be included in the file.

In many languages, source code files can include other source code files.
This typically comes in two variants: explicit inclusions, such as an `INCLUDE` directive in assembly or `#include` in
C, and implicit forms of transcluding code from a file (not necessarily a different file), such as macro expansion in
a macro assembler.
Inclusions need not transclude the entire file being included; for example, macro expansion will typically only
include a small portion of the file, that portion being the body of the macro.
(For the purpose of locating source lines of code, which is what this block type is designed for, it makes no
difference whether the inclusion refers to the entire file or part of it.)

Source file inclusion creates a file stack, where a single line of code isn't identified just by its position in the
file that contains it, but also by the line that included that file, the line that included its parent, and so on.
Preserving the entire stack is key to debuggability, because users often need to see layers that aren't the bottommost
one: in many cases, some previous file in the stack is the most relevant one or has the most context.
This is particularly true with implicit inclusions: for example, if a language accepts macros, each macro expansion is
implicitly including the file that defines the macro (even if it is the same file that expands it); however, it is
very likely that, for most macros, the user will want to see the macro invocation itself, not its definition.

This block therefore defines such a stack, so that each entry represents a single combination of a file and its
parents; each instance of a file being included from a different location in the source code SHOULD create a separate
entry in this table.
Likewise, files that are direct inputs to the build process are included in the table as well, so that they can also
be referenced for line number assignment.
Entries in the table MUST be unique: no two entries may have the same exact combination of values across all fields.

The table entries have the following format:

|Offset|Size|Description|
|-----:|---:|:----------|
|     0|   4|File       |
|     4|   4|Parent     |
|     8|   4|Line number|

* **File**: index into the [source file table][sect4.5] linked by the linked block field which indicates the file that
  will be referenced by this entry.
  This is the file "at the top" of the file stack for this entry.
  If the entry refers to something that is not a file (for example, because the input to the build process comes from
  a non-file source), this field MAY also be set to the [invalid index][def-invalid].
* **Parent**: parent of the current entry, i.e., the next entry in the file stack.
  If the current entry has no parent (i.e., it is a direct input to the build process, not included by another file),
  this field MUST be set to the [invalid index][def-invalid].
  Otherwise, this field contains the index of another entry in the same table; that index MUST be smaller than this
  entry's index.
  This ensures that file stacks cannot become circular chains.
* **Line number**: line at which the parent includes the current file.
  If the parent field is set to the [invalid index][def-invalid] (i.e., if the current entry has no parent), this
  field is meaningless and SHOULD be set to the [invalid value][def-invalid] as well.
  Otherwise, this is the line number in the parent file that includes the file referenced by this entry.
  Line numbers start at 1; line number 0 MAY be used to indicate special inclusions before the beginning of the file,
  or if the line number is unknown.
  This field MUST NOT be greater than the line count declared for the parent file in its corresponding entry in the
  linked [source file table][sect4.5].

### 4.8. Section table block type

* **Numeric type identifier**: $4007
* **Element size**: 32 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: optional [string table block][sect4.2]
* **Reference field**: optional [file stack block][sect4.7]

This block contains information about the sections that make up the program image.
A section is a contiguous portion of addressing space (in ROM, console RAM, cartridge RAM, or any other area) that is
treated as a unit by the toolchain, particularly the linking process.
The file SHOULD NOT contain more than one block of this type.

This block contains links to two other blocks through its linked block and reference fields.
The linked block is a [string table block][sect4.2] used for section names and section type strings; the reference
field contains a link to a [file stack block][sect4.7] used for file references.
For each of these links, if all entries in the table contain the [invalid index][def-invalid] for the corresponding
fields, the corresponding linked block MAY be set to the [invalid block index][def-invalid] as well.
Otherwise, the corresponding linked block MUST point to a block of the correct type.

This block contains a table where each entry represents a section.
Each section has a name; sections SHOULD have unique and non-empty names.

The table entries have the following format:

|Offset|Size|Description                              |
|-----:|---:|:----------------------------------------|
|     0|   4|Section name                             |
|     4|   4|Section type string                      |
|     8|   2|Bank number                              |
|    10|   2|Starting address                         |
|    12|   2|Size                                     |
|    14|   1|Flags                                    |
|    15|   1|Reserved                                 |
|    16|   4|ROM image offset                         |
|    20|   4|File                                     |
|    24|   4|Line number                              |
|    28|   4|[Address-to-line mapping table][sect4.11]|

The flags are a bit-packed field with the following subfields:

|Start|Bits|Description            |
|----:|---:|:----------------------|
|    0|   4|Alignment requirement  |
|    4|   2|Section kind           |
|    6|   1|Multiple locations flag|
|    7|   1|Autogenerated flag     |

* **Section name**: index into the linked [string table block][sect4.2] indicating the name of the section.
  If the section has no name, this field MAY be set to the [invalid string index][def-invalid].
  (This is not the same as having the empty string as its name.)
* **Section type string**: index into the linked [string table block][sect4.2] indicating the type of the section.
  This is a short string that describes the kind of section it is, such as `ROM0` or `HRAM`; possible values for this
  field are defined by the toolchain that builds the ROM image.
  If the section has no type (for example, because the toolchain doesn't support section types in the first place),
  this field MAY be set to the [invalid string index][def-invalid].
* **Bank number**: bank number for the section's starting address.
  If the section's starting address is in unbanked memory, this field MUST be set to zero.
* **Starting address**: absolute address of the first byte of the section.
  If the section's size is zero (i.e., the section is empty), this field SHOULD be set to the address the first byte
  of the section would have if the section wasn't empty.
* **Size**: size of the section, in bytes.
* **Flags**: bit-packed field containing the following values:
    * **Alignment requirement**: number of low-order bits constrained by alignment requirements in the section's
      placement.
      (For example, a section which must be aligned to a multiple of 16 bytes constrains the lower 4 bits to zero.)
      Since some toolchains support aligning sections at a non-zero offset, the corresponding low-order bits of the
      starting address MAY be non-zero: for example, a section that requests an alignment of 4 bytes away from a
      multiple of 8 bytes would have a value of 3 for this field, but the lower 3 bits of the starting address would
      be set to 4.
      If the alignment requirement of the section is not known, this field must be set to 15, which is the
      [invalid value][def-invalid] for a field of this size.
      A value of 15 MUST NOT be interpreted as an actual alignment requirement.
    * **Section kind**: value that describes the type of section represented by this entry.
      This field may have the following values:
        * **0 (text)**: represents a section of code or data contained in the ROM image.
          The section contains user-defined contents that are mapped into memory from the ROM image.
          ROM sections are generally of this kind.
        * **1**: reserved.
          This value is assigned no meaning by this version of the specification and MUST NOT be used.
        * **2 (memory)**: represents a section of uninitialized memory, which does not correspond to any data in the
          ROM image.
          The section is mapped to memory, but the data in it doesn't come from the ROM image; nevertheless, the
          region of memory assigned to this section is exclusive to it.
          RAM sections are generally of this kind.
          Sections of this kind SHOULD have the ROM image offset field set to the [invalid value][def-invalid].
        * **3 (common)**: represents a section of uninitialized memory shared with other sections.
          The section is mapped to memory, potentially overlaid with other sections; since the region of memory is
          assigned to multiple sections, the data in it cannot come from the ROM image.
          Some RAM sections can be of this kind.
          Sections of this kind SHOULD have the ROM image offset field set to the [invalid value][def-invalid].
    * **Multiple locations flag**: if set, it indicates that the section is declared in multiple places in the source
      code.
      This implies that a single location in source code cannot be given for the section.
      Sections which have this flag set MUST have the file and line number fields set to the
      [invalid value][def-invalid].
    * **Autogenerated flag**: if set, it indicates that the section was generated by the toolchain without it being
      requested by the source code.
      Sections which have this flag set MUST have the multiple locations flag cleared, and the file and line number
      fields set to the [invalid value][def-invalid].
* The reserved field has no meaning assigned to it by this version of the specification.
  Like all reserved fields, it MUST be set to the [invalid value][def-invalid].
* **ROM image offset**: offset into the ROM image file where this section begins.
  For sections with a section kind value of 2 or 3, this field is meaningless, and it SHOULD be set to the
  [invalid value][def-invalid].
  If the size field is set to zero (i.e., the section is empty) and the section kind field is also set to zero, this
  field SHOULD be set to the offset into the ROM image file where the section would begin if it wasn't empty.
* **File**: index into the linked [file stack block][sect4.7] indicating the file where the section begins.
  This field is only meaningful if the language supports explicit section declarations; the line of code referenced by
  the file and line number fields is the line of code that declares the section.
  This field MAY be set to the [invalid index][def-invalid] to indicate that the line of code where the section begins
  is unknown; it MUST be set to the [invalid index][def-invalid] if the multiple locations flag or the autogenerated
  flag are set.
* **Line number**: line number for the line of code where the section begins; this is a line in the file indicated by
  the file field.
  The first line of the file is line 1.
  If the file field is set to the [invalid index][def-invalid], this field MUST also be set to the
  [invalid value][def-invalid].
  Otherwise, this field MUST be set to a value no greater than the line count for the specified file (as indicated in
  the [source file table block][sect4.5] linked by the [file stack block][sect4.7]); line number 0 MAY be used to
  indicate special code before the beginning of the file, or if the actual line number is unknown.
* **Address-to-line mapping table**: index number of an [address-to-line mapping table block][sect4.11] that maps the
  addresses in the section to their corresponding source code line numbers.
  If the file contains an [address-to-line mapping table block][sect4.11] that maps the entire section this way, this
  field MAY be set to the block index for that block, even if the block maps some additional addresses that don't
  belong to this section.
  Otherwise (for example, if there is no such block, or if the section is split across many such blocks), this field
  MUST be set to the [invalid block index][def-invalid].

### 4.9. Symbol table block type

* **Numeric type identifier**: $4008
* **Element size**: 16 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [string table block][sect4.2]
* **Reference field**: optional [section table block][sect4.8]

This block contains the symbols exported by the program.
Symbols are public identifiers that map to values, typically addresses in the program image.
The file SHOULD NOT contain more than one block of this type.

This block contains links to two other blocks through the linked block and reference fields.
The linked block is a [string table block][sect4.2] used for the symbol names themselves; this block is REQUIRED.
On the other hand, the reference field contains a link to a [section table block][sect4.8] used to specify the
sections that each symbol belongs to.
If all entries in the table contain the [invalid index][def-invalid] in the section field, the reference field MAY be
set to the [invalid block index][def-invalid].
Otherwise, the field MUST point to a [section table block][sect4.8] containing entries for the sections that the
symbols in the table belong to.

The table entries have the following format:

|Offset|Size|Description|
|-----:|---:|:----------|
|     0|   4|Name       |
|     4|   2|Value      |
|     6|   2|Bank       |
|     8|   4|Section    |
|    12|   2|Size       |
|    14|   1|Flags      |
|    15|   1|Reserved   |

The flags are a bit-packed field with the following subfields:

|Start|Bits|Description       |
|----:|---:|:-----------------|
|    0|   2|Symbol type       |
|    2|   1|Boot ROM flag     |
|    3|   1|Address flag      |
|    4|   1|Banked symbol flag|
|    5|   1|Invalid size flag |
|    6|   2|Reserved          |

* **Name**: index into the linked [string table block][sect4.2] referencing the name of the symbol.
  The name MUST be different from all other names in the table, and it MUST NOT be the empty string.
  (Names are compared as [strings][def-strings]: two different index values referencing equal strings are considered
  equal.)
  This field MUST NOT be set to the [invalid string index][def-invalid].
* **Value**: value of the symbol.
  If the address flag is set, the symbol represents an address, and this field contains that address.
  Otherwise, if the address flag is cleared, the symbol represents a 4-byte integer, and this field contains the lower
  16 bits of that integer.
* **Bank**: bank of the symbol.
  If the address flag and the banked symbol flag are both set, the symbol represents a banked address, and this field
  contains the bank for that address.
  If the address flag is set, but the banked symbol flag is cleared, the symbol represents an unbanked address; in
  that case, this field is meaningless and MUST be set to zero.
  (Zero is used instead of the [invalid value][def-invalid] to maximize compatibility with implementations that treat
  all addresses as banked.)
  If the address flag is cleared, the symbol represents a 4-byte integer; in that case, this field contains the upper
  16 bits of that integer.
* **Section**: index into the linked [section table block][sect4.8] indicating the section that the symbol belongs to.
  If the address flag is cleared, this field MUST be set to the [invalid index value][def-invalid].
  Otherwise, if the address flag is set, and the symbol belongs to a section, this field contains the index of that
  section into the [section table block][sect4.8]; if the symbol doesn't belong to any section, or the section it
  belongs to is unknown, this field is set to the [invalid index value][def-invalid].
* **Size**: size of the object referenced by the symbol.
  (An "object" in this context is any data, code or structure in memory; it does not necessarily mean an object in the
  context of object-oriented programming.)
  If the object has a known size, and that size is $FFFF or less, this field is set to that size and the invalid size
  flag is cleared.
  Otherwise, the invalid size flag is set, indicating that this field doesn't actually contain a true size.
  If that flag is set, and the true size is greater than $FFFF (i.e., it would overflow the field), this field is set
  to the [invalid value][def-invalid]; if the size is unknown, this field is set to zero.
  Values other than zero and the [invalid value][def-invalid] MUST NOT be used if the invalid size flag is set.
  If the address flag is cleared (indicating that the symbol doesn't represent an object in memory at all), the
  invalid size flag MUST be set and this field MUST be set to zero.
* **Flags**: bit-packed field containing the following values:
    * **Symbol type**: value that describes the kind of object referenced by the symbol.
      Possible values are 0 for code, 1 for data, 2 for empty space (such as padding or uninitialized memory), and 3
      for mixed or unknown contents.
    * **Boot ROM flag**: if set, it indicates that the symbol's address represents an address in the boot ROM.
      If this flag is set, the address flag MUST be set, the banked symbol flag MUST be cleared, and the bank field
      MUST be set to zero; the value field SHOULD contain an address in boot ROM space.
    * **Address flag**: if set, it indicates that the symbol represents an address.
      Otherwise, the symbol represents a 4-byte integer, not an address.
      If this flag is cleared, the size field MUST be set to zero, the invalid size flag MUST be set, the boot ROM and
      banked symbol flags MUST be cleared, the symbol type field MUST be set to 3, and the section field MUST be set
      to the [invalid index value][def-invalid].
      If this flag is cleared, the value field contains the lower 16 bits of the symbol's value, and the bank field
      contains the upper 16 bits of that value.
    * **Banked symbol flag**: if set, it indicates that the symbol is banked.
      Otherwise, the symbol is unbanked; in that case, the bank field is meaningless and MUST be set to zero.
      If the address flag is cleared, this flag is meaningless and MUST be cleared as well.
    * **Invalid size flag**: if set, it indicates that the size field contains a special value, not the size of an
      object.
      When this flag is set, the size field MUST be set to either zero or the [invalid value][def-invalid]; the former
      value indicates an unknown size, while the latter value indicates an overflowing size.
      If the address flag is cleared, this flag MUST be set, as a non-address symbol cannot have a size.
* Both reserved fields (the standalone one and the one in the bit-packed flags field) have no meaning assigned to them
  by this version of the specification.
  Like all reserved fields, they MUST be set to the [invalid value][def-invalid], which is an all-bits-set value in
  all cases.

### 4.10. Line-to-address mapping table block type

* **Numeric type identifier**: $4009
* **Element size**: 20 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [file stack block][sect4.7]
* **Reference field**: index into the linked file stack table

This block maps each line of a file to an address in the program image.
It is the inverse of the [address-to-line mapping table][sect4.11] block type.

Since each inclusion of a file (or of part of it, such as with macro expansion) will most likely map to a different
address, the block references a file in the [file stack][sect4.7] instead of referencing the [source file][sect4.5]
directly.
Each block of this type corresponds to a single entry in the file stack; the reference field contains the index of the
entry in the linked [file stack block][sect4.7] (linked through the linked block field) indicating which entry in the
file stack the block corresponds to.
A file MUST NOT contain more than one block of this type with the same combination of values for its linked block and
reference fields.

Each entry in the table contained in this block corresponds to one or more lines of code in the corresponding file; an
entry can refer to multiple consecutive lines of code if they are being handled as a single logical unit.
Not all lines of code need to be referenced by the table: lines of code that don't correspond to any addresses in the
program image SHOULD NOT be included in the table.
Likewise, lines of code that correspond to multiple disjoint ranges of addresses (such as file inclusion statements
that include code from multiple [sections][sect4.8]) MUST NOT be included in the table, with the exception of repeated
blocks as defined below.
(Lines in repeated blocks that by themselves correspond to multiple disjoint ranges of addresses MUST NOT be included
in the table: the exception only applies when the repeated block itself is the cause for the multiplicity.)

It is possible for a line of source code to be emitted multiple times into the final program text.
For example, assemblers often support repetition directives, and higher-level compilers can often unroll loops.
This can happen both within the file and from outside the file (e.g., because the file itself was included from within
a repeated block in another file).
A sequence of source lines repeated as a unit (even if some of those lines aren't emitted in every iteration) is
called a "repeated block" for the purposes of this specification.
If the entire file is repeated (for example, because it is included from within a repeated block in another file),
then the entire contents of the file make up a repeated block.
Repeated blocks require special handling, because they result in multiple line-to-address mappings for the lines they
contain.
Therefore, this block's table contains fields to designate repeated blocks, allowing consumers to process them
correctly.

Entries in the table MUST appear in "logical program order".
This order is defined by the following rules:

1. Lines that don't belong to any repeated blocks MUST appear in ascending order of line numbers.
   The line numbers in those entries MUST NOT overlap with any other entries in the table.
2. Repeated blocks MUST appear in the position their contents would appear (according to rule 1) if they weren't
   repeated.
   That means that they appear after all entries representing lines that come before the repeated block, and before
   all entries representing lines that come after the repeated block.
3. If a file contains nested repeated blocks, the position of the inner block is defined by applying rule 2 in the
   context of each iteration of the outer block; in other words, the rule applies recursively to all nested repeated
   blocks.
4. Within a repeated block, lines in each iteration that don't belong to any nested repeated blocks MUST appear in
   ascending order of line numbers.
   The line numbers in those entries MUST NOT overlap with any other entries in the same iteration of that repeated
   block, or with any entries outside the repeated block.
5. Iterations of a repeated block SHOULD appear in the order they appear in the program image (i.e., in ascending
   order of addresses).
   If the corresponding addresses belong to the same section (as defined in any [section table][sect4.8] contained in
   the file), this requirement is a MUST.
   On the other hand, if some or all iterations of the repetition define new sections, the iterations MAY appear in
   the order defined by the source code instead.
6. The first entry corresponding to each iteration of a repeated block MUST be designated as such through the use of
   the repeated block length and remaining iteration count fields.
7. It is possible for two or more nested repeated blocks to begin at the same line.
   In this case, there MUST be an entry for each layer of repeated blocks beginning at that line; this is an exception
   to rule 4 in terms of overlapping line numbers.
   These entries MUST be ordered according to their nesting order, with the outermost repeated block coming first and
   the innermost repeated block coming last.
   Entries other than the innermost MUST have their size and line count fields set to zero in this case.

The table entries have the following format:

|Offset|Size|Description              |
|-----:|---:|:------------------------|
|     0|   4|Line number              |
|     4|   4|Line count               |
|     8|   2|Starting address         |
|    10|   2|Bank number              |
|    12|   2|Size                     |
|    14|   2|Remaining iteration count|
|    16|   4|Repeated block length    |

* **Line number**: initial line that is being referenced by the entry.
  The first line of the file is line 1.
  This field MUST NOT be less than 1 or greater than the number of lines in the file (as defined by the
  [source file table block][sect4.5] linked by the [file stack block][sect4.7] in turn linked by this block).
  The order of the entries according to this field MUST obey all the restrictions laid out above by the logical
  program order rules.
  This field MUST NOT be set to the [invalid value][def-invalid].
* **Line count**: number of lines being referenced by the entry.
  The last line referenced by the entry is the sum of the line number field and this field minus one.
  That last line MUST NOT be greater than the number of lines in the file, defined in the same way as for the previous
  field.
  This field MUST NOT be set to the [invalid value][def-invalid], and it MUST NOT be set to zero except in the case
  mentioned in rule 7 of the logical program order rules laid out above.
* **Starting address**: starting address of the range of addresses that the lines referenced by the entry map to.
  If the size field is zero, this is the starting address that the lines in question would map to if the entry had a
  non-zero size.
* **Bank number**: bank number for the starting address referenced by the previous field.
  If that address lies in unbanked memory, this field MUST be set to zero.
* **Size**: number of bytes that the lines referenced by the entry map to.
  This field SHOULD NOT be set to zero except in the case mentioned in rule 7 of the logical program order rules laid
  out above.
* **Remaining iteration count**: number of iterations remaining in the current repeated block.
  If the repeated block length field is set to zero (i.e., the current entry does not begin an iteration of a repeated
  block), this field MUST be set to zero as well.
  Otherwise, this field indicates the number of iterations remaining in the current repeated block, not including the
  current one.
  (For instance, if a repeated block is repeated three times, the first entry for each iteration would respectively
  have values of 2, 1 and 0 for this field.)
  If the remaining iteration count overflows this field, the field MUST be set to the [invalid value][def-invalid];
  as usual, this value is also the maximal value the field can have.
* **Repeated block length**: number of table entries representing the current iteration of a repeated block.
  If the current entry is not the first entry of some iteration in a repeated block (see rule 6 of the logical program
  order rules laid out above), this field MUST be set to zero.
  For the first entry of an iteration in a repeated block, this field MUST be set to the number of table entries that
  make up that iteration.
  (Note that this is the number of **table entries**, not the number of **lines**.)
  The starting entry itself is included in that count, so the minimum value for this field in that case is 1.
  This means that the first entry of the next iteration (or the first entry after the end of the repeated block, if
  the remaining iteration count field is zero) can be found by adding the value of this field to the current entry's
  index number.
  This field MUST NOT be set to the [invalid value][def-invalid].

### 4.11. Address-to-line mapping table block type

* **Numeric type identifier**: $400A
* **Element size**: 20 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [file stack block][sect4.7]
* **Reference field**: starting address and bank (see below)

This block maps a range of addresses to the source code lines that represent those addresses.
It is the inverse of the [line-to-address mapping table][sect4.10] block type.
An address-to-line mapping block MUST map a single contiguous range of addresses to source code lines; the ranges
mapped by any two such blocks in the file MUST NOT overlap.
The linked block is a [file stack block][sect4.7], linked through the linked block field, which is used to specify the
file that each address maps to.

If the mapped range of addresses fully contains one or more sections, as defined by any [section table block][sect4.8]
in the file, the table entries for those sections in that block MAY reference this block in their address-to-line
mapping table field, indicating that this block maps the entire section.
Generators SHOULD NOT split sections across multiple address-to-line mapping table blocks.

The reference field contains the starting address for the range mapped by the block: the upper 16 bits determine the
bank and the lower 16 bits determine the address within the bank.
All addresses mapped by the block MUST belong to the same bank.
If the mapped region of memory is unbanked, the upper 16 bits of the reference field MUST be set to zero and all
addresses mapped by the block MUST be in unbanked regions of memory.
(The final address of the range can be determined by inspecting the block.)
The address determined by the reference field MUST NOT fall within the range mapped by any other address-to-line
mapping block; this condition is sufficient to prevent overlaps, since, if two ranges overlap, at least one of their
starting addresses will fall within the other range.

A single address MAY map to multiple files across the [file stack][sect4.7] as files include one another.
An address-to-line mapping table can represent going down the [file stack][sect4.7], and thus it can contain many such
mappings: as many levels as desired can be represented within the block.

The entries' addresses MUST follow the following rules:

1. The first entry's starting address MUST be equal to the starting address of the mapped range (i.e., the starting
   address in the reference field of the block).
2. For the purpose of these rules, each entry's "next address" is defined to be the sum of its starting address and
   its size.
   The next address of all entries MUST NOT go beyond the end of the addressing space, and, for banked addresses, it
   MUST NOT go beyond the end of the bank.
   However, the next address MAY be exactly at the end of the addressing space and/or the bank.
   (For example, an entry can have a starting address of $FFFF and a size of 1, giving it a next address of $10000,
   which is exactly at the end of the addressing space.)
3. If an entry does not represent a file inclusion (i.e., its included entries field is zero), and it is not the last
   entry in the table, the entry's next address MUST be equal to the following entry's starting address.
   (In other words, all entries in the table at the same file inclusion level MUST represent consecutive addresses.)
   If the entry's next address is exactly at the end of the addressing space and/or the bank, it MUST be the last
   entry in the table.
4. If an entry does represent a file inclusion (i.e., its included entries field is not zero), then the entry
   implicitly defines an "included span": if the entry's index number is N, and its included entries field's value is
   S, then entries with indexes N+1 through N+S in the table are the entry's included span.
   (The included span MUST NOT go beyond the end of the table: in other words, the table MUST have at least N+S+1
   entries.)
   If the included span is not at the end of the table, then the entry's next address MUST be equal to the starting
   address of the entry right after the included span.
   If the entry's next address is exactly at the end of the addressing space and/or the bank, there MUST NOT be any
   entries in the table after the included span.
   (These restrictions are equivalent to those defined in rule 3, but ignoring the included span.)
5. If an entry represents a file inclusion, all entries in its included span MUST have a starting address greater than
   or equal to the parent's starting address and a next address less than or equal to the parent's next address.
   (In other words, the addresses in the included span MUST be contained within the parent entry's address range.)
   The first entry in the included span MUST have a starting address equal to the parent's starting address, and the
   last entry in the included span MUST have a next address equal to the parent's next address.
6. If an entry within another entry's included span also represents a file inclusion, the inner entry's included span
   MUST be fully contained within the outer entry's included span.

The rules above do not allow for gaps in the addresses in the table.
However, it is possible to indicate that a certain entry does not map to any file.
Therefore, generators MUST indicate "gaps" in the source code by using these entries (i.e., entries with their file
field set to the [invalid index][def-invalid]) instead.
If a gap spans multiple bytes, generators MAY use a single entry to indicate the gap or split it into as many entries
as desired.

Each entry in the table that does not belong to one of those "gap" entries corresponds to one or more lines of code in
some source code file in the file stack.
An entry can refer to multiple consecutive lines of code if they are being handled as a single logical unit.

The table entries have the following format:

|Offset|Size|Description     |
|-----:|---:|:---------------|
|     0|   2|Starting address|
|     2|   2|Size            |
|     4|   4|File            |
|     8|   4|Line number     |
|    12|   4|Line count      |
|    16|   4|Included entries|

* **Starting address**: starting address for the range of addresses referenced by this entry.
  This address MUST follow all the rules laid out above.
  The bank number for this address is given by the block's reference field; all addresses in a single table belong to
  the same bank.
* **Size**: size of the range of addresses referenced by this entry, in bytes.
  This field MUST NOT be set to zero.
  The resulting range (and in particular, the "next address", as it results from this field and the starting address
  field) MUST follow all the rules laid out above.
* **File**: index into the linked [file stack block][sect4.7] indicating the file that this entry maps to.
  This field MAY be set to the [invalid index][def-invalid] if this entry corresponds to a range of addresses that
  doesn't map to any source code lines.
  In that case, the line number field MUST be set to the [invalid value][def-invalid] as well, and the line count
  field MUST be set to zero.
* **Line number**: initial line number that this entry maps to.
  The first line of the file is line 1.
  If the file field is set to the [invalid index][def-invalid], this field MUST also be set to the
  [invalid value][def-invalid].
  Otherwise, this field MUST NOT be greater than the number of lines in the file that the file field refers to (as
  defined by the [source file table block][sect4.5] linked by the [file stack block][sect4.7] in turn linked by this
  block); it MAY be set to zero to indicate any special code being inserted before the beginning of the file.
* **Line count**: number of lines that this entry maps to.
  If the file field is set to the [invalid index][def-invalid], this field MUST be set to zero.
  Otherwise, this field MUST NOT be set to zero, and the last line that the entry maps to (calculated as the sum of
  the line number field and this field minus one) MUST NOT be greater than the number of lines in the file, as defined
  for the field above.
  If the line number field is set to zero, this field SHOULD be set to one.
  This field MUST NOT be set to the [invalid value][def-invalid].
* **Included entries**: number of entries in this entry's included span.
  If this entry does not include other entries, as defined in rule 4 of the rules laid out above, this field MUST be
  set to zero.
  Otherwise, it MUST be set to a non-zero value as defined in that rule.
  A non-zero value for this field implies that the entries in this entry's included span are "contained" by this
  entry: for example, a file inclusion will "contain" the lines it includes from another file.
  This field MUST NOT be set to the [invalid value][def-invalid].

### 4.12. Preferred address-to-line mappings block type

* **Numeric type identifier**: $400B
* **Element size**: 12 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: not used (set to [invalid][def-invalid])
* **Reference field**: starting address and bank (see below)

This block indicates which entry in some [address-to-line mapping table][sect4.11] is the preferred mapping for any
given address in the program image.
The purpose of this block is to indicate to consumers how to find a single line of code to map to any given address:
since [address-to-line mapping tables][sect4.11] can contain file inclusions that map the same address to multiple
lines of code, this block provides a way for consumers to determine which one of those entries to choose when a single
line needs to be shown.

Each block of this type MUST map an entire bank or an entire region of unbanked memory.
(This allows consumers to quickly determine if a block of this type is available for any given location in memory.)
The reference field contains the starting address for the bank or region; the lower 16 bits contain the address and
the upper 16 bits contain the bank number.
If the mapped region of memory is unbanked, the upper 16 bits of the reference field MUST be set to zero.
If the mapped region of memory is banked, all addresses referenced by the block belong to the same bank.
A file MUST NOT contain more than one block of this type with the same value in its reference field.

All addresses MUST represent contiguous ranges of memory, without gaps: for all entries other than the first, the
starting address of the entry MUST be equal to the sum of the starting address and the size for the previous entry.
The starting address of the first entry MUST be equal to the starting address indicated in the reference field (i.e.,
to its lower 16 bits), and the last entry of the table MUST be at the end of the bank or region mapped by the block
(i.e., the sum of its starting address and size MUST be equal to the sum of the bank or region's starting address and
size).

The table entries have the following format:

|Offset|Size|Description                        |
|-----:|---:|:----------------------------------|
|     0|   2|Starting address                   |
|     2|   2|Size                               |
|     4|   4|[Preferred mapping table][sect4.11]|
|     8|   4|Preferred mapping entry            |

* **Starting address**: starting address for the range of addresses referenced by this entry.
  For the first entry in the table, this value MUST be equal to the starting address of the bank or region mapped by
  the block (i.e., the lower 16 bits of the block's reference field); for other entries, this value MUST be equal to
  the sum of the previous entry's starting address and size fields.
* **Size**: size of the range of addresses referenced by this entry, in bytes.
  This field MUST NOT be set to zero.
  For the last entry in the table, the resulting range MUST extend to the end of the bank or region mapped by the
  block; in other words, the sum of this field and the starting address field MUST be equal to the sum of the starting
  address for the bank or region (as determined by the lower 16 bits of the block's reference field) and the size of
  that bank or region.
* **[Preferred mapping table][sect4.11]**: index of the [address-to-line mapping table block][sect4.11] that contains
  the preferred address-to-line mapping table entry for this entry's range of addresses.
  If there is no such mapping, this field MAY be set to the [invalid block index][def-invalid] instead; in that case,
  the preferred mapping entry field MUST also be set to the [invalid index][def-invalid].
* **Preferred mapping entry**: index into the table referenced by the preferred mapping table field indicating the
  entry that contains the preferred address-to-line mapping for this entry's range of addresses.
  If the preferred mapping table field is set to the [invalid block index][def-invalid], this field MUST also be set
  to the [invalid index][def-invalid].
  Otherwise, this field MUST be set to the index of some entry in the table referenced by the block index in the
  preferred mapping table field; the range of addresses specified by this entry's starting address and size fields
  MUST be fully contained within the range of addresses specified in the referenced entry in the corresponding
  [address-to-line mapping table][sect4.11].

### 4.13. Checksum table block type

* **Numeric type identifier**: $400C
* **Element size**: 8 bytes
* **Alignment constraint**: 4 bytes
* **Linked block field**: [data block][sect4.1] containing checksum data
* **Reference field**: not used (set to [invalid][def-invalid])

This block is used to contain checksums for files.
These checksums MAY be used by consumers to verify that the files they are handling are the same files that were used
to generate the debug information file.
The block does not specify what file or data the checksum belongs to: this block is intended for other blocks to
reference whenever they need to specify a checksum.

For each checksum, both the algorithm used to calculate it and its size are indicated in this block.
Different entries in the table MAY use different algorithms.
The size is normally determined by the algorithm; however, it is included in the table so that consumers that don't
implement the algorithm can still use the checksum in some other way (for instance, by displaying it to the user).
The actual checksum data is contained in a [data block][sect4.1], linked through this block's linked block field.
The data MAY be contained anywhere in that block; in particular, there are no alignment requirements for it, and
consumers MUST be prepared to deal with checksum data that is not aligned to any particular width.

Each checksum algorithm is identified by a 2-byte numeric identifier.
Valid algorithms are listed in the following table, along with the size expected by each one of them; these algorithms
are defined in [the corresponding annex][annexA].
This version of the specification does not contain any provisions for user-defined algorithms; numeric identifiers not
listed in this table MUST NOT be used.

|  ID   |Name                      |Size|
|:-----:|:-------------------------|---:|
|`$F000`|Byte sum                  |   1|
|`$F001`|2-byte sum (little-endian)|   2|
|`$F002`|4-byte sum (little-endian)|   4|
|`$F003`|Game Boy header checksum  |   2|
|`$F004`|CRC-32 (ITU-T V.42)       |   4|
|`$F005`|Adler-32                  |   4|
|`$F006`|MD5                       |  16|
|`$F007`|SHA-1                     |  20|
|`$F008`|SHA-256                   |  32|
|`$F009`|SHA-512                   |  64|

The table entries have the following format:

|Offset|Size|Description|
|-----:|---:|:----------|
|     0|   2|Algorithm  |
|     2|   2|Size       |
|     4|   4|Location   |

* **Algorithm**: numeric identifier of the algorithm used to calculate the checksum.
  This value MUST be one of the numeric identifiers listed in the table above; values not listed in that table MUST
  NOT be used.
  In particular, this field MUST NOT be set to the [invalid value][def-invalid].
* **Size**: size of the checksum data, in bytes.
  This value MUST match the size expected by the algorithm.
* **Location**: offset into the linked [data block][sect4.1] where the checksum data is found.
  This value MUST be smaller than the linked block's size, and the combination of this field and the size field MUST
  NOT cause the checksum data to extend beyond the end of the linked block.
  Consumers MUST NOT expect any particular alignment for the location specified by this value.

## 5. General constraints

Some constraints apply to all parts of the file where they are relevant.
These constraints mostly refer to invalid or inconsistent data: the goal is to avoid data that cannot be parsed at
all, potentially even leading to security issues.
While these constraints are generally listed where they are relevant and not obvious, in the interest of brevity, they
are sometimes omitted when the context would imply them.

In order to ensure that implementations are aware of these constraints, they are all listed here.
Generators MUST follow all these constraints when generating a file, and consumers MAY reject any file that doesn't
adhere to these constraints as invalid.

* All block references (i.e., block indexes contained either in the linked block or reference fields for some other
  block or within a block's contents) MUST point to a block of the required type, unless the reference is set to the
  [invalid block index][def-invalid].
* All table references (i.e., indexes that refer to some table, either in the same block or in another block) MUST
  point to valid table entries, unless they are set to the [invalid index][def-invalid].
  A valid table entry index MUST be smaller than the containing block's size as specified in the corresponding entry
  of the [master block table][sect3.5] (i.e., the number of **entries** in the block, not the number of **bytes**); if
  the block in question has some way of marking entries as unused, valid table entry indexes MUST NOT point to unused
  entries.
* If a pair of fields in a table specifies a range, and the field that indicates the start of the range is not set to
  the [invalid value][def-invalid], the entire range MUST be contained within the object being referenced.
  In other words, ranges MUST NOT extend beyond the end of their containing object.
* If a field can only contain values from a specified list, that field MUST NOT be set to a value not in that list; in
  particular, unassigned values MUST NOT be used for extensions.
  (The block type field in the [master block table][sect3.5] is an exception to this rule, as it has a range
  explicitly made available for extensions.)
  Extensions MUST use the mechanisms described in [the corresponding section][sect3.6].

## 6. Versioning

This section describes the versioning policy for this document (i.e., the way version numbers are assigned to it), as
well as the semantics of the version information field in the [header][sect3.4] of a debug information file.

### 6.1. Versioning policy

The versioning policy for this document is based on [Semantic Versioning][semver].

Semantic Versioning, as it is, cannot be used directly: the Semantic Versioning specification is written for software,
which this document is not, and it requires a public API, which would either be non-existent or be the entire contents
of this document.
Therefore, a policy that is equivalent in spirit is defined here.

Version numbers consist of three separate numbers: the **major** version number, the **minor** version number and the
**revision** number (equivalent to the patch number in Semantic Versioning).
These numbers are written separated by dots, as in `1.2.3` (meaning major version 1, minor version 2, revision 3).
The current version number for this document is listed at the beginning, under the title heading, along with the date
of publication.
(The word "draft" instead of a date indicates that the document is a draft for the indicated version, not considered
to be the published stated version.)

As long as the major version number is 1 or greater, the following rules are followed when updating the version
number for the document:

* If an update only makes corrections that don't change the intended semantics of the specification, the update will
  increment only the **revision** number.
  This is typically used to indicate clarifications and similar updates that should not affect interoperability.
  Implementations SHOULD accept files with higher revision numbers than expected.
* If an update makes changes to the semantics of the specification, but these changes are compatible with the previous
  version, the **minor** version number is incremented (and the revision number reset to zero).
  A change is considered compatible if an implementation based on the previous version can handle the new version
  without changes, although perhaps with reduced functionality.
  Implementations MAY accept files with higher minor version numbers than expected, although perhaps with reduced
  functionality; implementations that do SHOULD check all [reserved fields][def-reserved] and enumeration values to
  ensure that they are set as they expect; differences in these values indicate behavior they are not aware of.
  For a change to be considered compatible, it MUST follow the following rules:
    * If new fields are added to a structure, the [invalid value][def-invalid] for those fields MUST indicate
      semantics that are identical to the previous version without those fields.
      (This applies both to reserved fields being unreserved and to new fields being added at the end of a structure.)
      As a result, implementations based on the older version can treat these fields as [reserved][def-reserved],
      resulting in identical behavior.
    * New values MAY be added to any enumeration defined by the specification, such as block type identifiers, but
      existing values MUST NOT be modified unless they are designated as reserved.
      This ensures that implementations based on the older version can still process the values they know and use the
      presence of values unknown to them to indicate features they are not aware of.
    * If new block types are defined, those new blocks MUST NOT affect the semantics of existing block types, unless
      a block of the new type is referenced by a block of an existing type either via a new field in a structure or
      via a previously unused linked block or reference field in the [master block table][sect3.5] entry for the block
      of the existing type.
      Therefore, implementations based on the older version can safely ignore blocks of new types they do not
      recognize and process the ones that they do, provided that they check that [reserved fields][def-reserved] and
      unused linked block and reference fields in those blocks are set to the [invalid value][def-invalid], thus
      ensuring that no blocks of new types are being linked by those fields.
    * The semantics of existing fields in existing structures MUST NOT be changed unless a new field in the same
      structure is set to a value other than the [invalid value][def-invalid] or an existing field whose value is an
      enumeration is set to a value that was reserved in the previous version.
      This allows implementations based on the older version to parse the fields they know about as usual, provided
      that they check that [reserved fields][def-reserved] are set to the [invalid value][def-invalid].
* If an update makes incompatible changes to the semantics of the specification, not following the rules laid out
  above, the **major** version number is incremented (and the minor version and revision numbers are reset to zero).
  This is considered a breaking change.
  Implementations MUST NOT accept files with higher major version numbers than expected.

If the major version number is 0, this indicates a work-in-progress version of this specification.
In this case, incompatible changes result in the minor version number being incremented (and the revision number reset
to zero), and compatible changes result in the revision number being incremented.
Implementations that accept version numbers with a major version number of 0 MUST expect the specification to change
at any time in significant ways and SHOULD NOT accept version numbers with a major version of 0 that they don't know
about, regardless of which component of the version number has changed.

[semver]: https://semver.org/

### 6.2. Version information field

The version information field is a 4-byte bit-packed field in the file's [header][sect3.4] that indicates which
version of this specification a debug information file conforms to.
It has the following subfields:

|Start|Bits|Description         |
|----:|---:|:-------------------|
|    0|   8|Revision number     |
|    8|  12|Minor version number|
|   20|  12|Major version number|

If all subfields are set to the [invalid value][def-invalid], then the entire version information field is considered
to be set to the [invalid value][def-invalid].
This value MAY be used internally for testing, to indicate that a certain file does not conform to any published
version of this specification; implementations MUST NOT accept files with a version information field set to the
[invalid value][def-invalid] except in this scenario.

Values for this field where the major version number subfield is set to the [invalid value][def-invalid] (i.e., $FFF)
but at least one of the other subfields isn't are reserved for future versions of this specification; such a
representation can be used by a future version of the specification that needs to change the layout of the field.
Implementations MUST NOT accept files with a reserved value in the version information field.

If the version information field is ever moved to a different location in the file, either the current version
information field will be set to a reserved representation, or the magic number field in the [header][sect3.4] will be
changed.
This ensures that implementations can rely on the version information field to determine whether they can process a
file, without accidentally parsing an unrelated value from a future version of this specification as a version number.

## 7. Implementation guidance

While any implementation that follows the rules and requirements laid out in this document can be considered
conformant, this section contains advice for applications that wish to implement it, in order to maximize
compatibility with other implementations.
Implementations SHOULD follow these guidelines when possible.

### 7.1. Guidance for generators

Generators SHOULD generate as much debug information as possible given the information they have access to.

The file format is designed for easy consumption over easy generation, so it is possible that some information would
be prohibitively expensive to generate.
(For example, [preferred address-to-line mappings][sect4.12] might be non-trivial to compute.)
However, in the general case, this bias towards consumption should not render the file impractical to generate:
building a ROM image is a slow process already, so it should generally be possible to generate a debug information
file alongside it.

On the other hand, as all blocks are optional, generators need not attempt to invent information that they simply
cannot determine from their input.
In those cases, it is usually preferrable to leave out that information entirely.

If a generator does not support all the block types and features defined by this specification, it can still attempt
to generate the information it does support.
Not supporting all the features in this specification does not compromise conformance: as long as the supported
features behave as required, unsupported blocks can be simply left out of the file and unsupported fields can be set
to the [invalid value][def-invalid] (or other values labelled as "unknown") where this specification allows that.

Build toolchains that generate debug information files SHOULD attempt to identify themselves in a machine-readable way
in the toolchain ID string field of the [ROM image information block][sect4.6].
This allows potential consumers to identify both the language in which the program was written in and the specific
flavour or variant of it that the toolchain accepts.

[Address-to-line mappings][sect4.11] and [line-to-address mappings][sect4.10] can be inherently inaccurate, due to
edge cases with a number of language features (such as complicated cases of macro expansion) that lead to an ambiguous
relationship between a line of code and the addresses in the program image it maps to.
Nonetheless, generators SHOULD attempt to generate mappings that are as accurate as possible; they SHOULD also ensure,
whenever possible, that the mappings are true inverses of one another.

### 7.2. Guidance for consumers

Consumers SHOULD process as much information as they can from a debug information file.
In the presence of unknown [extension blocks][sect3.6], they SHOULD ignore the unknown blocks and process the ones
that they do know about.
Consumers SHOULD NOT reject a file due to the presence of such unknown extensions.
Likewise, if a file makes use of features that the consumer does not implement, consumers SHOULD NOT reject the file
for that reason; instead, they SHOULD ignore those features and make use of the features they do implement.

For the reasons explained in [the previous section][sect7.1], [address-to-line mappings][sect4.11] and
[line-to-address mappings][sect4.10] can be slightly inconsistent.
Consumers MUST be aware of this potential problem and SHOULD fail gracefully in that case.

Consumers MUST NOT expect debug information files to fulfill conditions that are not required in this specification,
such as a specific order of blocks.
Instead, they MUST use the mechanisms described in this specification to discover and access the data in a debug
information file.

The file format is designed for fast random access, not sequential access.
This enables consumers to locate the data they want directly while minimizing the amount of unneeded data they need to
parse to do so.
Therefore, consumers SHOULD be prepared to access the file in a random manner (for example, by keeping the entire file
in memory if possible).

### 7.3. Guidance for defining extensions

Extensions to this format MUST be defined by making use of [extension blocks][sect3.6].
Other features, such as [reserved fields][def-reserved] or [comment blocks][sect4.4], MUST NOT be used to define
extensions.

Generators that emit [extension blocks][sect3.6] MUST be prepared for consumers to ignore those blocks.
Therefore, the information in those blocks MUST NOT change the semantics of [standard blocks][sect4]; it SHOULD NOT
change the semantics of other unrelated [extension blocks][sect3.6].

In this specification, the relationship between blocks is explicit.
[Extension blocks][sect3.6] SHOULD follow this guideline as well.
In particular, if a block needs to make use of information in another block, it SHOULD reference that block, either
through the linked block or reference fields of the first block, or through some field in its contents.
(In other words, blocks SHOULD NOT assume that other blocks that they do not reference explicitly exist at all.)
[Standard blocks][sect4] defined in this specification always follow this rule.

[Extension blocks][sect3.6] that need to reference data that can be defined by a [standard block][sect4] SHOULD do so
instead of defining a new extension block to contain identical data.
For example, if an [extension block][sect3.6] needs to make use of strings, it SHOULD use a [string table][sect4.2] as
defined by this specification instead of defining another [extension block][sect3.6] to contain strings.

[Extension blocks][sect3.6] are identified by their identification strings.
Of these strings, the originator string acts as a namespace controlled by the author of the extension; this string
SHOULD be chosen so as to be unique.
(For example, the URL of the document that defines the extension is a good choice of an originator string.)
On the other hand, the type string only needs to be unique within the namespace defined by the originator string;
therefore, a short, human-readable name SHOULD be chosen for the type string.

Extensions SHOULD publish documents that specify them.
Following the [conventions][sect2.1] and layout of this specification (as much as possible) is RECOMMENDED for
consistency with this specification and with other extension documents.

* * *

## Annex A — Checksum algorithms

**This annex is normative.**

This annex specifies and defines all of the checksum algorithms that [checksum table blocks][sect4.13] can use.
Those algorithms are grouped into separate sections in this annex for reading convenience.

### A.1. Simple summation algorithms

These algorithms are true checksums: their values result from adding up the values in the data being checksummed.
The ones supported by this specification are:

* **Byte sum**: simple sum of all the bytes in the data, as a 1-byte value.
  The checksum is the lower 8 bits of the sum of all individual bytes in the data; this is equivalent to adding all
  the bytes and handling overflows via wrap-around.
* **2-byte sum**: sum of all the 2-byte values in the data; values are read in little-endian format.
  This is equivalent to the byte sum, but handling the input two bytes at a time.
  If there is an odd number of bytes in the data, it is padded by appending a $00 byte at the end.
  The result of this checksum is a 2-byte value; it is stored in little-endian form.
* **4-byte sum**: sum of all the 4-byte values in the data; values are read in little-endian format.
  This is equivalent to the one above, but handling the input four bytes at a time.
  Input data is likewise padded to a multiple of four bytes by appending $00 bytes as needed.
* **Game Boy header checksum**: sum of all the bytes in the data as a 2-byte value.
  This algorithm is supported because it is likely to be already implemented by implementations.
  (Note that applying this algorithm to a ROM image without changes will not result in the ROM image's header checksum
  because the ROM image already contains its own checksum.)
  The checksum is the lower 16 bits of the sum of all individual bytes in the data; this is equivalent to
  zero-extending all the bytes to 2-byte values and adding them all up, handling overflows via wrap-around.
  Unlike in the ROM header, the checksum is stored as a little-endian value.

### A.2. CRC-32

This is one of the most common cyclic redundancy checks currently in use, and therefore one that implementors are
likely to find existing implementations for.

The algorithm is fully specified by [ITU-T recommendation V.42][itu-t-v42], section 8.1.1.6.2, with an initial value
of $FFFFFFFF and the result complemented.
A practical implementation of this algorithm can be found in the [PNG specification][png], which uses it to verify the
contents of its chunks.

The result of this calculation is a 4-byte integer, which is stored in little-endian format.

[itu-t-v42]: https://www.itu.int/rec/T-REC-V.42-200203-I/en
[png]: http://www.libpng.org/pub/png/spec/1.2/PNG-CRCAppendix.html

### A.3. Adler-32 checksum

This is a checksum that is simple to define, but more robust than the [simple summation checksums][annexA.1] given
above.
While many algorithms like this exist, this one was chosen because it is generally well-known and implementations are
often already available, as it is used by [the ZLIB compressed data format][rfc1950].

The checksum calculates two values, referred as the _first-order sum_ and _second-order sum_ in this section.
The first-order sum is initialized to 1 and the second-order sum is initialized to 0.
For each byte in the input data, the byte (as an unsigned integer) is added to the first-order sum, and the new value
of the first-order sum is added to the second-order sum.
Finally, both sums are reduced modulo $FFF1.
(The modulo reduction can be done after any step to prevent overflow.
If the sums are tracked as 4-byte integers, reductions will be necessary after no less than 5,552 steps; if the sums
are tracked as 8-byte integers, reductions will be necessary after no less than 380,368,439 steps.)

The final checksum is this pair of sums after modulo reduction, stored as 2-byte little-endian values: the first-order
sum is stored first, and the second-order sum is stored afterwards.
(Some specifications indicate the output as a single 4-byte value with the second-order sum left-shifted by 16 bits
and added to the first-order sum.
This formulation is equivalent.)

[rfc1950]: https://www.rfc-editor.org/rfc/rfc1950

### A.4. Hash algorithms

These are standard hash algorithms that are designed for security purposes, such as digital signatures.
However, their usage is common to validate whether the contents of a file have been tampered with, and thus they are
provided as options here.

These algorithms are defined by separate specification documents:

* **MD5**: defined by IETF's [RFC 1321][rfc1321].
* **SHA-1**: defined by NIST's [FIPS 180-1][fips180-1], as well as some subsequent editions of that publication.
* **SHA-256** and **SHA-512**: defined by NIST's [FIPS 180-2][fips180-2], as well as some subsequent editions of that
  publication.

These algorithms result in a number of 4-byte or 8-byte integers that must be converted to a byte string to determine
the final checksum.
This conversion is done as specified by those documents: in particular, MD5 specifies a little-endian conversion,
while SHA-1, SHA-256 and SHA-512 specify a big-endian conversion.
This matches the way these checksums are normally specified and rendered by users.
The length of each checksum's byte string is given by the table of valid algorithms in the
[checksum table block type][sect4.13] section.

[rfc1321]: https://www.rfc-editor.org/rfc/rfc1321
[fips180-1]: https://csrc.nist.gov/pubs/fips/180-1/final
[fips180-2]: https://csrc.nist.gov/pubs/fips/180-2/final

[sect1]: #1-objective-and-scope
[sect2]: #2-introduction
[sect2.1]: #21-conventions
[sect3]: #3-definitions-and-components
[sect3.1]: #31-format-overview
[sect3.2]: #32-definitions
[sect3.3]: #33-blocks
[sect3.4]: #34-header
[sect3.5]: #35-master-block-table
[sect3.6]: #36-extension-blocks
[sect4]: #4-standard-block-types
[sect4.1]: #41-data-block-type
[sect4.2]: #42-string-table-block-type
[sect4.3]: #43-extension-block-descriptor-block-type
[sect4.4]: #44-comment-block-type
[sect4.5]: #45-source-file-table-block-type
[sect4.6]: #46-rom-image-information-block-type
[sect4.7]: #47-file-stack-block-type
[sect4.8]: #48-section-table-block-type
[sect4.9]: #49-symbol-table-block-type
[sect4.10]: #410-line-to-address-mapping-table-block-type
[sect4.11]: #411-address-to-line-mapping-table-block-type
[sect4.12]: #412-preferred-address-to-line-mappings-block-type
[sect4.13]: #413-checksum-table-block-type
[sect5]: #5-general-constraints
[sect6]: #6-versioning
[sect6.1]: #61-versioning-policy
[sect6.2]: #62-version-information-field
[sect7]: #7-implementation-guidance
[sect7.1]: #71-guidance-for-generators
[sect7.2]: #72-guidance-for-consumers
[sect7.3]: #73-guidance-for-defining-extensions
[annexA]: #annex-a--checksum-algorithms
[annexA.1]: #a1-simple-summation-algorithms
[annexA.2]: #a2-crc-32
[annexA.3]: #a3-adler-32-checksum
[annexA.4]: #a4-hash-algorithms
[def-invalid]: #invalid-value
[def-numbers]: #numbers
[def-reserved]: #reserved-fields
[def-strings]: #strings
