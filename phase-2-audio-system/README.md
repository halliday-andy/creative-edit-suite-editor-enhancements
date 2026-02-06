# Phase 2: Audio System & Voiceover

**Duration:** 2-3 weeks
**Priority:** P0 (Must-Have)
**Status:** 🔴 Not Started
**Depends On:** Phase 1

---

## Overview

Phase 2 adds professional audio capabilities including multi-track audio lanes, in-browser voiceover recording, audio mixing controls, and waveform visualization.

---

## Goals

1. ✅ Add audio track lanes to timeline
2. ✅ Enable in-browser voiceover recording
3. ✅ Support audio mixing (gain, fades, trim)
4. ✅ Include audio in exports

---

## Database Tables (2 new)

### 1. `audio_assets`
Audio file metadata and storage references.

**Key Fields:**
- `type`: music, voiceover, sfx
- `storage_url`: Supabase Storage path
- `duration`: Audio length in seconds
- `waveform_data`: JSONB amplitude data for visualization

### 2. `audio_segments`
Audio placement in timeline with mixing parameters.

**Key Fields:**
- `start_time`: Position in project timeline
- `in_point`, `out_point`: Trim points
- `gain_db`: Volume adjustment (-60 to +20 dB)
- `fade_in_ms`, `fade_out_ms`: Fade durations

---

## Components

### New Components (4)
1. **AudioTrackLane** - Timeline lane with waveform visualization
2. **VoiceoverRecorder** - In-browser recording interface
3. **AudioInspector** - Gain/fade/trim controls
4. **AudioMixerPanel** - Track-level mute/solo/gain faders

### Updated Components (2)
1. **Timeline** - Add audio track lanes
2. **SourceMonitor** - Add Record Voiceover button

---

## Acceptance Criteria

- [ ] User can add audio track lanes to timeline
- [ ] User can record voiceover in-browser
- [ ] Waveforms display correctly
- [ ] Audio segments can be trimmed
- [ ] Gain adjustments work (-60dB to +20dB)
- [ ] Fade in/out controls work
- [ ] Track mute/solo buttons work
- [ ] Exported video includes mixed audio

---

## Resources

- **Main Spec:** Phase 2 section in [CES-EDITOR-IMPLEMENTATION-PLAN.md](../docs/CES-EDITOR-IMPLEMENTATION-PLAN.md)
- **Checklist:** Phase 2 items in [CES-EDITOR-IMPLEMENTATION-CHECKLIST.md](../docs/CES-EDITOR-IMPLEMENTATION-CHECKLIST.md)

---

**Phase Status:** 🔴 Not Started
**Last Updated:** 2026-02-06
