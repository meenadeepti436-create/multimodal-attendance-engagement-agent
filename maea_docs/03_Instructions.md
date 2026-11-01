# Instructions — Build, Run, and Use MAEA

## Part A — Setup
1. Install Python 3.10/3.11 and ffmpeg (`ffmpeg -version` should work).
2. Create project folder and put the 3 docs in `docs/`.
3. Create environment:
   ```bash
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```
4. Download model weights once (internet needed only now):
   ```bash
   python -c "from ultralytics import YOLO; YOLO('yolov8n.pt')"
   python -c "import insightface; from insightface.app import FaceAnalysis; FaceAnalysis(name='buffalo_l').prepare(ctx_id=-1)"
   ```
5. (Optional) Local LLM: install Ollama, then `ollama pull llama3.2:3b`.
6. Disconnect from the internet — the system must work offline from here.

## Part B — Build using the Master Prompt
1. Open Claude Code / Cursor in the empty project folder.
2. Paste the prompt from `00_MASTER_PROMPT.md`.
3. Let the agent work phase by phase. After each phase run `pytest` and review.
4. If a phase fails, paste the error and say: "Fix this, keep the architecture in docs/01."

## Part C — Run
| Task | Command |
|---|---|
| Demo with synthetic data | `make demo` |
| Enrol students (optional) | `python scripts/enroll.py --name "Asha" --images data/enroll/asha/` |
| Process a recorded video | `python scripts/run_session.py --video data/class.mp4 --title "DSA Lecture 5"` |
| Live webcam session | `python scripts/run_session.py --webcam 0 --title "Live class"` |
| Open dashboard | `streamlit run src/maea/dashboard/app.py` |
| Purge old data | `python scripts/purge_data.py --older-than-days 30` |
| Tests | `pytest --cov=src/maea` |

## Part D — Configuration (`configs/default.yaml`)
```yaml
attendance:
  min_continuous_seconds: 20
  min_presence_ratio: 0.60
  flicker_tolerance_s: 3
  leave_after_s: 60
  late_after_minutes: 10
engagement:
  window_s: 60
  hop_s: 30
  weights: {attention: 0.45, movement: 0.20, inactivity: 0.20, audio: 0.15}
  ema_alpha: 0.4
  levels: {high: 70, medium: 40}
  yaw_limit_deg: 30
  pitch_range_deg: [-25, 20]
audio:
  vad_aggressiveness: 2
  turn_gap_s: 1.5
agent:
  use_llm: false
  llm_model: llama3.2:3b
  drop_threshold_points: 15
  silent_stretch_s: 240
privacy:
  store_debug_frames: false
  retention_days: 30
  encrypt_embeddings: true
performance:
  process_every_n_frames: 2
  max_width: 960
```

## Part E — Recommended Recording Setup (for good results)
- Camera at front-top of the room facing students; avoid backlight from windows.
- Students' faces should be at least ~60 px wide.
- Place mic near centre of room; avoid heavy fan/AC noise.
- Record 5–10 min test clips first and tune thresholds.

## Part F — Validation Checklist
- [ ] Person visible 2 s is not marked present.
- [ ] Person present throughout is marked present with correct first-seen time.
- [ ] Leaving and returning creates left + re-entered events.
- [ ] Looking away for a long stretch lowers attention score.
- [ ] Loud discussion raises speech_ratio and audio context.
- [ ] Silence + low movement triggers pacing suggestion.
- [ ] Dashboard loads all pages; CSV/PDF exports open correctly.
- [ ] Works with LLM disabled; works with audio missing.
- [ ] Network-off test passes.

## Part G — Demo Script (for presentation / viva)
1. Show problem: manual roll calls and basic biometrics miss engagement.
2. Run `make demo`; show attendance with sustained-presence logic.
3. Show engagement timeline; point to an attention drop.
4. Show suggestion generated for that drop with rationale.
5. Show privacy page: offline, no raw video stored, pseudonymous IDs.
6. Show evaluation metrics and limitations.

## Part H — Troubleshooting
| Problem | Fix |
|---|---|
| Webcam not opening | Try index 1/2; close other apps using camera |
| Slow FPS | Increase `process_every_n_frames`, lower `max_width`, use yolov8n |
| `webrtcvad` install fails | Use `webrtcvad-wheels` or switch to Silero VAD |
| Head pose unstable | Improve lighting; smooth with EMA; ignore faces < 60 px |
| Many ID switches | Raise tracker buffer; enable identity enrolment |
| Ollama not found | Set `agent.use_llm: false` — rules still work |

## Part I — Suggested 2-Week Timeline
- Days 1–2: scaffold, IO, demo data
- Days 3–4: detection, tracking, attendance
- Days 5–6: head pose, motion, audio features
- Days 7–8: fusion, trends
- Days 9–10: agent + LLM
- Days 11–12: dashboard + reports
- Days 13–14: testing, evaluation, documentation, demo video
