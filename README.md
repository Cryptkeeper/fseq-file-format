# FSEQ File Format
A work-in-progress decoding and documentation of the *version 2* FSEQ (`.fseq`, "Falcon sequence") file format, derived from the [Falcon Player/fpp](https://github.com/FalconChristmas/fpp) software implementation and its usage in [xLights](https://github.com/smeighan/xLights). FSEQ is a time series file format used to describe channel output states, typically for controlling lighting equipment.

While documentation for sister formats or previous versions (such as ESEQ, PSEQ or FSEQ version 1) will be eventually added, they are currently undocumented. Until then, this documentation should assist the brave in decoding them manually.

## Encoding
FSEQ files are encoded in [little-endian](https://en.wikipedia.org/wiki/Endianness) format.

## Structure
### Header
Length is 32 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `[4]uint8` | Identifier | Always `FSEQ`* |
| 4 | `uint16` | Channel Data Offset | Byte index of the channel data portion of the file |
| 6 | `uint8` | Minor Version | Currently `0x00` |
| 7 | `uint8` | Major Version | Currently `0x02` |
| 8 | `uint16` | Header Length | Length of the header (32 bytes) + `Frame Block Count` * length of a `Frame Block` (8 bytes) |
| 10 | `uint32` | Channel Count | |
| 14 | `uint32` | Frame Count | |
| 18 | `uint8` | Step Time | Timing interval in milliseconds |
| 19 | `uint8` | Flags | Unused by the [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights) implementation |
| 20 | `uint8` | Compression Type | 0 = none, 1 = [zstd](https://github.com/facebook/zstd), 2 = [zlib](https://www.zlib.net/) |
| 21 | `uint8` | Frame Block Count | Ignored if `Compression Type` = 0 |
| 22 | `uint8` | Sparse Range Count | |
| 23 | `uint8` | | (Reserved for future use) |
| 24 | `uint64` | Unique ID | Implemented as the creation time in microseconds |

*`FSEQ` and `PSEQ` appear to be interchangable versions of the same file format. However, their structures may differ according to their major or minor version fields.

### Data
Variable length, determined by `Header->Channel Data Offset` - length of `Header` (32 bytes).

| Data Type | Corresponding Count Field |
| --- | --- |
| `[]Frame Block` | `Header->Frame Block Count` |
| `[]Sparse Range` | `Header->Sparse Range Count` |
| `[]Variable`| None |

When reading the `[]Variable` data, a software implementation should instead continue to read as long as there is at least 4 bytes free between the current reader index and `Header->Channel Data Offset`. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L425) code for more context.

If (after reading all fields) the current reader index is less than the `Header->Channel Data Offset` value, it should seek forward (up to 4 bytes) to `Header->Channel Data Offset`. A difference of more than 4 bytes likely denotes a decoding error in your implementation. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1458-L1462) code for more context.

Both of these implementation details appear to stem from the [rounding behavior](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1433) for the `Channel Data Offset`.

#### Frame Block
Length is 8 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `uint32` | Frame | |
| 4 | `uint32` | Length | |

#### Sparse Range
Length is 6 bytes.

| Byte Index | Data Type | Field Name |
| --- | --- | --- |
| 0 | `uint24` | Start Channel |
| 3 | `uint24` | End Channel Offset |

##### Example
To denote channels 16-32, `Start Channel` would have a value of 15 (16 - 1 since channel indexes start at 0) and an `End Channel Offset` of 16 (32 - 16 = 16).

#### Variable
Variable length, at least 4 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `uint16` | Length | Length of `Data` + a 4 byte header | 
| 2 | `[2]uint8` | Code | Two character string name |
| 4 | `[Data Length - 4]uint8` | Data | Typically used as a string |

While effectively useless, the [fpp](https://github.com/FalconChristmas/fpp) implementation seems to support zero length variables. However when reading it skips forward 4 bytes, ignoring the code field and resulting in a variable named "NULNUL" (`[0x00, 0x00]`).

##### Common Variable Codes
| Bytes | Code | Name | Description |
| --- | --- | --- | --- |
| `[0x6D, 0x66]` | `mf` | Media File | File path of the audio to play |
| `[0x73, 0x70]` | `sp` | Sequencing Program | Identifies the program used to create the sequence |

xLights sample 

##### Example
The variable `mf` (Media File) with a value of "xy" would be encoded in 6 bytes.

| Bytes | Description |
| --- | --- |
| `[0x00, 0x06]` | 6 byte length (2 bytes of data + 4 byte header) |
| `[0x6D, 0x66]` | 2 byte code (`mf`) |
| `[0x78, 0x79]` | 2 bytes of data ("xy") |

## Reference Implementations
* [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp) is a C++ implementation of the FSEQ file format. It is the project which also originated the file format and maintains it.
* [xLights](https://github.com/smeighan/xLights/blob/master/xLights/FSEQFile.cpp) is a C++ sequencing program which uses the FSEQ file format. However, its implementation is a copy/paste of the fpp code and provides no additional context.