# Submitting

## Modes (HLSL pixel shaders)

A mode submission is one `.hlsl` file against the contract in
[MODE-SHADER-CONTRACT.md](MODE-SHADER-CONTRACT.md):

- `main(float4 pos : SV_Position, float2 uv : TEXCOORD0)` entry
- Reads `b0`/`b1` (`b2`/`b3` optional), samples `t0` spectrum
- Passes all seven quality-gate sections
- Compiles under DXC `ps_6_6`
- Within the performance budget (see contract)

Open an issue in
[RTX-Audio-Visualizer-OP](https://github.com/harrythebear18-lab/RTX-Audio-Visualizer-OP/issues)
titled `[MODE] <name>` with the shader attached or linked, plus a short
note on which audio features it maps and why. Accepted modes ship
credited in release notes.

## External consumers (RDMA)

Anything that opens `RTXAudioVisualizer_RDMA` and plays by the seqlock
rules in [RDMA-TRANSPORT.md](RDMA-TRANSPORT.md) is a valid consumer —
no approval needed. If you build something on the feed, show it off in
the [Discord](https://discord.gg/denat8G6ze).

If your use case needs fields that aren't in `VisualFrame`, file an
issue describing what you need — the frame format can grow at minor
contract versions, and existing readers are unaffected by appended
fields (they read by offset).
