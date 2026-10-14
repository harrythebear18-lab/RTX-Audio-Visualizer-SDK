# Mode Shader Contract

How a visualization mode receives audio data in the RTX Audio
Visualizer. Contract version: **1.0** (matches v1.0.0).

## Entry point

Every mode is a fullscreen pixel shader:

```hlsl
float4 main(float4 pos : SV_Position, float2 uv : TEXCOORD0) : SV_Target
```

- Compiled at runtime by DXC as `ps_6_6` — **not** fxc `ps_5_0`
- Output: linear HDR color into an `R16G16B16A16_Float` layer-0 target
- The post pipeline (bloom → composite → tonemap) runs after your shader —
  output HDR, do not tonemap yourself

## Constant buffers

### `b0` — AudioBrainCB (the primary contract)

```hlsl
cbuffer AudioBrainCB : register(b0)
{
    BrainDynamics      Dynamics;   // Beat, Trans, Env, Overall
    FrequencySpectrum  Spectrum;   // Sub Bass LMid Mid HMid Pres Bril Air
    SpatialTelemetry   Spatial;    // StereoLR, Balance, Width, Phase, Anticip
    PerformanceRhythm  Rhythm;     // BPM, Conf, KickWeight, MoveSpeed

    float4 ColorPrimary;    // rgb = primary palette, w = hueBase
    float4 ColorSecondary;  // rgb = secondary palette, w = hueCenter
    float4 ColorTertiary;   // rgb = tertiary palette, w = hueRange

    float4 VisualModifiers; // x: Bright, y: Beam, z: Bloom, w: Ambient
    float4 SystemState;     // x: phrase progress, y: effect, z: pulse, w: sectionID
};
```

| Field | Range | Meaning |
|-------|-------|---------|
| `Dynamics.Beat` | 0..1 | tracked rhythm peak |
| `Dynamics.Trans` | 0..1 | instantaneous transient spike |
| `Dynamics.Env` | 0..1 | smoothed envelope follower |
| `Dynamics.Overall` | 0..1 | global RMS power |
| `Spectrum.*` | 0..1 | 8 bands: Sub(0-60) Bass(60-250) LMid(250-500) Mid(500-2k) HMid(2-4k) Pres(4-6k) Bril(6-12k) Air(12-20k Hz) |
| `Spatial.Balance` | -1..1 | stereo balance |
| `Spatial.Width` | 0..1 | stereo image width |
| `Spatial.Phase` | -1..1 | L/R phase correlation |
| `Spatial.Anticip` | 0..1 | predictive transient lookahead |
| `Rhythm.BPM` | 60..200 | tracked tempo |
| `Rhythm.Conf` | 0..1 | tempo confidence |
| `Rhythm.KickWeight` | 0..1 | kick drum trigger probability |
| `Rhythm.MoveSpeed` | ~0..2 | audio velocity constant for motion |

### `b1` — TimeCB

```hlsl
cbuffer TimeCB : register(b1)
{
    float GlobalTime;          // seconds since start
    float DeltaTime;           // real frame delta (clamped 0..0.25s)
    float2 RenderResolution;   // px
};
```

### `b2` — DspCB (opt-in: professional analysis)

```hlsl
cbuffer DspCB : register(b2)
{
    float DspMomentaryLUFS;      // -70..0
    float DspShortTermLUFS;
    float DspIntegratedLUFS;
    float DspTHDPercentage;      // %
    float DspPhaseCorrelation;   // -1..1
    float DspPeakDbL, DspPeakDbR;
    float DspCrestFactorDb;      // headroom indicator
    float DspBand0..DspBand7;    // biquad band-pass RMS, same 8 bands
};
```

### `b3` — VRCB (VR only)

Head pose, orientation quaternion, IPD/eye-index/FOV. When
`VRHeadPos.w == 0` fall back to a computed camera. **Never ignore this** —
a mode that hardcodes a camera breaks in VR.

## Textures & samplers

| Register | Binding | Contents |
|----------|---------|----------|
| `t0` | `u_spectrum` | 1024×3 float spectrum texture — row 0 = left, row 1 = mono, row 2 = right |
| `t5` | `u_feedback` | previous frame output — persistent state/trails |
| `s0` | `u_sampler` | bilinear (named `PointSampler` historically — it is **linear**) |
| `s1` | `u_feedbackSampler` | feedback sampler |

## The quality gate (all mandatory)

A mode that ships must pass all seven sections of the quality gate:

1. **≥3 visible layers** — background / midground / foreground with
   depth separation (fog, brightness falloff, or parallax)
2. **≥6 independent audio mappings** — bass drives structure, mids drive
   color/motion, treble drives detail, beat triggers a discrete event,
   BPM drives motion speed, stereo drives L/R behavior, plus spectrum
   *texture* sampling (not just UBO scalars)
3. **Structure** — focal point, cohesive palette, negative space.
   No random color fills.
4. **Anti-fatigue** — exponential decay on flashes, rest periods,
   smooth easing, gradual hue shifts
5. **Post-friendly** — outputs HDR-safe (use `hdrLimiter()`), designed
   for bloom
6. **Code quality** — shared `#include` headers, no magic numbers,
   ~200–400 lines, compiles under DXC `ps_6_6`
7. **≥1 DX12U feature** — wave intrinsics, min16float, VRS-friendly
   structure, or compute dependency

**Instant reject:** flat background, <3 audio parameters, everything
flat-2D, noise fills, strobing without decay, under 100 lines.

## Performance budget (16ms frame)

| Technique | Max objects | Max raymarch steps |
|-----------|------------|--------------------|
| 2D projection | ~100 | 0 |
| 3D projection | ~50 | 0 |
| Raymarch (simple SDF) | ~20 | 40 |
| Raymarch (complex SDF) | ~10 | 28 |
| Raymarch (volumetric) | ~5 | 36 |

- `[loop]` not `[unroll]` for loops >4 iterations
- Early-out on silent bands: `if (band < 0.005) continue;`
- Depth fog via `exp(-depth * density)` — cheap, sells scale
- Foreground particles: 12–18
- Wide FOV (1.2–1.5) exaggerates depth for free
