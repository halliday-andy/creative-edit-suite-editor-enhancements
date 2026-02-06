# Phase 1: Export Pipeline & Captions v1

**Duration:** 3-4 weeks
**Priority:** P0 (Must-Have Foundation)
**Status:** 🔴 Not Started

---

## Overview

Phase 1 establishes the foundation for all editor features by implementing the export pipeline and basic caption system. This phase enables users to render timeline compositions into downloadable video files and add styled captions.

**Blocks:** All downstream phases depend on this foundation.

---

## Goals

1. ✅ Enable users to export rendered videos from timeline
2. ✅ Add authorable caption layers to timeline
3. ✅ Establish timeline serialization and state management
4. ✅ Support caption auto-generation from transcripts

---

## Database Tables (4 new)

### 1. `render_jobs`
Tracks export job status and progress.

**Key Fields:**
- `status`: queued, running, completed, failed, cancelled
- `render_plan`: JSONB timeline serialization
- `progress`: 0-100%
- `output_urls`: Array of download URLs

### 2. `project_timeline_state`
Stores timeline composition for each project.

**Key Fields:**
- `timeline`: JSONB with tracks, segments, transitions, captions
- `updated_at`: Last modification timestamp

### 3. `caption_tracks`
Caption track metadata per project.

**Key Fields:**
- `name`: Track name (e.g., "English", "Spanish")
- `style_preset`: clean, bold, minimal, custom

### 4. `caption_segments`
Individual captions with timing and text.

**Key Fields:**
- `start_time`, `end_time`: Caption timing in seconds
- `text`: Caption text content
- `style`: JSONB with position, size, box, colors

---

## Components

### New Components (4)

1. **ExportDialog** - Modal for export settings (format, resolution, quality)
2. **CaptionControls** - Caption style and preset controls
3. **CaptionTrack** - Timeline lane for caption segments
4. **CaptionOverlay** - Caption rendering in program monitor

### Updated Components (3)

1. **VideoEditor** - Add Export button to toolbar
2. **ProgramMonitor** - Render caption overlay during playback
3. **Timeline** - Add caption track lane

---

## Services

### captionService
- `getTracks()` - Fetch caption tracks for project
- `createTrack()` - Create new caption track
- `getSegments()` - Fetch segments for track
- `createSegment()` - Add new caption segment
- `updateSegment()` - Modify caption timing/text
- `deleteSegment()` - Remove caption segment
- `generateFromTranscript()` - Auto-generate captions from transcript

### renderService
- `startRender()` - Create render job and trigger Edge Function
- `getRenderStatus()` - Poll job status and progress
- `cancelRender()` - Cancel running render
- `pollRenderStatus()` - Poll until completion
- `saveTimeline()` - Persist timeline state
- `loadTimeline()` - Restore timeline state

---

## Edge Functions

### render-video (New)
**Responsibilities:**
- Load timeline from database
- Download video segments, audio, captions
- Compose video using FFmpeg
- Burn in captions (if enabled)
- Upload output to storage
- Update render job with download URL

**Technology:** FFmpeg (in Deno Edge Function or external worker)

---

## Acceptance Criteria

- [ ] User can click "Export" button in editor
- [ ] Export dialog displays format options (MP4, resolution, quality)
- [ ] Render job starts and shows progress (0-100%)
- [ ] User can cancel in-progress render
- [ ] Completed render provides download link
- [ ] Exported video matches timeline composition
- [ ] User can add caption track to project
- [ ] User can create/edit/delete caption segments
- [ ] Captions display in program monitor during playback
- [ ] Captions can be toggled on/off in preview
- [ ] Captions are included in exported video (when enabled)
- [ ] Caption styling (position, size, box) works correctly
- [ ] Auto-generate captions from transcript works

---

## Testing Checklist

- [ ] Export 1-minute test video successfully
- [ ] Export video with captions enabled
- [ ] Export video with captions disabled
- [ ] Cancel render mid-progress and verify cleanup
- [ ] Edit caption text and verify in export
- [ ] Change caption position (top/center/bottom) and verify
- [ ] Auto-generate captions from transcript
- [ ] Timeline state persists across page refresh
- [ ] Multiple caption tracks work correctly
- [ ] Caption timing is accurate (±50ms tolerance)

---

## Dependencies

### Required
- Supabase PostgreSQL database
- Supabase Edge Functions support
- FFmpeg (in Edge Function or external worker)
- Supabase Storage for output files

### Optional
- Transcript data (for auto-generation)
- Existing timeline segments

---

## Implementation Steps

### Week 1: Database & Services
1. Run database migrations (15 minutes)
2. Create TypeScript types (10 minutes)
3. Implement `captionService.ts` (2 hours)
4. Implement `renderService.ts` (2 hours)
5. Test services with console (1 hour)

### Week 2: Export Pipeline
1. Create `ExportDialog` component (4 hours)
2. Connect to `renderService` (2 hours)
3. Implement basic `render-video` Edge Function (8 hours)
4. Test end-to-end export (4 hours)

### Week 3: Caption UI
1. Create `CaptionControls` component (4 hours)
2. Create `CaptionTrack` timeline lane (6 hours)
3. Create `CaptionOverlay` component (4 hours)
4. Update `Timeline` and `ProgramMonitor` (4 hours)

### Week 4: Testing & Polish
1. Integration testing (8 hours)
2. Fix bugs and edge cases (8 hours)
3. Performance optimization (4 hours)
4. Documentation (4 hours)

---

## Resources

- **Main Spec:** [CES-EDITOR-IMPLEMENTATION-PLAN.md](../docs/CES-EDITOR-IMPLEMENTATION-PLAN.md) (Phase 1 section)
- **Quick Start:** [CES-EDITOR-QUICK-START-GUIDE.md](../docs/CES-EDITOR-QUICK-START-GUIDE.md)
- **Checklist:** [CES-EDITOR-IMPLEMENTATION-CHECKLIST.md](../docs/CES-EDITOR-IMPLEMENTATION-CHECKLIST.md) (Phase 1 items)

---

## Next Phase

After completing Phase 1:
- ✅ **[Phase 2: Audio System →](../phase-2-audio-system/)**

---

**Phase Status:** 🔴 Not Started
**Last Updated:** 2026-02-06
