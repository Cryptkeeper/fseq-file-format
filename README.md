# FSEQ File Format
A third-party reverse engineering of the *version 2* FSEQ (`.fseq`) "Falcon sequence" file format, derived from the [Falcon Player](https://github.com/FalconChristmas/fpp) ("fpp") software implementation and its usage in [xLights](https://github.com/smeighan/xLights). FSEQ is a time series file format used to describe channel output states, typically for controlling lighting equipment.

First-party documentation concerning the file format, as well as the [ESEQ](https://github.com/FalconChristmas/fpp/blob/master/docs/ESEQ_Effect_Sequence_file_format.txt) file format, is available in the [fpp repository](https://github.com/FalconChristmas/fpp/blob/master/docs/FSEQ_Sequence_File_Format.txt).

Given the reverse engineered nature, this documentation should be considered incomplete, incorrect and outdated.

## Encoding
FSEQ files are encoded in [little-endian](https://en.wikipedia.org/wiki/Endianness) format.

## Diagram
```
                            ┌────────────────┬────────────────┬────────────────┬────────────────┐
                            │'P'         0x50│'S'         0x53│'E'         0x45│'Q'         0x51│
                            ├────────────────┴────────────────┼────────────────┼────────────────┤
                            │Channel Data Offset        uint16│Minor Version   │Major Version  2│
┌────────────────────────┐  ├─────────────────────────────────┼────────────────┴────────────────┤
│Extended Compression    │  │Variable Data Offset       uint16│Channel Count              uint32
│Block Count (v2.1) uint4│  ├─────────────────────────────────┼─────────────────────────────────┤
├────────────────────────┤   Channel Count (contd)      uint32│Frame Count                uint32
│Compression Type   uint4│  ├─────────────────────────────────┼────────────────┬────────────────┤
└────────────────────────┘   Frame Count (contd)        uint32│Step Time Ms.   │Flags (unused)  │
             │              ├────────┬───────┬────────────────┼────────────────┼────────────────┤
             └─────────────▶│ECBC    │CT     │Compr. Block Cnt│Sparse Range Cnt│Reserved        │
                            ├────────┴───────┴────────────────┴────────────────┴────────────────┤
                            │Unique ID/Creation Time Microseconds                         uint64
                            ├───────────────────────────────────────────────────────────────────┤
                             Unique ID/Creation Time Microseconds (contd)                 uint64│
                            └───────────────────────────────────────────────────────────────────┘
```

## Structure
### Header
Length is 32 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `[4]uint8` | Identifier | Always `PSEQ` (older encodings may contain `FSEQ`) |
| 4 | `uint16` | Channel Data Offset | Byte index of the channel data portion of the file |
| 6 | `uint8` | Minor Version | Normally `0x00`, optionally `0x01` is required to enable support for [Extended Compression Blocks](#extended-compression-blocks) (see [xLights@e33c065](https://github.com/smeighan/xLights/commit/e33c0651aa6886d2ab10c04cb83ef1d1fdd25062)) |
| 7 | `uint8` | Major Version | Currently `0x02` |
| 8 | `uint16` | Header Length | Address of first variable, length of the header (32 bytes) + `Compression Block Count` * length of a `Compression Block` (8 bytes) + `Sparse Range Count` * length of a `Sparse Range` (12 bytes) |
| 10 | `uint32` | Channel Count | Sum of `Sparse Range` lengths |
| 14 | `uint32` | Frame Count | |
| 18 | `uint8` | Step Time | Timing interval in milliseconds |
| 19 | `uint8` | Flags | Unused by the [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights) implementations |
| 20 | `uint4` | Compression Block Count | Upper 4 bits, likely 0 |
| 20 | `uint4` | Compression Type | 0 = none, 1 = [zstd](https://github.com/facebook/zstd), 2 = [zlib](https://www.zlib.net/) |
| 21 | `uint8` | Compression Block Count | Lower 8 bits, ignored if `Compression Type` = 0 |
| 22 | `uint8` | Sparse Range Count | |
| 23 | `uint8` | | (Reserved for future use) |
| 24 | `uint64` | Unique ID | Implemented as the creation time in microseconds |

### Data Tables
Variable length, determined by `Header->Channel Data Offset` - length of `Header` (32 bytes).

| Data Type | Corresponding Count Field |
| --- | --- |
| `[]Compression Block` | `Header->Compression Block Count` |
| `[]Sparse Range` | `Header->Sparse Range Count` |
| `[]Variable`| None |

When reading the `[]Variable` data, a software implementation should instead continue to read as long as there is at least 4 bytes free between the current reader index and `Header->Channel Data Offset`. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L425) source code for more context.

If (after reading all fields) the current reader index is less than the `Header->Channel Data Offset` value, it should seek forward (up to 4 bytes) to `Header->Channel Data Offset`. A difference of more than 4 bytes likely denotes a decoding error in your implementation. See the [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1458-L1462) source code for more context.

Both of these implementation details appear to stem from the [rounding behavior](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1433) for the `Channel Data Offset`.

#### Compression Block
Length is 8 bytes.

| Byte Index | Data Type | Field Name | Notes |
| --- | --- | --- | --- |
| 0 | `uint32` | First Frame Number | Index of the first frame encoded in the block |
| 4 | `uint32` | Length | Length in bytes of the compressed block |

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

The `Data` field may be null terminated depending on the encoding program. If your programming language does not null terminate its strings, your `Data` array should instead be `[Data Length - 4 - 1]uint8`.

##### Common Variable Codes
| Code | Name | Description | Example |
| --- | --- | --- | --- |
| `mf` | Media File | File path of the audio to play | `C:\test.mp3` |
| `sp` | Sequence Producer | Identifies the program used to create the sequence | `xLights Macintosh 2019.22` |

These (2) variable codes are currently the only codes referenced by the [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights) implementations. However, there is no validation that prevents third-party software or users from defining and using their own codes. The caveat being that they may clash with future additions given the limited namespace availability.

**Author's Note**: Use of the `mf` variable should be discouraged. Encoding the media file path directly into the sequence results in path breaks when moving files, requiring a re-export of the sequence file. As a minor security flaw, `mf` may expose the user's file structure unintentionally when distributing sequences. Software implementations should instead store this in an external configuration, only using the `mf` value if present and valid, as a configuration fallback. 

##### Recommended Variable Codes
| Code | Name | Description |
| --- | --- | --- |
| `an` | Author Name | Name of the file's author |
| `ae` | Author Email | Email address of the file's author |
| `aw` | Author Website | Website URL of the file's author |
| `ad` | Authorship Date | ISO-8601 compliant date & timestamp of the file's authorship |

These variable codes are not supported by either the [fpp](https://github.com/FalconChristmas/fpp) or the [xLights](https://github.com/smeighan/xLights) implementations, but recommended for new software implementations as a method for optionally storing authorship metadata within the existing file format.

##### Encoding Example
The variable `mf` (Media File) with a value of "xy" would be encoded in 6 bytes.

| Bytes | Description |
| --- | --- |
| `[0x00, 0x06]` | 6 byte length (2 bytes of data + 4 byte header) |
| `[0x6D, 0x66]` | 2 byte code (`mf`) |
| `[0x78, 0x79]` | 2 bytes of data ("xy") |

### Compressed Channel Data
For compressed FSEQ files, the channel data is written normally as uncompressed channel data, and then split into fixed-size chunk allocations which are then individually compressed (with up to 4095 of these chunks per file). This enables software implementations to decompress chunks of the file without buffering the full file length.

#### Extended Compression Blocks
The `Compression Block Count` value is split across two seperate fields. [This change](https://github.com/smeighan/xLights/commit/9d09555728f43c863ab24118ba901f4ae45dc3c5#diff-6f85e85fd47664285c3b8811d79aff161ebce060186e33ddea44e1f38f121283R1412) was done to increase the compression block limit to 4095 (previously 255) without extreme breakage in backwards compatability. Although the current minor version is `0x00`, a minor version greater than or equal to `0x01` is required to enable support within xLights. See [xLights@e33c065](https://github.com/smeighan/xLights/commit/e33c0651aa6886d2ab10c04cb83ef1d1fdd25062).

`Compression Block Count` should be treated as a `uint16` value, however only the lower 12 bits are used. The upper 4 bits (of the lower 12 used bits) are stored as the upper 4 bits of `Compression Type` (the "extended" compression block count). As such, it is important when working with the `Compression Type` field to ignore the lower 4 bits (which store the compression type enum value) and shift the upper 4 bits to index 0. The lower 8 bits are stored within the pre-existing `Compression Block Count` field.

##### Code Examples
```
// Prefix the 8-bit compressionBlockCount field with the upper 4 bits of compressionType as a 12-bit int within a uint16_t,
//  this 4 highest order bits will remain zero
//
//   +-> unused highest 4 bits,
//   |   set to zero
//   |
// +------------------+
// |0000|    |        +--> original 8 bit
// +------------------+    `compressionBlockCount`
//         |
//         |
//         +-> upper 4 bits of
//             `compressionType`
//
uint16_t extendedCompressionBlockCount = ((compressionType & 0xF0) << 4) | compressionBlockCount;
```

```
// Strip the 4-bit prefix and re-align it with the 4-bit compressionType, encode the remaining 8 bits as the compressionBlockCount
uint8_t encodedCompressionType = ((extendedCompressionBlockCount & 0x0F00) >> 4) | (compressionType & 0xF);
uint8_t encodedCompressionBlockCount = extendedCompressionBlockCount & 0x00FF;
```

#### Odds & Ends
- The first `Compression Block` will only contain 10 frames. Comments within the [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1129) indicates this is done to ensure the program can be started quicker.
- When compressed using zstd, [fpp](https://github.com/FalconChristmas/fpp) & [xLights](https://github.com/smeighan/xLights/blob/master/xLights/FSEQFile.cpp) do not include the [Zstandard frame header](https://github.com/facebook/zstd/blob/master/doc/zstd_compression_format.md#zstandard-frames). In its absence, some libraries (such as the [Python zstandard library](https://pypi.org/project/zstandard/#id2)) may require an explicit content length be provided.
- [Depending on the zlib version](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1086), the first `Compression Block` might not be compressed, even if the remaining data is.
- [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L781) attempts to allocate 64KB compression blocks. However instead of `64 * 1024`, it uses `64 * 2014` which allocates 125.8KB compression blocks. **Update: This bug has been fixed. This note is preserved for sequences encoded prior to the fix.**

#### Encoding Example
A compressed file (in this example, with `Compression Type` = 1/zstd) with a length of 50,454 bytes reports a `Compression Block Count` of 4. A software implementation should begin by reading the `Compression Block` values that follow the 32 byte `Header`.

```
Block #0
First Frame Number: 0
Length: 18

Block #1
First Frame Number: 10
Length: 50268

Block #2
First Frame Number: 0
Length: 0

Block #3
First Frame Number: 0
Length: 0
```

The [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1508) immediately discards any `Compression Block` values with a length of 0 (they appear to be a product of memory alignment), leaving us with the initial 2 values.

The start address (relative to `Header->Channel Data Offset`) of each `Compression Block` can be calculated by summing the `Length` value of all previous `Compression Block` values. The end address can be calculated by adding the `Length` value to the previously calculated start address.

```
Block #0
First Frame Number: 0
Length: 18
Relative Start Address: 0
Relative End Address: 18

Block #1
First Frame Number: 10
Length: 50268
Relative Start Addressing: 18
Relative End Address: 50286

(Blocks #2 & #3 ignored due to Length = 0)
```

To help validate software implementations, the end address of the last `Compression Block` + `Header->Channel Data Offset` should match the byte length of the file.

If no `Compression Block` values were read, the [fpp source code](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp#L1522) will consider the file corrupted. It will attempt to recover the file by inserting a `Compression Block` with a `First Frame Number` value of 0 and a `Length` value of the file's byte length - `Header->Channel Data Offset`.

### Uncompressed Channel Data
Variable length, `Header->Channel Count` * `Header->Frame Count` bytes. 

For each frame between [0, `Header->Frame Count`) a software implementation should read a  `[Header->Channel Count]uint8` array. Each `uint8` within this array represents the channel state for its given index. If applicable, remember to apply its corresponding `Sparse Range->Start Channel` offset.

Uncompressed channel data starts at `Header->Channel Data Offset` and continues to the end of the file. A software implementation may easily validate the file by ensuring that `Header->Channel Data Offset` + `Header->Channel Count` * `Header->Frame Count` matches the full byte length of the file.

#### Seeking
A software implementation can seek to a specific frame by taking the product of the frame index and the length of each frame.

```
var targetFrameIndex = 9 // Frame #10
var absoluteSeek = Header->Channel Data Offset // Skip the header and data tables
absoluteSeek += targetFrameIndex * Header->Channel Count
```

#### Encoding Example
A controller with 4 channels (indexes 0-3) would have its data encoded as `[4]uint8` _per frame_. Given a sequence with 10 total frames, the channel data length would be 40 bytes. How that data is interpreted is controlled by the controller's corresponding [channeloutput](https://github.com/FalconChristmas/fpp/tree/master/src/channeloutput) class. For example, the [Light-O-Rama channeloutput](https://github.com/FalconChristmas/fpp/blob/master/src/channeloutput/LOR.cpp#L243), and many others, interpret the `uint8` as a brightness level (with RGB simply using a channel per color).

## Reference Implementations
* [fpp](https://github.com/FalconChristmas/fpp/blob/master/src/fseq/FSEQFile.cpp) is a C++ implementation of the FSEQ file format. It is the project which also originated the file format and maintains it.
* [xLights](https://github.com/smeighan/xLights/blob/master/xLights/FSEQFile.cpp) is a C++ sequencing program which uses the FSEQ file format. However, its implementation is a copy/paste of the fpp source code and provides no additional context.
* [libtinyfseq](https://github.com/Cryptkeeper/libtinyfseq) is a header-only C library for decoding fseq files.
