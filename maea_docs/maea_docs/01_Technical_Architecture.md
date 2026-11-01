# Technical Architecture — Multimodal Attendance & Engagement Agent (MAEA)

## 1. Overview
MAEA is an offline, privacy-preserving classroom intelligence system. It ingests a live or recorded
video stream (with audio), and produces (1) attendance logs, (2) engagement analytics over time,
(3) assistive teaching suggestions, and (4) student/session analytics.

## 2. High-Level Architecture
```
 ┌────────────┐   ┌────────────┐
 │ Video Src  │   │ Audio Src  │   (webcam / file; ffmpeg audio extract)
 └─────┬──────┘   └─────┬──────┘
       │ frames+ts      │ 16kHz PCM chunks+ts
┌──────▼───────────────┐│┌───────────────────────┐
│ VISION PIPELINE      │││ AUDIO PIPELINE        │
│ detect -> track      │││ VAD -> energy ->      │
│ identity (optional)  │││ speech ratio, turns,  │
│ head pose, motion    │││ silence streaks       │
└──────┬───────────────┘│└──────────┬────────────┘
       │ per-track feats│           │ class-level feats
       ├────────────────┴───────────┤
┌──────▼───────────┐        ┌───────▼────────────┐
│ ATTENDANCE ENGINE│        │ FUSION ENGINE      │
│ sustained-presence│       │ windowing, weights,│
│ state machine    │        │ EMA, trend detect  │
└──────┬───────────┘        └───────┬────────────┘
       │                            │
       └──────────┬─────────────────┘
            ┌─────▼──────┐
            │ SQLite DB  │  (derived features only, pseudonymous)
            └─────┬──────┘
      ┌───────────┼─────────────────┐
┌─────▼──────┐ ┌──▼──────────────┐ ┌▼──────────────┐
│ AGENT      │ │ DASHBOARD       │ │ REPORT EXPORT │
│ rules + opt│ │ Streamlit/Plotly│ │ CSV / PDF     │
│ local LLM  │ └─────────────────┘ └───────────────┘
└────────────┘
```

## 3. Components

### 3.1 I/O Layer
| Module | Responsibility |
|---|---|
| `io/video_source.py` | Read webcam (`cv2.VideoCapture(0)`) or file; attach monotonic timestamps; frame skipping (`process_every_n_frames`); resize to max width. |
| `io/audio_source.py` | Extract audio with ffmpeg to 16 kHz mono WAV; yield 30 ms frames aligned to video timestamps; for webcam use `sounddevice` stream. |

### 3.2 Vision Pipeline
- **Detector**: YOLOv8n/s (person class) for body boxes; MediaPipe/InsightFace face detector for faces.
- **Tracker**: ByteTrack gives stable `track_id`; re-link lost tracks within `relink_window_s` using IoU + appearance embedding.
- **Identity (optional)**: ArcFace embedding (512-d) per face; cosine similarity >= 0.45 matches enrolled student; otherwise anonymous `S01..Sn`.
- **Head Pose**: MediaPipe Face Mesh -> 6 key landmarks -> `cv2.solvePnP` -> yaw, pitch, roll. `attention_flag = |yaw| < 30° and pitch in [-25°, 20°]`.
- **Motion / Inactivity**: displacement of landmark centroid + frame-difference energy inside the person box; `inactivity_seconds` increments while motion < threshold, resets on movement.

### 3.3 Audio Pipeline
- WebRTC VAD (aggressiveness 2) on 30 ms frames; RMS energy in dBFS.
- Window features: `speech_ratio`, `mean_rms`, `turn_count` (speech segments separated by > 1.5 s silence), `longest_silence_s`.
- Audio is **class-level only** (single microphone cannot reliably attribute speakers). No transcription is performed.

### 3.4 Attendance Engine
State machine per track/identity:
```
ABSENT --seen--> CANDIDATE --continuous>=T_cont OR cumulative>=ratio--> PRESENT
PRESENT --missing>T_leave--> LEFT --seen again--> PRESENT (re-entered)
CANDIDATE --missing>flicker_tolerance--> ABSENT
```
Defaults: `min_continuous_seconds=20`, `min_presence_ratio=0.60`, `flicker_tolerance_s=3`, `leave_after_s=60`, `late_after_minutes=10`.

### 3.5 Fusion Engine
Per student, per window (60 s, hop 30 s):
```
attention      = mean(attention_flag)
movement_norm  = clip(mean(movement_energy)/M_ref, 0, 1)
inactivity_norm= clip(max(inactivity_seconds)/window_s, 0, 1)
audio_context  = f(class speech_ratio, turn_count)  # class-level, same for all students
score = 100*(0.45*attention + 0.20*movement_norm + 0.20*(1-inactivity_norm) + 0.15*audio_context)
score = EMA(score, alpha=0.4)
```
Levels: High >= 70, Medium 40–69, Low < 40. Weights/thresholds live in `configs/default.yaml`.

### 3.6 Trend Detection (`engagement/trends.py`)
- **Attention drop**: class mean falls >= 15 points within 5 min.
- **Interaction peak**: speech_ratio and turn_count both above 80th percentile of session.
- **Silent stretch**: no speech > 4 min.
- **Pacing issue**: low movement + low audio + dropping attention for >= 3 consecutive windows.
- **Cluster disengagement**: >= 30 % of students Low simultaneously.

### 3.7 Agent (Decision Support)
1. **Rule engine** converts findings to structured suggestions `{category, title, rationale, time_range, confidence}`.
2. **Optional local LLM** (Ollama, `llama3.2:3b`/`phi3`) paraphrases suggestions using only structured data; temperature 0.3; fallback to template text if Ollama unavailable.
3. All outputs carry the label *"Decision-support only — instructor decides."*

### 3.8 Storage Schema (SQLite)
```
sessions(id, title, started_at, ended_at, source, config_hash)
participants(id, session_id, pseudo_id, display_name NULL, enrolled BOOL)
attendance(id, session_id, participant_id, first_seen, marked_present_at, left_at,
           total_visible_s, presence_ratio, status, late BOOL)
engagement_windows(id, session_id, participant_id NULL, t_start, t_end,
           attention, movement, inactivity, audio_ctx, score, level)
audio_windows(id, session_id, t_start, t_end, speech_ratio, mean_rms, turn_count, longest_silence_s)
findings(id, session_id, type, t_start, t_end, severity, payload_json)
suggestions(id, session_id, category, title, rationale, t_start, t_end, confidence, source)
```

### 3.9 Dashboard (Streamlit)
Pages: **Live/Run**, **Attendance**, **Engagement Timeline**, **Heatmap**, **Suggestions**, **Student Analytics**, **Compare Sessions**, **Privacy & Settings**.

### 3.10 Privacy Layer
Pseudonymisation, encrypted embeddings (Fernet), no raw media persisted, retention purge job, consent notice, offline enforcement test.

## 4. Data Flow (single session)
1. Load config -> create session row.
2. Stream frames/audio -> extract per-second features -> update attendance state.
3. Aggregate windows -> fusion -> write to DB.
4. On session end: trend detection -> rules (+LLM) -> suggestions -> report export.
5. Dashboard reads from DB.

## 5. Technology Choices & Rationale
| Need | Choice | Why |
|---|---|---|
| Detection/tracking | YOLOv8n + ByteTrack | Fast on CPU, offline weights |
| Head pose | MediaPipe FaceMesh + PnP | Lightweight, no training needed |
| Identity | InsightFace ArcFace (ONNX) | Accurate, offline |
| VAD | webrtcvad | Tiny, robust, CPU-only |
| LLM | Ollama local models | Offline GenAI, optional |
| DB | SQLite | Zero setup |
| UI | Streamlit + Plotly | Fast to build, interactive |

## 6. Performance Plan
Frame skipping (every 2–3 frames), resize to 960 px width, ONNX runtime, batch face-mesh per tracked face crop, run audio in separate thread, cache embeddings per track.

## 7. Error Handling & Degradation
No audio -> vision-only (audio weight redistributed). No face -> body-only motion + presence. Low light -> warn and lower confidence. LLM down -> templates. Corrupt frames -> skipped and logged.

## 8. Testing Strategy
Unit tests per module; synthetic videos (moving rectangles/faces) for attendance logic; golden-file tests for fusion; end-to-end on a 60 s clip; offline-network assertion test.

## 9. Limitations & Ethics
Head pose is a proxy for attention, not proof of learning. Camera angle, lighting and occlusion affect accuracy. Audio cannot attribute individual speakers. Scores are advisory and must not be used for grading or punishment.
