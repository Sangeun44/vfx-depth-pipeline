# VFX Environment Generation Pipeline
**Depth-Guided Production Asset Generation using ComfyUI**

## Overview

A proof-of-concept pipeline for generating consistent environment extensions from production plates using depth-conditioned diffusion models. Built to explore multi-model orchestration patterns for AI-driven VFX workflows.

**Use Case**: VFX studio needs environment extensions across multiple camera angles. Instead of 2 weeks of matte painting, generate variations in hours for supervisor review, then final cleanup on chosen shots only.

**Status**: Prototype demonstrating core orchestration patterns. Not production-ready, but shows the integration architecture for coordinating depth estimation → ControlNet conditioning → generation → metadata logging.

---

## Architecture
Input Plate (JPG/PNG)
↓
MiDaS Depth Estimation
↓
ControlNet Depth Conditioning
↓
SDXL Generation (3 seed variations)
↓
Output (PNG + Metadata JSON)


### Why This Stack?

**MiDaS for depth estimation**: Works on production plates without camera metadata. Handles challenging cases (reflections, dark areas) better than geometric reconstruction.

**ControlNet for spatial control**: VFX supervisors need control. Depth conditioning ensures generated elements respect scene geometry (no floating objects, proper perspective).

**Seed-based variation strategy**: Instead of one "perfect" generation, produce 3-5 variations. Supervisor picks winner, reducing iteration cycles. All seeds logged for reproducibility.

**Metadata logging**: Production requirement. If supervisor requests changes to Variation B two weeks later, you need exact parameters (seed, model version, ControlNet strength) to iterate from same starting point.

---

## Installation

```bash
# Clone ComfyUI
git clone https://github.com/comfyanonymous/ComfyUI
cd ComfyUI

# Install dependencies
pip install -r requirements.txt

# Models auto-download on first run:
# - SDXL 1.0 base (~6.9GB)
# - ControlNet Depth (~5GB)
# - MiDaS depth estimator (~400MB)

# Start ComfyUI server
python main.py --listen
```

---

## Usage

### 1. Build Workflow in ComfyUI UI

Open `http://localhost:8188` and create this node graph:

- **Load Image** → hero plate
- **MiDaS Depth Estimator** → extract depth map
- **ControlNet Apply (Depth)** → condition generation
- **SDXL Sampler** → generate image
- **Save Image** → output result

Save workflow as `workflow_api.json` (File → Save API Format).

### 2. Run Pipeline Script

```bash
python vfx_pipeline.py --input inputs/hero_shot_001.jpg --variations 3
```

**Output**:
outputs/
├── shot_001_var_A.png
├── shot_001_var_A_meta.json
├── shot_001_var_B.png
├── shot_001_var_B_meta.json
├── shot_001_var_C.png
└── shot_001_var_C_meta.json

### 3. Review Variations

VFX supervisor reviews contact sheet, selects preferred variation. Metadata JSON contains exact parameters for iteration.

---

## Technical Decisions

### Why Python API instead of UI-only workflow?

**Batch processing**: 30 shots × 3 variations = 90 runs. Manual UI clicking is not scalable.

**Reproducibility**: Script ensures consistent parameters across shots. UI workflows are prone to accidental setting changes.

**Integration potential**: Python script can be wrapped as Nuke plugin, Maya shelf tool, or Shotgrid action. UI workflow cannot.

### Why PNG output instead of EXR?

**Prototype constraint**: 
ComfyUI's default Save Image node outputs PNG. Production version would add EXR export with:
- 16-bit float color depth
- ACEScc color space transform
- Alpha channel for compositing
- Camera metadata injection (FOV, focal length, lens distortion)

This is a 20-line addition once core orchestration is validated.

### Why metadata JSON instead of embedded EXIF?

**VFX pipeline compatibility**: Many studios use custom asset management systems that ingest JSON metadata (Shotgrid, ftrack, custom databases). EXIF embedding is fragile across file format conversions.

**Human readability**: Artist can `cat shot_001_var_A_meta.json` to debug. EXIF requires specialized tools.

### Why ControlNet Depth specifically?

Tested alternatives:
- **ControlNet Canny (edge detection)**: Too rigid. Locked composition exactly, no creative freedom.
- **ControlNet OpenPose (skeleton detection)**: Wrong domain. Designed for character poses, not environments.
- **Depth**: Best balance. Constrains spatial geometry while allowing texture/lighting variation.

**ControlNet strength tuning**:
- 0.4-0.5: Too loose, ignores plate geometry
- **0.6-0.7: Sweet spot** for environment work
- 0.8-1.0: Too rigid, photocopies input

---

## Known Limitations (Production Blockers)

### 1. Temporal Coherence
**Problem**: Generating frame sequences produces flickering/morphing between frames.

**Solution Path**: 
- AnimateDiff integration for motion consistency
- Frame-to-frame optical flow constraints
- Seed interpolation strategies

**Why not in prototype**: Single-frame generation validates core orchestration. Temporal models add 3x complexity.

### 2. Color Space Handling
**Problem**: Generated images are sRGB. VFX pipelines expect ACEScc/linear color.

**Solution Path**:
- OpenColorIO integration
- Post-generation color transform
- EXR export with proper metadata

**Why not in prototype**: PNG output sufficient for validation. Color pipeline is plumbing, not architecture.

### 3. VRAM Management
**Problem**: Loading SDXL + ControlNet + MiDaS = ~14GB VRAM. Batch runs can OOM.

**Solution Path**:
- Model offloading between stages
- Queue management (run depth estimation batch, unload, run generation batch)
- Multi-GPU orchestration for parallel processing

**Why not in prototype**: Single-shot generation fits in 24GB GPU. Batching is optimization, not validation.

### 4. Art Direction Control
**Problem**: Text prompts alone are insufficient. VFX supervisor needs: "Make it more like this reference, less like that."

**Solution Path**:
- IP-Adapter for reference image conditioning
- Style LoRA training on show-specific matte paintings
- Multi-reference blending (weight different style influences)

**Why not in prototype**: Validates base generation quality first. Style control is layer 2.

---

## The Orchestration Problem

This pipeline coordinates **three models** in sequence:

1. **MiDaS** (depth estimation) → produces depth map
2. **ControlNet** (conditioning) → binds depth to generation
3. **SDXL** (generation) → produces final image

**Failure modes**:
- MiDaS fails on dark/reflective surfaces → depth map is garbage → generation ignores geometry
- ControlNet strength misconfigured → either too rigid (photocopies input) or too loose (ignores depth)
- SDXL produces off-brand results → wasted generation cycles

**Coordination requirements**:
- Intermediate state passing (depth map → ControlNet → SDXL)
- Parameter validation (ControlNet strength in valid range)
- Error handling (retry logic for generation failures)
- Logging (which model version produced which output)

**This is the same pattern in virtual production pipelines**: coordinating real-time rendering + camera tracking + LED wall content. Same orchestration problem, different modalities.

---

## Next Steps (Production Path)

### Phase 1: Integration 
- [ ] EXR output with ACEScc color space
- [ ] Nuke plugin wrapper (Python Panel integration)
- [ ] Shotgrid API integration (auto-populate shot metadata)

### Phase 2: Quality 
- [ ] Style LoRA training on show-specific references
- [ ] IP-Adapter multi-reference conditioning
- [ ] Automatic prompt generation from shot metadata

### Phase 3: Scale 
- [ ] Batch processing queue system
- [ ] VRAM management / model offloading
- [ ] Multi-GPU orchestration

### Phase 4: Temporal 
- [ ] AnimateDiff integration
- [ ] Optical flow constraints
- [ ] Frame sequence coherence validation

---

## Why This Matters

**VFX is adoption-blocked on generative AI** because:
- Models don't integrate with existing tools (Nuke, Houdini, Maya)
- Quality is inconsistent (temporal flickering, style drift)
- No reproducibility (can't iterate on specific results)
- No art direction (text prompts insufficient for professional work)

**This pipeline addresses the integration and reproducibility problems**. 
It's not production-ready, but it demonstrates the orchestration architecture needed for generative AI in VFX pipelines.

The hard problems (temporal coherence, style control, ACES color) are **additive layers on top of this foundation**, not architectural rewrites.

---

Built as exploration of AI-driven VFX orchestration patterns. 
Demonstrates multi-model coordination, production metadata requirements, and integration for creative pipelines.

---

## License

MIT License - Free to use, modify, and distribute.

**Model Licenses**:
- SDXL 1.0: CreativeML Open RAIL++-M License
- ControlNet: Apache 2.0
- MiDaS: MIT License
