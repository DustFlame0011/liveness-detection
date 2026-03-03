# Tech Stack: Liveness Detection Streamlit Demo
## AI PM Portfolio Project — LiveShield

---

## Overview — What We're Building

```
[Webcam / Camera Input]
     │
     ▼
[Streamlit Web App]
     │
     ├── OpenCV → capture frames from camera
     ├── MediaPipe → face detection + 468 facial landmarks
     ├── Detection Modules → analyse each layer
     │       ├── Layer 1: Texture + Reflection Analysis
     │       ├── Layer 2: Active Challenge (blink / head turn)
     │       └── Layer 3: Temporal Frame Analysis
     │
     └── UI Output → Score + Verdict + Signal Breakdown
```

---

## Full Stack

### Core Libraries

| Library | Purpose | Install |
|---------|---------|---------|
| `streamlit` | Web UI framework — runs app in browser | `pip install streamlit` |
| `opencv-python-headless` | Image processing, frame manipulation | `pip install opencv-python-headless` |
| `mediapipe` | Google's face detection + 468 facial landmark points | `pip install mediapipe` |
| `numpy` | Array operations for pixel analysis | `pip install numpy` |
| `Pillow` | Image format conversion | `pip install Pillow` |
| `scipy` | Statistical analysis for texture metrics | `pip install scipy` |

### Optional (for advanced Layer 3)

| Library | Purpose | When to use |
|---------|---------|------------|
| `torch` + `torchvision` | Run pre-trained deepfake detection model | When using a real ML model |
| `facenet-pytorch` | Face embeddings for consistency comparison | Layer 3 face consistency |
| `plotly` | Interactive charts in Streamlit | Confidence score visualisation |

---

## What Each Layer Actually Does

### Layer 1: Passive Analysis — OpenCV + NumPy

```python
import cv2
import numpy as np

def analyze_texture(frame):
    """
    Real skin has micro-texture and subsurface light scatter.
    Printed / screen faces have flatter, more uniform texture.
    Measured with Laplacian variance — high value = more texture = likely real.
    """
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    laplacian_var = cv2.Laplacian(gray, cv2.CV_64F).var()
    return laplacian_var


def detect_screen_reflection(frame):
    """
    Phone screens and monitors produce a flat, uniform specular highlight.
    Real skin produces diffuse, irregular reflections.
    Detected by looking for large uniform high-brightness regions in HSV space.
    """
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    # High V (brightness) + low S (saturation) = uniform screen glow pattern
    high_value_mask = hsv[:, :, 2] > 240
    reflection_ratio = np.sum(high_value_mask) / high_value_mask.size
    return reflection_ratio  # High ratio = suspected screen replay
```

### Layer 2: Active Challenge — MediaPipe

```python
import mediapipe as mp

mp_face_mesh = mp.solutions.face_mesh


def detect_blink(landmarks):
    """
    MediaPipe provides 468 landmark points across the face.
    Eye landmarks are used to compute Eye Aspect Ratio (EAR).
    EAR < 0.22 = eye is closing (blink detected).

    Reference: Soukupova & Cech (2016) — Real-Time Eye Blink Detection
    using Facial Landmarks.
    """
    # Left eye landmark indices
    LEFT_EYE = [362, 382, 381, 380, 374, 373,
                390, 249, 263, 466, 388, 387, 386, 385, 384, 398]

    # EAR = sum(vertical distances) / (2 * horizontal distance)
    pass


def detect_head_turn(landmarks):
    """
    Compare nose tip position relative to the midpoint between eye centres.
    Head turn = nose shifts left or right of the centre line.
    """
    pass
```

### Layer 3: Temporal Analysis — Multi-frame consistency

```python
from collections import deque


class TemporalAnalyzer:
    """
    Maintains a rolling buffer of recent frames.
    Deepfakes commonly exhibit temporal artifacts:
      - Edge flicker at the face boundary (deepfake seam)
      - Unnatural skin smoothness across all frames (GAN over-smoothing)
      - Lighting inconsistency between face and background
    """

    def __init__(self, buffer_size=15):
        self.frame_buffer = deque(maxlen=buffer_size)
        self.edge_scores = deque(maxlen=buffer_size)

    def analyze_edge_consistency(self, frame, face_bbox):
        """
        Real-time deepfake overlays often leave an edge artifact at the
        face boundary. Measured using edge gradient magnitude around the
        face perimeter — high or erratic values indicate a deepfake seam.
        """
        pass

    def compute_temporal_score(self):
        """
        Average edge consistency score across all frames in the buffer.
        Abnormally low OR high variance across frames suggests deepfake.
        """
        pass
```

---

## App Architecture

```
app.py                        ← main Streamlit UI and session flow
│
├── liveness/
│   ├── __init__.py
│   ├── layer1_passive.py     ← texture, reflection, colour analysis
│   ├── layer2_active.py      ← blink detection, head pose tracking
│   ├── layer3_temporal.py    ← multi-frame edge + texture consistency
│   └── scorer.py             ← combine layers → composite score → verdict
│
├── utils/
│   ├── __init__.py
│   ├── camera.py             ← OpenCV camera handling
│   ├── visualizer.py         ← draw landmarks, bounding boxes, score overlay
│   └── logger.py             ← session logging
│
├── .streamlit/
│   └── config.toml           ← dark theme + server config
│
└── requirements.txt
```

---

## UI Layout

```
┌─────────────────────────────────────────────┐
│  LiveShield — AI Liveness Detection         │
├──────────────────────┬──────────────────────┤
│                      │  LIVENESS SCORE      │
│   [CAMERA FEED]      │  ████████░░  0.82    │
│                      │                      │
│   ● Face detected    │  Layer 1: Passive    │
│   ◉ Blink detected   │  Texture    ✓ 0.91   │
│   ○ Nod pending      │  Reflection ✓ 0.88   │
│                      │                      │
│                      │  Layer 2: Active     │
│                      │  Blink      ✓        │
│                      │  Head Turn  ○ wait   │
│                      │                      │
│                      │  Layer 3: Temporal   │
│                      │  Frames: 12/15       │
│                      │  Edge: ✓ consistent  │
│                      │                      │
│                      │  VERDICT: REVIEW     │
│                      │  Confidence: 0.82    │
└──────────────────────┴──────────────────────┘
│  Signals: slight_edge_artifact_detected      │
└──────────────────────────────────────────────┘
```

---

## Build Timeline

### Week 1 — Setup + Layer 1
- [ ] Install dependencies, set up `st.camera_input()` in Streamlit
- [ ] Implement texture analysis (Laplacian variance)
- [ ] Implement screen reflection detection (HSV high-value mask)
- [ ] Build basic UI with camera feed display

**Checkpoint:** App opens camera + displays texture score

---

### Week 2 — Layer 2 Active Challenge
- [ ] Integrate MediaPipe Face Mesh (468 landmarks)
- [ ] Implement Eye Aspect Ratio (EAR) for blink detection
- [ ] Implement nose position tracking for head turn detection
- [ ] Build challenge prompt UI ("Please blink twice")
- [ ] Add challenge randomisation (pool of 4+ actions)

**Checkpoint:** App requests a blink and detects it correctly

---

### Week 3 — Layer 3 + Score Combination
- [ ] Build frame buffer (`deque`, 15 frames)
- [ ] Implement edge consistency analysis (Canny + MAD across frames)
- [ ] Build `scorer.py` combining Layer 1 + 2 + 3 into composite score
- [ ] Implement threshold logic: REAL / REVIEW / FAKE
- [ ] Add confidence breakdown UI with per-layer scores

**Checkpoint:** App returns verdict with full signal breakdown

---

### Week 4 — Polish + Deploy
- [ ] Build Attack Simulation Mode — test against photo / screen replay
- [ ] Add session logging (score history table)
- [ ] Deploy to Streamlit Cloud (free hosting)
- [ ] Write README documenting every PM threshold decision

**Checkpoint:** Live demo link ready to share in portfolio

---

## What This Demo Signals to Recruiters

| What the demo shows | PM signal recruiters read |
|--------------------|--------------------------|
| Multi-layer architecture | Systems thinking — not just a single feature |
| Confidence score + breakdown | Understanding of AI explainability |
| REAL / REVIEW / FAKE threshold | Awareness of security vs. UX tradeoff |
| Attack simulation mode | Adversarial case thinking, not just happy path |
| Skin tone note in README | Understanding of AI fairness and bias |

---

## Deployment Notes

### Streamlit Cloud Constraints

| Constraint | Cause | Workaround in prototype |
|-----------|-------|------------------------|
| No `cv2.VideoCapture()` | Server has no camera access | Use `st.camera_input()` — snapshot based |
| Layer 2 real-time blink tracking unavailable | Requires continuous video stream | 2-snapshot comparison: baseline vs. challenge |
| mediapipe version pinning required | Dependency conflicts on cloud | Pin to `mediapipe==0.10.30` in `requirements.txt` |

### Quick Start (local)

```bash
# Requires Python 3.10 – 3.12 (mediapipe does not support 3.13+)
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
streamlit run app.py
```

---

## References

| Topic | Source |
|-------|--------|
| MediaPipe Face Mesh | https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker |
| Eye Aspect Ratio (blink detection) | Soukupova & Cech, 2016 — "Real-Time Eye Blink Detection using Facial Landmarks" |
| ISO/IEC 30107-3 (PAD standard) | https://www.iso.org/standard/79520.html |
| FaceForensics++ dataset | https://github.com/ondyari/FaceForensics |
| Streamlit camera input | https://docs.streamlit.io/develop/api-reference/media/st.camera_input |
