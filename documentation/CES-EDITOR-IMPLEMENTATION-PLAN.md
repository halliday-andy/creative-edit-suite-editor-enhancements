# Creative Edit Suite (CES) Editor Implementation Plan

## Executive Summary

This document provides a comprehensive, phased implementation plan for upgrading the Creative Edit Suite editor with professional video editing capabilities. The plan is organized into 5 phases spanning 12-16 weeks, prioritizing core editing functionality before advanced features.

**Target Repository:** `creative-edit-suite` (Lovable-managed)

**Based On:** CES_EDITOR_FEATURE_BACKLOG.md

---

## Implementation Philosophy

### Approach
- **Phase-based delivery:** Each phase delivers working, user-testable features
- **Database-first:** Schema changes precede UI implementation
- **Progressive enhancement:** Build foundation before advanced features
- **User-centric:** Prioritize features that unblock creative workflows

### Success Metrics
1. Users can export rendered videos with captions and audio
2. Caption styling matches professional video editing standards
3. Audio mixing and voiceover recording work in-browser
4. Face-aware reframing enables vertical video optimization
5. Search and auto-select features accelerate edit decisions

---

## Phase Overview

| Phase | Focus | Duration | Priority | Deliverables |
|-------|-------|----------|----------|--------------|
| **Phase 1** | Export Pipeline & Captions v1 | 3-4 weeks | P0 | Render exports, basic captions |
| **Phase 2** | Audio System & Voiceover | 2-3 weeks | P0 | Audio tracks, recording, mixing |
| **Phase 3** | Advanced Captions & Transitions | 2-3 weeks | P1 | Pro caption styling, transitions |
| **Phase 4** | Face-Aware Features | 3-4 weeks | P1 | Reframing controls, person search |
| **Phase 5** | AI-Powered Assistance | 2-3 weeks | P2 | Auto-selects, interstitials, versioning |

**Total Timeline:** 12-17 weeks (sequential) or 8-12 weeks (parallel tracks)

---

## Phase 1: Export Pipeline & Captions v1
**Duration:** 3-4 weeks
**Priority:** P0 (Must-Have)
**Blocks:** All downstream features

### Goals
1. Enable users to export rendered videos from timeline
2. Add authorable caption layers to timeline
3. Establish foundation for all export features

### Database Schema Changes

#### New Table: `render_jobs`
```sql
CREATE TABLE render_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status TEXT NOT NULL CHECK (status IN ('queued', 'running', 'completed', 'failed', 'cancelled')),
    render_plan JSONB NOT NULL, -- timeline serialization
    progress INTEGER DEFAULT 0 CHECK (progress >= 0 AND progress <= 100),
    output_urls JSONB DEFAULT '[]'::jsonb, -- [{type: 'video', url: '...', format: 'mp4'}]
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,

    -- Indexes
    CONSTRAINT valid_progress CHECK (progress BETWEEN 0 AND 100)
);

CREATE INDEX idx_render_jobs_project ON render_jobs(project_id);
CREATE INDEX idx_render_jobs_status ON render_jobs(status) WHERE status IN ('queued', 'running');
```

#### New Table: `project_timeline_state`
```sql
CREATE TABLE project_timeline_state (
    project_id UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    timeline JSONB NOT NULL, -- {tracks: [], segments: [], transitions: [], captions: []}
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Timeline schema structure (JSONB):
-- {
--   "tracks": [
--     {"id": "uuid", "type": "video", "segments": [...]}
--   ],
--   "segments": [
--     {"id": "uuid", "track_id": "uuid", "clip_id": "uuid", "start_time": 0, "duration": 10}
--   ],
--   "transitions": [],
--   "caption_tracks": []
-- }
```

#### New Table: `caption_tracks`
```sql
CREATE TABLE caption_tracks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name TEXT NOT NULL DEFAULT 'Captions',
    style_preset TEXT NOT NULL CHECK (style_preset IN ('clean', 'bold', 'minimal', 'custom')),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(project_id, name)
);

CREATE INDEX idx_caption_tracks_project ON caption_tracks(project_id);
```

#### New Table: `caption_segments`
```sql
CREATE TABLE caption_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    track_id UUID NOT NULL REFERENCES caption_tracks(id) ON DELETE CASCADE,
    segment_id UUID REFERENCES segments(id) ON DELETE SET NULL, -- optional link to video segment
    start_time NUMERIC NOT NULL, -- seconds from project start
    end_time NUMERIC NOT NULL,
    text TEXT NOT NULL,
    style JSONB DEFAULT '{}'::jsonb, -- {position: 'bottom', size: 'medium', box: true}
    created_at TIMESTAMPTZ DEFAULT NOW(),

    CONSTRAINT valid_time_range CHECK (end_time > start_time)
);

CREATE INDEX idx_caption_segments_track ON caption_segments(track_id);
CREATE INDEX idx_caption_segments_time ON caption_segments(start_time, end_time);
```

### Store Changes (`/src/stores/editorStore.ts`)

```typescript
interface EditorState {
  // Existing state...

  // NEW: Render state
  renderStatus: 'idle' | 'queued' | 'rendering' | 'completed' | 'failed';
  renderProgress: number;
  activeRenderJobId: string | null;

  // NEW: Caption state
  captionMode: 'off' | 'on' | 'preview';
  captionStyle: CaptionStylePreset;
  captionSettings: CaptionSettings;
  activeCaptionTrackId: string | null;

  // NEW: Timeline state
  timeline: TimelineState;
}

interface TimelineState {
  tracks: Track[];
  segments: Segment[];
  transitions: Transition[];
  captionTracks: CaptionTrack[];
}

interface CaptionSettings {
  position: 'top' | 'center' | 'bottom';
  size: 'small' | 'medium' | 'large';
  boxed: boolean;
  fontFamily: string;
  fontSize: number;
}

// NEW: Actions
const actions = {
  // Render actions
  startRender: async (exportSettings: ExportSettings) => { /* ... */ },
  cancelRender: async () => { /* ... */ },
  pollRenderStatus: async () => { /* ... */ },

  // Caption actions
  setCaptionMode: (mode: 'off' | 'on' | 'preview') => { /* ... */ },
  setCaptionStyle: (preset: CaptionStylePreset) => { /* ... */ },
  addCaptionSegment: (segment: CaptionSegment) => { /* ... */ },
  updateCaptionSegment: (id: string, updates: Partial<CaptionSegment>) => { /* ... */ },
  deleteCaptionSegment: (id: string) => { /* ... */ },

  // Timeline serialization
  serializeTimeline: (): TimelineState => { /* ... */ },
  loadTimeline: (timeline: TimelineState) => { /* ... */ },
};
```

### New Components

#### 1. Export Dialog (`/src/components/editor/export/ExportDialog.tsx`)
```typescript
interface ExportDialogProps {
  projectId: string;
  onClose: () => void;
}

export function ExportDialog({ projectId, onClose }: ExportDialogProps) {
  // Export settings: format, resolution, quality, include captions/audio
  // Progress indicator
  // Cancel button
  // Download link when complete
}
```

#### 2. Caption Controls (`/src/components/editor/captions/CaptionControls.tsx`)
```typescript
interface CaptionControlsProps {
  trackId: string;
  onStyleChange: (style: CaptionStyle) => void;
}

export function CaptionControls({ trackId, onStyleChange }: CaptionControlsProps) {
  // Style preset selector (clean, bold, minimal, custom)
  // Position selector
  // Size controls
  // Box toggle
  // Live preview
}
```

#### 3. Caption Track Lane (`/src/components/editor/timeline/CaptionTrack.tsx`)
```typescript
interface CaptionTrackProps {
  trackId: string;
  segments: CaptionSegment[];
  zoom: number;
}

export function CaptionTrack({ trackId, segments, zoom }: CaptionTrackProps) {
  // Render caption segments as timeline blocks
  // Enable drag to reposition
  // Enable resize handles for timing
  // Click to edit text
}
```

#### 4. Caption Overlay (`/src/components/editor/captions/CaptionOverlay.tsx`)
```typescript
interface CaptionOverlayProps {
  currentTime: number;
  segments: CaptionSegment[];
  style: CaptionStyle;
}

export function CaptionOverlay({ currentTime, segments, style }: CaptionOverlayProps) {
  // Render active caption at current playhead position
  // Apply styling (position, size, box, font)
  // Match export rendering
}
```

### New Services

#### Render Service (`/src/services/renderService.ts`)
```typescript
export const renderService = {
  /**
   * Start a new render job
   */
  async startRender(projectId: string, settings: ExportSettings): Promise<RenderJob> {
    // 1. Serialize timeline from editorStore
    // 2. Create render_job record
    // 3. Trigger supabase edge function: render-video
    // 4. Return job ID for polling
  },

  /**
   * Poll render job status
   */
  async getRenderStatus(jobId: string): Promise<RenderJob> {
    // Query render_jobs table
    // Return status, progress, output_urls
  },

  /**
   * Cancel running render
   */
  async cancelRender(jobId: string): Promise<void> {
    // Update render_job status to 'cancelled'
    // Signal edge function to abort
  },
};
```

#### Caption Service (`/src/services/captionService.ts`)
```typescript
export const captionService = {
  /**
   * Get all caption tracks for project
   */
  async getTracks(projectId: string): Promise<CaptionTrack[]> {
    const { data, error } = await supabase
      .from('caption_tracks')
      .select('*')
      .eq('project_id', projectId);
    return data || [];
  },

  /**
   * Create new caption track
   */
  async createTrack(projectId: string, name: string, stylePreset: string): Promise<CaptionTrack> {
    const { data, error } = await supabase
      .from('caption_tracks')
      .insert({ project_id: projectId, name, style_preset: stylePreset })
      .select()
      .single();
    return data;
  },

  /**
   * Get segments for caption track
   */
  async getSegments(trackId: string): Promise<CaptionSegment[]> {
    const { data, error } = await supabase
      .from('caption_segments')
      .select('*')
      .eq('track_id', trackId)
      .order('start_time');
    return data || [];
  },

  /**
   * Auto-generate captions from transcript
   */
  async generateFromTranscript(trackId: string, clipId: string): Promise<void> {
    // Fetch clip transcript
    // Split into caption-sized chunks (2-3 seconds each)
    // Insert caption_segments
  },
};
```

### Edge Functions

#### New: `render-video` Edge Function
**File:** `/supabase/functions/render-video/index.ts`

**Responsibilities:**
1. Receive render job ID
2. Load timeline from `project_timeline_state`
3. Download video segments, audio tracks, caption data
4. Compose final video using FFmpeg:
   - Concatenate segments
   - Mix audio tracks
   - Burn in captions (if enabled)
   - Apply transitions
5. Upload output to storage
6. Update `render_jobs` with output URL

**Technology Options:**
- **Option A:** FFmpeg in Edge Function (requires Deno FFmpeg)
- **Option B:** Queue to external render worker (AWS Lambda, Replicate)
- **Recommended:** Start with Option A for MVP, migrate to Option B for scale

### Component Updates

#### Update: `VideoEditor.tsx`
```typescript
// Add Export button to toolbar
<Button onClick={handleExportClick}>
  <Download className="h-4 w-4" />
  Export Video
</Button>

// Add Export Dialog
{showExportDialog && (
  <ExportDialog projectId={projectId} onClose={() => setShowExportDialog(false)} />
)}
```

#### Update: `ProgramMonitor.tsx`
```typescript
// Add caption overlay rendering
<div className="relative">
  <VideoPlayer src={videoSrc} currentTime={currentTime} />
  {captionMode !== 'off' && (
    <CaptionOverlay
      currentTime={currentTime}
      segments={captionSegments}
      style={captionStyle}
    />
  )}
</div>
```

#### Update: `Timeline.tsx`
```typescript
// Add caption track lane below video tracks
<div className="timeline-tracks">
  {videoTracks.map(track => <VideoTrack key={track.id} {...track} />)}
  {audioTracks.map(track => <AudioTrack key={track.id} {...track} />)}
  {captionTracks.map(track => <CaptionTrack key={track.id} {...track} />)}
</div>
```

### Phase 1 Acceptance Criteria

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

### Phase 1 Testing Checklist

- [ ] Export 1-minute video with 3 segments
- [ ] Export video with captions enabled
- [ ] Export video with captions disabled
- [ ] Cancel render mid-progress
- [ ] Edit caption text and verify in export
- [ ] Change caption position and verify in export
- [ ] Auto-generate captions from transcript
- [ ] Timeline state persists across page refresh

---

## Phase 2: Audio System & Voiceover
**Duration:** 2-3 weeks
**Priority:** P0 (Must-Have)
**Depends On:** Phase 1

### Goals
1. Add audio track lanes to timeline
2. Enable in-browser voiceover recording
3. Support audio mixing (gain, fades)
4. Include audio in exports

### Database Schema Changes

#### New Table: `audio_assets`
```sql
CREATE TABLE audio_assets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    type TEXT NOT NULL CHECK (type IN ('music', 'voiceover', 'sfx')),
    storage_url TEXT NOT NULL,
    duration NUMERIC NOT NULL, -- seconds
    sample_rate INTEGER DEFAULT 48000,
    channels INTEGER DEFAULT 2,
    waveform_data JSONB, -- [{time: 0, amplitude: 0.5}, ...]
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audio_assets_project ON audio_assets(project_id);
CREATE INDEX idx_audio_assets_type ON audio_assets(type);
```

#### New Table: `audio_segments`
```sql
CREATE TABLE audio_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    audio_asset_id UUID NOT NULL REFERENCES audio_assets(id) ON DELETE CASCADE,
    track_index INTEGER NOT NULL DEFAULT 0, -- multiple audio lanes
    start_time NUMERIC NOT NULL, -- project timeline position
    in_point NUMERIC NOT NULL DEFAULT 0, -- trim start
    out_point NUMERIC NOT NULL, -- trim end
    gain_db NUMERIC DEFAULT 0 CHECK (gain_db BETWEEN -60 AND 20),
    fade_in_ms INTEGER DEFAULT 0 CHECK (fade_in_ms >= 0),
    fade_out_ms INTEGER DEFAULT 0 CHECK (fade_out_ms >= 0),
    created_at TIMESTAMPTZ DEFAULT NOW(),

    CONSTRAINT valid_in_out CHECK (out_point > in_point)
);

CREATE INDEX idx_audio_segments_project ON audio_segments(project_id);
CREATE INDEX idx_audio_segments_asset ON audio_segments(audio_asset_id);
```

### Store Changes

```typescript
interface EditorState {
  // NEW: Audio state
  audioTracks: AudioTrack[];
  selectedAudioSegmentId: string | null;
  audioMixerVisible: boolean;

  // NEW: Recording state
  isRecording: boolean;
  recordingAssetId: string | null;
  recordingDuration: number;
}

interface AudioTrack {
  id: string;
  name: string;
  segments: AudioSegment[];
  muted: boolean;
  solo: boolean;
  gain: number; // track-level gain
}

interface AudioSegment {
  id: string;
  assetId: string;
  startTime: number;
  inPoint: number;
  outPoint: number;
  gainDb: number;
  fadeInMs: number;
  fadeOutMs: number;
}

// NEW: Actions
const actions = {
  // Audio track management
  addAudioTrack: () => { /* ... */ },
  removeAudioTrack: (trackId: string) => { /* ... */ },

  // Audio segment management
  addAudioSegment: (trackId: string, segment: AudioSegment) => { /* ... */ },
  updateAudioSegment: (id: string, updates: Partial<AudioSegment>) => { /* ... */ },
  removeAudioSegment: (id: string) => { /* ... */ },

  // Voiceover recording
  startRecording: async () => { /* ... */ },
  stopRecording: async () => { /* ... */ },
  insertRecording: (trackId: string, startTime: number) => { /* ... */ },

  // Audio mixing
  setTrackGain: (trackId: string, gain: number) => { /* ... */ },
  setTrackMute: (trackId: string, muted: boolean) => { /* ... */ },
  setTrackSolo: (trackId: string, solo: boolean) => { /* ... */ },
};
```

### New Components

#### 1. Audio Track Lane (`/src/components/editor/audio/AudioTrackLane.tsx`)
```typescript
interface AudioTrackLaneProps {
  track: AudioTrack;
  zoom: number;
  onSegmentMove: (segmentId: string, newStartTime: number) => void;
}

export function AudioTrackLane({ track, zoom, onSegmentMove }: AudioTrackLaneProps) {
  // Render waveform visualization
  // Enable drag-and-drop positioning
  // Show trim handles
  // Display gain envelope
}
```

#### 2. Voiceover Recorder (`/src/components/editor/audio/VoiceoverRecorder.tsx`)
```typescript
interface VoiceoverRecorderProps {
  onRecordingComplete: (audioBlob: Blob, duration: number) => void;
}

export function VoiceoverRecorder({ onRecordingComplete }: VoiceoverRecorderProps) {
  // Microphone input selector
  // Record button with live timer
  // Waveform preview during recording
  // Stop button
  // Upload to storage and create audio_asset
}
```

#### 3. Audio Inspector (`/src/components/editor/audio/AudioInspector.tsx`)
```typescript
interface AudioInspectorProps {
  segmentId: string;
}

export function AudioInspector({ segmentId }: AudioInspectorProps) {
  // Gain slider (-60dB to +20dB)
  // Fade in/out duration inputs
  // Trim controls (in-point, out-point)
  // Delete button
}
```

#### 4. Audio Mixer Panel (`/src/components/editor/audio/AudioMixerPanel.tsx`)
```typescript
export function AudioMixerPanel() {
  // Track-level gain faders
  // Mute/Solo buttons per track
  // Master output level meter
  // Peak/RMS indicators
}
```

### New Services

#### Audio Service (`/src/services/audioService.ts`)
```typescript
export const audioService = {
  /**
   * Record voiceover from microphone
   */
  async recordVoiceover(): Promise<AudioRecording> {
    // Use MediaRecorder API
    // Capture audio stream
    // Return blob and duration
  },

  /**
   * Upload audio asset to storage
   */
  async uploadAudio(blob: Blob, projectId: string, type: string): Promise<AudioAsset> {
    // Upload to Supabase Storage
    // Create audio_assets record
    // Generate waveform data
    return asset;
  },

  /**
   * Get audio segments for project
   */
  async getSegments(projectId: string): Promise<AudioSegment[]> {
    const { data, error } = await supabase
      .from('audio_segments')
      .select(`
        *,
        audio_asset:audio_assets(*)
      `)
      .eq('project_id', projectId)
      .order('start_time');
    return data || [];
  },

  /**
   * Generate waveform visualization data
   */
  async generateWaveform(audioUrl: string): Promise<WaveformData[]> {
    // Decode audio
    // Sample amplitude at regular intervals
    // Return normalized amplitude array
  },
};
```

### Component Updates

#### Update: `Timeline.tsx`
```typescript
// Add audio track lanes
<div className="timeline-tracks">
  {videoTracks.map(track => <VideoTrack key={track.id} {...track} />)}
  <Separator />
  {audioTracks.map(track => (
    <AudioTrackLane key={track.id} track={track} zoom={zoom} />
  ))}
  <Button variant="ghost" onClick={addAudioTrack}>
    <Plus className="h-4 w-4" />
    Add Audio Track
  </Button>
</div>
```

#### Update: `SourceMonitor.tsx`
```typescript
// Add voiceover recording button
<div className="source-monitor-controls">
  <Button onClick={handleRecordVoiceover}>
    <Mic className="h-4 w-4" />
    Record Voiceover
  </Button>
</div>

{showRecorder && (
  <VoiceoverRecorder onRecordingComplete={handleInsertRecording} />
)}
```

#### Update: `render-video` Edge Function
```typescript
// Add audio mixing to render pipeline
async function renderVideo(jobId: string) {
  // ... existing video composition

  // Load audio segments
  const audioSegments = await getAudioSegments(projectId);

  // Download audio assets
  // Apply gain, fades, timing
  // Mix audio tracks
  // Merge with video

  // Final output with mixed audio
}
```

### Phase 2 Acceptance Criteria

- [ ] User can add audio track lanes to timeline
- [ ] User can drag audio files into timeline
- [ ] User can click "Record Voiceover" and start recording
- [ ] Recording shows live duration timer
- [ ] Recorded voiceover auto-inserts at playhead position
- [ ] Audio segments display waveform visualization
- [ ] User can trim audio segment (in/out points)
- [ ] User can adjust segment gain (-60dB to +20dB)
- [ ] User can add fade in/out to audio segments
- [ ] Track-level mute/solo buttons work correctly
- [ ] Exported video includes mixed audio tracks
- [ ] Multiple audio tracks mix correctly in export

### Phase 2 Testing Checklist

- [ ] Record 10-second voiceover
- [ ] Insert voiceover at specific timeline position
- [ ] Adjust voiceover gain to -6dB
- [ ] Add 500ms fade in and fade out
- [ ] Trim first 2 seconds of voiceover
- [ ] Add music track under voiceover
- [ ] Solo voiceover track (mute music)
- [ ] Export video with mixed audio
- [ ] Verify audio timing matches timeline
- [ ] Test with 3+ simultaneous audio tracks

---

## Phase 3: Advanced Captions & Transitions
**Duration:** 2-3 weeks
**Priority:** P1 (High-Value Enhancement)
**Depends On:** Phase 1, Phase 2

### Goals
1. Add professional caption styling options
2. Enable transition effects between segments
3. Support keyframe-based motion graphics

### Database Schema Changes

#### Update Table: `caption_segments` - Add Advanced Styling
```sql
-- Extend style JSONB to support:
ALTER TABLE caption_segments
  ADD COLUMN style_v2 JSONB DEFAULT '{}'::jsonb;

-- style_v2 schema:
-- {
--   "rotation_deg": 0,
--   "font_family": "Inter",
--   "font_color": "#FFFFFF",
--   "font_size": 48,
--   "stroke_color": "#000000",
--   "stroke_width": 2,
--   "bg_shape": "none" | "box" | "pill" | "highlight",
--   "bg_color": "#000000",
--   "bg_opacity": 0.8,
--   "enter_anim": "fade" | "slide" | "scale" | "none",
--   "exit_anim": "fade" | "slide" | "scale" | "none",
--   "animation_duration_ms": 300
-- }
```

#### New Table: `segment_transitions`
```sql
CREATE TABLE segment_transitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    from_segment_id UUID REFERENCES segments(id) ON DELETE CASCADE,
    to_segment_id UUID REFERENCES segments(id) ON DELETE CASCADE,
    type TEXT NOT NULL CHECK (type IN ('cut', 'dissolve', 'dip_black', 'dip_white', 'wipe_left', 'wipe_right')),
    duration_ms INTEGER DEFAULT 500 CHECK (duration_ms > 0 AND duration_ms <= 5000),
    created_at TIMESTAMPTZ DEFAULT NOW(),

    CONSTRAINT unique_transition UNIQUE(from_segment_id, to_segment_id)
);

CREATE INDEX idx_transitions_project ON segment_transitions(project_id);
```

#### New Table: `segment_keyframes`
```sql
CREATE TABLE segment_keyframes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    t NUMERIC NOT NULL CHECK (t >= 0 AND t <= 1), -- normalized time within segment (0 = start, 1 = end)
    transform JSONB NOT NULL, -- {x, y, scale, rotation, opacity}
    created_at TIMESTAMPTZ DEFAULT NOW(),

    CONSTRAINT unique_keyframe_time UNIQUE(segment_id, t)
);

-- transform JSONB schema:
-- {
--   "x": 0,           // horizontal position offset (pixels)
--   "y": 0,           // vertical position offset (pixels)
--   "scale": 1.0,     // scale factor (1.0 = 100%)
--   "rotation": 0,    // rotation in degrees
--   "opacity": 1.0    // opacity (0.0 - 1.0)
-- }

CREATE INDEX idx_keyframes_segment ON segment_keyframes(segment_id);
```

### Store Changes

```typescript
interface EditorState {
  // NEW: Advanced caption state
  selectedCaptionStyle: CaptionStyleV2;
  captionPresets: CaptionPreset[];

  // NEW: Transition state
  transitions: Map<string, Transition>; // key: "from_id:to_id"
  selectedTransitionType: TransitionType;

  // NEW: Keyframe state
  keyframes: Map<string, Keyframe[]>; // key: segment_id
  keyframeEditorVisible: boolean;
  selectedKeyframe: string | null;
}

interface CaptionStyleV2 {
  rotationDeg: number;
  fontFamily: string;
  fontColor: string;
  fontSize: number;
  strokeColor: string;
  strokeWidth: number;
  bgShape: 'none' | 'box' | 'pill' | 'highlight';
  bgColor: string;
  bgOpacity: number;
  enterAnim: 'fade' | 'slide' | 'scale' | 'none';
  exitAnim: 'fade' | 'slide' | 'scale' | 'none';
  animationDurationMs: number;
}

interface Transition {
  id: string;
  fromSegmentId: string;
  toSegmentId: string;
  type: 'cut' | 'dissolve' | 'dip_black' | 'dip_white' | 'wipe_left' | 'wipe_right';
  durationMs: number;
}

interface Keyframe {
  id: string;
  segmentId: string;
  t: number; // 0-1
  transform: {
    x: number;
    y: number;
    scale: number;
    rotation: number;
    opacity: number;
  };
}

// NEW: Actions
const actions = {
  // Caption styling
  applyCaptionPreset: (preset: CaptionPreset) => { /* ... */ },
  updateCaptionStyle: (segmentId: string, style: Partial<CaptionStyleV2>) => { /* ... */ },

  // Transitions
  addTransition: (fromId: string, toId: string, type: TransitionType) => { /* ... */ },
  updateTransition: (id: string, updates: Partial<Transition>) => { /* ... */ },
  removeTransition: (id: string) => { /* ... */ },

  // Keyframes
  addKeyframe: (segmentId: string, t: number, transform: Transform) => { /* ... */ },
  updateKeyframe: (id: string, transform: Partial<Transform>) => { /* ... */ },
  removeKeyframe: (id: string) => { /* ... */ },
  interpolateTransform: (segmentId: string, currentTime: number) => Transform,
};
```

### New Components

#### 1. Caption Style Inspector (`/src/components/editor/captions/CaptionStyleInspector.tsx`)
```typescript
interface CaptionStyleInspectorProps {
  segmentId: string;
  currentStyle: CaptionStyleV2;
  onStyleChange: (style: CaptionStyleV2) => void;
}

export function CaptionStyleInspector({ segmentId, currentStyle, onStyleChange }: CaptionStyleInspectorProps) {
  return (
    <div className="space-y-4">
      {/* Font controls */}
      <div>
        <Label>Font Family</Label>
        <Select value={currentStyle.fontFamily} onValueChange={handleFontChange}>
          <SelectItem value="Inter">Inter</SelectItem>
          <SelectItem value="Montserrat">Montserrat</SelectItem>
          <SelectItem value="Bebas Neue">Bebas Neue</SelectItem>
        </Select>
      </div>

      <div>
        <Label>Font Size</Label>
        <Slider value={[currentStyle.fontSize]} onValueChange={handleFontSizeChange} min={24} max={120} />
      </div>

      {/* Color controls */}
      <div>
        <Label>Text Color</Label>
        <ColorPicker value={currentStyle.fontColor} onChange={handleFontColorChange} />
      </div>

      <div>
        <Label>Stroke Color</Label>
        <ColorPicker value={currentStyle.strokeColor} onChange={handleStrokeColorChange} />
      </div>

      <div>
        <Label>Stroke Width</Label>
        <Slider value={[currentStyle.strokeWidth]} onValueChange={handleStrokeWidthChange} min={0} max={10} />
      </div>

      {/* Background shape */}
      <div>
        <Label>Background Shape</Label>
        <ToggleGroup value={currentStyle.bgShape} onValueChange={handleBgShapeChange}>
          <ToggleGroupItem value="none">None</ToggleGroupItem>
          <ToggleGroupItem value="box">Box</ToggleGroupItem>
          <ToggleGroupItem value="pill">Pill</ToggleGroupItem>
          <ToggleGroupItem value="highlight">Highlight</ToggleGroupItem>
        </ToggleGroup>
      </div>

      {/* Rotation */}
      <div>
        <Label>Rotation</Label>
        <Slider value={[currentStyle.rotationDeg]} onValueChange={handleRotationChange} min={-45} max={45} />
      </div>

      {/* Animations */}
      <div>
        <Label>Enter Animation</Label>
        <Select value={currentStyle.enterAnim} onValueChange={handleEnterAnimChange}>
          <SelectItem value="none">None</SelectItem>
          <SelectItem value="fade">Fade</SelectItem>
          <SelectItem value="slide">Slide</SelectItem>
          <SelectItem value="scale">Scale</SelectItem>
        </Select>
      </div>

      <div>
        <Label>Exit Animation</Label>
        <Select value={currentStyle.exitAnim} onValueChange={handleExitAnimChange}>
          <SelectItem value="none">None</SelectItem>
          <SelectItem value="fade">Fade</SelectItem>
          <SelectItem value="slide">Slide</SelectItem>
          <SelectItem value="scale">Scale</SelectItem>
        </Select>
      </div>

      {/* Preset buttons */}
      <div className="flex gap-2">
        <Button onClick={() => applyCaptionPreset('bold')}>Bold</Button>
        <Button onClick={() => applyCaptionPreset('minimal')}>Minimal</Button>
        <Button onClick={() => applyCaptionPreset('highlight')}>Highlight</Button>
      </div>
    </div>
  );
}
```

#### 2. Transition Editor (`/src/components/editor/transitions/TransitionEditor.tsx`)
```typescript
interface TransitionEditorProps {
  fromSegmentId: string;
  toSegmentId: string;
  transition: Transition | null;
  onUpdate: (transition: Transition) => void;
}

export function TransitionEditor({ fromSegmentId, toSegmentId, transition, onUpdate }: TransitionEditorProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Transition</CardTitle>
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          <div>
            <Label>Type</Label>
            <Select value={transition?.type || 'cut'} onValueChange={handleTypeChange}>
              <SelectItem value="cut">Cut</SelectItem>
              <SelectItem value="dissolve">Dissolve</SelectItem>
              <SelectItem value="dip_black">Dip to Black</SelectItem>
              <SelectItem value="dip_white">Dip to White</SelectItem>
              <SelectItem value="wipe_left">Wipe Left</SelectItem>
              <SelectItem value="wipe_right">Wipe Right</SelectItem>
            </Select>
          </div>

          {transition?.type !== 'cut' && (
            <div>
              <Label>Duration (ms)</Label>
              <Slider
                value={[transition?.durationMs || 500]}
                onValueChange={handleDurationChange}
                min={100}
                max={2000}
                step={100}
              />
              <span className="text-sm text-muted-foreground">{transition?.durationMs || 500}ms</span>
            </div>
          )}

          {/* Preview area */}
          <div className="border rounded p-4">
            <TransitionPreview transition={transition} />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 3. Transform Inspector (`/src/components/editor/keyframes/TransformInspector.tsx`)
```typescript
interface TransformInspectorProps {
  segmentId: string;
  keyframes: Keyframe[];
  currentTime: number;
  onKeyframeAdd: (t: number, transform: Transform) => void;
  onKeyframeUpdate: (id: string, transform: Transform) => void;
}

export function TransformInspector({ segmentId, keyframes, currentTime, onKeyframeAdd, onKeyframeUpdate }: TransformInspectorProps) {
  const currentTransform = interpolateTransform(keyframes, currentTime);

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <Label>Position X</Label>
        <Input
          type="number"
          value={currentTransform.x}
          onChange={handleXChange}
          className="w-20"
        />
      </div>

      <div className="flex items-center justify-between">
        <Label>Position Y</Label>
        <Input
          type="number"
          value={currentTransform.y}
          onChange={handleYChange}
          className="w-20"
        />
      </div>

      <div className="flex items-center justify-between">
        <Label>Scale</Label>
        <Slider
          value={[currentTransform.scale * 100]}
          onValueChange={handleScaleChange}
          min={10}
          max={300}
        />
        <span className="text-sm">{(currentTransform.scale * 100).toFixed(0)}%</span>
      </div>

      <div className="flex items-center justify-between">
        <Label>Rotation</Label>
        <Slider
          value={[currentTransform.rotation]}
          onValueChange={handleRotationChange}
          min={-180}
          max={180}
        />
        <span className="text-sm">{currentTransform.rotation}°</span>
      </div>

      <div className="flex items-center justify-between">
        <Label>Opacity</Label>
        <Slider
          value={[currentTransform.opacity * 100]}
          onValueChange={handleOpacityChange}
          min={0}
          max={100}
        />
        <span className="text-sm">{(currentTransform.opacity * 100).toFixed(0)}%</span>
      </div>

      <Separator />

      <div>
        <Label>Keyframes</Label>
        <KeyframeTimeline
          keyframes={keyframes}
          currentTime={currentTime}
          onKeyframeSelect={handleKeyframeSelect}
          onKeyframeAdd={onKeyframeAdd}
        />
      </div>

      <div className="flex gap-2">
        <Button onClick={handleAddKeyframe}>
          <Plus className="h-4 w-4" />
          Add Keyframe
        </Button>
        {selectedKeyframe && (
          <Button variant="destructive" onClick={handleDeleteKeyframe}>
            <Trash className="h-4 w-4" />
            Delete
          </Button>
        )}
      </div>
    </div>
  );
}
```

#### 4. Caption Keyframe Mini Editor (`/src/components/editor/captions/CaptionKeyframeMiniEditor.tsx`)
```typescript
interface CaptionKeyframeMiniEditorProps {
  segmentId: string;
  currentStyle: CaptionStyleV2;
}

export function CaptionKeyframeMiniEditor({ segmentId, currentStyle }: CaptionKeyframeMiniEditorProps) {
  // Simplified keyframe controls for caption-specific animations
  // Position path editor
  // Rotation timeline
  // Scale/opacity curves
}
```

### Component Updates

#### Update: `CaptionOverlay.tsx`
```typescript
// Add support for advanced styling
export function CaptionOverlay({ currentTime, segments, style }: CaptionOverlayProps) {
  const activeSegment = findActiveSegment(segments, currentTime);
  if (!activeSegment) return null;

  const styleV2 = activeSegment.style_v2 || style;
  const progress = (currentTime - activeSegment.start_time) / (activeSegment.end_time - activeSegment.start_time);

  // Apply enter/exit animations
  const animationStyle = calculateAnimationStyle(styleV2, progress);

  return (
    <div
      className="caption-overlay"
      style={{
        transform: `rotate(${styleV2.rotationDeg}deg)`,
        fontFamily: styleV2.fontFamily,
        fontSize: styleV2.fontSize,
        color: styleV2.fontColor,
        textStroke: `${styleV2.strokeWidth}px ${styleV2.strokeColor}`,
        ...animationStyle,
      }}
    >
      {styleV2.bgShape !== 'none' && (
        <div className={`caption-bg caption-bg-${styleV2.bgShape}`} style={{
          backgroundColor: styleV2.bgColor,
          opacity: styleV2.bgOpacity,
        }} />
      )}
      <span>{activeSegment.text}</span>
    </div>
  );
}
```

#### Update: `Timeline.tsx`
```typescript
// Show transition indicators between segments
{segments.map((segment, index) => {
  const nextSegment = segments[index + 1];
  const transition = nextSegment ? transitions.get(`${segment.id}:${nextSegment.id}`) : null;

  return (
    <React.Fragment key={segment.id}>
      <TimelineSegment segment={segment} />
      {transition && <TransitionIndicator transition={transition} />}
    </React.Fragment>
  );
})}
```

#### Update: `ProgramMonitor.tsx`
```typescript
// Apply keyframe transforms to video segments
const transform = interpolateTransform(keyframes, currentTime);

<div
  className="video-container"
  style={{
    transform: `translate(${transform.x}px, ${transform.y}px) scale(${transform.scale}) rotate(${transform.rotation}deg)`,
    opacity: transform.opacity,
  }}
>
  <VideoPlayer src={videoSrc} currentTime={currentTime} />
</div>
```

### Phase 3 Acceptance Criteria

- [ ] User can change caption font family and size
- [ ] User can set caption text color and stroke
- [ ] User can choose background shape (none/box/pill/highlight)
- [ ] User can rotate captions (-45° to +45°)
- [ ] User can apply enter/exit animations to captions
- [ ] Caption presets (Bold, Minimal, Highlight) work correctly
- [ ] User can add transition between two segments
- [ ] Dissolve, dip to black, and wipe transitions work
- [ ] Transition duration is adjustable (100ms-2000ms)
- [ ] User can add keyframe at current playhead position
- [ ] User can adjust position, scale, rotation, opacity per keyframe
- [ ] Keyframe interpolation produces smooth motion
- [ ] Exported video includes caption styling and animations
- [ ] Exported video includes transitions
- [ ] Exported video includes keyframe motion

### Phase 3 Testing Checklist

- [ ] Create caption with custom font and colors
- [ ] Apply 15° rotation to caption
- [ ] Add slide-in enter animation
- [ ] Add fade-out exit animation
- [ ] Apply "Highlight" preset to caption
- [ ] Add dissolve transition between two segments (500ms)
- [ ] Add keyframe at segment start (scale 1.0)
- [ ] Add keyframe at segment mid (scale 1.5)
- [ ] Add keyframe at segment end (scale 1.0)
- [ ] Verify smooth scale animation during playback
- [ ] Export video with styled captions and transitions
- [ ] Verify all effects render correctly in export

---

## Phase 4: Face-Aware Features
**Duration:** 3-4 weeks
**Priority:** P1 (High-Value Enhancement)
**Depends On:** Phase 1, Entity/Face Recognition System (separate track)

### Goals
1. Add face-aware reframing controls to editor
2. Enable person-based search filters
3. Support vertical video optimization presets

### Prerequisites
- Entity/Face Recognition System must be deployed (separate implementation)
- `face_detections` table populated with real face data
- `person_clusters` table with labeled individuals

### Database Schema Changes

#### Update: Extend `segments` table with reframing metadata
```sql
ALTER TABLE segments
  ADD COLUMN reframe_config JSONB DEFAULT '{}'::jsonb;

-- reframe_config schema:
-- {
--   "mode": "center" | "face_auto" | "manual_keyframe",
--   "preset": "single_speaker" | "two_speaker" | "reaction" | "custom",
--   "target_aspect": "9:16" | "1:1" | "4:5",
--   "crop_keyframes": [
--     {"t": 0, "x": 0, "y": 0, "width": 1080, "height": 1920},
--     {"t": 1, "x": 0, "y": 0, "width": 1080, "height": 1920}
--   ],
--   "face_tracking_enabled": true,
--   "min_face_size": 0.2
-- }
```

#### New: Person-based search filters (uses existing person tables)
No new tables needed - uses:
- `person_clusters` (from face recognition system)
- `person_appearances` (from face recognition system)
- `person_entity_links` (from face recognition system)

### Store Changes

```typescript
interface EditorState {
  // NEW: Reframe state
  reframeMode: 'off' | 'preview' | 'enabled';
  reframePreset: ReframePreset | null;
  activeCropPath: CropKeyframe[];

  // NEW: Person filter state
  selectedPeople: string[]; // person_cluster_ids
  peopleFilterMode: 'any' | 'all' | 'exact';
  minPeopleCount: number | null;
  maxPeopleCount: number | null;
}

type ReframePreset =
  | 'single_speaker'
  | 'two_speaker'
  | 'reaction'
  | 'custom';

interface CropKeyframe {
  t: number; // normalized time (0-1)
  x: number;
  y: number;
  width: number;
  height: number;
}

// NEW: Actions
const actions = {
  // Reframing
  setReframeMode: (mode: 'off' | 'preview' | 'enabled') => { /* ... */ },
  applyReframePreset: (segmentId: string, preset: ReframePreset) => { /* ... */ },
  addCropKeyframe: (segmentId: string, t: number, crop: Crop) => { /* ... */ },
  updateCropKeyframe: (segmentId: string, keyframeId: string, crop: Partial<Crop>) => { /* ... */ },
  generateFaceAutoCrop: async (segmentId: string) => { /* ... */ },

  // Person filters
  setSelectedPeople: (personIds: string[]) => { /* ... */ },
  setPeopleFilterMode: (mode: 'any' | 'all' | 'exact') => { /* ... */ },
  setPeopleCountRange: (min: number | null, max: number | null) => { /* ... */ },
  searchClipsByPeople: async () => { /* ... */ },
};
```

### New Components

#### 1. Reframe Panel (`/src/components/editor/reframe/ReframePanel.tsx`)
```typescript
interface ReframePanelProps {
  segmentId: string;
  onReframeApply: (config: ReframeConfig) => void;
}

export function ReframePanel({ segmentId, onReframeApply }: ReframePanelProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Reframe for Vertical</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Target aspect ratio */}
        <div>
          <Label>Target Format</Label>
          <ToggleGroup value={targetAspect} onValueChange={setTargetAspect}>
            <ToggleGroupItem value="9:16">9:16 (TikTok/Reels)</ToggleGroupItem>
            <ToggleGroupItem value="1:1">1:1 (Instagram)</ToggleGroupItem>
            <ToggleGroupItem value="4:5">4:5 (Instagram)</ToggleGroupItem>
          </ToggleGroup>
        </div>

        {/* Reframe mode */}
        <div>
          <Label>Reframe Mode</Label>
          <Select value={reframeMode} onValueChange={setReframeMode}>
            <SelectItem value="center">Center (Static)</SelectItem>
            <SelectItem value="face_auto">Face Auto-Track</SelectItem>
            <SelectItem value="manual_keyframe">Manual Keyframes</SelectItem>
          </Select>
        </div>

        {/* Preset buttons */}
        <div>
          <Label>Smart Presets</Label>
          <div className="grid grid-cols-2 gap-2">
            <Button onClick={() => applyPreset('single_speaker')}>
              <User className="h-4 w-4 mr-2" />
              Single Speaker
            </Button>
            <Button onClick={() => applyPreset('two_speaker')}>
              <Users className="h-4 w-4 mr-2" />
              Two Person
            </Button>
            <Button onClick={() => applyPreset('reaction')}>
              <Smile className="h-4 w-4 mr-2" />
              Reaction Shot
            </Button>
          </div>
        </div>

        {reframeMode === 'face_auto' && (
          <div>
            <Label>Face Tracking</Label>
            <div className="space-y-2">
              <div className="flex items-center justify-between">
                <span className="text-sm">Enable face tracking</span>
                <Switch checked={faceTrackingEnabled} onCheckedChange={setFaceTrackingEnabled} />
              </div>
              <div>
                <Label>Min Face Size</Label>
                <Slider
                  value={[minFaceSize * 100]}
                  onValueChange={(v) => setMinFaceSize(v[0] / 100)}
                  min={10}
                  max={50}
                />
                <span className="text-sm text-muted-foreground">{(minFaceSize * 100).toFixed(0)}%</span>
              </div>
            </div>
          </div>
        )}

        {reframeMode === 'manual_keyframe' && (
          <div>
            <Label>Crop Keyframes</Label>
            <CropKeyframeEditor
              segmentId={segmentId}
              keyframes={cropKeyframes}
              onUpdate={handleKeyframeUpdate}
            />
          </div>
        )}

        {/* Preview */}
        <div>
          <Label>Preview</Label>
          <div className="border rounded aspect-[9/16] bg-black">
            <CropPathPreview
              segmentId={segmentId}
              cropPath={cropPath}
              currentTime={currentTime}
            />
          </div>
        </div>

        <Button onClick={handleApply} className="w-full">
          Apply Reframe
        </Button>
      </CardContent>
    </Card>
  );
}
```

#### 2. Crop Path Preview (`/src/components/editor/reframe/CropPathPreview.tsx`)
```typescript
interface CropPathPreviewProps {
  segmentId: string;
  cropPath: CropKeyframe[];
  currentTime: number;
}

export function CropPathPreview({ segmentId, cropPath, currentTime }: CropPathPreviewProps) {
  const currentCrop = interpolateCropPath(cropPath, currentTime);

  return (
    <div className="relative w-full h-full">
      {/* Full frame video */}
      <video ref={videoRef} className="absolute inset-0 w-full h-full object-contain" />

      {/* Crop region overlay */}
      <div
        className="absolute border-2 border-primary"
        style={{
          left: `${(currentCrop.x / originalWidth) * 100}%`,
          top: `${(currentCrop.y / originalHeight) * 100}%`,
          width: `${(currentCrop.width / originalWidth) * 100}%`,
          height: `${(currentCrop.height / originalHeight) * 100}%`,
        }}
      >
        {/* Crop handles */}
        <div className="absolute -top-2 -left-2 w-4 h-4 bg-primary rounded-full" />
        <div className="absolute -top-2 -right-2 w-4 h-4 bg-primary rounded-full" />
        <div className="absolute -bottom-2 -left-2 w-4 h-4 bg-primary rounded-full" />
        <div className="absolute -bottom-2 -right-2 w-4 h-4 bg-primary rounded-full" />
      </div>

      {/* Face detection overlays (if enabled) */}
      {faceDetections.map(face => (
        <div
          key={face.id}
          className="absolute border border-green-400"
          style={{
            left: `${face.bbox_x * 100}%`,
            top: `${face.bbox_y * 100}%`,
            width: `${face.bbox_w * 100}%`,
            height: `${face.bbox_h * 100}%`,
          }}
        />
      ))}
    </div>
  );
}
```

#### 3. People Filter Panel (`/src/components/search/PeopleFilterPanel.tsx`)
```typescript
interface PeopleFilterPanelProps {
  onFilterChange: (filter: PeopleFilter) => void;
}

export function PeopleFilterPanel({ onFilterChange }: PeopleFilterPanelProps) {
  const { data: people } = usePeople(); // Fetch labeled person clusters

  return (
    <Card>
      <CardHeader>
        <CardTitle>Filter by People</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Multi-select people */}
        <div>
          <Label>Select People</Label>
          <MultiSelect
            options={people.map(p => ({ value: p.id, label: p.display_name }))}
            value={selectedPeople}
            onChange={setSelectedPeople}
            placeholder="All people"
          />
        </div>

        {selectedPeople.length > 1 && (
          <div>
            <Label>Match Mode</Label>
            <Select value={matchMode} onValueChange={setMatchMode}>
              <SelectItem value="any">Any selected person</SelectItem>
              <SelectItem value="all">All selected people</SelectItem>
              <SelectItem value="exact">Exactly these people</SelectItem>
            </Select>
          </div>
        )}

        {/* People count range */}
        <div>
          <Label>Number of People</Label>
          <div className="flex gap-2 items-center">
            <Input
              type="number"
              placeholder="Min"
              value={minPeople || ''}
              onChange={(e) => setMinPeople(e.target.value ? parseInt(e.target.value) : null)}
              className="w-20"
            />
            <span>to</span>
            <Input
              type="number"
              placeholder="Max"
              value={maxPeople || ''}
              onChange={(e) => setMaxPeople(e.target.value ? parseInt(e.target.value) : null)}
              className="w-20"
            />
          </div>
        </div>

        {/* Quick filters */}
        <div>
          <Label>Quick Filters</Label>
          <div className="flex flex-wrap gap-2">
            <Badge variant="outline" className="cursor-pointer" onClick={() => setFilter({ minPeople: 1, maxPeople: 1 })}>
              Solo
            </Badge>
            <Badge variant="outline" className="cursor-pointer" onClick={() => setFilter({ minPeople: 2, maxPeople: 2 })}>
              Duo
            </Badge>
            <Badge variant="outline" className="cursor-pointer" onClick={() => setFilter({ minPeople: 3 })}>
              Group (3+)
            </Badge>
          </div>
        </div>

        <Button onClick={handleApplyFilter} className="w-full">
          Apply Filter
        </Button>
      </CardContent>
    </Card>
  );
}
```

#### 4. Person Timeline View (`/src/components/people/PersonTimeline.tsx`)
```typescript
interface PersonTimelineProps {
  personId: string;
}

export function PersonTimeline({ personId }: PersonTimelineProps) {
  const { data: appearances } = usePersonAppearances(personId);
  const { data: person } = usePerson(personId);

  return (
    <div className="space-y-4">
      <div className="flex items-center gap-4">
        <Avatar>
          <AvatarImage src={person.representative_thumbnail_url} />
          <AvatarFallback>{person.display_name[0]}</AvatarFallback>
        </Avatar>
        <div>
          <h3 className="font-semibold">{person.display_name}</h3>
          <p className="text-sm text-muted-foreground">
            {appearances.length} appearances
          </p>
        </div>
      </div>

      <div className="space-y-2">
        {appearances.map(appearance => (
          <Card key={appearance.id} className="p-4">
            <div className="flex items-center gap-4">
              <img
                src={appearance.clip.thumbnail_url}
                className="w-24 h-16 object-cover rounded"
              />
              <div className="flex-1">
                <p className="font-medium">{appearance.clip.title}</p>
                <p className="text-sm text-muted-foreground">
                  {formatTime(appearance.start_time)} - {formatTime(appearance.end_time)}
                </p>
                <p className="text-xs text-muted-foreground">
                  Dominance: {(appearance.dominance_score * 100).toFixed(0)}%
                </p>
              </div>
              <Button onClick={() => jumpToClip(appearance.clip_id, appearance.start_time)}>
                View
              </Button>
            </div>
          </Card>
        ))}
      </div>
    </div>
  );
}
```

### New Services

#### Reframe Service (`/src/services/reframeService.ts`)
```typescript
export const reframeService = {
  /**
   * Generate face-aware crop path for segment
   */
  async generateFaceAutoCrop(
    segmentId: string,
    targetAspect: '9:16' | '1:1' | '4:5',
    options: { minFaceSize: number }
  ): Promise<CropKeyframe[]> {
    // 1. Load face detections for segment time range
    const faces = await getFaceDetections(segmentId);

    // 2. For each frame with faces:
    //    - Calculate bounding box containing all faces
    //    - Apply padding
    //    - Constrain to target aspect ratio
    //    - Smooth transitions between frames

    // 3. Return crop keyframes at key moments
    return cropPath;
  },

  /**
   * Apply reframe preset
   */
  async applyPreset(
    segmentId: string,
    preset: ReframePreset
  ): Promise<ReframeConfig> {
    switch (preset) {
      case 'single_speaker':
        // Center on largest face, keep stable
        return {
          mode: 'face_auto',
          preset: 'single_speaker',
          target_aspect: '9:16',
          face_tracking_enabled: true,
          min_face_size: 0.25,
        };

      case 'two_speaker':
        // Frame both speakers, allow slight panning
        return {
          mode: 'face_auto',
          preset: 'two_speaker',
          target_aspect: '9:16',
          face_tracking_enabled: true,
          min_face_size: 0.20,
        };

      case 'reaction':
        // Emphasize most active/emotive face
        return {
          mode: 'face_auto',
          preset: 'reaction',
          target_aspect: '9:16',
          face_tracking_enabled: true,
          min_face_size: 0.30,
        };
    }
  },
};
```

#### People Search Service (`/src/services/peopleSearchService.ts`)
```typescript
export const peopleSearchService = {
  /**
   * Search clips by people filter
   */
  async searchClipsByPeople(filter: PeopleFilter): Promise<Clip[]> {
    const { data, error } = await supabase.rpc('search_clips_by_people', {
      people_ids: filter.selectedPeople,
      min_people: filter.minPeopleCount,
      max_people: filter.maxPeopleCount,
      match_mode: filter.matchMode,
    });

    return data || [];
  },

  /**
   * Get person appearances timeline
   */
  async getPersonTimeline(personId: string): Promise<PersonAppearance[]> {
    const { data, error } = await supabase
      .from('person_appearances')
      .select(`
        *,
        clip:clips(*)
      `)
      .eq('cluster_id', personId)
      .order('start_time');

    return data || [];
  },

  /**
   * Get all labeled people
   */
  async getLabeledPeople(): Promise<PersonCluster[]> {
    const { data, error } = await supabase
      .from('person_clusters')
      .select('*')
      .eq('status', 'labeled')
      .order('appearance_count', { ascending: false });

    return data || [];
  },
};
```

### Component Updates

#### Update: `Search.tsx`
```typescript
// Add people filter panel
<div className="search-page">
  <div className="search-sidebar">
    <SearchFilters />
    <PeopleFilterPanel onFilterChange={handlePeopleFilter} />
  </div>

  <div className="search-results">
    <SearchResults filters={{ ...textFilters, ...peopleFilter }} />
  </div>
</div>
```

#### Update: `VideoEditor.tsx`
```typescript
// Add reframe panel to right sidebar
<div className="editor-right-panel">
  <Tabs>
    <TabsList>
      <TabsTrigger value="inspector">Inspector</TabsTrigger>
      <TabsTrigger value="reframe">Reframe</TabsTrigger>
    </TabsList>
    <TabsContent value="inspector">
      <Inspector />
    </TabsContent>
    <TabsContent value="reframe">
      <ReframePanel segmentId={selectedSegmentId} onReframeApply={handleReframeApply} />
    </TabsContent>
  </Tabs>
</div>
```

#### Update: `render-video` Edge Function
```typescript
// Add reframe/crop processing to render pipeline
async function renderVideo(jobId: string) {
  // ... existing composition

  // Load reframe configs for segments
  const reframeConfigs = await getReframeConfigs(segments);

  // For each segment with reframe:
  for (const segment of segments) {
    if (segment.reframe_config?.mode === 'face_auto') {
      // Generate crop filter with crop path
      const cropFilter = buildCropFilter(segment.reframe_config.crop_keyframes);
      // Apply to video segment
    }
  }

  // Final output with reframed video
}
```

### Phase 4 Acceptance Criteria

- [ ] User can select reframe preset (Single Speaker, Two Person, Reaction)
- [ ] Face auto-tracking generates smooth crop path
- [ ] Manual keyframe mode allows custom crop positioning
- [ ] Crop preview shows real-time framing
- [ ] Reframe config saves with segment
- [ ] Exported video applies reframing correctly
- [ ] People filter panel displays labeled individuals
- [ ] User can filter clips by selected people
- [ ] "Any", "All", "Exact" match modes work correctly
- [ ] People count range filter (min/max) works
- [ ] Person timeline shows all appearances
- [ ] Clicking appearance jumps to clip at timestamp
- [ ] Search results respect people filters

### Phase 4 Testing Checklist

- [ ] Apply "Single Speaker" preset to interview segment
- [ ] Verify crop centers on speaker's face
- [ ] Apply "Two Person" preset to dialogue segment
- [ ] Verify crop includes both speakers
- [ ] Create manual crop keyframes for custom framing
- [ ] Preview reframe in program monitor
- [ ] Export video with reframe applied
- [ ] Verify exported video matches preview
- [ ] Filter clips by specific person
- [ ] Filter clips with exactly 2 people
- [ ] Filter clips with 3+ people
- [ ] View person timeline for frequent speaker
- [ ] Jump to specific appearance from timeline
- [ ] Combine text search with people filter

---

## Phase 5: AI-Powered Assistance
**Duration:** 2-3 weeks
**Priority:** P2 (Polish and Differentiators)
**Depends On:** Phases 1-4

### Goals
1. Add auto-select assistant for intelligent clip suggestions
2. Enable AI-generated interstitials (image/video)
3. Add project versioning and review workflow

### Database Schema Changes

#### New Table: `auto_select_runs`
```sql
CREATE TABLE auto_select_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    preset TEXT NOT NULL CHECK (preset IN ('highlights', 'emotional', 'vertical_ready', 'custom')),
    input_filters JSONB NOT NULL, -- {topics, people, min_confidence, emotion_types}
    suggested_segments JSONB NOT NULL, -- [{segment_id, score, reason}]
    status TEXT DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_auto_select_runs_project ON auto_select_runs(project_id);
```

#### New Table: `generated_interstitials`
```sql
CREATE TABLE generated_interstitials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    prompt TEXT NOT NULL,
    asset_url TEXT NOT NULL, -- storage URL for generated image/video
    asset_type TEXT NOT NULL CHECK (asset_type IN ('image', 'video')),
    duration NUMERIC, -- for videos
    voiceover_url TEXT, -- optional TTS voiceover
    voiceover_text TEXT,
    generation_model TEXT, -- e.g., "dall-e-3", "runway-gen3"
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_interstitials_project ON generated_interstitials(project_id);
```

#### New Table: `project_versions`
```sql
CREATE TABLE project_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    version_number INTEGER NOT NULL,
    label TEXT NOT NULL,
    timeline_snapshot JSONB NOT NULL, -- full timeline state
    created_by UUID REFERENCES auth.users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(project_id, version_number)
);

CREATE INDEX idx_project_versions_project ON project_versions(project_id);
```

#### New Table: `timeline_comments`
```sql
CREATE TABLE timeline_comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    timestamp NUMERIC NOT NULL, -- timeline position in seconds
    comment TEXT NOT NULL,
    author UUID NOT NULL REFERENCES auth.users(id),
    status TEXT DEFAULT 'open' CHECK (status IN ('open', 'resolved')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_timeline_comments_project ON timeline_comments(project_id);
CREATE INDEX idx_timeline_comments_timestamp ON timeline_comments(timestamp);
```

### Store Changes

```typescript
interface EditorState {
  // NEW: Auto-select state
  autoSelectRunId: string | null;
  autoSelectSuggestions: AutoSelectSuggestion[];
  autoSelectFilters: AutoSelectFilters;

  // NEW: Interstitials state
  generatedInterstitials: GeneratedInterstitial[];
  interstitialGeneratorOpen: boolean;

  // NEW: Versioning state
  currentVersion: number;
  versions: ProjectVersion[];
  commentsVisible: boolean;
  timelineComments: TimelineComment[];
}

interface AutoSelectSuggestion {
  segmentId: string;
  score: number; // 0-1
  reason: string;
  confidence: number;
  selected: boolean;
}

interface AutoSelectFilters {
  preset: 'highlights' | 'emotional' | 'vertical_ready' | 'custom';
  topics?: string[];
  people?: string[];
  minConfidence?: number;
  emotionTypes?: string[];
  minDuration?: number;
  maxDuration?: number;
}

interface GeneratedInterstitial {
  id: string;
  prompt: string;
  assetUrl: string;
  assetType: 'image' | 'video';
  duration?: number;
  voiceoverUrl?: string;
  voiceoverText?: string;
}

interface ProjectVersion {
  id: string;
  versionNumber: number;
  label: string;
  timelineSnapshot: TimelineState;
  createdBy: string;
  createdAt: string;
}

interface TimelineComment {
  id: string;
  timestamp: number;
  comment: string;
  author: string;
  status: 'open' | 'resolved';
  createdAt: string;
}

// NEW: Actions
const actions = {
  // Auto-select
  runAutoSelect: async (filters: AutoSelectFilters) => { /* ... */ },
  acceptSuggestion: (segmentId: string) => { /* ... */ },
  rejectSuggestion: (segmentId: string) => { /* ... */ },
  insertAcceptedSuggestions: () => { /* ... */ },

  // Interstitials
  generateInterstitial: async (prompt: string, options: InterstitialOptions) => { /* ... */ },
  insertInterstitial: (interstitialId: string, position: number) => { /* ... */ },

  // Versioning
  createVersion: async (label: string) => { /* ... */ },
  loadVersion: async (versionId: string) => { /* ... */ },

  // Comments
  addComment: async (timestamp: number, comment: string) => { /* ... */ },
  resolveComment: async (commentId: string) => { /* ... */ },
};
```

### New Components

#### 1. Auto-Selects Panel (`/src/components/editor/panels/AutoSelectsPanel.tsx`)
```typescript
interface AutoSelectsPanelProps {
  projectId: string;
}

export function AutoSelectsPanel({ projectId }: AutoSelectsPanelProps) {
  const [preset, setPreset] = useState<AutoSelectPreset>('highlights');
  const [filters, setFilters] = useState<AutoSelectFilters>({});
  const { suggestions, runAutoSelect, acceptSuggestion, rejectSuggestion } = useAutoSelect();

  return (
    <Card>
      <CardHeader>
        <CardTitle>Auto-Select Assistant</CardTitle>
        <CardDescription>AI-powered clip selection</CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        {/* Preset selector */}
        <div>
          <Label>Preset</Label>
          <Select value={preset} onValueChange={setPreset}>
            <SelectItem value="highlights">Highlights</SelectItem>
            <SelectItem value="emotional">Emotional Moments</SelectItem>
            <SelectItem value="vertical_ready">Vertical-Ready Clips</SelectItem>
            <SelectItem value="custom">Custom Filters</SelectItem>
          </Select>
        </div>

        {preset === 'custom' && (
          <>
            <div>
              <Label>Topics</Label>
              <MultiSelect
                options={availableTopics}
                value={filters.topics}
                onChange={(topics) => setFilters({ ...filters, topics })}
              />
            </div>

            <div>
              <Label>People</Label>
              <MultiSelect
                options={availablePeople}
                value={filters.people}
                onChange={(people) => setFilters({ ...filters, people })}
              />
            </div>

            <div>
              <Label>Min Confidence</Label>
              <Slider
                value={[filters.minConfidence || 0.5]}
                onValueChange={([v]) => setFilters({ ...filters, minConfidence: v })}
                min={0}
                max={1}
                step={0.1}
              />
            </div>
          </>
        )}

        <Button onClick={() => runAutoSelect({ preset, ...filters })} className="w-full">
          <Sparkles className="h-4 w-4 mr-2" />
          Generate Suggestions
        </Button>

        {suggestions.length > 0 && (
          <div className="space-y-2">
            <Label>Suggestions ({suggestions.filter(s => s.selected).length}/{suggestions.length})</Label>
            {suggestions.map(suggestion => (
              <SuggestionCard
                key={suggestion.segmentId}
                suggestion={suggestion}
                onAccept={() => acceptSuggestion(suggestion.segmentId)}
                onReject={() => rejectSuggestion(suggestion.segmentId)}
              />
            ))}

            <Button
              onClick={insertAcceptedSuggestions}
              disabled={suggestions.filter(s => s.selected).length === 0}
              className="w-full"
            >
              Insert {suggestions.filter(s => s.selected).length} Selected
            </Button>
          </div>
        )}
      </CardContent>
    </Card>
  );
}
```

#### 2. Suggestion Card (`/src/components/editor/panels/SuggestionCard.tsx`)
```typescript
interface SuggestionCardProps {
  suggestion: AutoSelectSuggestion;
  onAccept: () => void;
  onReject: () => void;
}

export function SuggestionCard({ suggestion, onAccept, onReject }: SuggestionCardProps) {
  return (
    <Card className={cn(
      "p-3",
      suggestion.selected && "border-primary"
    )}>
      <div className="flex items-center gap-3">
        <img
          src={suggestion.thumbnailUrl}
          className="w-20 h-12 object-cover rounded"
        />
        <div className="flex-1">
          <p className="text-sm font-medium line-clamp-1">{suggestion.title}</p>
          <p className="text-xs text-muted-foreground">{suggestion.reason}</p>
          <div className="flex items-center gap-2 mt-1">
            <Badge variant="outline">Score: {(suggestion.score * 100).toFixed(0)}%</Badge>
            <Badge variant="outline">{formatDuration(suggestion.duration)}</Badge>
          </div>
        </div>
        <div className="flex gap-1">
          {!suggestion.selected ? (
            <>
              <Button size="sm" variant="ghost" onClick={onAccept}>
                <Check className="h-4 w-4" />
              </Button>
              <Button size="sm" variant="ghost" onClick={onReject}>
                <X className="h-4 w-4" />
              </Button>
            </>
          ) : (
            <Button size="sm" variant="ghost" onClick={onReject}>
              <Check className="h-4 w-4 text-primary" />
            </Button>
          )}
        </div>
      </div>
    </Card>
  );
}
```

#### 3. Interstitial Generator Panel (`/src/components/editor/interstitials/InterstitialGeneratorPanel.tsx`)
```typescript
interface InterstitialGeneratorPanelProps {
  projectId: string;
}

export function InterstitialGeneratorPanel({ projectId }: InterstitialGeneratorPanelProps) {
  const [prompt, setPrompt] = useState('');
  const [assetType, setAssetType] = useState<'image' | 'video'>('image');
  const [includeVoiceover, setIncludeVoiceover] = useState(false);
  const [voiceoverText, setVoiceoverText] = useState('');
  const { generate, isGenerating } = useInterstitialGenerator();

  return (
    <Card>
      <CardHeader>
        <CardTitle>Generate Interstitial</CardTitle>
        <CardDescription>AI-generated b-roll, title cards, and transitions</CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <Label>Type</Label>
          <ToggleGroup value={assetType} onValueChange={setAssetType}>
            <ToggleGroupItem value="image">Image</ToggleGroupItem>
            <ToggleGroupItem value="video">Video (5s)</ToggleGroupItem>
          </ToggleGroup>
        </div>

        <div>
          <Label>Prompt</Label>
          <Textarea
            value={prompt}
            onChange={(e) => setPrompt(e.target.value)}
            placeholder="Describe the visual you want to generate..."
            rows={4}
          />
        </div>

        <div className="flex items-center justify-between">
          <Label>Include Voiceover</Label>
          <Switch checked={includeVoiceover} onCheckedChange={setIncludeVoiceover} />
        </div>

        {includeVoiceover && (
          <div>
            <Label>Voiceover Script</Label>
            <Textarea
              value={voiceoverText}
              onChange={(e) => setVoiceoverText(e.target.value)}
              placeholder="Text to be spoken..."
              rows={3}
            />
          </div>
        )}

        <Button
          onClick={() => generate({ prompt, assetType, includeVoiceover, voiceoverText })}
          disabled={isGenerating || !prompt}
          className="w-full"
        >
          {isGenerating ? (
            <>
              <Loader2 className="h-4 w-4 mr-2 animate-spin" />
              Generating...
            </>
          ) : (
            <>
              <Sparkles className="h-4 w-4 mr-2" />
              Generate
            </>
          )}
        </Button>

        {/* Generated interstitials gallery */}
        <div>
          <Label>Generated Assets</Label>
          <div className="grid grid-cols-2 gap-2 mt-2">
            {generatedInterstitials.map(interstitial => (
              <InterstitialThumbnail
                key={interstitial.id}
                interstitial={interstitial}
                onInsert={(position) => insertInterstitial(interstitial.id, position)}
              />
            ))}
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 4. Version History Panel (`/src/components/editor/versioning/VersionHistoryPanel.tsx`)
```typescript
interface VersionHistoryPanelProps {
  projectId: string;
}

export function VersionHistoryPanel({ projectId }: VersionHistoryPanelProps) {
  const { versions, currentVersion, createVersion, loadVersion } = useVersioning(projectId);
  const [newVersionLabel, setNewVersionLabel] = useState('');

  return (
    <Card>
      <CardHeader>
        <CardTitle>Version History</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="flex gap-2">
          <Input
            placeholder="Version label..."
            value={newVersionLabel}
            onChange={(e) => setNewVersionLabel(e.target.value)}
          />
          <Button onClick={() => createVersion(newVersionLabel)}>
            <Save className="h-4 w-4 mr-2" />
            Save
          </Button>
        </div>

        <div className="space-y-2">
          {versions.map(version => (
            <Card
              key={version.id}
              className={cn(
                "p-3 cursor-pointer",
                version.versionNumber === currentVersion && "border-primary"
              )}
              onClick={() => loadVersion(version.id)}
            >
              <div className="flex items-center justify-between">
                <div>
                  <p className="font-medium">v{version.versionNumber}: {version.label}</p>
                  <p className="text-xs text-muted-foreground">
                    {formatDate(version.createdAt)}
                  </p>
                </div>
                {version.versionNumber === currentVersion && (
                  <Badge>Current</Badge>
                )}
              </div>
            </Card>
          ))}
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 5. Timeline Comments Panel (`/src/components/editor/comments/TimelineCommentsPanel.tsx`)
```typescript
interface TimelineCommentsPanelProps {
  projectId: string;
  currentTime: number;
}

export function TimelineCommentsPanel({ projectId, currentTime }: TimelineCommentsPanelProps) {
  const { comments, addComment, resolveComment } = useTimelineComments(projectId);
  const [newComment, setNewComment] = useState('');

  return (
    <Card>
      <CardHeader>
        <CardTitle>Timeline Comments</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="flex gap-2">
          <Input
            placeholder="Add comment at current time..."
            value={newComment}
            onChange={(e) => setNewComment(e.target.value)}
          />
          <Button onClick={() => addComment(currentTime, newComment)}>
            <MessageSquare className="h-4 w-4" />
          </Button>
        </div>

        <div className="space-y-2 max-h-96 overflow-y-auto">
          {comments.map(comment => (
            <Card key={comment.id} className="p-3">
              <div className="flex items-start justify-between gap-2">
                <div className="flex-1">
                  <div className="flex items-center gap-2 mb-1">
                    <Badge variant="outline" className="text-xs">
                      {formatTime(comment.timestamp)}
                    </Badge>
                    {comment.status === 'resolved' && (
                      <Badge variant="secondary" className="text-xs">Resolved</Badge>
                    )}
                  </div>
                  <p className="text-sm">{comment.comment}</p>
                  <p className="text-xs text-muted-foreground mt-1">
                    {comment.author} · {formatDate(comment.createdAt)}
                  </p>
                </div>
                {comment.status === 'open' && (
                  <Button size="sm" variant="ghost" onClick={() => resolveComment(comment.id)}>
                    <Check className="h-4 w-4" />
                  </Button>
                )}
              </div>
            </Card>
          ))}
        </div>
      </CardContent>
    </Card>
  );
}
```

### New Services

#### Auto-Select Service (`/src/services/autoSelectService.ts`)
```typescript
export const autoSelectService = {
  /**
   * Run auto-select algorithm
   */
  async runAutoSelect(projectId: string, filters: AutoSelectFilters): Promise<AutoSelectRun> {
    // Call edge function: auto-select-clips
    const { data, error } = await supabase.functions.invoke('auto-select-clips', {
      body: {
        project_id: projectId,
        preset: filters.preset,
        filters: filters,
      }
    });

    return data;
  },

  /**
   * Get auto-select run results
   */
  async getRunResults(runId: string): Promise<AutoSelectSuggestion[]> {
    const { data, error } = await supabase
      .from('auto_select_runs')
      .select('suggested_segments')
      .eq('id', runId)
      .single();

    return data.suggested_segments || [];
  },
};
```

#### Interstitial Service (`/src/services/interstitialService.ts`)
```typescript
export const interstitialService = {
  /**
   * Generate interstitial asset
   */
  async generate(options: InterstitialOptions): Promise<GeneratedInterstitial> {
    // Call edge function: generate-interstitial
    const { data, error } = await supabase.functions.invoke('generate-interstitial', {
      body: {
        prompt: options.prompt,
        asset_type: options.assetType,
        include_voiceover: options.includeVoiceover,
        voiceover_text: options.voiceoverText,
      }
    });

    return data;
  },

  /**
   * Get all generated interstitials for project
   */
  async getAll(projectId: string): Promise<GeneratedInterstitial[]> {
    const { data, error } = await supabase
      .from('generated_interstitials')
      .select('*')
      .eq('project_id', projectId)
      .order('created_at', { ascending: false });

    return data || [];
  },
};
```

### New Edge Functions

#### `auto-select-clips` Edge Function
**File:** `/supabase/functions/auto-select-clips/index.ts`

**Responsibilities:**
1. Receive project ID and filters
2. Query atoms with embeddings
3. Apply filters (topics, people, confidence, emotion)
4. Score each atom/segment based on criteria:
   - Semantic relevance to topics
   - Person prominence
   - Emotion intensity
   - Clip quality/confidence
   - Vertical-readiness (for vertical preset)
5. Rank and return top N suggestions

#### `generate-interstitial` Edge Function
**File:** `/supabase/functions/generate-interstitial/index.ts`

**Responsibilities:**
1. Receive prompt and options
2. Call image generation API (DALL-E, Midjourney, or Stability AI)
3. If video requested, call video generation API (Runway, Pika)
4. If voiceover requested, call TTS API (ElevenLabs, OpenAI TTS)
5. Upload assets to storage
6. Create `generated_interstitials` record
7. Return asset URLs

### Phase 5 Acceptance Criteria

- [ ] User can select auto-select preset (Highlights, Emotional, Vertical-Ready)
- [ ] Custom filters support topics, people, min confidence
- [ ] Auto-select generates ranked suggestions with scores
- [ ] User can accept/reject individual suggestions
- [ ] Accepted suggestions can be batch-inserted to timeline
- [ ] User can generate image interstitials with AI
- [ ] User can generate 5-second video interstitials with AI
- [ ] Optional TTS voiceover can be added to interstitials
- [ ] Generated interstitials display in gallery
- [ ] User can insert interstitial at specific timeline position
- [ ] User can save current timeline as versioned snapshot
- [ ] User can load previous version (restores timeline state)
- [ ] Version history shows all saved versions with labels
- [ ] User can add comment at specific timeline timestamp
- [ ] Comments display in chronological order
- [ ] User can resolve comments
- [ ] Timeline shows comment markers at timestamps

### Phase 5 Testing Checklist

- [ ] Run "Highlights" auto-select on 30-minute project
- [ ] Verify suggestions ranked by relevance
- [ ] Accept 5 suggestions and insert to timeline
- [ ] Run "Vertical-Ready" preset on landscape footage
- [ ] Verify suggestions have face prominence
- [ ] Generate image interstitial with prompt "Mountain sunset"
- [ ] Generate video interstitial with prompt "Typing on keyboard"
- [ ] Add TTS voiceover to interstitial
- [ ] Insert generated interstitial between two segments
- [ ] Save timeline version with label "V1 - First Cut"
- [ ] Make edits to timeline
- [ ] Save new version "V2 - With B-Roll"
- [ ] Load V1 and verify timeline restored
- [ ] Add comment at 01:23 "Fix audio levels here"
- [ ] Resolve comment after fixing
- [ ] Export final project with all features

---

## Implementation Strategy

### Sequential Approach (12-17 weeks)
Execute phases one after another:
1. Phase 1 (3-4 weeks)
2. Phase 2 (2-3 weeks)
3. Phase 3 (2-3 weeks)
4. Phase 4 (3-4 weeks)
5. Phase 5 (2-3 weeks)

**Pros:** Lower complexity, easier to manage, thorough testing
**Cons:** Longer time to market

### Parallel Approach (8-12 weeks)
Run multiple tracks simultaneously:
- **Track A:** Export + Captions (Phase 1) → Advanced Captions (Phase 3a)
- **Track B:** Audio System (Phase 2) → Transitions (Phase 3b)
- **Track C:** Face-Aware Features (Phase 4, starts Week 5)
- **Track D:** AI Assistance (Phase 5, starts Week 9)

**Pros:** Faster delivery
**Cons:** Requires more developers, coordination overhead

### MVP-First Approach (6-8 weeks)
Focus on core P0 features, delay P1/P2:
1. **Weeks 1-4:** Phase 1 + Phase 2 (Export, Captions, Audio)
2. **Weeks 5-6:** Basic reframing (Phase 4 subset)
3. **Weeks 7-8:** Polish and testing
4. **Post-MVP:** Phase 3, Phase 4 (full), Phase 5

**Pros:** Fastest path to usable editor
**Cons:** Missing advanced features initially

### Recommended: Hybrid Approach (10-14 weeks)
1. **Weeks 1-4:** Phase 1 + Phase 2 sequentially (foundation)
2. **Weeks 5-9:** Phase 3 + Phase 4 in parallel
3. **Weeks 10-12:** Phase 5 (AI features)
4. **Weeks 13-14:** Integration testing, polish, documentation

---

## Testing Strategy

### Unit Testing
- Test database migrations with sample data
- Test Zustand store actions and state updates
- Test service functions with mocked Supabase
- Test component rendering with Vitest + Testing Library

### Integration Testing
- Test end-to-end export pipeline (timeline → render → download)
- Test caption generation from transcript
- Test voiceover recording → upload → insert flow
- Test face detection → reframe generation
- Test auto-select algorithm with real clips

### User Acceptance Testing
- Export 5-minute video with captions and audio
- Record and mix voiceover with music track
- Apply face-aware reframing to vertical clip
- Use auto-select to build 60-second highlights reel
- Generate and insert AI interstitial

### Performance Testing
- Render job with 20+ segments (< 5 minutes)
- Timeline with 50+ segments (smooth playback)
- Search with people filters (< 2 seconds)
- Auto-select on 2-hour source material (< 30 seconds)

---

## Rollback Plan

Each phase includes migration files. If issues arise:

1. **Database Rollback:**
   ```sql
   -- Example rollback for Phase 1
   DROP TABLE IF EXISTS caption_segments;
   DROP TABLE IF EXISTS caption_tracks;
   DROP TABLE IF EXISTS render_jobs;
   DROP TABLE IF EXISTS project_timeline_state;
   ```

2. **Feature Flags:**
   Use Supabase feature flags to disable incomplete features:
   ```typescript
   const FEATURE_FLAGS = {
     EXPORT_ENABLED: false,
     CAPTIONS_ENABLED: false,
     AUDIO_ENABLED: false,
     REFRAME_ENABLED: false,
     AUTO_SELECT_ENABLED: false,
   };
   ```

3. **Version Control:**
   Tag each phase completion:
   ```bash
   git tag phase-1-export-captions
   git tag phase-2-audio-system
   # etc.
   ```

---

## Success Metrics

### Phase 1
- [ ] 90%+ of test exports complete successfully
- [ ] Caption rendering matches preview
- [ ] Export time < 2x video duration

### Phase 2
- [ ] Voiceover recording success rate > 95%
- [ ] Audio mixing accurate within ±0.5dB
- [ ] Multi-track audio exports correctly

### Phase 3
- [ ] Caption styling matches professional standards
- [ ] Transitions render smoothly (no artifacts)
- [ ] Keyframe interpolation is smooth

### Phase 4
- [ ] Face auto-tracking accuracy > 85%
- [ ] Reframe presets work on 90%+ of clips
- [ ] People search recall > 90%

### Phase 5
- [ ] Auto-select relevance score > 75% (user feedback)
- [ ] Interstitial generation success rate > 90%
- [ ] Version restore works 100% of time

---

## Next Steps

1. **Review this plan** with development team
2. **Estimate effort** for each phase (story points or hours)
3. **Prioritize** based on business needs and dependencies
4. **Set up project tracking** (GitHub Projects, Linear, or Jira)
5. **Begin Phase 1 implementation** using detailed specifications above

---

## Appendix: File Structure

```
creative-edit-suite/
├── supabase/
│   ├── migrations/
│   │   ├── 20260206_phase1_export_captions.sql
│   │   ├── 20260206_phase2_audio_system.sql
│   │   ├── 20260206_phase3_advanced_features.sql
│   │   ├── 20260206_phase4_face_aware.sql
│   │   └── 20260206_phase5_ai_assistance.sql
│   └── functions/
│       ├── render-video/
│       ├── auto-select-clips/
│       └── generate-interstitial/
├── src/
│   ├── stores/
│   │   ├── editorStore.ts
│   │   ├── peopleStore.ts
│   │   └── autoSelectStore.ts
│   ├── services/
│   │   ├── renderService.ts
│   │   ├── captionService.ts
│   │   ├── audioService.ts
│   │   ├── reframeService.ts
│   │   ├── peopleSearchService.ts
│   │   ├── autoSelectService.ts
│   │   └── interstitialService.ts
│   ├── components/
│   │   └── editor/
│   │       ├── export/
│   │       │   └── ExportDialog.tsx
│   │       ├── captions/
│   │       │   ├── CaptionControls.tsx
│   │       │   ├── CaptionOverlay.tsx
│   │       │   └── CaptionStyleInspector.tsx
│   │       ├── audio/
│   │       │   ├── AudioTrackLane.tsx
│   │       │   ├── VoiceoverRecorder.tsx
│   │       │   └── AudioInspector.tsx
│   │       ├── transitions/
│   │       │   └── TransitionEditor.tsx
│   │       ├── keyframes/
│   │       │   └── TransformInspector.tsx
│   │       ├── reframe/
│   │       │   ├── ReframePanel.tsx
│   │       │   └── CropPathPreview.tsx
│   │       ├── panels/
│   │       │   └── AutoSelectsPanel.tsx
│   │       ├── interstitials/
│   │       │   └── InterstitialGeneratorPanel.tsx
│   │       ├── versioning/
│   │       │   └── VersionHistoryPanel.tsx
│   │       └── comments/
│   │           └── TimelineCommentsPanel.tsx
│   └── hooks/
│       ├── useRender.ts
│       ├── useCaptions.ts
│       ├── useAudio.ts
│       ├── useReframe.ts
│       ├── usePeople.ts
│       ├── useAutoSelect.ts
│       └── useVersioning.ts
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-06
**Target Codebase:** `creative-edit-suite` (Lovable)
**Prepared By:** Claude (Anthropic)
