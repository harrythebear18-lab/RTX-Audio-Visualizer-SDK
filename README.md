# RTX Audio Visualizer — SDK

The contract surface for the **RTX Audio Visualizer** (a Sentinal Systems UK product
under Sentinal Systems). This repo holds interface specifications — not
implementation source — for two things:

1. **Mode shaders** — how a visualization mode receives audio data and
   what it must satisfy to ship
2. **The RDMA analysis feed** — a shared-memory transport that lets an
   external process consume the visualizer's real-time audio analysis

Get the app itself from
[RTX-Audio-Visualizer-OP](https://github.com/harrythebear18-lab/RTX-Audio-Visualizer-OP/releases).

## Contents

| Doc | Contract |
|-----|----------|
| [docs/MODE-SHADER-CONTRACT.md](docs/MODE-SHADER-CONTRACT.md) | Constant buffers, spectrum texture, entry point, quality gate |
| [docs/RDMA-TRANSPORT.md](docs/RDMA-TRANSPORT.md) | Memory-map layout, frame struct, seqlock read protocol |
| [docs/SUBMITTING.md](docs/SUBMITTING.md) | How to propose a mode or an external consumer |

## The mental model

```
ResonateX DSP engine (FFT, bands, LUFS, THD, phase, mixer)
        │
        ▼  analysis feed
RTX Audio Visualizer
        ├── mode shaders consume the feed as cbuffers + spectrum texture
        └── external processes consume it via the RDMA memory map
```

A "mode" is a pixel shader that reads the audio contract and produces
pixels. An "external consumer" is any process that opens the shared map
and reads frames — lighting rigs, overlays, recording tools, alternative
renderers.
