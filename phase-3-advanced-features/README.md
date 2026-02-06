# Phase 3: Advanced Captions & Transitions

**Duration:** 2-3 weeks
**Priority:** P1 (High-Value Enhancement)
**Status:** 🔴 Not Started
**Depends On:** Phase 1, Phase 2

---

## Overview

Phase 3 adds professional-grade caption styling, transition effects, and keyframe-based motion graphics.

---

## Goals

1. ✅ Add professional caption styling options
2. ✅ Enable transition effects between segments
3. ✅ Support keyframe-based motion graphics

---

## Database Tables (2 new)

### 1. `segment_transitions`
Transition effects between video segments.

**Key Fields:**
- `type`: cut, dissolve, dip_black, dip_white, wipe_left, wipe_right
- `duration_ms`: Transition length (100-5000ms)

### 2. `segment_keyframes`
Keyframe animation data for motion graphics.

**Key Fields:**
- `t`: Normalized time (0-1)
- `transform`: JSONB with x, y, scale, rotation, opacity

---

## Components

### New Components (6)
1. **CaptionStyleInspector** - Full caption styling controls
2. **CaptionKeyframeMiniEditor** - Caption animation keyframes
3. **TransitionEditor** - Transition type and duration
4. **TransformInspector** - Position/scale/rotation/opacity
5. **KeyframeTimeline** - Keyframe visualization
6. **TransitionPreview** - Real-time transition preview

---

## Acceptance Criteria

- [ ] Caption font/size/color customization works
- [ ] Caption rotation works (-45° to +45°)
- [ ] Background shapes (box/pill/highlight) work
- [ ] Enter/exit animations work
- [ ] Transition effects render smoothly
- [ ] Keyframe interpolation is smooth
- [ ] All effects export correctly

---

## Resources

- **Main Spec:** Phase 3 section in [CES-EDITOR-IMPLEMENTATION-PLAN.md](../docs/CES-EDITOR-IMPLEMENTATION-PLAN.md)

---

**Phase Status:** 🔴 Not Started
**Last Updated:** 2026-02-06
