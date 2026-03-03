<div align="center">

# 🛡️ LiveShield

### AI-Powered Liveness Detection for eKYC

**A multi-layer deepfake detection prototype — built as an AI PM portfolio project**

[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.32-FF4B4B?style=flat-square&logo=streamlit&logoColor=white)](https://streamlit.io)
[![OpenCV](https://img.shields.io/badge/OpenCV-4.9-5C3EE8?style=flat-square&logo=opencv&logoColor=white)](https://opencv.org)
[![MediaPipe](https://img.shields.io/badge/MediaPipe-0.10-0097A7?style=flat-square)](https://mediapipe.dev)
[![ISO](https://img.shields.io/badge/Standard-ISO%2FIEC%2030107--3-22C55E?style=flat-square)](https://www.iso.org/standard/79520.html)
[![License](https://img.shields.io/badge/License-MIT-6B7280?style=flat-square)](LICENSE)

[Demo Flow](#-demo-flow) · [Architecture](#-architecture) · [PM Decisions](#-pm-decisions) · [Roadmap](#-roadmap)

</div>

---

## Why This Exists

Most eKYC systems ask users to **blink or smile** to prove they are real.

Real-time deepfake tools can replicate a blink in under 200ms. Deepfake fraud attempts quadrupled between 2023 and 2024, now accounting for 7% of all fraud attempts globally. A single challenge-response layer is no longer sufficient.

**LiveShield uses three independent detection layers** — passive biometric analysis, active challenge-response, and temporal frame consistency — to detect not just whether a face responds, but how it was rendered.

> This project is the working prototype for a full product specification: [`04-liveness-detection-PRD.md`](https://github.com/DustFlame0011/liveness-detection/blob/main/PRD.md), written to ISO/IEC 30107-3 and aligned to Thailand PDPA.

---

## 🎬 Demo Flow

The app runs a 3-step browser-based session:

```
Step 1 — Baseline Capture
  └── User takes a neutral photo → Layer 1 passive analysis runs immediately

Step 2 — Active Challenge
  └── System issues a randomised prompt: BLINK · LOOK LEFT · LOOK RIGHT · SMILE
  └── User performs the action, takes a second photo
  └── Layer 2 (landmark comparison) + Layer 3 (frame consistency) run

Step 3 — Verdict
  └── Composite score 0.00–1.00
  └── REAL (≥0.85) · REVIEW (0.70–0.84) · FAKE (<0.70)
  └── Full signal breakdown + webhook JSON payload
```

The sidebar lets you adjust thresholds in real time to see how the verdict changes — demonstrating the security vs. UX tradeoff as a live product decision.

---

## 🏗 Architecture

```
[Browser Camera — st.camera_input()]
            │
            ▼
┌───────────────────────────────────────────────┐
│  Layer 1 · Passive Analysis (no user action)  │
│                                               │
│  Texture      Laplacian variance of face ROI  │
│  Reflection   HSV uniform brightness mask     │
│  Colour       YCbCr skin distribution check   │
└───────────────────────┬───────────────────────┘
                        │ weight: 45%
                        ▼
┌───────────────────────────────────────────────┐
│  Layer 2 · Active Challenge-Response          │
│                                               │
│  Blink        Eye Aspect Ratio (EAR) < 0.22   │
│  Head turn    Nose-to-eye-center offset delta  │
│  Smile        Mouth corner width increase      │
│  Timing       150ms < response < 500ms         │
└───────────────────────┬───────────────────────┘
                        │ weight: 35%
                        ▼
┌───────────────────────────────────────────────┐
│  Layer 3 · Temporal Frame Analysis            │
│                                               │
│  Edge consistency   Canny MAD across frames   │
│  Texture variance   Laplacian per frame        │
│  Lighting ratio     Face brightness vs BG      │
└───────────────────────┬───────────────────────┘
                        │ weight: 20%
                        ▼
┌───────────────────────────────────────────────┐
│  Scorer · Composite Score (0.00 – 1.00)       │
│                                               │
│  ≥ 0.85  →  REAL      auto-approve            │
│  0.70–0.84  →  REVIEW  human queue (2hr SLA)  │
│  < 0.70  →  FAKE      auto-reject + log       │
└───────────────────────────────────────────────┘
```

### Attack Coverage

| Attack Type | Generation | Detected by |
|------------|-----------|-------------|
| Printed photo | Gen 1 | Layer 1 — Texture + Reflection |
| Screen replay (video on phone) | Gen 2 | Layer 1 — Reflection + Colour |
| Real-time deepfake overlay | Gen 3 | Layer 2 timing + Layer 3 edge seam |
| Injection attack (bypass camera) | Gen 4 | Device attestation *(mobile SDK — not in this prototype)* |

---

## 📁 Project Structure

```
liveshield/
├── app.py                      ← Streamlit app — 3-step session flow, UI, webhook output
├── requirements.txt            ← Pinned dependencies for Streamlit Cloud compatibility
│
├── liveness/
│   ├── layer1_passive.py       ← Texture, reflection, YCbCr colour analysis
│   ├── layer2_active.py        ← EAR blink, head pose, smile detection
│   ├── layer3_temporal.py      ← Edge consistency, texture variance, lighting ratio
│   └── scorer.py               ← Composite score → verdict + recommendation
│
├── utils/
│   └── visualizer.py           ← Draw bounding box, landmarks, PIL ↔ OpenCV conversion
│
├── .streamlit/
│   └── config.toml             ← Dark theme
│
├── 04-liveness-detection-PRD.md    ← Full product specification
└── 04-liveness-tech-stack.md       ← Algorithm documentation
```

---

## 🧪 Test Scenarios

### ✅ Genuine User
1. Capture baseline — face forward, good lighting, neutral expression
2. Challenge: BLINK → blink naturally, then capture
3. Expected: Score ≥ 0.85, Verdict = `REAL`

### 🚫 Print Attack Simulation
1. Capture baseline normally
2. Challenge → open a photo of a different person on another device, point your camera at that screen
3. Expected: Layer 1 low (reflection detected), Score < 0.70, Verdict = `FAKE`

### ⚠️ Challenge Fail
1. Capture baseline normally
2. Challenge: LOOK LEFT → do not turn, stay facing forward
3. Expected: Layer 2 low, Score likely in REVIEW band

### 🔧 Threshold Tuning (PM Demo)
1. Complete any session to get a verdict
2. In the sidebar, drag the REAL threshold slider from `0.85` → `0.75`
3. Observe the verdict reclassify — this shows the threshold as a live product decision, not a fixed engineering constant

---

## 📐 PM Decisions

These are product decisions embedded in the code. They are not engineering defaults — each has explicit business reasoning and should be re-evaluated after 30 days of production data.

| Decision | Current Value | Location | Rationale |
|---------|-------------|----------|-----------|
| REAL threshold | `0.85` | `scorer.py` | Targets APCER < 5% — ISO/IEC 30107-3 Level 2 requirement |
| REVIEW threshold | `0.70` | `scorer.py` | Keeps manual review queue below 10% of sessions |
| Layer 1 weight | `45%` | `scorer.py` | Most reliable signal for print and screen attacks |
| Layer 2 weight | `35%` | `scorer.py` | Strong signal for deepfake overlay detection |
| Layer 3 weight | `20%` | `scorer.py` | Supplementary — limited frames available in 2-snapshot prototype |
| Blink timing floor | `150ms` | `layer2_active.py` | Below this = too fast to be human (Soukupova & Cech 2016) |
| Blink timing ceiling | `500ms` | `layer2_active.py` | Above this = consistent with pre-recorded replay |
| Texture target | `800` Laplacian var | `layer1_passive.py` | Average real skin at 720p in normal lighting |

> **Recalibration policy:** Any threshold change requires PM + Risk sign-off. Do not adjust based on a single edge case.

---

## 📊 Success Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **APCER** | Attack Presentation Classification Error Rate — % of attacks incorrectly accepted | < 5% |
| **BPCER** | Bona fide Presentation Classification Error Rate — % of real users incorrectly rejected | < 3% |
| **ACER** | (APCER + BPCER) / 2 | < 4% |
| **Verdict latency** | Time from session start to final verdict | < 5s |
| **Skin tone variance** | BPCER difference across Fitzpatrick I–VI groups | < 2% |

> Standard accuracy hides asymmetric costs. 5% false acceptance rate means 1 in 20 fraudsters passes — catastrophic for a fraud system. APCER/BPCER, defined in ISO/IEC 30107-3, forces an honest evaluation of the security vs. UX tradeoff.

---

## 🌍 Fairness & Bias

Most public deepfake training datasets (FaceForensics++, DFDC) skew toward lighter skin tones (Fitzpatrick I–II). A model trained primarily on this data produces higher false rejection rates for users with darker skin — not because they are committing fraud, but because the training data did not represent them.

This PRD requires:

- Minimum **15% SEA skin tone representation** in training data (Fitzpatrick III–IV)
- BPCER reported **separately per Fitzpatrick group**, not averaged
- Automated alert if any group's rejection rate deviates **> 2 percentage points** from the mean
- Monthly bias audit — P1 retraining ticket triggered if threshold is breached

---

## ⚠️ Known Limitations

| Limitation | Impact | Planned Fix |
|-----------|--------|------------|
| Snapshot-based capture, no real-time video | Layer 2 accuracy ~70–80% vs ~94% in production | Native mobile SDK with video stream (v2.0) |
| No device attestation | Cannot detect Gen 4 injection attacks | iOS DeviceCheck / Android Play Integrity in mobile SDK (v1.3) |
| Heuristic-based Layer 3 | May miss sophisticated Gen 3 deepfakes | Fine-tune Vision Transformer on FaceForensics++ (v2.0) |
| No custom training data | Thresholds calibrated on general webcam conditions | Recalibrate after 30-day production pilot |

---

## 🗺 Roadmap

| Version | Feature | Complexity |
|---------|---------|-----------|
| **v1.0** | 3-layer prototype · Streamlit · Session log · Webhook output | ✅ Done |
| v1.1 | Expand challenge pool from 4 → 8 actions | Low |
| v1.2 | Fine-tune model on SEA skin tone dataset | Medium |
| v1.3 | Mobile SDK with device attestation (iOS / Android) | High |
| v2.0 | Real-time video stream via WebRTC · ViT deepfake model | High |

---

## 📄 Related Documents

| File | Description |
|------|-------------|
| [`04-liveness-detection-PRD.md`](https://github.com/DustFlame0011/liveness-detection/blob/main/PRD.md) | Full product spec — problem statement, personas, metrics, acceptance criteria, AI model specification |
| [`04-liveness-tech-stack.md`](https://github.com/DustFlame0011/liveness-detection/blob/main/liveness-tech-stack.md) | Algorithm documentation — layer logic, architecture, build timeline |

---

## 📚 References

- Soukupova & Cech (2016) — [Real-Time Eye Blink Detection using Facial Landmarks](http://vision.fe.uni-lj.si/cvww2016/proceedings/papers/05.pdf)
- [ISO/IEC 30107-3](https://www.iso.org/standard/79520.html) — Biometric Presentation Attack Detection
- [MediaPipe Face Mesh](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker) — 468 landmark documentation
- [FaceForensics++ Dataset](https://github.com/ondyari/FaceForensics) — Deepfake training benchmark
- Sumsub Identity Fraud Report 2024

---

## 👤 Author

**Sukumal Plaengrat** — Technical Product Manager

[![LinkedIn](https://img.shields.io/badge/LinkedIn-sukumal--p-0A66C2?style=flat-square&logo=linkedin)](https://www.linkedin.com/in/sukumal-p/)
[![GitHub](https://img.shields.io/badge/GitHub-DustFlame0011-181717?style=flat-square&logo=github)](https://github.com/DustFlame0011)

---

<div align="center">
<sub>Built as part of an AI PM portfolio · Not a production system · PRD aligned to ISO/IEC 30107-3 · Thailand PDPA B.E. 2562</sub>
</div>
