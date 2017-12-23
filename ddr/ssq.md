# Dance Dance Revolution SSQ Format

This document serves to explain the SSQ format used by Konami's Dance Dance
Revolution series of video games.

## Terminology
```
Byte: 8 bits of data.
Word: 16 bits, or 2 bytes, of data.
Dword: 32 bits, or 4 bytes, or 2 words, of data
```

## About this format

To date, SSQ is the most recent format used by Konami's Dance Dance Revolution
series. Its earliest occurrence is in unreferenced files in DDR 3rd Mix Plus.
They are first officially used in DDR 4th Mix.

The SSQ format appears to have been originally developed without freeze notes 
in mind, as the SSQ format pre-dates the inclusion of freeze arrows - which
occurred in in DDRMAX -DanceDanceRevolution 6thMIX-.  This is also evident by 
the hacky way in which freeze arrow support was added to the SSQ format.  Shock 
arrows are also implemented in a similarly hacky, although less intrusive, manner.

The SSQ format supports BPM changes and multiple variable sized chunks, much like 
a WAV file.

For Playstation2 and arcade versions of DDR, the game's SSQ files tend to be 
stored in a giant block of data.  Because of this, the game needs a way to 
locate the individual SSQ files within that block, and load that data.

Preceding a game's SSQ file data is an array of 32 bit values.  Each value 
represents the offset of an SSQ file, relative to the start of the data 
chunk.  There is one entry for each song in the game.  The first offset 
value is the size, in bytes, of the array.  Each following entry value
will be the size of the array + the sum of the previous SSQ file(s)' size(s).

With this table, the game can obtain the size of each song's SSQ file, as well
as locate, and extract individual SSQ files.

Chunks are `dword` (`double-word`) aligned, meaning if the data length is not 
that of a dword, or 32 bits, it is padded out with 0s until the data chunk is
the proper length.

## In a nutshell

A SSQ file consists of a series of "chunks." Each chunk has a header defining 
what it is, as well as containing some chunk-specific metadata.

Often, you'll find at least three chunks. The first two chunks are the tempo 
data, and some unknown yet consistent data.  Following these two chunks are 
the chunks containing the actual stepchart data.  

There are (number of difficulties * 2) charts, one for each difficulty in single
play mode, and one for each difficulty in doubles play.  If a chart does not
contain actual step data for a particular difficulty, that stepchart will just
have a 12 byte header, with no stepdata following it.


## Chunk format

```
Offset(h)     Type                     Length (in bytes)    Description
+00           Dword                           4             Chunk size, in bytes - including the size of this value
+04           Word                            2             Parameter 1: Type of chunk
+06           Word                            2             Parameter 2: Type-dependent metadata
+08           Word                            2             Parameter 3: Type-dependent metadata
+0A           Word                            2             Parameter 4: Type-dependent metadata
+0C           Dependent on chunk type       Varies          The rest of the data, specifics depend on the chunk type
```

## Chunk types

The first word, or 16 bits, following the chunk size value defines the type of 
chunk you are working with.

Format specifics depend on the value found in parameter 1.  Data will have all its
time offsets first, then all its corresponding data second.

### Type 01: Tempo Changes

- Parameter 2: Time frames per second
- Parameter 3: Number of BPM change/stop entries

After the time offsets, tempo data is dword-aligned, and will be 4 bytes per
entry. The time offset determines what point the song needs to change tempo
or stop, and the data determines how many time frames are supposed to have
passed.

You can use the following formula to convert this kind of time table into BPM.
Note that `i` must be at least 1 because deltas are used.

```
DeltaOffset = TimeOffset[i] - TimeOffset[i - 1];
DeltaTicks = TempoData[i] - TempoData[i - 1];
TicksPerSecond = Parameter2;
MeasureLength = 4096;

BPM = (DeltaOffset / MeasureLength) / ((DeltaTicks / TicksPerSecond) / 240);
```

Stops will be encoded as two consecutive entries with the same time offset, but
different data. You can use the following formula to convert this kind of
time into seconds.

```
DeltaTicks = TempoData[i] - TempoData[i - 1];
TicksPerSecond = Parameter2;

StopLengthInSeconds = DeltaTicks / TicksPerSecond;
```

### Type 02: unknown

- Parameter 3: Number of entries

After the time offsets, this data is word-aligned, and will be 2 bytes per
entry. Not much else is known about this chunk type. It appears in every
observed SSQ file so far. It could possibly be linked to what tells the game
when to end the song and show the results screen.

The observed pattern in hex:
```
0104
0201
0202
0205
0203
0204
```

### Type 03: steps

- Parameter 2: Chart difficulty type
- Parameter 3: Number of steps

After the time offsets, this data and will be at least one byte
per entry (see below for details about why this varies.)

#### Difficulty types

Use Parameter 2 with the following table to determine the chart type.

```
Value(h)  Type
0114      Single Basic
0214      Single Standard
0314      Single Heavy
0414      Single Beginner
0614      Single Challenge

0118      Double Basic
0218      Double Standard
0318      Double Heavy
0418      Double Beginner
0618      Double Challenge
```

#### Decoding steps

Each arrow is represented by one nibble of the data. Assuming the least
significant bit is 0, and the most significant bit is 7, you can use this table
to determine which arrow is to be pressed:

```
Nibble    Arrow
0         Player 1 Left
1         Player 1 Down
2         Player 1 Up
3         Player 1 Right
4         Player 2 Left
5         Player 2 Down
6         Player 2 Up
7         Player 2 Right
```

**Note:** As of DDR X, shock arrows are part of the format, and are indicated
by having all bits set (a value of 0xFF).

On DDR MAX and later mixes, it is possible to encounter data where the step
data has no bits set (0x00). This indicates a freeze arrow. Freeze arrow data
is present after all the normal data. For example: a chunk with 300 steps would
consist of 300 time offsets, 300 steps, and the freeze step data would come
afterwards, if any. You can easily determine the number of freeze steps by using
this formula:

```
StepCount = Parameter3;
FreezeStepCount = ChunkLength - 0x0C - (StepCount * 5)
```

The freeze data indicates which arrows are to start the freeze, and should be
read one by one as zero-bytes are encountered in the regular step data.

### Type 05: background change script

- Parameter 2: Chart difficulty type
- Parameter 3: Number of steps

The background scripts are actually kept separate from the rest of the SSQ data.

Like with the block of step data, the block containing background scripts begins
with an array of 32 bit values.  Each value represents the offset of a background
change script, relative to the start of the data chunk.

There is one entry for each song in the game.  The first offset value is the size, 
in bytes, of the array.  Each following entry value will be the size of the array 
+ the sum of the previous background script file(s)' size(s).

With this table, the game can obtain the size of the data that needs to be extracted, 
as well as actually locate, and extract individual background script.

Following this array will be the background change scripts.  Like with SSQ data, each 
script is compressed using Konami's implementation of LZ (Lempel–Ziv) compression.

Like with an SSQ file, background change scripts are stored using chunks of data.  The 
only significant difference is that a song has a single script in a single chunk, as opposed 
to SSQ files being made up of stepdata stored in multiple chunks.  The song order for each 
script is the same as the song order for the SSQ data.

Within each background script, the first 4 bytes contain the size of the script file, and 
the 2 bytes following dictate the chunk type, just like with an SSQ file.  With the back-
ground scripts, however, the chunk type value appears to be padded out to 32 bits in length.

Following this is another word-length piece of data containing the number of background
changes that occur. 

More investigation is needed on the specifics to the data following this, but there appear to 
be 2 sets of N entries, N being the number of changes.  The first set could be timing related,
and the second set could be specific information about the background change that contains

### Type 09: song metadata

This section contains at least the title and artist of a song. This kind of chunk was discovered by looking at Dance Dance Revolution S+. Further details pending for this section.



