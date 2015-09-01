# The USNO NOMAD1 Star Catalog #

This document describes the binary data format used in [the USNO NOMAD1 Star Catalog](http://vizier.cfa.harvard.edu/viz-bin/ftp-index?/ftp/cats/bincats/NOMAD1).
### Overview ###

|Offset |Length   |Description
|:-----:|:-------:|:----------
|`0x000`|160 bytes|The file header
|`0x0A0`|640 bytes|The chunk index
|`0x320`|variable |The chunk table

### Header ###
The file begins with an ASCII header, `0xA0` bytes long, ending in a {LF}. The header contains data important in interpreting the remainder of the file. If all of the information does not fill the available 159 characters, it is right-padded with spaces before the {LF}, which is always the 160th byte of the file.

For example, here is the header from `bindat/060/N0608.bin`:

```
NOMAD-1.0(25) 060/N0608.bin 0000001-1076940  pm=-7500 mag=-10000 Ep=+19010 xtra=2938,0 UCAC2=48868(9999999/19808600) Tyc2=1998                                 {LF}
```

| Component          | Description
|:------------------:|:------------
|`NOMAD‑1.0`         |The format identifier. Always `NOMAD‑1.0`.
|`(25)`              |The length, in bytes, of the fixed portion of each star record in the file. Usually `25`, but sometimes `26` near the poles. `25`-type files contain at most 79 chunks, while `26`-type files contain only one chunk.
|`060/N0608.bin`     |The file identifier.
|`0000001-1076940`   |The first and last star record numbers in the file. The first star record number of each file is always `0000001`. The last record number also indicates the number of star records in the file.
|`pm=-7500`          |The proper motion offset, which must be added to the value of each proper motion field in each star record in the file.
|`mag=-10000`        |The magnitude offset, which must be added to the value of each magnitude field in each star record in the file.
|`Ep=+19010`         |The epoch offset, which must be added to the value of each epoch field in each star record in the file.
|`xtra=2938,0`       |The total number each of two-byte and four-byte value overflow fields included in the file across all chunks.
|`UCAC2=48868`       |The number of star records that include UCAC2 identifiers in their variable records.
|`(9999999/19808600)`|The minimum and maximum UCAC2 identifiers in this file.
|`Tyc2=1998`         |The number of star records that include TYCHO2 or HIPPARCOS identifiers in their variable records.

### Chunk Index ###

The header is followed immediately by the chunk index. The chunk index is always 640 bytes long, even in files with 26-byte records, which only have one chunk.

|Offset|Type                |Description
|:----:|:------------------:|-----------
|`0x00`|ChunkIndexRecord[80]|A list of chunk index records

##### Chunk Index Record #####

|Offset|Type  |Example      |Description
|:----:|:----:|:-----------:|:----------
|`0x00`|UInt32|`00 00 03 38`|Offset, from the beginning of the file, to the beginning of the binary portion of the described chunk.
|`0x04`|UInt32|`00 00 00 01`|The record number of the first star record in the identified chunk.

If the file contains fewer chunks than would fill the chunk index, the remaining record positions are nulled so that the chunk index always has a length of 640 bytes.
The chunk index contains n+1 non-zero chunk index records, where n is the number of chunks in the file. The final non-zero chunk index record has an offset value equal to the length of the file (thus pointing to the byte after the last byte of the file) and a star record number one more than the last star record number identified in the file header. This final, one-beyond entry in the index is required, so each file may have up to 79 chunks.
Note that files with a 26-byte fixed record length have only one chunk each. Thus the chunk index will contain only two non-zero chunk index records, with the remainder of the index nulled.

### Chunk ###

#### Chunk Header ####

Each chunk begins with a variable-length header. `offset` is relative to the position given for the chunk by the offset field in the chunk's chunk index record:

|Field   |Offset          |Type          |Example                   |Description
|:-------|:--------------:|:------------:|:------------------------:|:-------------------------------------------
|`head`  |`-0x18`         |Char[26]      |`Chunk#00 0000001‑0000002`|The chunk's ASCII header, always 26 bytes, identifying the chunk number (in the file), the first record in the chunk, and the last record in the chunk. **Note this field begins 24 bytes _before_ the position indicated in the chunk index!**
|`hlen`  |`0x00`          |UInt32        |`00 00 00 30`             |Length in bytes of the chunk header; doubles as relative offset to beginning of accelerator
|`min_id`|`0x04`          |UInt32        |`00 00 00 01`             |Minimum star record ID in the chunk (1)
|`min_ra`|`0x08`          |UInt32        |`00 00 00 00`             |Minimum right ascension in the chunk in mas (0 mas = 0h00m00.000s RA)
|`min_sd`|`0x10`          |UInt32        |`0D 0B D8 00`             |Minimum SPD (south polar distance) in the chunk in mas (218880000 mas = 60º48′00.000″ SPD = -29º12′00.000″ dec)
|`max_id`|`0x14`          |UInt32        |`00 00 0D D8`             |Maximum star record ID in the chunk (3544)
|`max_ra`|`0x18`          |UInt32        |`00 FF FF FF`             |Maximum right ascension in the chunk in mas (16777215mas = 4º39′37.215″ = 0h18m38.481s RA)
|`max_sd`|`0x20`          |UInt32        |`0D 11 56 40`             |Maximum SPD in the chunk in mas (219240000 mas = 60º54′00.000″ SPD = -29º06′00.000″ dec)
|`nxtra2`|`0x24`          |UInt16        |`00 07`                   |Number of short (2-byte) extra values in the header (7)
|`nxtra4`|`0x26`          |UInt16        |`00 00`                   |Number of long (4-byte) extra values in the header (0)
|`xtra4` |`0x28`          |UInt32[nxtra4]|                          |Array of long (4-byte) extra values
|`xtra2` |`0x28` +4*nxtra4|UInt16[nxtra2]|                          |Array of short (2-byte) extra values

Note that there may be (up to two?) null bytes of padding after the end of `xtra2`.

#### Accelerator ####

The header is followed by the accelerator, a lookup structure used by the original NOMAD software for rapidly locating stars by RA within a chunk. The accelerator begins `hlen` bytes after the beginning (discounting the `head` string) of the chunk header. The accelerator is a variable-length table of 12-byte entries of the following format:

|Field   |Type  |Description
|:------:|:----:|:----------
|`offset`|UInt32|Offset from beginning of accelerator to beginning of identified star record
|`id`    |UInt32|ID number of the identified star record
|`ra`    |UInt32|RA in mas of the identified star record

To determine the number of accelerator records, divide `offset` of the first accelerator entry by 12. For example, the first entry in `bindat/060/N0608.bin`'s accelerator for chunk #00 is:

|Field   |Value
|:------:|:----:
|`offset`|`00 00 00 90`
|`id`    |`00 00 00 01`
|`ra`    |`00 00 0D 2A`

This entry gives the offset from the beginning of the accelerator to the beginning of the first star record as `0x90`, or 144 bytes. Because each accelerator record is 12 bytes long, expect a total of 144/12 = 12 accelerator entries. There should be one accelerator entry for every 500 star records in the chunk, but this is not guaranteed.

#### Star Records ####

TODO