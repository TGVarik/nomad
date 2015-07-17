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

|Offset|Type  |Example|Description
|:----:|:----:|:-----:|:----------
|`0x00`|UInt32|`00 00 03 38`|Offset, from the beginning of the file, to the beginning of the binary portion of the described chunk.
|`0x04`|UInt32|`00 00 00 01`|The record number of the first star record in the identified chunk.

If the file contains fewer chunks than would fill the chunk index, the remaining record positions are nulled so that the chunk index always has a length of 640 bytes.
The chunk index contains n+1 non-zero chunk index records, where n is the number of chunks in the file. The final non-zero chunk index record has an offset value equal to the length of the file (thus pointing to the byte after the last byte of the file) and a star record number one more than the last star record number identified in the file header. This final, one-beyond entry in the index is required, so each file may have up to 79 chunks.
Note that files with a 26-byte fixed record length have only one chunk each. Thus the chunk index will contain only two non-zero chunk index records, with the remainder of the index nulled.

