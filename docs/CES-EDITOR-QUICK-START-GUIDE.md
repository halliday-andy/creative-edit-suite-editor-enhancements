# Creative Edit Suite Editor - Quick Start Guide

Get started with Phase 1 (Export Pipeline & Captions) in 1-2 hours.

---

## Prerequisites

- Supabase project with creative-edit-suite database
- Access to Supabase SQL Editor
- Local development environment running
- Basic familiarity with React, TypeScript, and Zustand

---

## Step 1: Database Setup (15 minutes)

### 1.1 Run Migration SQL

Open Supabase SQL Editor and execute:

```sql
-- ============================================
-- PHASE 1: Export Pipeline & Captions v1
-- ============================================

-- Table: render_jobs
CREATE TABLE render_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    status TEXT NOT NULL CHECK (status IN ('queued', 'running', 'completed', 'failed', 'cancelled')),
    render_plan JSONB NOT NULL,
    progress INTEGER DEFAULT 0 CHECK (progress >= 0 AND progress <= 100),
    output_urls JSONB DEFAULT '[]'::jsonb,
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);

CREATE INDEX idx_render_jobs_project ON render_jobs(project_id);
CREATE INDEX idx_render_jobs_status ON render_jobs(status) WHERE status IN ('queued', 'running');

-- Table: project_timeline_state
CREATE TABLE project_timeline_state (
    project_id UUID PRIMARY KEY REFERENCES projects(id) ON DELETE CASCADE,
    timeline JSONB NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Table: caption_tracks
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

-- Table: caption_segments
CREATE TABLE caption_segments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    track_id UUID NOT NULL REFERENCES caption_tracks(id) ON DELETE CASCADE,
    segment_id UUID REFERENCES segments(id) ON DELETE SET NULL,
    start_time NUMERIC NOT NULL,
    end_time NUMERIC NOT NULL,
    text TEXT NOT NULL,
    style JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT valid_time_range CHECK (end_time > start_time)
);

CREATE INDEX idx_caption_segments_track ON caption_segments(track_id);
CREATE INDEX idx_caption_segments_time ON caption_segments(start_time, end_time);

-- Grant permissions (adjust role as needed)
GRANT ALL ON render_jobs TO authenticated;
GRANT ALL ON project_timeline_state TO authenticated;
GRANT ALL ON caption_tracks TO authenticated;
GRANT ALL ON caption_segments TO authenticated;
```

### 1.2 Verify Tables

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
AND table_name IN ('render_jobs', 'project_timeline_state', 'caption_tracks', 'caption_segments');
```

You should see all 4 tables listed.

---

## Step 2: TypeScript Types (10 minutes)

### 2.1 Create Types File

Create `/src/types/editor.ts`:

```typescript
// Render Types
export interface RenderJob {
  id: string;
  project_id: string;
  status: 'queued' | 'running' | 'completed' | 'failed' | 'cancelled';
  render_plan: RenderPlan;
  progress: number;
  output_urls: OutputUrl[];
  error_message: string | null;
  created_at: string;
  started_at: string | null;
  completed_at: string | null;
}

export interface RenderPlan {
  tracks: Track[];
  segments: TimelineSegment[];
  transitions: Transition[];
  captionTracks: CaptionTrack[];
  settings: ExportSettings;
}

export interface OutputUrl {
  type: 'video' | 'audio' | 'subtitle';
  url: string;
  format: string;
  size_bytes?: number;
}

export interface ExportSettings {
  format: 'mp4' | 'webm' | 'mov';
  resolution: '1080p' | '720p' | '480p';
  quality: 'high' | 'medium' | 'low';
  includeCaptions: boolean;
  includeAudio: boolean;
  fps: 24 | 30 | 60;
}

// Timeline Types
export interface Track {
  id: string;
  type: 'video' | 'audio' | 'caption';
  name: string;
  segments: string[]; // segment IDs
}

export interface TimelineSegment {
  id: string;
  trackId: string;
  clipId: string;
  startTime: number; // seconds from project start
  duration: number; // seconds
  inPoint: number; // trim start within clip
  outPoint: number; // trim end within clip
  transform?: Transform;
}

export interface Transform {
  x: number;
  y: number;
  scale: number;
  rotation: number;
  opacity: number;
}

export interface Transition {
  id: string;
  fromSegmentId: string;
  toSegmentId: string;
  type: 'cut' | 'dissolve' | 'dip_black' | 'dip_white';
  durationMs: number;
}

// Caption Types
export interface CaptionTrack {
  id: string;
  project_id: string;
  name: string;
  style_preset: CaptionStylePreset;
  is_active: boolean;
  created_at: string;
}

export type CaptionStylePreset = 'clean' | 'bold' | 'minimal' | 'custom';

export interface CaptionSegment {
  id: string;
  track_id: string;
  segment_id: string | null;
  start_time: number;
  end_time: number;
  text: string;
  style: CaptionStyle;
  created_at: string;
}

export interface CaptionStyle {
  position: 'top' | 'center' | 'bottom';
  size: 'small' | 'medium' | 'large';
  boxed: boolean;
  fontFamily?: string;
  fontSize?: number;
  color?: string;
  backgroundColor?: string;
}

// Timeline State
export interface TimelineState {
  tracks: Track[];
  segments: TimelineSegment[];
  transitions: Transition[];
  captionTracks: CaptionTrack[];
}

export interface ProjectTimelineState {
  project_id: string;
  timeline: TimelineState;
  updated_at: string;
}
```

---

## Step 3: Caption Service (15 minutes)

### 3.1 Create Caption Service

Create `/src/services/captionService.ts`:

```typescript
import { supabase } from '@/lib/supabase';
import { CaptionTrack, CaptionSegment, CaptionStylePreset } from '@/types/editor';

export const captionService = {
  /**
   * Get all caption tracks for a project
   */
  async getTracks(projectId: string): Promise<CaptionTrack[]> {
    const { data, error } = await supabase
      .from('caption_tracks')
      .select('*')
      .eq('project_id', projectId)
      .order('created_at');

    if (error) {
      console.error('Error fetching caption tracks:', error);
      throw error;
    }

    return data || [];
  },

  /**
   * Create a new caption track
   */
  async createTrack(
    projectId: string,
    name: string = 'Captions',
    stylePreset: CaptionStylePreset = 'clean'
  ): Promise<CaptionTrack> {
    const { data, error } = await supabase
      .from('caption_tracks')
      .insert({
        project_id: projectId,
        name,
        style_preset: stylePreset,
      })
      .select()
      .single();

    if (error) {
      console.error('Error creating caption track:', error);
      throw error;
    }

    return data;
  },

  /**
   * Update caption track
   */
  async updateTrack(
    trackId: string,
    updates: Partial<Pick<CaptionTrack, 'name' | 'style_preset' | 'is_active'>>
  ): Promise<CaptionTrack> {
    const { data, error } = await supabase
      .from('caption_tracks')
      .update(updates)
      .eq('id', trackId)
      .select()
      .single();

    if (error) {
      console.error('Error updating caption track:', error);
      throw error;
    }

    return data;
  },

  /**
   * Delete caption track
   */
  async deleteTrack(trackId: string): Promise<void> {
    const { error } = await supabase
      .from('caption_tracks')
      .delete()
      .eq('id', trackId);

    if (error) {
      console.error('Error deleting caption track:', error);
      throw error;
    }
  },

  /**
   * Get segments for a caption track
   */
  async getSegments(trackId: string): Promise<CaptionSegment[]> {
    const { data, error } = await supabase
      .from('caption_segments')
      .select('*')
      .eq('track_id', trackId)
      .order('start_time');

    if (error) {
      console.error('Error fetching caption segments:', error);
      throw error;
    }

    return data || [];
  },

  /**
   * Create caption segment
   */
  async createSegment(segment: {
    track_id: string;
    start_time: number;
    end_time: number;
    text: string;
    style?: object;
    segment_id?: string;
  }): Promise<CaptionSegment> {
    const { data, error } = await supabase
      .from('caption_segments')
      .insert(segment)
      .select()
      .single();

    if (error) {
      console.error('Error creating caption segment:', error);
      throw error;
    }

    return data;
  },

  /**
   * Update caption segment
   */
  async updateSegment(
    segmentId: string,
    updates: Partial<Pick<CaptionSegment, 'start_time' | 'end_time' | 'text' | 'style'>>
  ): Promise<CaptionSegment> {
    const { data, error } = await supabase
      .from('caption_segments')
      .update(updates)
      .eq('id', segmentId)
      .select()
      .single();

    if (error) {
      console.error('Error updating caption segment:', error);
      throw error;
    }

    return data;
  },

  /**
   * Delete caption segment
   */
  async deleteSegment(segmentId: string): Promise<void> {
    const { error } = await supabase
      .from('caption_segments')
      .delete()
      .eq('id', segmentId);

    if (error) {
      console.error('Error deleting caption segment:', error);
      throw error;
    }
  },

  /**
   * Auto-generate captions from transcript
   */
  async generateFromTranscript(
    trackId: string,
    clipId: string
  ): Promise<CaptionSegment[]> {
    // TODO: Implement transcript → caption conversion
    // This would:
    // 1. Fetch transcript for clip
    // 2. Split into caption-sized chunks (2-3 seconds each)
    // 3. Create caption_segments in batch
    throw new Error('Not implemented yet');
  },
};
```

---

## Step 4: Render Service (15 minutes)

### 4.1 Create Render Service

Create `/src/services/renderService.ts`:

```typescript
import { supabase } from '@/lib/supabase';
import { RenderJob, ExportSettings, TimelineState } from '@/types/editor';

export const renderService = {
  /**
   * Start a new render job
   */
  async startRender(
    projectId: string,
    timeline: TimelineState,
    settings: ExportSettings
  ): Promise<RenderJob> {
    // Create render job record
    const { data, error } = await supabase
      .from('render_jobs')
      .insert({
        project_id: projectId,
        status: 'queued',
        render_plan: {
          ...timeline,
          settings,
        },
        progress: 0,
      })
      .select()
      .single();

    if (error) {
      console.error('Error creating render job:', error);
      throw error;
    }

    // Trigger edge function (async - don't wait)
    supabase.functions
      .invoke('render-video', {
        body: { job_id: data.id },
      })
      .catch((err) => {
        console.error('Error invoking render function:', err);
      });

    return data;
  },

  /**
   * Get render job status
   */
  async getRenderStatus(jobId: string): Promise<RenderJob> {
    const { data, error } = await supabase
      .from('render_jobs')
      .select('*')
      .eq('id', jobId)
      .single();

    if (error) {
      console.error('Error fetching render status:', error);
      throw error;
    }

    return data;
  },

  /**
   * Cancel a running render job
   */
  async cancelRender(jobId: string): Promise<void> {
    const { error } = await supabase
      .from('render_jobs')
      .update({ status: 'cancelled' })
      .eq('id', jobId);

    if (error) {
      console.error('Error cancelling render:', error);
      throw error;
    }
  },

  /**
   * Poll render status until completion
   */
  async pollRenderStatus(
    jobId: string,
    onProgress?: (progress: number) => void,
    intervalMs: number = 2000
  ): Promise<RenderJob> {
    return new Promise((resolve, reject) => {
      const interval = setInterval(async () => {
        try {
          const job = await this.getRenderStatus(jobId);

          if (onProgress && job.progress !== undefined) {
            onProgress(job.progress);
          }

          if (job.status === 'completed') {
            clearInterval(interval);
            resolve(job);
          } else if (job.status === 'failed' || job.status === 'cancelled') {
            clearInterval(interval);
            reject(new Error(job.error_message || `Render ${job.status}`));
          }
        } catch (error) {
          clearInterval(interval);
          reject(error);
        }
      }, intervalMs);
    });
  },

  /**
   * Save timeline state
   */
  async saveTimeline(projectId: string, timeline: TimelineState): Promise<void> {
    const { error } = await supabase
      .from('project_timeline_state')
      .upsert({
        project_id: projectId,
        timeline,
        updated_at: new Date().toISOString(),
      });

    if (error) {
      console.error('Error saving timeline:', error);
      throw error;
    }
  },

  /**
   * Load timeline state
   */
  async loadTimeline(projectId: string): Promise<TimelineState | null> {
    const { data, error } = await supabase
      .from('project_timeline_state')
      .select('timeline')
      .eq('project_id', projectId)
      .single();

    if (error) {
      if (error.code === 'PGRST116') {
        // No timeline saved yet
        return null;
      }
      console.error('Error loading timeline:', error);
      throw error;
    }

    return data.timeline;
  },
};
```

---

## Step 5: Update Editor Store (20 minutes)

### 5.1 Add State to Editor Store

Update `/src/stores/editorStore.ts`:

```typescript
import { create } from 'zustand';
import { CaptionTrack, CaptionSegment, CaptionStylePreset, ExportSettings } from '@/types/editor';

interface EditorState {
  // Existing state...

  // NEW: Render state
  renderStatus: 'idle' | 'queued' | 'rendering' | 'completed' | 'failed';
  renderProgress: number;
  activeRenderJobId: string | null;

  // NEW: Caption state
  captionMode: 'off' | 'on' | 'preview';
  activeCaptionTrackId: string | null;
  captionTracks: CaptionTrack[];
  captionSegments: Map<string, CaptionSegment[]>; // key: track_id

  // Actions
  setRenderStatus: (status: EditorState['renderStatus']) => void;
  setRenderProgress: (progress: number) => void;
  setActiveRenderJobId: (jobId: string | null) => void;

  setCaptionMode: (mode: EditorState['captionMode']) => void;
  setActiveCaptionTrack: (trackId: string | null) => void;
  setCaptionTracks: (tracks: CaptionTrack[]) => void;
  setCaptionSegments: (trackId: string, segments: CaptionSegment[]) => void;
  addCaptionSegment: (segment: CaptionSegment) => void;
  updateCaptionSegment: (trackId: string, segmentId: string, updates: Partial<CaptionSegment>) => void;
  removeCaptionSegment: (trackId: string, segmentId: string) => void;
}

export const useEditorStore = create<EditorState>((set, get) => ({
  // Existing state...

  // NEW: Render state
  renderStatus: 'idle',
  renderProgress: 0,
  activeRenderJobId: null,

  // NEW: Caption state
  captionMode: 'off',
  activeCaptionTrackId: null,
  captionTracks: [],
  captionSegments: new Map(),

  // Actions
  setRenderStatus: (status) => set({ renderStatus: status }),
  setRenderProgress: (progress) => set({ renderProgress: progress }),
  setActiveRenderJobId: (jobId) => set({ activeRenderJobId: jobId }),

  setCaptionMode: (mode) => set({ captionMode: mode }),
  setActiveCaptionTrack: (trackId) => set({ activeCaptionTrackId: trackId }),
  setCaptionTracks: (tracks) => set({ captionTracks: tracks }),

  setCaptionSegments: (trackId, segments) => {
    const newMap = new Map(get().captionSegments);
    newMap.set(trackId, segments);
    set({ captionSegments: newMap });
  },

  addCaptionSegment: (segment) => {
    const segments = get().captionSegments.get(segment.track_id) || [];
    const newMap = new Map(get().captionSegments);
    newMap.set(segment.track_id, [...segments, segment]);
    set({ captionSegments: newMap });
  },

  updateCaptionSegment: (trackId, segmentId, updates) => {
    const segments = get().captionSegments.get(trackId) || [];
    const newSegments = segments.map((seg) =>
      seg.id === segmentId ? { ...seg, ...updates } : seg
    );
    const newMap = new Map(get().captionSegments);
    newMap.set(trackId, newSegments);
    set({ captionSegments: newMap });
  },

  removeCaptionSegment: (trackId, segmentId) => {
    const segments = get().captionSegments.get(trackId) || [];
    const newSegments = segments.filter((seg) => seg.id !== segmentId);
    const newMap = new Map(get().captionSegments);
    newMap.set(trackId, newSegments);
    set({ captionSegments: newMap });
  },
}));
```

---

## Step 6: Test Database & Services (10 minutes)

### 6.1 Test Caption Service

Create a test file or use browser console:

```typescript
import { captionService } from '@/services/captionService';

// Test: Create caption track
const track = await captionService.createTrack('your-project-id', 'Test Captions', 'clean');
console.log('Created track:', track);

// Test: Create caption segment
const segment = await captionService.createSegment({
  track_id: track.id,
  start_time: 0,
  end_time: 3,
  text: 'Hello, world!',
  style: {
    position: 'bottom',
    size: 'medium',
    boxed: true,
  },
});
console.log('Created segment:', segment);

// Test: Get segments
const segments = await captionService.getSegments(track.id);
console.log('All segments:', segments);
```

---

## Step 7: Next Steps

### Immediate Tasks
1. **Build Export Dialog UI** - Create the `ExportDialog.tsx` component
2. **Build Caption Controls** - Create the `CaptionControls.tsx` component
3. **Integrate with Timeline** - Add caption track to timeline view
4. **Create render-video Edge Function** - Implement FFmpeg export logic

### Testing Checklist
- [ ] Create caption track successfully
- [ ] Add caption segment with text
- [ ] Update caption timing (start/end)
- [ ] Delete caption segment
- [ ] Caption track persists across page refresh
- [ ] Multiple caption tracks work correctly

### Resources
- Full implementation plan: `CES-EDITOR-IMPLEMENTATION-PLAN.md`
- Complete checklist: `CES-EDITOR-IMPLEMENTATION-CHECKLIST.md`
- Supabase Docs: https://supabase.com/docs
- FFmpeg Docs: https://ffmpeg.org/documentation.html

---

## Troubleshooting

### Database Errors

**Error: `relation "render_jobs" does not exist`**
- Solution: Re-run the migration SQL from Step 1.1

**Error: `permission denied for table render_jobs`**
- Solution: Run the GRANT commands from the migration SQL

### TypeScript Errors

**Error: `Cannot find module '@/types/editor'`**
- Solution: Ensure the types file is created at `/src/types/editor.ts`
- Check your tsconfig.json has the `@/` path alias configured

### Service Errors

**Error: `supabase is not defined`**
- Solution: Ensure you have a Supabase client instance exported from `@/lib/supabase`

```typescript
// /src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

---

## Need Help?

If you encounter issues:
1. Check the Supabase logs for database errors
2. Verify all tables exist with `\dt` in SQL Editor
3. Ensure Row Level Security (RLS) policies allow access
4. Review the full implementation plan for context
5. Test services individually before integration

**Happy coding!** 🚀
