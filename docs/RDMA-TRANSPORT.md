# RDMA Analysis-Feed Transport

The visualizer publishes its real-time audio analysis into a named
memory-mapped ring buffer. Any local process — lighting rigs, overlay
tools, recorders, alternative renderers — can consume it with zero
serialization and no socket.

Transport contract version: **1.1** (v1.0.0 ships seqlock-guarded reads).

## Shared memory map

- **Map name:** `RTXAudioVisualizer_RDMA` (Windows named section)
- **Access:** `ReadWrite`
- **Layout:** 64-byte header + 4 frame slots

### Header (offset 0, 64 bytes, packed)

| Offset | Type | Field |
|--------|------|-------|
| 0 | int32 | `WriteIndex` — next slot to be written |
| 4 | int32 | `ReadIndex` — last consumed slot |
| 8 | int64 | `Sequence` — **seqlock**: odd = write in progress, even = stable |
| 16 | int32 | `FrameTotalSize` — bytes per slot |
| 20 | int32 | `SpectrumBins` — spectrum floats per slot |
| 24..63 | — | padding |

### Frame slot (HeaderSize + i × FrameTotalSize)

Each slot is `VisualFrame` + spectrum:

```
[0 .. FrameStructSize)            VisualFrame struct (packed)
[FrameStructSize .. FrameTotal)   float[SpectrumBins] spectrum (1024 bins)
```

`VisualFrame` carries: 8 bands, beat/transient/envelope/overall, BPM +
confidence, kick, stereo balance/width/L/R energy, intensity group
(effect/movement/brightness/beam/bloom/dynamic/ambient), triggers,
section + phrase progress, spectral centroid/spread/dominant frequency.

## Read protocol (seqlock)

Never read a slot without checking the sequence — a torn read looks like
a glitch, not an error:

```
loop (bounded — 3 retries is plenty):
    seq1 = read int64 @ offset 8
    if seq1 is odd: spin briefly, retry          // writer mid-publish
    writeIdx = read int32 @ offset 0
    readIdx  = read int32 @ offset 4
    if writeIdx == readIdx: return false          // nothing new
    latestIdx = (writeIdx - 1 + 4) % 4
    read VisualFrame + spectrum from slot latestIdx
    seq2 = read int64 @ offset 8
    if seq2 != seq1: retry                        // writer lapped us
    write int32 @ offset 4 = writeIdx             // mark consumed
    return frame
return false                                       // contention — next tick
```

Rules for consumers:

- **Bounded retry.** If three attempts tear, skip the frame — the next
  publish arrives within ~16ms anyway.
- **Latest-wins.** The map is a freshest-frame broadcast, not a queue.
  If you fall behind, you get the newest frame, not a backlog.
- **Read-only discipline.** Only write offset 4 (your read cursor).
  Everything else belongs to the writer.
- **No lifetime guarantee.** The map exists only while the visualizer's
  writer is alive. `OpenExisting` failing means nothing is publishing.

## Why this exists

The visualizer's analysis (ResonateX DSP) is the interesting part — 8
bands, psychoacoustic telemetry, tempo tracking, loudness. If you want
that data in your own process, you shouldn't have to run your own FFT.
Open the map, read frames, build whatever you want on top.
