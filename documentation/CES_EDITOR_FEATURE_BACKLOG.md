# Creative Edit Suite (CES) Editor Feature Backlog

## Goal

Define a concrete P0/P1/P2 backlog for editor capability upgrades, including exact schema, store, and component changes.

Primary codebase target:
- `/Users/andyhalliday/Documents/CODEX Video Project/VideoEditGitRepos/creative-edit-suite`

---

## P0 (Must-Have Foundation)

## P0.1 Export pipeline (renderable timeline to output files)

### Why
- Current CES editor lacks an in-repo export/render path.

### Schema changes
- New table `render_jobs`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `status text check (queued,running,completed,failed,cancelled)`
  - `render_plan jsonb`
  - `progress int`
  - `output_urls jsonb`
  - `error_message text`
  - `created_at`, `started_at`, `completed_at`

- New table `project_timeline_state`:
  - `project_id uuid pk fk projects`
  - `timeline jsonb not null` (tracks, segments, transitions, caption layers)
  - `updated_at timestamptz`

### Store changes
- Update `/src/stores/editorStore.ts`:
  - add `renderStatus`, `renderProgress`, `activeRenderJobId`
  - add `serializeTimeline()` and `loadTimeline()`

### Component/service changes
- New:
  - `/src/components/editor/export/ExportDialog.tsx`
  - `/src/services/renderService.ts`
- Update:
  - `/src/components/editor/VideoEditor.tsx` (add Export action)

---

## P0.2 Caption system v1 (port + integrate)

### Why
- CES has no authorable caption layer in editor timeline/export.

### Schema changes
- New table `caption_tracks`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `name text`
  - `style_preset text` (`clean`,`bold`,`minimal`,`custom`)
  - `created_at`

- New table `caption_segments`:
  - `id uuid pk`
  - `track_id uuid fk caption_tracks`
  - `segment_id uuid null` (optional link to video segment)
  - `start_time numeric`
  - `end_time numeric`
  - `text text`
  - `style jsonb` (position/size/box)
  - `created_at`

### Store changes
- Update `/src/stores/editorStore.ts`:
  - `captionMode`, `captionStyle`, `captionSettings` (add from origin model)
  - actions: `setCaptionMode`, `setCaptionStyle`, `setCaptionSettings`

### Component changes
- New:
  - `/src/components/editor/captions/CaptionControls.tsx`
  - `/src/components/editor/timeline/CaptionTrack.tsx`
  - `/src/components/editor/captions/CaptionOverlay.tsx`
- Update:
  - `/src/components/editor/monitors/ProgramMonitor.tsx` (render caption overlay)
  - `/src/components/editor/timeline/Timeline.tsx` (include caption track lane)

---

## P0.3 Audio track model + voiceover recording v1

### Why
- CES timeline is video-only; no native voiceover workflow.

### Schema changes
- New table `audio_assets`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `type text` (`music`,`voiceover`,`sfx`)
  - `storage_url text`
  - `duration numeric`
  - `created_at`

- New table `audio_segments`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `audio_asset_id uuid fk audio_assets`
  - `start_time numeric`
  - `in_point numeric`
  - `out_point numeric`
  - `gain_db numeric default 0`
  - `fade_in_ms int default 0`
  - `fade_out_ms int default 0`

### Store changes
- Update `/src/stores/editorStore.ts`:
  - add `audioTracks` and `selectedAudioSegmentId`
  - actions: `addAudioSegment`, `updateAudioSegment`, `removeAudioSegment`

### Component changes
- New:
  - `/src/components/editor/audio/AudioTrackLane.tsx`
  - `/src/components/editor/audio/VoiceoverRecorder.tsx`
  - `/src/components/editor/audio/AudioInspector.tsx`
- Update:
  - `/src/components/editor/timeline/Timeline.tsx` (audio lanes)
  - `/src/components/editor/monitors/SourceMonitor.tsx` (record VO and insert)

---

## P0.4 Processing status consistency

### Why
- UI reliability depends on predictable job states.

### Schema changes
- Align `processing_jobs.job_type` CHECK with actual job types used in edge functions.
- Add `completed_at`, `runtime_ms`, `worker_version`.

### Store/service changes
- Update `/src/hooks/useAssets.ts`:
  - compute progress from dependency graph, not only first running job.
- Update `/src/components/assets/ProcessingStatusBadge.tsx`:
  - support `face_detect`, `face_embed`, `face_cluster`, `embedding`.

---

## P1 (High-Value Enhancements)

## P1.1 Pro captions (rotation/color/shape/animations)

### Schema changes
- Extend `caption_segments.style` with:
  - `rotation_deg`
  - `font_family`
  - `font_color`
  - `stroke_color`
  - `stroke_width`
  - `bg_shape` (`none`,`box`,`pill`,`highlight`)
  - `enter_anim`, `exit_anim`

### Store changes
- Update caption state in `/src/stores/editorStore.ts` for per-segment overrides.

### Components
- New:
  - `/src/components/editor/captions/CaptionStyleInspector.tsx`
  - `/src/components/editor/captions/CaptionKeyframeMiniEditor.tsx`

---

## P1.2 Transitions and keyframe motion

### Schema changes
- New table `segment_transitions`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `from_segment_id uuid`
  - `to_segment_id uuid`
  - `type text` (`cut`,`dissolve`,`dip_black`,`dip_white`)
  - `duration_ms int`

- New table `segment_keyframes`:
  - `id uuid pk`
  - `segment_id uuid`
  - `t numeric`
  - `transform jsonb` (`x`,`y`,`scale`,`rotation`,`opacity`)

### Store changes
- Update `/src/stores/editorStore.ts`:
  - `transitions`, `keyframes`
  - actions for insert/update/delete keyframes.

### Components
- New:
  - `/src/components/editor/transitions/TransitionEditor.tsx`
  - `/src/components/editor/keyframes/TransformInspector.tsx`

---

## P1.3 Face-aware reframing controls in editor

### Schema changes
- Extend segment/timeline JSON with:
  - `reframe_mode` (`center`,`face_auto`,`manual_keyframe`)
  - `reframe_preset` (`single_speaker`,`two_speaker`,`reaction`)
  - `crop_keyframes jsonb`

### Store changes
- Update `/src/stores/editorStore.ts`:
  - add `reframeMode`, `reframePreset` per segment
  - actions: `applyReframePreset`, `setCropKeyframe`

### Components
- New:
  - `/src/components/editor/reframe/ReframePanel.tsx`
  - `/src/components/editor/reframe/CropPathPreview.tsx`

---

## P1.4 Person-aware search filters in editor flow

### Schema changes
- Uses person tables from main implementation spec (`person_clusters`, `person_appearances`).

### Store/service changes
- New `peopleService` queries for clip filters.
- Update `/src/services/atomSearch.ts` and `/src/services/clipSearch.ts` to pass people filters.

### Components
- Update `/src/pages/Search.tsx`:
  - people multi-select
  - people-count slider/input

---

## P2 (Polish and Differentiators)

## P2.1 Auto-selects assistant

### Schema changes
- New table `auto_select_runs`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `preset text`
  - `input_filters jsonb`
  - `suggested_segments jsonb`
  - `created_at`

### Store changes
- New `/src/stores/autoSelectStore.ts`:
  - run history, current suggestions, accepted/rejected flags.

### Components
- New:
  - `/src/components/editor/panels/AutoSelectsPanel.tsx`
  - `/src/components/editor/panels/SuggestionList.tsx`

---

## P2.2 Interstitial generator (image/video + optional narration)

### Schema changes
- New table `generated_interstitials`:
  - `id uuid pk`
  - `project_id uuid fk projects`
  - `prompt text`
  - `asset_url text`
  - `asset_type text` (`image`,`video`)
  - `voiceover_url text null`
  - `created_at`

### Store changes
- Update editor store for generated assets catalog.

### Components
- New:
  - `/src/components/editor/interstitials/InterstitialGeneratorPanel.tsx`
  - `/src/components/editor/interstitials/InterstitialInsertDialog.tsx`

---

## P2.3 Review workflow and versioning

### Schema changes
- New table `project_versions` (`project_id`, `timeline_snapshot`, `label`, `created_at`)
- New table `timeline_comments` (`project_id`, `timestamp`, `comment`, `author`, `created_at`)

### Components
- New version history and timeline comment UI.

---

## P1.5 Timeline Sequence Thumbnail Generation

### Problem
Currently, thumbnails only exist at the source clip level. When you compose a timeline sequence from multiple source clips (especially starting from a specific atom), there's no way to generate a representative thumbnail for the composed output. The timeline and any exported sequence just inherits whatever thumbnail the first source clip has.

### Solution
Add a "Capture Thumbnail" action that grabs a frame from the Program Monitor at the current playhead position and saves it as the sequence/project thumbnail.

### Schema changes
- Add `thumbnail_url text null` column to the `projects` table.

### Component/service changes
- Update `/src/components/editor/monitors/ProgramMonitor.tsx`:
  - Add a camera icon "Capture Thumbnail" button in the header bar
  - On click: grabs current `<video>` frame, uploads blob, updates project record
  - Show small preview badge confirming thumbnail was set
  - Show toast: "Sequence thumbnail captured"
- Update `/src/components/editor/VideoPlayer.tsx`:
  - Expose `captureFrame(): Promise<Blob | null>` via `forwardRef` + `useImperativeHandle`
  - Internally: `canvas.drawImage(videoEl, ...) -> canvas.toBlob()`
- Update `/src/stores/editorStore.ts`:
  - Add `sequenceThumbnailUrl` field to editor state
- New `/src/services/thumbnailExtractor.ts`:
  - Add `captureFrameFromVideo(videoElement: HTMLVideoElement): Promise<Blob>` utility
  - Reuses canvas logic but operates on an already-playing video element (not a File)

### Storage path
```
media/projects/{projectId}/sequence-thumbnail.jpg
```

### Crop-aware capture
Canvas applies current aspect ratio crop (pillarbox/letterbox) so the thumbnail accurately represents the final output, not the full 16:9 source frame.

### User flow
1. Arrange timeline sequence from source clips
2. Scrub playhead to the frame you want as the thumbnail
3. Click camera icon in Program Monitor header
4. Toast confirms "Sequence thumbnail captured"
5. Thumbnail URL stored on project record (used for export metadata, project cards, etc.)

---

## Delivery Order (Recommended)

1. P0.1 Export pipeline
2. P0.2 Caption v1
3. P0.3 Audio + voiceover v1
4. P0.4 Job-state consistency
5. P1.2 Transitions/keyframes
6. P1.3 Face-aware reframing controls
7. P1.4 Person-aware search
8. P2 differentiators

---

## Definition of Done (Editor Track)

1. Timeline can render deterministic exports with video + captions + audio.
2. Captions are visible in preview and preserved in exports.
3. Voiceover can be recorded in-browser and placed on audio lanes.
4. Face-aware reframing presets are previewable and exportable.
5. Person filters materially improve search and select workflows.

