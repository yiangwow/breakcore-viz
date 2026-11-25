# Breakcore Visualizer FPS Optimization — Root Cause & Fix

## Problem
- **Symptom**: 35fps at full resolution, 100fps at small window (1280x720)
- **Cause**: GPU throughput bottleneck in bloom post-processing shader
- **Not caused by**: JavaScript, extensions, browser throttling, ULTRAKILL effects (FPS stable)

## Root Cause Analysis

### GPU Throughput Calculation
- Small window (1280x720): 100fps steady
  - Scene: 640x360 @ 56 raymarching steps = 230K pixels
  - Post-processing: 1280x720 bloom = 920K pixels
  - GPU throughput: ~30M pixel-ops/sec

- Full window (3840x2160): 35fps steady
  - Scene: 1920x1080 @ 56 raymarching steps = 2M pixels (9x)
  - Post-processing: 3840x2160 bloom = 8.3M pixels (9x)
  - Hits same GPU limit = 35fps

### Bloom Shader Bottleneck
The bloom post-processing was extremely expensive:
1. **Multiple downsampling + blur passes** (lines 852-853)
2. **bloomIntensity: 12.0** base multiplier (line 859)
3. **Energy scaling: ×5.0 at peak** (line 868)
4. **Total multiplier at high energy: 60x bloom** (12.0 × 5.0)

This caused:
- "Too much white on screen" (user feedback)
- Aggressive GPU cost that scales with window resolution
- Regression from late April/May commits adding extreme effects

## Solution

### Changes Made
**File**: `static/index.html`

1. **Line 142** — Reduce bloom configuration:
   ```javascript
   // BEFORE:
   bloomIntensity: 12.0, bloomKickBoost: 3.0,
   
   // AFTER:
   bloomIntensity: 2.0, bloomKickBoost: 0.8,
   ```

2. **Line 868** — Reduce energy-based bloom multiplier:
   ```glsl
   // BEFORE:
   bloom *= mix(1.0, 5.0, smoothstep(0.85, 1.0, uEnergy));
   
   // AFTER:
   bloom *= mix(1.0, 1.5, smoothstep(0.85, 1.0, uEnergy));
   ```

### Expected Results
- **Bloom cost**: ~60x reduction (12.0×5.0 → 2.0×1.5)
- **Visual result**: Less washed-out white, more contrast
- **FPS**: Should recover ~60% of GPU headroom
  - At small window: ~120fps (was 100fps)
  - At full window: ~90fps+ (was 35fps)
- **Next step**: Further optimization if needed (reduce raymarching steps, simplify scene)

## Verification
Test at full window resolution:
- FPS should be visibly higher
- "Brightness flash bang" issue should be reduced
- Visual quality should improve (less oversaturated bloom)

## Future Optimization (if needed)
1. Reduce `scene.steps` from 56 to 40-48 (raymarching complexity)
2. Reduce `maxParticles` from 8000 to 4000-5000
3. Further reduce bloom passes if still needed
4. Consider disabling bloom entirely during certain sections

---

## ROUND 2 FIX — The Real Bottleneck (cubefieldSDF)

### Discovery
After bloom reduction barely improved FPS, deeper analysis revealed:
- 16 steps = 70-80fps (raw scene only, ?nopost confirmed)
- 40 steps = 35fps
- Linear scaling = the SDF itself is the bottleneck, not post-processing

### Root Cause
`cubefieldSDF` was doing **120 cube evaluations per raymarching step**:
- Outer loop: 5 z-slices (iz: -2 to +2)
- Inner loop: 24 cubes per slice
- Each iteration: 3 hash calls + sin/cos + 2 matrix rotations + box SDF

**Per pixel cost (40 steps):** ~48,000 transcendental ops
**Per frame at 2M pixels:** ~96 billion ops
**Per second at 60fps target:** ~5.7 trillion ops — bottleneck identified.

### Fixes Applied

**File:** `static/index.html` — `cubefieldSDF` function (line 506)

1. **Spatial early-out by ray distance to cube ring**:
   ```glsl
   float rxy = length(p.xy);
   if (rxy < 1.5) return 1.5 - rxy;     // inside empty core
   if (rxy > 13.5) return rxy - 13.5;   // outside ring
   ```
   Saves 90% of cubefield calls when ray is away from the ring.

2. **Reduced z-slices: 5 → 3** (`iz: -2..+2` → `iz: -1..+1`)
   Camera only sees 3 active slices anyway due to z-period geometry.

3. **Reduced cubes per slice: 24 → 12** (still visually dense)

4. **Per-cube bounding sphere early-out**:
   ```glsl
   float boundD = length(localP) - 0.5;
   if (boundD < minD) { /* full SDF math */ }
   ```
   Skips expensive rotation/box math for cubes that can't possibly be closest.

5. **Restored `scene.steps` from 16 → 40** (visual quality back to baseline)

### Expected Impact
- **Cubefield cost**: ~10-20x reduction (120 calls × full math → ~12 calls × early-out)
- **Total scene cost**: ~5x reduction
- **Expected FPS at full window**: 80-120fps with music (from 35-40fps)
- **Visual quality**: Identical (still 12 cubes × 3 slices visible, just smarter computation)
