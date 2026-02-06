# Phase 5: AI-Powered Assistance

**Duration:** 2-3 weeks
**Priority:** P2 (Polish and Differentiators)
**Status:** 🔴 Not Started
**Depends On:** Phases 1-4

---

## Overview

Phase 5 adds AI-powered features including auto-select clip suggestions, AI-generated interstitials, project versioning, and review workflow.

---

## Goals

1. ✅ Add auto-select assistant for intelligent clip suggestions
2. ✅ Enable AI-generated interstitials (image/video)
3. ✅ Add project versioning and review workflow

---

## Database Tables (4 new)

### 1. `auto_select_runs`
AI selection algorithm results.

**Key Fields:**
- `preset`: highlights, emotional, vertical_ready, custom
- `input_filters`: JSONB with topics, people, confidence
- `suggested_segments`: JSONB with scores and reasons

### 2. `generated_interstitials`
AI-generated assets (images/video).

**Key Fields:**
- `prompt`: Generation prompt
- `asset_url`: Storage URL
- `asset_type`: image, video
- `voiceover_url`: Optional TTS voiceover

### 3. `project_versions`
Timeline version snapshots.

**Key Fields:**
- `version_number`: Incremental version
- `label`: User-provided label
- `timeline_snapshot`: Full timeline JSONB

### 4. `timeline_comments`
Review comments with timestamps.

**Key Fields:**
- `timestamp`: Timeline position in seconds
- `comment`: Comment text
- `status`: open, resolved

---

## Components

### New Components (8)
1. **AutoSelectsPanel** - Preset selector and suggestions list
2. **SuggestionCard** - Individual suggestion with accept/reject
3. **InterstitialGeneratorPanel** - AI generation interface
4. **InterstitialThumbnail** - Generated asset preview
5. **VersionHistoryPanel** - Version list and restore
6. **TimelineCommentsPanel** - Comments list
7. **CommentMarker** - Timeline comment indicator
8. **TransitionPreview** - Transition effect preview

---

## AI Services

### Auto-Select
- **Algorithm:** Semantic relevance + person prominence + emotion + quality
- **Presets:** Highlights, Emotional, Vertical-Ready, Custom

### Image Generation
- **Providers:** DALL-E 3, Stability AI
- **Use Cases:** Title cards, b-roll, backgrounds

### Video Generation
- **Providers:** Runway Gen-3, Pika
- **Duration:** 5-second clips

### TTS Voiceover
- **Providers:** ElevenLabs, OpenAI TTS
- **Use Cases:** Narration, intros, outros

---

## Acceptance Criteria

- [ ] Auto-select generates relevant suggestions (>75% satisfaction)
- [ ] Image generation succeeds >90% of time
- [ ] Video generation works (5-second clips)
- [ ] TTS voiceover quality is acceptable
- [ ] Version save/restore works 100% reliably
- [ ] Comments display at correct timestamps
- [ ] Can resolve comments

---

## Resources

- **Main Spec:** Phase 5 section in [CES-EDITOR-IMPLEMENTATION-PLAN.md](../docs/CES-EDITOR-IMPLEMENTATION-PLAN.md)

---

**Phase Status:** 🔴 Not Started
**Last Updated:** 2026-02-06
