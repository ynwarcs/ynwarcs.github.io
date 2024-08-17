---
layout: post
title: pdbconv - PDB compression with MSFZ
---
<script>
addCustomTypes(['MsfzFragment', 'MsfzChunk', 'MsfzStream', 'MsfzHeader']);
addCustomNames(['curFragment', 'chunkIndex', 'chunkData', 'fragmentData', 'outputFragmentData']);
</script>

## contents
---
- [1.0 intro](#10-intro)
- [2.0 msf - recap](#20-msf---recap)
- [3.0 msfz](#30-msfz)
	+ [3.1 definition](#31-definition)
		* [3.1.1 header](#311-header)
		* [3.1.2 directory data](#312-directory-data)
		* [3.1.3 chunk data](#313-chunk-data)
		* [3.1.4 fragment data](#314-fragment-data)
- [4.0 pdbconv](#40-pdbconv)

## 1.0 intro
----
A few weeks back I spotted some new code in msdia140.dll that provides support for a new format for PDB files that is unlike the standard "MSF" file format that's been used for the previous 30 years. I made a short post on [mastodon](https://infosec.exchange/@ynwarcs/112746816073863886) about it and wanted to follow up with more details later, but I had to shelve my ideas for a few weeks due to something else taking precedence. I finally managed to write up a little bit about the new format and even create a converter for it, so this blog post will talk about that.

The new format is dubbed "MSFZ" throughout the library and its main advantage is that it supports flexible generic compression, whereas MSF stores data uncompressed. So far there haven't been any public details released about the format by MS or other researchers, nor any suggestion that a different format had been introduced or was going to be introduced in the future. On top of that, the format is fully supported for reading in msdia140.dll and other MS DLLs that link MSDIA statically, but is not supported for writing in modules where that would be expected (mspdb140.dll, mspdbst.dll, mspdbcore.dll etc.). When (if?) MS do decide to officially roll out the format,I imagine they'll have to do it slowly, as there are external PDB parsing implementations that are quite popular but don't rely on msdia to do their work: LLVM's [pdb-util](https://llvm.org/docs/CommandGuide/llvm-pdbutil.html) (which can also produce PDB files) and [raw_pdb](https://github.com/MolecularMatters/raw_pdb) by MolecularMatters. Both of these would need to be able to parse the file format in case a non-small percentage of the users decide to use this format for their PDB files, or at least be able to convert it to a MSF representation before parsing.

I felt it'd be interesting to reverse the format & write up a converter (MSF <-> MSFZ) to test it out and see how it turns up in practice. I had known from the past that PDB files can usually be compressed with a compression ratio as low as 12%, so I was interested in how close to that benchmark the new format could come to. And seeing as PDBs can grow to a few gigs in size for large codebases, the format was definitely solving a real world problem. The post below introduces describes the new format and introduces **pdbconv**, the converter.


## 2.0 msf - recap
----
A PDB file has so far known only one format - MSF, standing for Multi Stream File. It is well explained in [LLVM Docs](https://llvm.org/docs/PDB/MsfFile.html) but we'll do a short recap here. The idea behind MSF is to separate data into streams, with each stream containing some specific type of data that can latter be fetched. Streams themselves are serialized in blocks (also called pages):
- each MSF file defines a fixed block size (which must be a power of two between 0x200 and 0x2000)
- each stream is composed of a certain number of blocks
- the mapping of stream index -> block indices is serialized in a "stream directory" which is itself serialized via blocks

For example, the stream at index 1 is always the "PDB Info" stream which contains information about the PDB file, such as its signature that debuggers use to match against the executable file during debugging. The stream at index 2 is always the "DBI stream" which is the stream that contains basic information about symbols, and further information on how to access detailed information from other streams. Most streams don't have fixed indices and are dynamically referenced throughout the other streams.

If we wanted to read data from the PDB Info stream, we'd do something like:
- Parse the MSF header at the beginning of the PDB file, find out where the "stream directory block indices" block is
- Read data from that block and parse stream directory block indices to find out where (in which blocks) the stream directory is located
- Read data from those blocks and parse the stream directory to find out where (in which blocks) the PDB Info stream is located
- Read data from those blocks and parse the PDB info stream

Since blocks have a fixed size, it's fairly simple to just seek to the {blockIndex * blockSize} offset in the memory mapped file and read the amount of data we need from the disk. This reduces the memory footprint and the amount of operations we need to do to get to the data. However, this comes at the cost of:
- Wasting a non-negligible amount of memory, as one block always belongs to one stream. For example, if fixed block size is 0x1000 and the size of the stream is 0x1001, two full blocks will be used to serialize the stream, using 0x2000 bytes of memory, 0xFFF of which is wasted.
- Limitations on the number of blocks, number of streams, size of file, etc. Since the format contains some extra structures (e.g. [the free block map](https://llvm.org/docs/PDB/MsfFile.html#the-free-block-map)) and these are serialized a bit awkwardly, as well as a few streams that also rely on the format not breaking some limits, there are limitations on the number of blocks (max `1 << 20`), number of streams (max 4096) and size of the file (8.0GB). A famous example is the fact that a PDB file using block size 0x1000 can only grow up to 4.0Gb, which [broke chromium's debug builds](https://randomascii.wordpress.com/2023/03/08/when-debug-symbols-get-large/) and prompted MS to introduced a /pdbpagesize:8192 option to their compiler toolchain.

## 3.0 msfz
----
MSFZ format seems to be intended to work around these problems, shifting costs to the other side. The format also recognizes streams, but it doesn't serialize them in blocks. Instead, it separates each stream into one or more **fragments**, each of which is serialized in a **chunk**. A single fragment can only be serialized within a single chunk, but a single chunk can contain multiple fragments, even from within different streams. But the main feature is not this new split which reduces memory waste, but the fact that chunks can be compressed (via [zstd](https://github.com/facebook/zstd)) and decompressed on-demand at runtime to grab the data stored in them.

The flexibility of the implementation means that:
- Each stream can be compressed with an appropriate strategy, i.e. some streams can be split into multiple fragments (if we know that common use cases include reading small pieces of data) or into a single large fragment (if we know we're going to read all data from the stream anyway).
- Data from different streams can be compressed together. If streams are likely to contain similarly formatted data, this could mean greater compression ratio.

On the other hand, there are some drawbacks:
- The time required to decompress a chunk may be non negligible, especially if they're big. Since decompression is done on-demand, this could happen in a sensitive place where one doesn't expect it (e.g. on an input thread in some program) and cause inconvenience for the user.
- Chunks are decompressed into memory, which will inevitably increase the memory usage of the program. Depending on the usage, this may have the effect of the entire original PDB file being loaded into memory, which could be several GBs of memory!

#### 3.1 definition
----
Below is the precise definition of the format, as reversed by me. The name of the structures is provided in symbols shipped with msdia140.dll but not the member fields, so I was the one that named those.

##### 3.1.1 header
The MSFZ header is located at the beginning of the file and has the following structure:
```cpp
struct MsfzHeader
{
    uint8_t m_Signature[0x20];                         // must be "Microsoft MSFZ Container\x0D\x0AALD"
    uint64_t m_Unknown1_64t;                           // must be zero
    uint32_t m_StreamDirectoryDataOffset;              // offset to the stream directory data
    uint32_t m_StreamDirectoryDataOrigin;              // origin of the stream directory data (explained below)
    uint32_t m_ChunkMetadataOffset;                    // offset to chunk metadata
    uint32_t m_ChunkMetadataOrigin;                    // origin of the chunk metadata
    uint32_t m_NumMSFStreams;                          // total number of streams in the file
    uint32_t m_IsStreamDirectoryDataCompressed;        // must be 0-1, denotes if the stream directory data is compressed
    uint32_t m_StreamDirectoryDataLengthCompressed;    // compressed length of the stream directory data
    uint32_t m_StreamDirectoryDataLengthDecompressed;  // decompressed length of the stream directory data
    uint32_t m_NumChunks;                              // total number of chunks in the file
    uint32_t m_ChunkMetadataLength;                    // total length of chunk metadata
};
```

Most of the fields are self-explanatory, with the exception of "origin" fields. These values relate to the offset values that are located above them in the header, and are meant to be interpreted as [`STREAM_SEEK`](https://learn.microsoft.com/en-us/windows/win32/api/objidl/ne-objidl-stream_seek) enumeration values. They decide whether the specified offset is offset from the beginning of the file, the current position in the file or the end of the file.

##### 3.1.2 directory data
The first thing that's parsed after the header is the stream directory data. This data contains information on how the streams are split into fragments and in which chunks these fragments are located. It can be compressed or decompressed, as denoted by the `m_IsStreamDirectoryDataCompressed` field in the header. The implementation will automatically decompress the entire directory data into a local buffer if it's compressed and work from there. 

The data is basically just a list of `MsfzStream` objects:

```
struct MsfzStream
{
    MsfzFragment m_Fragments[]; // dynamically-sized array of fragments
    uint32_t m_Separator;       // zero value separating the streams
}
```

Each fragment is defined as:
```
struct MsfzFragment
{
    uint32_t m_DataSize;               // total size of raw (decompressed) data contained in this fragment
    uint32_t m_DataOffset;             // offset of the data belonging to this fragment, may be file offset or chunk offset
    uint32_t m_ChunkIndexOrDataOrigin; // index of the chunk where this fragment's data is located, or origin of the data
};
```


Since the number of the fragments in the stream is not serialized and cannot be inferred via the stream size value as they're not of fixed length, the data contains "separators", which are basically just 32-bit zero values. When attempting to parse the next `MsfzFragment` entry for the current stream, if it's determined that its `m_SizeOfFragment` is zero, this is taken to mean that the list for the current stream ends and we start parsing data for the next stream. A bit odd, but it does the job nevertheless.

The streams are always serialized in order, meaning that the first encountered stream is the one with index zero, then the one with index one etc. The parsing stops once the end of data is reached, and the code checks that the number of streams found is equal to the `m_NumMSFStreams` value specified in the header.

##### 3.1.3 chunk data
The other part of the data that's always parsed is chunk metadata. This data contains information about the chunks, which is separate from the streams & fragments. This data is never compressed, it's serialized in its raw form and read as such. The position and length of the data is determined by `m_ChunkMetadataOffset` and `m_ChunkMetadataLength` fields in the header. The format of the data is very simple - it's just an array of `MsfzChunk` objects:
```
struct MsfzChunk
{
    uint32_t m_OffsetToChunkData;           // offset to chunk data in the file
    uint32_t m_OriginToChunkData;           // origin of chunk data in the file
    uint32_t m_IsCompressed;                // whether chunk data is compressed or not, must be 0 or 1
    uint32_t m_CompressedSize;              // size of the compressed data in the chunk
    uint32_t m_DecompressedSize;            // size of the decompressed data
};
```

The fields are pretty self explanatory. The code ensures that the `m_NumChunks` field in the header is valid, i.e. `m_NumChunks * sizeof(MsfzChunk) == m_ChunkMetadataLength`. After this data is parsed, it's stored for the remainder of the session so that it can be used later for fetching stream data.

##### 3.1.4 fragment data
So what happens once we need to read data from some stream? Let's say we want to read 0x1000 bytes from stream at index 0x10, at offset 0x100. The program will first access the `MsfzStream` object at index 0x10 to find out how it's split across fragments. It's important to mention here that data stored in fragments is **sequential**, meaning that the first fragment contains data from the beginning of the stream, the second fragment contains data that's adjacent to it, and so on. When calculating which fragment(s) should be used to fetch the data, the program will simply do a linear walk through the list of fragments and add up previously encountered sizes to calculate the offsets of data that fragments hold.

Once the span of fragments which should be taken into account is calculated, the program will attempt to read the data belonging to these fragments and copy it into the output buffer that was reserved for reading the stream data. This operation is not entirely trivial either. Earlier we gave the definition of the `MsfzFragment` object with a brief explanation of what each field represents, but we need to clarify a little further. The last field in the struct, `m_ChunkIndexOrDataOrigin` may be a little confusing. It turns out that fragments can be serialized in chunks, but also plainly somewhere in the file. This field is supposed to denote whether that's the case. The highest-order bit of the value denotes whether the other 31 bits represent the index of the chunk where the data is located (the bit is set), or the origin value of the data serialized plainly in the file (the bit is unset). Depending on this, the `m_DataOffset` field has different meanings too - it can be the offset of this fragment's data within the chunk it belongs to, or the offset in the file where this data is located. The code that reads fragment data then looks something like:

```cpp
...
const MsfzFragment& curFragment = ...;
if (curFragment.m_ChunkIndexOrDataOrigin & (1 << 31))
{
    // fragment data is in a chunk
    const uint32_t chunkIndex = curFragment.m_ChunkIndexOrDataOrigin & ~(1 << 31);
    
    // fetch chunk data
    uint8_t* chunkData = ...;
    memcpy(outputFragmentData, chunkData + curFragment.m_DataOffset, curFragment.m_DataSize);
}
else
{
    // fragment data is stored raw in the file
    // read data from file, using curFragment.m_ChunkIndexOrDataOrigin as origin and curFragment.m_DataOffset as offset
    uint8_t* fragmentData = ...;
    memcpy(outputFragmentData, fragmentData, curFragment.m_DataSize);
}
...
```

### 4.0 pdbconv
----
After reversing the format, which didn't take too long, I wanted to write a small converter program to see how the format fares in real world situations. I expected it wouldn't take more than a few days but I ended up developing the converter into a full-fledged program that can do both compression and decompression and performs well even on large files.

The program is **pdbconv** and is available at [github](https://github.com/ynwarcs/pdbconv). You can find most details about the usage of the program, caveats and benchmarks on the github page. 

As a showcase, here is a demonstration of using pdbconv to compress **chrome.dll.pdb** from **3.1GB** to **500MB** in around 6 seconds:

<br>
<iframe preload="none" style="border:2px solid white; display:block; margin:auto; max-width:520px;" align="center" width="100%" height="300" src="/assets/pdbconv-demo/demo.mp4" frameborder="0" type="video/mp4"> </iframe>