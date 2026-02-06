# Phase 4: Face-Aware Features

**Duration:** 3-4 weeks
**Priority:** P1 (High-Value Enhancement)
**Status:** 🔴 Not Started
**Depends On:** Phase 1, Face Recognition System

---

## Overview

Phase 4 adds face-aware reframing for vertical video optimization and person-based search filters.

**Prerequisites:** Face Recognition System must be deployed (separate track).

---

## Goals

1. ✅ Add face-aware reframing controls to editor
2. ✅ Enable person-based search filters
3. ✅ Support vertical video optimization presets

---

## Database Changes

### No New Tables
Uses existing face recognition system tables:
- `face_detections`
- `person_clusters`
- `person_appearances`

### Schema Updates
- Extend `segments` table with `reframe_config` JSONB field

---

## Components

### New Components (4)
1. **ReframePanel** - Reframe mode and preset selector
2. **CropPathPreview** - Visual crop overlay with face detection boxes
3. **PeopleFilterPanel** - Multi-select people filter
4. **PersonTimeline** - Person appearance timeline view

---

## Reframe Presets

1. **Single Speaker** - Center on largest face, stable framing
2. **Two Person** - Frame both speakers, allow slight panning
3. **Reaction** - Emphasize most active/emotive face

---

## Acceptance Criteria

- [ ] Face auto-tracking generates smooth crop paths
- [ ] Reframe presets work on 85%+ of clips
- [ ] Manual keyframe mode allows custom crops
- [ ] People filter displays labeled individuals
- [ ] Person timeline shows all appearances
- [ ] Search by people works accurately
- [ ] Reframe exports match preview

---

## Resources

- **Main Spec:** Phase 4 section in [CES-EDITOR-IMPLEMENTATION-PLAN.md](../docs/CES-EDITOR-IMPLEMENTATION-PLAN.md)
- **Face Recognition Docs:** [creative-edit-suite-enhancements](https://github.com/yourusername/creative-edit-suite-enhancements)

---

**Phase Status:** 🔴 Not Started
**Last Updated:** 2026-02-06
