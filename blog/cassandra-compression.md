---
title: How Cassandra Compression Actually Works (Chunks, Offsets, and Reads)
date: 2026-07-01
---

I got asked about Cassandra compression by someone recently and didn't do it justice on the spot. The questions were good ones: what does `chunk_length_in_kb` really control, what happens on a write, and on a read how does Cassandra know how many bytes to pull off disk before it can decompress anything? I work on a database with a Cassandra backend, but we forked Cassandra years ago, before table compression existed, and our data is local so we never leaned on it. So I went and read the actual mechanics. Here's the version I wish I'd had in my head.

## The setup

Cassandra stores data in SSTables, which are immutable once written. Compression happens when the SSTable is written and never changes after that. If you `ALTER` the compression settings, nothing happens to existing data until those SSTables get rewritten by compaction.

When compression is on, two files matter:

- `Data.db` holds the compressed bytes
- `CompressionInfo.db` holds the metadata Cassandra needs to find and decompress those bytes

That second file is the whole trick. Hold onto it.

## chunk_length_in_kb is the uncompressed size

This is the part I had backwards in my head. `chunk_length_in_kb` is **not** how big each chunk is on disk. It's the size of the *uncompressed* buffer Cassandra fills before it compresses and flushes.

So with the default of 16 KB (it was 64 KB before Cassandra 4.0), Cassandra buffers 16 KB of real data, compresses that block, and writes the result. The compressed output might be 4 KB or 9 KB depending on how squishy the data is. The chunks on disk are all different sizes. But every chunk represents exactly one fixed slice of the uncompressed stream.

That "fixed in the uncompressed world, variable on disk" split is what the he kept circling, and it's the thing that makes everything else work.

## The write path

Writing is the easy direction:

1. Buffer incoming data until you hit `chunk_length_in_kb` worth of uncompressed bytes.
2. Compress that buffer (LZ4 by default).
3. Append the compressed bytes to `Data.db`, followed by a 4-byte checksum.
4. Record the starting byte offset of this chunk in `CompressionInfo.db`.

Repeat until the SSTable is done. The checksum is a CRC over the compressed bytes; it's how Cassandra catches bitrot later, and `crc_check_chance` controls how often it bothers to verify on read.

So `CompressionInfo.db` ends up looking roughly like this:

```plaintext
compressor name        e.g. "LZ4Compressor"
chunk_length           e.g. 16384   (uncompressed bytes per chunk)
data_length            total uncompressed length of the file
chunk_count            N
chunk_offsets[]        long[N]   <-- the important bit
```

`chunk_offsets` is just an array of byte positions into `Data.db`. Offset `i` tells you where compressed chunk `i` starts.

## The read path

Now the question that actually matters. Cassandra has resolved a partition through its index and knows the **uncompressed** byte position it wants, call it `position`. The data on disk is compressed and every chunk is a different size, so it can't just seek there. Here's how it gets from an uncompressed position to actual bytes.

First, figure out which chunk holds that position. Because chunks are a fixed size *in uncompressed terms*, this is plain division:

```java
int chunkIndex = (int) (position / chunkLength);
int offsetInChunk = (int) (position % chunkLength);
```

Then look up where that chunk lives on disk, and figure out how many bytes to read. The length of a compressed chunk isn't stored directly. You get it by subtracting consecutive offsets (minus the 4 checksum bytes):

```java
long start = chunkOffsets[chunkIndex];

long end = (chunkIndex + 1 < chunkCount)
    ? chunkOffsets[chunkIndex + 1]   // next chunk starts here
    : compressedFileLength;          // last chunk runs to EOF

int compressedLength = (int) (end - start - 4); // 4 = CRC checksum
```

That answers the "how do we know how many bytes / offsets to read" question directly. You don't store the compressed length, you derive it from the gap between this offset and the next one.

After that the rest is mechanical:

```java
file.seek(start);
file.read(buffer, 0, compressedLength);   // read exactly this chunk

if (shouldCheck(crcCheckChance))
    verifyCrc(buffer, file.readInt());    // the trailing 4 bytes

byte[] decompressed = lz4.decompress(buffer); // up to chunkLength bytes

return decompressed[offsetInChunk ...];   // jump to what we wanted
```

A concrete pass with 16 KB chunks (16384 bytes). Say Cassandra wants uncompressed `position = 50000`:

- `chunkIndex = 50000 / 16384 = 3`
- `offsetInChunk = 50000 % 16384 = 848`
- read the compressed bytes between `chunkOffsets[3]` and `chunkOffsets[4]`
- decompress that one chunk back into ~16 KB
- skip to byte 848 in the result

That's it. One division to find the chunk, one subtraction to size the read, one decompress, one in-memory skip.

## Why fixed uncompressed size, and not fixed disk size

You can't index into compressed data, because the compressor changes the size unpredictably. If chunks were a fixed size *on disk*, you'd have no idea which uncompressed byte each one started at, and random reads would mean decompressing from the front of the file every time.

By fixing the uncompressed size instead, the mapping from "byte I want" to "chunk number" becomes a single divide. The offset array handles the other direction, telling you where that chunk sits on disk. The two together give you O(1) random access into compressed data, which is the whole point.

## The tradeoff in chunk size

To read one tiny cell, Cassandra still has to read and decompress the *entire* chunk that contains it. So chunk size is a real knob:

- **Bigger chunks** give the compressor more context, so better compression ratio and a smaller file. But every small read drags a big block off disk and decompresses it. That's read amplification.
- **Smaller chunks** mean less wasted I/O per read, but a worse ratio and more offheap memory, since you keep more offsets around.

That's why 4.0 dropped the default from 64 KB to 16 KB. For read-heavy or point-read workloads, dragging 64 KB off disk to return a few hundred bytes is mostly waste. If you're doing big sequential scans or your rows are large, bigger chunks can still win.

## The short version

`chunk_length_in_kb` sizes the uncompressed buffer. On write, Cassandra compresses one buffer at a time and records each chunk's disk offset in `CompressionInfo.db`. On read, it divides the wanted position by the chunk length to pick a chunk, subtracts neighbouring offsets to size the read, pulls exactly those bytes, checks the CRC, decompresses, and skips to the byte it wanted. Fixed uncompressed chunks are what let it do all that without scanning from the start of the file.

I should have been able to walk through this in the room. Now I can, and writing it down made it stick.

