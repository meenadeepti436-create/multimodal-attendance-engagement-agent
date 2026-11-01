# System Requirements — Multimodal Attendance & Engagement Agent

## 1. Hardware
| Tier | CPU | RAM | GPU | Storage | Notes |
|---|---|---|---|---|---|
| Minimum | 4-core (i5 / Ryzen 5) | 8 GB | none | 5 GB free | ~5–8 FPS, vision-only mode comfortable |
| Recommended | 6–8 core | 16 GB | NVIDIA 4 GB+ (optional) | 15 GB free | ~10–20 FPS, optional local LLM |
| Camera | 720p webcam or classroom camera (front-facing, students' faces visible) |
| Microphone | Any room mic (class-level audio only) |

## 2. Software
- OS: Windows 10/11, Ubuntu 20.04+, or macOS 12+
- Python 3.10 or 3.11
- ffmpeg (system install, on PATH)
- Optional: Ollama (https://ollama.com) with `llama3.2:3b` or `phi3:mini` pulled in advance
- Optional: CUDA 11.8/12.x for GPU acceleration

## 3. Python Dependencies (`requirements.txt`)
```
numpy>=1.24
pandas>=2.0
opencv-python>=4.8
ultralytics>=8.1
mediapipe>=0.10
onnxruntime>=1.16
insightface>=0.7        # optional: identity enrolment
webrtcvad-wheels>=2.0.14
librosa>=0.10
soundfile>=0.12
sounddevice>=0.4        # live mic
scipy>=1.10
scikit-learn>=1.3
pydantic>=2.5
pyyaml>=6.0
sqlalchemy>=2.0
cryptography>=41.0
streamlit>=1.30
plotly>=5.18
reportlab>=4.0
requests>=2.31          # only for local Ollama (localhost)
pytest>=7.4
pytest-cov>=4.1
ruff>=0.1
```

## 4. Functional Requirements
| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Accept live webcam or recorded video (mp4/avi/mkv) | Must |
| FR-02 | Detect and track multiple participants with stable IDs | Must |
| FR-03 | Mark attendance only after sustained presence (configurable thresholds) | Must |
| FR-04 | Timestamped attendance logs; late/left/re-entered events | Must |
| FR-05 | Estimate head pose per participant | Must |
| FR-06 | Compute movement and inactivity duration | Must |
| FR-07 | Compute class-level audio activity (VAD, speech ratio, turns) | Must |
| FR-08 | Fuse signals into windowed engagement scores | Must |
| FR-09 | Detect attention drops, interaction peaks, pacing issues | Must |
| FR-10 | Generate teaching suggestions with rationale and confidence | Must |
| FR-11 | Per-student and class session analytics dashboard | Must |
| FR-12 | Export CSV and PDF reports | Must |
| FR-13 | Optional identity enrolment and roster mapping | Should |
| FR-14 | Optional local-LLM phrasing of suggestions | Should |
| FR-15 | Compare multiple sessions | Could |
| FR-16 | Configurable thresholds/weights via YAML | Must |

## 5. Non-Functional Requirements
| ID | Requirement | Target |
|---|---|---|
| NFR-01 | Fully offline at runtime | 0 external network calls |
| NFR-02 | Throughput | >= 10 FPS @720p on recommended CPU (with frame skipping) |
| NFR-03 | Attendance accuracy on test clips | >= 90 % correct present/absent |
| NFR-04 | Flicker robustness | Appearance < 3 s never marks present |
| NFR-05 | Privacy | No raw media stored by default; pseudonymous IDs |
| NFR-06 | Reliability | Graceful degradation when audio/LLM/faces unavailable |
| NFR-07 | Maintainability | Typed code, ruff clean, >= 70 % test coverage on core |
| NFR-08 | Usability | Dashboard usable without CLI knowledge after session run |
| NFR-09 | Portability | Runs on Windows/Linux/macOS |

## 6. Privacy & Ethical Requirements
1. Obtain consent from participants/institution before use; display notice.
2. No cloud, no telemetry, no speech-to-text.
3. Store derived features only; encrypt face embeddings; provide purge command.
4. Scores are decision-support, not for grading or discipline.
5. No individual ranking displayed to the class.

## 7. Datasets / Assets for Development & Evaluation
- Self-recorded classroom-style clips (with consent), 2–10 min, 3–8 people.
- Synthetic generator (`scripts/generate_demo_data.py`) for deterministic tests.
- Optional public data for sanity checks: WIDER FACE (detection), 300W-LP/AFLW2000 (head pose), AMI Meeting Corpus / VoxConverse (audio activity).
- Model weights downloaded once (`yolov8n.pt`, MediaPipe bundled, InsightFace `buffalo_l`) then used offline.

## 8. Evaluation Metrics
- Attendance: precision, recall, F1 vs manual ground truth; false-positive rate for flickers.
- Head pose: MAE (degrees) on a labelled sample.
- Engagement: correlation with manually annotated attention labels (Spearman), window-level agreement.
- Audio: VAD F1 on annotated clip.
- System: FPS, memory, latency per window.
- Suggestions: instructor usefulness rating (1–5) in a small pilot.

## 9. Constraints & Assumptions
Front-facing camera with most faces visible; adequate lighting; one room mic; sessions up to 3 hours; up to ~40 participants (accuracy degrades with distance/occlusion).

## 10. Deliverables
Source code, config, tests, Streamlit dashboard, sample report PDF, README, Technical Architecture, System Requirements, Instructions, Privacy note, short demo video.
