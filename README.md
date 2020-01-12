# FSEQ File Format
A work-in-progress, third-party reverse engineering of the *version 2* FSEQ (`.fseq`) "Falcon sequence" file format, derived from the [Falcon Player](https://github.com/FalconChristmas/fpp) ("fpp") software implementation and its usage in [xLights](https://github.com/smeighan/xLights). FSEQ is a time series file format used to describe channel output states, typically for controlling lighting equipment.

Some first-party documentation (while sparse) concerning the file format, as well as its sister format [ESEQ](https://github.com/FalconChristmas/fpp/blob/master/docs/ESEQ_Effect_Sequence_file_format.txt), is available in the [fpp repository](https://github.com/FalconChristmas/fpp/blob/master/docs/FSEQ_Sequence_File_Format.txt).

Given the reverse engineered nature, this documentation should be considered incomplete, incorrect and outdated.

## Encoding
FSEQ files are encoded in [little-endian](https://en.wikipedia.org/wiki/Endianness) format.

## Structure
### Header
Length is 32 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `[4]uint8` | Identifier | Always `PSEQ` (older encodings may contain `FSEQ`) |
| 4 | `uint16` | Channel Data Offset | Byte index of the channel data portion of the file |
| 6 | `uint8` | Minor Version | Currently `0x00` |
| 7 | `uint8` | Major Version | Currently `0x02` |
| 8 | `uint16` | Header Length | Length of the header (32 bytes) + `Frame Block Count` * length of a `Frame Block` (8 bytes) |
| 10 | `uint32` | Channel Count | Sum of `Sparse Range` lengths |
| 14 | `uint32` | Frame Count | |
| 18 | `uint8` | Step Time | Timing interval in milliseconds |
| 19 | `uint8` | Flags | Unused by the [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights) implementations |
| 20 | `uint8` | Compression Type | 0 = none, 1 = [zstd](https://github.com/facebook/zstd), 2 = [zlib](https://www.zlib.net/) |
| 21 | `uint8` | Frame Block Count | Ignored if `Compression Type` = 0 |
| 22 | `uint8` | Sparse Range Count | |
| 23 | `uint8` | | (Reserved for future use) |
| 24 | `uint64` | Unique ID | Implemented as the creation time in microseconds |

### Data Tables
Variable length, determined by `Header->Channel Data Offset` - length of `Header` (32 bytes).

| Data Type | Corresponding Count Field |
| --- | --- |
| `[]Frame Block` | `Header->Frame Block Count` |
| `[]Sparse Range` | `Header->Sparse Range Count` |
| `[]Variable`| None |

When reading the `[]Variable` data, a software implementation should instead continue to read as long as there is at least 4 bytes free between the current reader index and `Header->Channel Data Offset`. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L425) source code for more context.

If (after reading all fields) the current reader index is less than the `Header->Channel Data Offset` value, it should seek forward (up to 4 bytes) to `Header->Channel Data Offset`. A difference of more than 4 bytes likely denotes a decoding error in your implementation. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1458-L1462) source code for more context.

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

A `Sparse Range` is a channel range, defined by its starting channel and the range count. Channel indexes start at 0.

##### Example
To denote channels 16-32, `Start Channel` would have a value of 15 (16 - 1 since channel indexes start at 0) and an `End Channel Offset` of 16 (32 - 16 = 16).

#### Variable
Variable length, at least 4 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `uint16` | Length | Length of `Data` + a 4 byte header | 
| 2 | `[2]uint8` | Code | Two character string name |
| 4 | `[Data Length - 4]uint8` | Data | Typically used as a string |

While effectively useless, the [fpp](https://github.com/FalconChristmas/fpp) implementation seems to support zero length variables. However when reading it skips forward 4 bytes, ignoring the `Code` field and resulting in a variable named "NULNUL" (`[0x00, 0x00]`).

##### Common Variable Codes
| Bytes | Code | Name | Description |
| --- | --- | --- | --- |
| `[0x6D, 0x66]` | `mf` | Media File | File path of the audio to play |
| `[0x73, 0x70]` | `sp` | Sequence Producer | Identifies the program used to create the sequence |

These (2) variable codes are currently the only codes referenced by the [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights) implementations. However, there is no validation that prevents third-party software or users from defining and using their own codes. The caveat being that they may clash with future additions given the limited namespace availability.

##### Data Sample
xLights `sp` ("Sequence Producer") data sample: `xLights Macintosh 2019.22`

##### Encoding Example
The variable `mf` (Media File) with a value of "xy" would be encoded in 6 bytes.

| Bytes | Description |
| --- | --- |
| `[0x00, 0x06]` | 6 byte length (2 bytes of data + 4 byte header) |
| `[0x6D, 0x66]` | 2 byte code (`mf`) |
| `[0x78, 0x79]` | 2 bytes of data ("xy") |

### Channel Data
The following describes _uncompressed_ channel data. Compressed channel data follows the same format, however it is split across compression blocks (with up to 255 compression blocks) which each contain several frames.

Variable length, `Header->Channel Count` * `Header->Frame Count` bytes. 

Each byte represents the channel state for its given index (remember to apply its corresponding `Sparse Range->Start Channel` offset, if applicable).

`Channel Data` starts at `Header->Channel Data Offset` and continues to the end of the file. A software implementation may easily validate a file by ensuring that `Header->Channel Data Offset` + `Header->Channel Count` * `Header->Frame Count` matches the full byte length of the file.

#### Seeking
A software implementation can seek to a specific frame by taking the product of the frame index and the length of each frame.

```
var targetFrameIndex = 9 // Frame #10
var absoluteSeek = Header->Channel Data Offset // Skip the header and data tables
absoluteSeek += targetFrameIndex * Header->Channel Count
```

#### Encoding Example
A controller with 4 channels (indexes 0-3) would have its data encoded as `[4]uint8` _per frame_. Given a sequence with 10 total frames, the channel data length would be 40 bytes. How that data is interpreted is controlled by the controller's corresponding [channeloutput](https://github.com/FalconChristmas/fpp/tree/master/src/channeloutput) class. For example, the [Light-O-Rama channeloutput](https://github.com/FalconChristmas/fpp/blob/master/src/channeloutput/LOR.cpp#L243), and many others, interpret the `uint8` as a brightness level (with RGB simply using a channel per color).

## Odds & Ends
- The first compression block will only contain 10 frames. Comments within the [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1129) indicates this is done to ensure the program can started quicker.
- Even when the full sequence is compressed, the first frame is excluded ([depending on the zlib version](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1086)).
- [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L774) attempts to allocate 64KB compression blocks. However instead of `64 * 1024`, it uses `64 * 2014` which allocates 125.8KB compression blocks. While this may be a typo, it enables compressed channel data to be up to 31.3MB in length.

## Reference Implementations
* [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp) is a C++ implementation of the FSEQ file format. It is the project which also originated the file format and maintains it.
* [xLights](https://github.com/smeighan/xLights/blob/master/xLights/FSEQFile.cpp) is a C++ sequencing program which uses the FSEQ file format. However, its implementation is a copy/paste of the fpp source code and provides no additional context.
* [go-fseq](https://github.com/Cryptkeeper/go-fseq) is a Go library implemented given this documentation.
