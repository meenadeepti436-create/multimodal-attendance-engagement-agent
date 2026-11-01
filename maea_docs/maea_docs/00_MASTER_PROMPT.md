# MASTER PROMPT — Multimodal Attendance & Engagement Agent

> Paste everything between the START and END markers into Claude Code / Cursor / any coding agent.
> Attach `01_Technical_Architecture.md`, `02_System_Requirements.md`, `03_Instructions.md` in the project root first
> (put them in `/docs`) so the agent can read them.

---------------------------- START OF PROMPT ----------------------------

You are a senior ML engineer and Python architect. Build a complete, runnable, OFFLINE,
privacy-preserving project called **Multimodal Attendance & Engagement Agent (MAEA)**.

## 1. Goal
An AI classroom-intelligence system that, from a live webcam or a recorded video (with its audio):
1. Automatically marks attendance based on SUSTAINED presence (not single-frame detection).
2. Analyses engagement over time by fusing head pose, movement, inactivity duration and audio activity.
3. Generates context-aware, AI-driven TEACHING SUGGESTIONS (assistive only, never enforcing decisions).
4. Provides per-student and class-level SESSION ANALYTICS via a dashboard and exportable reports.
Everything must run fully offline on a normal laptop (CPU is mandatory, GPU optional). No cloud APIs.

## 2. Read first
Read `/docs/01_Technical_Architecture.md`, `/docs/02_System_Requirements.md` and `/docs/03_Instructions.md`.
They are the source of truth. If anything is ambiguous, choose the simplest option that satisfies them and
record the decision in `docs/DECISIONS.md`.

## 3. Tech stack (fixed)
- Python 3.10+ ; OpenCV ; NumPy ; Pandas
- Person/face detection + tracking: Ultralytics YOLO (person) + ByteTrack, and a face detector (MediaPipe or InsightFace)
- Identity (optional enrolment): face embeddings (InsightFace ArcFace ONNX) matched by cosine similarity; fallback to anonymous IDs (S01, S02, ...)
- Head pose: MediaPipe Face Mesh landmarks + cv2.solvePnP -> yaw/pitch/roll
- Movement / inactivity: landmark + bounding-box displacement and frame-difference energy per track
- Audio: ffmpeg extraction -> 16 kHz mono; WebRTC VAD (or Silero VAD) + RMS energy; class-level speech ratio, turn-taking count, silence streaks
- Fusion: weighted, normalised, EMA-smoothed engagement score (config-driven)
- Agent: rule engine (always on) + OPTIONAL local LLM via Ollama (e.g. llama3.2:3b / phi3) that turns structured findings into natural-language suggestions. System must work with LLM disabled.
- Storage: SQLite (SQLAlchemy) ; Config: YAML (pydantic validated)
- UI: Streamlit dashboard (Plotly charts) ; Reports: CSV + PDF (reportlab) 
- Tests: pytest ; Lint: ruff ; Packaging: requirements.txt + Makefile

## 4. Required project structure
```
maea/
  docs/ (the 3 provided docs + DECISIONS.md)
  configs/default.yaml
  src/maea/
    __init__.py
    config.py                # pydantic settings loader
    pipeline.py              # orchestrates a session end-to-end
    io/video_source.py       # webcam / file reader, frame skipping, timestamps
    io/audio_source.py       # ffmpeg extraction + chunking
    vision/detector.py       # person/face detection
    vision/tracker.py        # ByteTrack wrapper, stable track IDs
    vision/identity.py       # enrolment + embedding matching (optional)
    vision/head_pose.py      # Face Mesh + solvePnP
    vision/motion.py         # movement + inactivity features
    audio/vad.py             # voice activity detection
    audio/features.py        # speech ratio, energy, turn-taking, silence streaks
    attendance/engine.py     # sustained-presence state machine
    engagement/fusion.py     # multimodal fusion + smoothing + windowing
    engagement/trends.py     # drops, peaks, pacing issues detection
    agent/rules.py           # rule-based findings
    agent/llm.py             # optional Ollama client (offline, graceful fallback)
    agent/suggestions.py     # merges rules + LLM into final suggestions
    storage/db.py, models.py # SQLite schema + DAO
    privacy/anonymizer.py    # pseudonymisation, retention purge, blur option
    reports/exporter.py      # CSV + PDF
    dashboard/app.py         # Streamlit UI
  scripts/
    enroll.py  run_session.py  generate_demo_data.py  purge_data.py
  tests/                     # unit tests per module + one end-to-end test on a short sample clip
  Makefile  requirements.txt  README.md
```

## 5. Functional requirements (implement ALL)
### 5.1 Attendance
- Track every person with a stable ID across frames (handle brief occlusion; re-link within N seconds).
- State machine per person: ABSENT -> CANDIDATE -> PRESENT -> (LEFT).
  Mark PRESENT only after continuous presence >= `min_continuous_seconds` (default 20 s) OR cumulative
  visibility >= `min_presence_ratio` (default 60 %) of the session. Ignore flickers shorter than `flicker_tolerance_s`.
- Log timestamped events: first_seen, marked_present, left, re_entered, total_visible_seconds, presence_ratio.
- Output attendance table (CSV + DB). Late arrivals flagged using `late_after_minutes`.
- Optional roster file (CSV) to map enrolled identities to names; unknown faces stay anonymous.

### 5.2 Engagement
- Compute per-track, per-second features: yaw, pitch, attention_flag (looking towards front within thresholds),
  movement_energy, inactivity_seconds (no meaningful motion), face_visible.
- Audio (class-level, because one mic cannot attribute speech to individuals): speech_ratio per window,
  RMS energy, number of speaker turns, longest silence streak.
- Fuse into `engagement_score` in [0,100] per student per window (default window 60 s, hop 30 s):
  `score = 100 * (w_att*attention + w_mov*movement_norm + w_inact*(1-inactivity_norm) + w_aud*audio_context)`,
  weights from config (defaults 0.45 / 0.20 / 0.20 / 0.15). Smooth with EMA. Document the formula in code.
- Class-level engagement = mean/median across present students + percentage "highly engaged / neutral / low".
- Trend detection: attention drops (>= X points over Y minutes), interaction peaks, long silent stretches,
  pacing issues (sustained low movement + low audio), late-session fatigue.
- Label engagement levels: High (>=70), Medium (40-69), Low (<40) — configurable.

### 5.3 AI teaching suggestions
- Rule engine maps findings to suggestion templates (e.g. engagement drop after 20 min -> short interactive activity;
  long monologue with no student speech -> ask a question/poll; high class-wide low movement + low audio -> change pacing/take a break;
  small cluster of disengaged students -> targeted check-in).
- Each suggestion includes: title, rationale (the evidence with timestamps), confidence, category
  (pacing / interaction / classroom-management / content), and the time range it applies to.
- Optional Ollama LLM rewrites findings into friendly, concise advice. Strict prompt: use ONLY provided data,
  no student names, no judgment of individuals, advisory tone ("consider...").
- Suggestions are ALWAYS labelled "Decision-support only — instructor decides".

### 5.4 Session analytics
- Class timeline chart (engagement, attention, speech ratio), attendance table, per-student timeline,
  heatmap (students x time), pacing summary, suggestion cards, session comparison across past sessions.
- Export: attendance.csv, engagement_timeseries.csv, session_report.pdf.

## 6. Privacy & ethics (non-negotiable)
- 100 % offline; block any network calls in runtime code (add a test that asserts no sockets are opened in the pipeline).
- Never store raw video/audio by default; keep only derived features. Option `store_debug_frames=false`.
- Pseudonymous IDs by default; face embeddings stored encrypted (Fernet key in local keyfile) and only if the user ran enrolment.
- No emotion/identity inference from audio content; no speech-to-text; audio is used only for activity levels.
- Consent banner in dashboard + `docs/PRIVACY.md`. Retention policy with `scripts/purge_data.py`.
- Engagement scores are indicators, not grades; show uncertainty and never rank students publicly.

## 7. Non-functional targets
- >= 10 FPS processing on a modern CPU laptop at 720p with frame skipping; configurable `process_every_n_frames`.
- Deterministic, seeded demo mode. Graceful degradation: if audio missing -> vision-only; if no faces -> body-only; if LLM missing -> rules only.
- Clear logging, typed code, docstrings, no dead code, config over constants.

## 8. Build order — do this step by step and STOP to run tests after each phase
1. Scaffold repo, config, DB schema, logging, Makefile.
2. Video/audio IO + synthetic demo-data generator.
3. Detection + tracking -> attendance engine + tests.
4. Head pose + motion features + audio features + tests.
5. Fusion + trend detection + tests.
6. Rule-based agent + optional Ollama LLM + tests.
7. Privacy module, exporter, PDF report.
8. Streamlit dashboard.
9. End-to-end test, README with screenshots instructions, final polish.
After each phase print: what was built, how to run it, test results.

## 9. Acceptance criteria
- `make demo` runs a full session on bundled/generated sample data and opens a populated dashboard.
- `python scripts/run_session.py --video path.mp4` produces attendance.csv, engagement_timeseries.csv, suggestions, report PDF.
- `pytest` passes; coverage of core logic >= 70 %.
- A person appearing for 2 seconds is NOT marked present; one present for the whole session IS.
- Turning the LLM off still produces valid suggestions.
- No network access required after dependencies and models are installed.

## 10. Output format for your work
Create all files, run the tests, fix failures, then give me: (a) a tree of the repo, (b) run instructions,
(c) known limitations, (d) next improvements. Do not leave TODO stubs for required features.

----------------------------- END OF PROMPT -----------------------------
