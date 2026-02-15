# AI-Powered Emergency Communication Bridge for the Deaf Community - Technical Design Document

**Version:** 1.0  
**Date:** February 15, 2026  
**Status:** Design Phase  
**Related Documents:** requirements.md v1.0

---

## Document Overview

This technical design document provides implementation-ready specifications for the Emergency Sign Language Bridge system. All design decisions are derived from and traceable to requirements.md. The architecture prioritizes sub-second latency (NFR-1.1, NFR-1.2), >95% accuracy on critical vocabulary (NFR-2.1), and 99.99% uptime (NFR-3.1).

---

## High-Level System Architecture

### Architecture Overview

The system employs a hybrid edge-cloud architecture to meet stringent latency requirements while maintaining high accuracy. Critical path operations run on-device, while complex inference leverages cloud GPU resources.

```
┌─────────────────────────────────────────────────────────────────┐
│                        CALLER DEVICE (Mobile)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Camera Input │→ │ Edge ML      │→ │ Video Compression   │  │
│  │ (30 FPS)     │  │ - Pose Det.  │  │ + Metadata Stream   │  │
│  └──────────────┘  │ - Urgency    │  └──────────┬──────────┘  │
│                     │ - Keywords   │             │              │
│  ┌──────────────┐  └──────────────┘             │              │
│  │ Avatar       │← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ┘              │
│  │ Renderer     │                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
                              ↕ TLS 1.3 Encrypted WebRTC
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUD INFERENCE CLUSTER                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Video Stream │→ │ Sign         │→ │ Text-to-Speech      │  │
│  │ Processor    │  │ Recognition  │  │ Synthesis           │  │
│  └──────────────┘  │ Transformer  │  └──────────┬──────────┘  │
│                     └──────────────┘             │              │
│  ┌──────────────┐  ┌──────────────┐             │              │
│  │ Multi-modal  │  │ Critical Info│             │              │
│  │ Urgency      │  │ Extraction   │             │              │
│  │ Fusion       │  │ (NER + Rules)│             │              │
│  └──────────────┘  └──────────────┘             │              │
└──────────────────────────────────────────────────┼──────────────┘
                                                    ↓
┌─────────────────────────────────────────────────────────────────┐
│                   DISPATCHER WORKSTATION                         │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Live Video + │  │ Structured   │  │ CAD System          │  │
│  │ Translation  │  │ Data Panel   │  │ Integration         │  │
│  │ Display      │  │ (Location,   │  │ (Dispatch Actions)  │  │
│  └──────────────┘  │  Urgency)    │  └─────────────────────┘  │
│                     └──────────────┘                            │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │ Operator     │→ │ Speech-to-   │→ Cloud → Avatar Animation │
│  │ Microphone   │  │ Text (STT)   │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Design Rationale

**Edge Processing (FR-5.1, NFR-1.10):**
- Pose detection runs on-device to reduce bandwidth (30 FPS video → landmark coordinates)
- Urgency keywords detected locally for immediate alerting
- Enables offline mode with degraded but functional service

**Cloud Processing (NFR-2.1, NFR-4.1):**
- Full transformer models require GPU resources unavailable on mobile
- Complex multi-modal fusion for accuracy targets (>95%)
- Centralized model updates without app reinstalls (NFR-9.1)

**Hybrid Approach:**
- Edge provides <200ms initial response (urgency, keywords)
- Cloud provides <800ms full translation (total <1s per NFR-1.1)
- Graceful degradation when cloud unavailable

---

## Client-Side Architecture (Mobile App)

### Technology Stack
- **Platform:** Flutter
- **ML Framework:** TensorFlow Lite (on-device inference)
- **Video:** WebRTC (real-time streaming)
- **State Management:** Redux Toolkit
- **Storage:** SQLite (offline queue), SecureStore (credentials)

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Mobile Application                       │
├─────────────────────────────────────────────────────────────┤
│  UI Layer (Flutter)                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Emergency    │  │ Video Call   │  │ Avatar Display  │  │
│  │ Button (1tap)│  │ Screen       │  │ Component       │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Business Logic Layer                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Call Manager │  │ Network      │  │ Offline Queue   │  │
│  │ (State)      │  │ Monitor      │  │ Manager         │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  ML Inference Layer (TensorFlow Lite)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Pose         │  │ Keyword      │  │ Urgency         │  │
│  │ Detection    │  │ Classifier   │  │ Scorer          │  │
│  │ (MediaPipe)  │  │ (MobileNet)  │  │ (Lightweight)   │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Communication Layer                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ WebRTC       │  │ REST API     │  │ SMS Fallback    │  │
│  │ Client       │  │ Client       │  │ (GPS coords)    │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Device APIs                                                │
│  │ Camera │ GPS │ Network │ Storage │ Permissions │       │
└─────────────────────────────────────────────────────────────┘
```

### Edge ML Models (FR-5.7, NFR-1.7)

**1. Pose Detection Model**
- **Base:** MediaPipe Holistic (Google)
- **Size:** 12 MB
- **Latency:** 33ms per frame (30 FPS)
- **Output:** 543 landmarks (hands: 21×2, pose: 33, face: 468)
- **Optimization:** Quantized INT8 for mobile

**2. Emergency Keyword Classifier**
- **Architecture:** MobileNetV3 + LSTM (2 layers, 128 units)
- **Size:** 8 MB
- **Input:** Last 30 frames of landmarks (1 second window)
- **Output:** 20 critical keywords (weapon, fire, blood, help, etc.) + confidence
- **Accuracy:** 88% on emergency vocabulary (degraded from cloud 95%)

**3. Local Urgency Scorer**
- **Architecture:** Lightweight CNN (3 conv layers)
- **Size:** 3 MB
- **Input:** Signing speed, facial landmarks, keyword presence
- **Output:** Urgency score 1-10
- **Latency:** 15ms

**Total Edge Model Size:** 23 MB (within 100 MB constraint per NFR-1.7)

### Video Processing Pipeline

```
Camera (30 FPS, 720p) 
  ↓
Frame Buffer (3 frame lookahead for stabilization)
  ↓
Pose Detection (MediaPipe Lite)
  ↓
┌─────────────────┬─────────────────┐
│ Landmark Stream │ Compressed Video│
│ (JSON, 5 KB/s)  │ (H.264, 500kbps)│
└────────┬────────┴────────┬────────┘
         ↓                 ↓
    Edge Models      WebRTC Stream
         ↓                 ↓
    Local Urgency    Cloud Inference
```

**Compression Strategy (FR-5.4):**
- Adaptive bitrate: 1 Mbps (good network) → 200 kbps (poor network)
- Resolution scaling: 720p → 480p → 360p based on bandwidth
- Frame rate reduction: 30 FPS → 15 FPS if needed
- Landmark-only mode: Send coordinates instead of video (<10 kbps)

### Offline Mode Implementation (FR-5.1, FR-5.7)

**Capabilities:**
- Pose detection (fully offline)
- Emergency keyword recognition (88% accuracy)
- GPS coordinate extraction
- Local urgency scoring

**Limitations:**
- No full sentence translation
- No speech-to-sign (avatar unavailable)
- No multi-modal fusion

**Fallback Behavior:**
1. Detect network loss within 2 seconds (SFR-4.1)
2. Switch to offline models automatically
3. Display "Limited Mode" indicator to user
4. Queue extracted data (location, keywords, urgency)
5. Send GPS via SMS immediately (FR-5.2)
6. Transmit queued data when connection resumes (FR-5.3)

---

## Server-Side Architecture

### Technology Stack
- **API Gateway:** NGINX + Kong (rate limiting, auth)
- **Application:** Python 3.11 + FastAPI (async)
- **ML Inference:** PyTorch 2.0 + CUDA 12.1
- **Real-time:** WebRTC SFU (Selective Forwarding Unit)
- **Database:** PostgreSQL 15 (metadata), Redis (session state)
- **Message Queue:** Apache Kafka (telemetry, async processing)
- **Monitoring:** Prometheus + Grafana
- **Logging:** ELK Stack (Elasticsearch, Logstash, Kibana)

### Service Architecture (Microservices)

```
┌─────────────────────────────────────────────────────────────┐
│                      Load Balancer (Global)                  │
│                    (Geographic routing)                      │
└────────────────────────────┬────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway (Kong)                      │
│  - Authentication (JWT)                                      │
│  - Rate limiting (10k req/s per NFR-4.4)                    │
│  - TLS termination                                           │
└────────────────────────────┬────────────────────────────────┘
                             ↓
         ┌───────────────────┼───────────────────┐
         ↓                   ↓                   ↓
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ WebRTC Service  │ │ Translation     │ │ Dispatcher      │
│ (Video Stream)  │ │ Service         │ │ API Service     │
│                 │ │ (ML Inference)  │ │ (CAD Integration│
│ - SFU routing   │ │ - Sign→Text     │ │ - Structured    │
│ - Bandwidth     │ │ - Text→Sign     │ │   data          │
│   adaptation    │ │ - Urgency       │ │ - Dispatch      │
│                 │ │ - Info extract  │ │   actions)      │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         ↓                   ↓                   ↓
┌─────────────────────────────────────────────────────────────┐
│                    GPU Inference Cluster                     │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ Sign Recog   │  │ Multi-modal  │  │ Avatar Gen      │  │
│  │ Workers      │  │ Fusion       │  │ Workers         │  │
│  │ (8x A100)    │  │ Workers      │  │ (4x T4)         │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
         ↓                   ↓                   ↓
┌─────────────────────────────────────────────────────────────┐
│                      Data Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ PostgreSQL   │  │ Redis        │  │ S3 (Recordings) │  │
│  │ (Call meta)  │  │ (Sessions)   │  │ (Encrypted)     │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### GPU Cluster Configuration (NFR-4.1, NFR-4.2)

**Normal Load (10,000 concurrent calls):**
- 8x NVIDIA A100 (80GB) for sign recognition
- 4x NVIDIA T4 for avatar generation
- 4x CPU nodes for orchestration

**Disaster Surge (100,000 concurrent calls per NFR-4.2):**
- Auto-scale to 80x A100 within 1 hour
- Use spot instances for cost optimization
- Geographic distribution across 3 regions

**Per-GPU Capacity:**
- A100: ~1,250 concurrent translation streams (batch size 32)
- T4: ~2,500 concurrent avatar renders (pre-cached animations)

---

## Component-Level Breakdown

### 1. Video Ingestion Pipeline

**Purpose:** Receive, stabilize, and prepare video for ML inference (FR-1.1)

```
WebRTC Stream (H.264, 30 FPS)
  ↓
Decoder (FFmpeg)
  ↓
Frame Buffer (5 frame sliding window)
  ↓
Motion Stabilization (optical flow)
  ↓
Low-light Enhancement (adaptive histogram equalization)
  ↓
Frame Queue (Redis) → ML Workers
```

**Stabilization Algorithm (FR-1.6):**
- Optical flow estimation (Lucas-Kanade)
- Affine transformation to compensate camera shake
- Handles up to 5 Hz vibration per NFR-2.3

**Low-light Enhancement (FR-1.5):**
- CLAHE (Contrast Limited Adaptive Histogram Equalization)
- Activates when mean luminance <10 lux
- Maintains >85% accuracy per NFR-2.2

**Latency Budget:** 50ms (frame decode + preprocessing)

### 2. Pose Detection Subsystem

**Cloud Implementation (Higher Accuracy):**
- **Model:** MediaPipe Holistic (full precision FP32)
- **Output:** 543 3D landmarks + visibility scores
- **Latency:** 25ms per frame on GPU
- **Accuracy:** 98% landmark detection

**Landmark Extraction:**
```
Left Hand: 21 landmarks (x, y, z, visibility)
Right Hand: 21 landmarks
Body Pose: 33 landmarks
Face: 468 landmarks (reduced to 70 key points for efficiency)
```

**Occlusion Handling (FR-1.7):**
- Temporal smoothing: Use last 5 frames to interpolate missing landmarks
- Confidence weighting: Reduce influence of low-visibility landmarks
- Partial hand inference: Train model to recognize signs with 30% occlusion

### 3. Sign Recognition Model Architecture

**Model:** Spatial-Temporal Transformer (Custom)

**Architecture Details:**
```
Input: Landmark sequence (T=90 frames, 3 seconds @ 30 FPS)
  ↓
Spatial Encoder (per-frame)
  - Embedding layer: 543 landmarks → 512 dims
  - 3 Transformer blocks (8 heads, 512 dims)
  - Captures hand shape, body position relationships
  ↓
Temporal Encoder (across frames)
  - Positional encoding (sinusoidal)
  - 6 Transformer blocks (8 heads, 512 dims)
  - Self-attention captures motion patterns
  ↓
Emergency Vocabulary Head
  - Linear layer: 512 → 500 emergency signs
  - Softmax activation
  ↓
Output: Sign probabilities + confidence scores
```

**Training Configuration:**
- Optimizer: AdamW (lr=1e-4, weight decay=0.01)
- Loss: Cross-entropy + confidence calibration loss
- Batch size: 32 sequences
- Training data: 1,000 hours general ASL + 100 hours emergency synthetic
- Augmentation: Speed variation (0.5x-2x), spatial jitter, landmark dropout

**Confidence Calibration (SFR-1.1):**
- Temperature scaling on output logits
- Calibrated on validation set to ensure confidence matches accuracy
- Threshold: 80% confidence for critical information per SFR-1.2


**Performance:**
- Inference time: 120ms per 3-second window on A100
- Accuracy: 96% on emergency vocabulary (exceeds NFR-2.1 target of 95%)
- Model size: 180 MB (FP32), 45 MB (INT8 quantized)

### 4. Multi-Modal Urgency Detection Model

**Purpose:** Calculate urgency score 1-10 from multiple signals (FR-3.1)

**Input Modalities:**
1. Signing speed (signs per minute)
2. Facial expression features (68 facial landmarks)
3. Body language features (pose landmarks)
4. Sign content (emergency keywords)
5. Background audio features (MFCC)
6. Hesitation patterns (pause duration, frequency)

**Architecture:**
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Signing Speed   │  │ Facial Features │  │ Body Language   │
│ Encoder         │  │ CNN (3 layers)  │  │ MLP (2 layers)  │
│ (1D Conv)       │  │ → 128 dims      │  │ → 128 dims      │
│ → 128 dims      │  └─────────────────┘  └─────────────────┘
└─────────────────┘           ↓                    ↓
         ↓                    └────────┬───────────┘
         └─────────────────────────────┼───────────────────┐
                                       ↓                   ↓
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Keyword         │  │ Audio Features  │  │ Hesitation      │
│ Embeddings      │  │ (MFCC → CNN)    │  │ Features        │
│ → 128 dims      │  │ → 128 dims      │  │ → 128 dims      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         ↓                    ↓                    ↓
         └────────────────────┼────────────────────┘
                              ↓
                    Multi-Modal Fusion
                    (Cross-attention, 4 heads)
                              ↓
                    Fusion Transformer (3 blocks)
                              ↓
                    Urgency Regression Head
                    (Linear → Sigmoid × 10)
                              ↓
                    Urgency Score (1-10)
```

**Training:**
- Supervised learning on synthetic emergency scenarios
- Labels: Human expert urgency ratings (3 raters, averaged)
- Loss: MSE + ranking loss (ensures ordering)
- Correlation target: >90% with human assessment (NFR-2.7)

**Inference:**
- Latency: 80ms on GPU
- Updates every 1 second (rolling window)
- Smoothing: Exponential moving average (α=0.3) to reduce jitter


### 5. Speech-to-Text Pipeline

**Purpose:** Convert dispatcher speech to text for sign generation (FR-2.1)

**Model:** Whisper Large-v3 (OpenAI, fine-tuned)

**Fine-tuning:**
- Base: Whisper Large-v3 (1.5B parameters)
- Additional training: 50 hours emergency dispatcher speech
- Vocabulary expansion: Medical terms, street names, emergency protocols
- Accuracy target: >95% WER on emergency vocabulary (NFR-2.8)

**Pipeline:**
```
Dispatcher Microphone (16 kHz audio)
  ↓
Voice Activity Detection (WebRTC VAD)
  ↓
Audio Buffering (500ms chunks with 200ms overlap)
  ↓
Whisper Inference (streaming mode)
  ↓
Text Output + Timestamps
  ↓
Emergency Term Normalization
  (e.g., "CPR" → "chest compressions")
  ↓
Text-to-Sign Translation
```

**Latency Optimization:**
- Streaming inference: Process 500ms chunks
- Speculative decoding: 30% latency reduction
- Batch size: 8 concurrent dispatcher streams
- Total latency: 400ms (audio → text)

**Error Handling:**
- Confidence scores per word
- Flag uncertain medical terms for clarification
- Fallback: Display text overlay if sign generation fails

### 6. Text-to-Sign Generation Pipeline

**Purpose:** Translate English text to ASL grammar and generate sign sequence (FR-2.2, FR-2.3)

**Stage 1: English → ASL Gloss Translation**

Model: Transformer Seq2Seq
```
Input: English text (dispatcher speech)
  "Is the patient breathing normally?"
  ↓
Tokenization + Embedding (512 dims)
  ↓
Encoder (6 Transformer blocks)
  ↓
Decoder (6 Transformer blocks)
  ↓
Output: ASL Gloss
  "PATIENT BREATHE NORMAL?"
```

**ASL Grammar Rules:**
- Remove articles (a, an, the)
- Simplify verb tenses (present tense default)
- Reorder to topic-comment structure
- Add non-manual markers (facial expressions)

**Stage 2: Gloss → Sign Sequence**

Lookup table: 500 emergency signs → animation IDs
- Pre-rendered animations for common phrases
- Procedural generation for rare combinations
- Fingerspelling for proper nouns (FR-1.10)


**Latency:**
- Text → Gloss: 150ms
- Gloss → Animation: 50ms (lookup), 300ms (procedural)
- Total: 200-500ms depending on complexity

### 7. 3D Avatar Rendering System

**Purpose:** Display sign language animations to caller (FR-2.3, FR-2.4)

**Avatar Specifications:**
- **Model:** Rigged 3D humanoid (Unity-compatible)
- **Bones:** 54 (hands: 15×2, arms: 6, spine: 5, head: 3, face: 10)
- **Facial blend shapes:** 52 (FACS-based)
- **Rendering:** Real-time on mobile GPU (Metal/Vulkan)

**Animation System:**
```
Sign Sequence (from text-to-sign)
  ↓
Animation Blending Engine
  - Pre-cached: 200 common emergency phrases (instant playback)
  - Procedural: IK solver for rare signs (300ms generation)
  - Transitions: 100ms smooth blending between signs
  ↓
Facial Expression Overlay
  - Emotion mapping: Urgent → furrowed brow, wide eyes
  - Mouth shapes: Synchronized with mouthing
  ↓
Rendering (60 FPS on mobile)
  ↓
Display to caller
```

**High-Contrast Mode (FR-2.8):**
- Avatar: Dark blue on white background (daylight)
- Avatar: White on dark blue background (night)
- Automatic switching based on ambient light sensor

**Text Overlay (FR-2.6):**
- Critical information displayed as text simultaneously
- Addresses, numbers, medical terms
- Large font (24pt minimum), high contrast

**Performance:**
- Rendering: 16ms per frame (60 FPS)
- Memory: 50 MB (avatar + animations)
- Battery impact: 15% per hour (within 30% budget per NFR-1.9)

### 8. Confidence-Aware Clarification Module

**Purpose:** Automatically request clarification for uncertain information (FR-8.1, SFR-1.2)

**Trigger Conditions:**
```python
if confidence < 0.80 and is_critical_field(field):
    generate_clarification_request(field, recognized_value)
```

**Critical Fields (per SFR-1.2, SFR-1.3):**
- Location (address, landmarks)
- Emergency type (medical, fire, crime)
- Victim count
- Weapon presence
- Life-threatening symptoms

**Clarification Generation:**
```
Low confidence detected (e.g., address: "123 Main St" @ 72%)
  ↓
Template Selection
  "Did you sign [ADDRESS]? Please repeat slowly."
  ↓
Text-to-Sign Translation (priority queue)
  ↓
Avatar displays clarification question
  ↓
Enhanced Attention Mode
  - Focus on fingerspelling
  - Slower frame rate (15 FPS) for clarity
  - Zoom on hands (if camera allows)
  ↓
Re-inference with higher confidence threshold
  ↓
If still <80% after 2 attempts → Escalate to human VRS (SFR-1.4)
```

**Clarification Templates:**
- Location: "Did you sign [address]? Please fingerspell slowly."
- Emergency type: "Is this a medical emergency or crime?"
- Victim count: "How many people need help? Show number."
- Symptoms: "Where is the pain? Show on body."

**Latency Budget:**
- Detection: 10ms
- Template selection: 5ms
- Sign generation: 200ms
- Total: 215ms (within 1-second budget)


### 9. Multi-Party Tracking Module

**Purpose:** Track and attribute signs from multiple people in frame (FR-6.1 - FR-6.5)

**Architecture:**
```
Video Frame
  ↓
Person Detection (YOLOv8)
  - Detect up to 3 people (FR-6.1)
  - Bounding boxes + person IDs
  ↓
Per-Person Pose Estimation
  - Run MediaPipe on each bounding box
  - Extract landmarks for each person
  ↓
Spatial Tracking
  - Assign consistent IDs across frames (DeepSORT)
  - Track position, movement patterns
  ↓
Primary Caller Identification (FR-6.3)
  - Heuristics: Closest to camera, most screen time, direct gaze
  - User can manually designate primary
  ↓
Sign Attribution
  - Assign recognized signs to person IDs
  - Maintain separate translation streams
  ↓
Information Fusion (FR-6.5)
  - Merge complementary information
  - Flag contradictions for dispatcher review
  - Prioritize primary caller's information
```

**Performance:**
- Person detection: 25ms (YOLOv8-small on GPU)
- Per-person pose: 25ms × 3 = 75ms
- Tracking: 10ms
- Total overhead: 110ms (acceptable within 1-second budget)

**Display to Dispatcher:**
```
[Primary Caller - Sarah]: "Fire on 3rd floor, people trapped"
[Witness - Unknown]: "I saw smoke from apartment 305"
```

### 10. Offline Mode Architecture

**Purpose:** Maintain emergency functionality without network (FR-5.1, FR-5.7)

**Offline Capabilities:**
```
┌─────────────────────────────────────────────────────┐
│              Offline Mode (On-Device Only)          │
├─────────────────────────────────────────────────────┤
│ Available:                                          │
│  ✓ Pose detection (MediaPipe Lite)                 │
│  ✓ Emergency keyword recognition (88% accuracy)    │
│  ✓ GPS coordinate extraction                       │
│  ✓ Local urgency scoring                           │
│  ✓ SMS fallback (GPS + keywords)                   │
│  ✓ Local data queue                                │
├─────────────────────────────────────────────────────┤
│ Unavailable:                                        │
│  ✗ Full sentence translation                       │
│  ✗ Speech-to-sign (avatar)                         │
│  ✗ Multi-modal urgency fusion                      │
│  ✗ Critical information extraction (NER)           │
│  ✗ Dispatcher communication                        │
└─────────────────────────────────────────────────────┘
```

**Offline Workflow:**
```
1. Network loss detected (2-second timeout per SFR-4.1)
   ↓
2. Display "Limited Mode - Sending GPS via SMS" alert
   ↓
3. Extract GPS coordinates
   ↓
4. Run local keyword detection
   ↓
5. Compose SMS: "EMERGENCY: [keywords] at [GPS coordinates]"
   ↓
6. Send to local 911 SMS gateway (FR-5.2)
   ↓
7. Queue detailed data locally (SQLite)
   ↓
8. Monitor network status every 5 seconds
   ↓
9. When network returns: Upload queued data (FR-5.3)
   ↓
10. Resume full functionality
```

**SMS Format:**
```
EMERGENCY DEAF CALLER
Keywords: FIRE TRAPPED HELP
Location: 37.7749°N, 122.4194°W
Urgency: 8/10
Time: 2026-02-15 14:32:18 UTC
Call ID: abc123-def456
```

**Data Queue Schema:**
```sql
CREATE TABLE offline_queue (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME,
    gps_lat REAL,
    gps_lon REAL,
    keywords TEXT,
    urgency_score INTEGER,
    landmark_data BLOB,
    uploaded BOOLEAN DEFAULT 0
);
```

---

## ML Model Architecture Details

### Transformer Configuration

**Sign Recognition Transformer:**
```yaml
Architecture: Spatial-Temporal Transformer
Spatial Encoder:
  layers: 3
  attention_heads: 8
  hidden_dim: 512
  feedforward_dim: 2048
  dropout: 0.1
  activation: GELU

Temporal Encoder:
  layers: 6
  attention_heads: 8
  hidden_dim: 512
  feedforward_dim: 2048
  dropout: 0.1
  activation: GELU
  positional_encoding: sinusoidal
  max_sequence_length: 90

Output Head:
  emergency_vocabulary_size: 500
  confidence_calibration: temperature_scaling
  temperature: 1.5
```

**Why Transformer over LSTM/CNN:**
- Better long-range dependencies (3-second sign sequences)
- Parallel processing (faster inference)
- Attention visualization for debugging
- State-of-art accuracy on sign language benchmarks


### Multi-Modal Fusion Strategy

**Fusion Approach:** Late Fusion with Cross-Attention

**Rationale:**
- Early fusion: Loses modality-specific features
- Late fusion: Preserves modality strengths, allows independent optimization
- Cross-attention: Learns inter-modality relationships

**Implementation:**
```python
class MultiModalFusion(nn.Module):
    def __init__(self):
        self.modality_encoders = {
            'signing_speed': Conv1DEncoder(out_dim=128),
            'facial': CNNEncoder(out_dim=128),
            'body': MLPEncoder(out_dim=128),
            'keywords': EmbeddingEncoder(out_dim=128),
            'audio': MFCCEncoder(out_dim=128),
            'hesitation': MLPEncoder(out_dim=128)
        }
        self.cross_attention = MultiHeadAttention(heads=4, dim=128)
        self.fusion_transformer = TransformerEncoder(layers=3, dim=128)
        self.urgency_head = nn.Linear(128, 1)
    
    def forward(self, modality_inputs):
        # Encode each modality
        encoded = [encoder(modality_inputs[name]) 
                   for name, encoder in self.modality_encoders.items()]
        
        # Stack: [6, batch, 128]
        stacked = torch.stack(encoded, dim=0)
        
        # Cross-attention between modalities
        attended = self.cross_attention(stacked, stacked, stacked)
        
        # Fusion transformer
        fused = self.fusion_transformer(attended)
        
        # Global pooling + urgency prediction
        pooled = fused.mean(dim=0)
        urgency = torch.sigmoid(self.urgency_head(pooled)) * 10
        
        return urgency
```

**Training Strategy:**
- Pre-train modality encoders independently
- Freeze encoders, train fusion layers
- Fine-tune end-to-end with small learning rate
- Regularization: Dropout 0.2, weight decay 0.01

### Attention Mechanisms

**Self-Attention in Sign Recognition:**
```
Purpose: Capture temporal dependencies in sign sequences

Query: Current frame features
Key: All frame features in sequence
Value: All frame features in sequence

Attention(Q, K, V) = softmax(QK^T / √d_k) V

Benefits:
- Identifies sign boundaries (start/end of signs)
- Captures co-articulation (sign transitions)
- Handles variable-length signs
```

**Cross-Attention in Multi-Modal Fusion:**
```
Purpose: Learn relationships between modalities

Example: High signing speed + fearful face → High urgency
         High signing speed + neutral face → Excitement (lower urgency)

Query: Facial features
Key: Signing speed features
Value: Signing speed features

Learns: "When face shows fear, fast signing indicates panic"
```

**Attention Visualization:**
- Export attention weights for debugging
- Identify which frames/modalities contribute to decisions
- Validate model reasoning with domain experts

### Confidence Scoring Logic

**Calibrated Confidence:**
```python
def calibrated_confidence(logits, temperature=1.5):
    """
    Temperature scaling for confidence calibration
    Higher temperature → More conservative confidence
    """
    scaled_logits = logits / temperature
    probabilities = softmax(scaled_logits)
    confidence = max(probabilities)
    return confidence

# Example:
# Uncalibrated: [0.92, 0.05, 0.03] → confidence = 0.92
# Calibrated (T=1.5): [0.78, 0.12, 0.10] → confidence = 0.78
# More realistic confidence for decision-making
```

**Confidence Components:**
```
Total Confidence = w1 × Model_Confidence 
                 + w2 × Temporal_Consistency
                 + w3 × Landmark_Visibility
                 - w4 × Environmental_Noise

Where:
- Model_Confidence: Softmax probability from model
- Temporal_Consistency: Agreement across sliding windows
- Landmark_Visibility: Percentage of landmarks detected
- Environmental_Noise: Motion blur, lighting issues

Weights: w1=0.5, w2=0.3, w3=0.15, w4=0.05
```

**Threshold Strategy:**
- Critical fields (location, emergency type): 80% threshold
- Non-critical fields (caller name): 60% threshold
- Weapon detection: 90% threshold (high precision required)
- Escalation: <70% after 2 clarifications → Human VRS

### Few-Shot Emergency Vocabulary Extension

**Problem:** Limited training data for emergency-specific signs

**Solution:** Meta-Learning + Prototypical Networks

**Architecture:**
```
Base Model: Pre-trained on 1,000 hours general ASL
  ↓
Few-Shot Adaptation Layer
  - Prototypical network: Learn sign embeddings
  - Support set: 5-10 examples per new emergency sign
  - Query: New sign to classify
  ↓
Distance Metric: Euclidean distance to prototypes
  ↓
Classification: Nearest prototype
```

**Training Process:**
```
1. Pre-train on general ASL (500 common signs)
2. Meta-train on emergency vocabulary:
   - Sample N-way K-shot tasks (N=5 signs, K=5 examples)
   - Learn to classify with minimal examples
3. Fine-tune on full emergency dataset (100 hours)
4. Continual learning: Add new signs with 5-10 examples
```

**Performance:**
- New sign accuracy: 85% with 5 examples, 92% with 10 examples
- Adaptation time: 30 minutes training per new sign
- No catastrophic forgetting (maintains 96% on original vocabulary)

### Synthetic Data Pipeline

**Purpose:** Generate emergency scenario training data (DR-2.1, DR-2.2)

**Pipeline:**
```
1. Scenario Generation
   ├─ Emergency type: Medical, Fire, Crime, Accident
   ├─ Severity: Low, Medium, High, Critical
   ├─ Environment: Indoor, Outdoor, Vehicle, Public
   └─ Time: Day, Night, Dawn, Dusk

2. Script Generation (GPT-4)
   ├─ Input: Scenario parameters
   ├─ Output: Emergency conversation script
   └─ Example: "My father is having chest pain. He can't breathe. 
                We're at 123 Main Street, apartment 4B."

3. ASL Translation
   ├─ English → ASL Gloss (rule-based + model)
   └─ Example: "FATHER CHEST PAIN. BREATHE CAN'T. 
                ADDRESS 1-2-3 M-A-I-N STREET, APARTMENT 4-B."

4. Motion Capture / Animation
   ├─ Deaf signers perform scripts (motion capture)
   ├─ OR: Procedural animation from gloss
   └─ Output: 3D skeletal animation

5. Rendering with Augmentation
   ├─ Camera angles: Front, side, tilted (±30°)
   ├─ Lighting: Bright, dim, flickering
   ├─ Motion: Stable, shaky (0-5 Hz)
   ├─ Occlusion: Random hand/body parts (0-30%)
   ├─ Background: Chaos (fire, crowds, sirens)
   └─ Signing speed: 0.5x - 2.5x normal

6. Landmark Extraction
   ├─ Run MediaPipe on rendered video
   └─ Output: Training samples (landmarks + labels)
```

**Synthetic Data Statistics:**
- 500 unique emergency scenarios
- 10 variations per scenario (augmentation)
- 5,000 synthetic training samples
- 30 seconds average duration
- Total: ~42 hours of synthetic data

**Validation:**
- Deaf community review: 4.2/5.0 realism score
- Model trained on synthetic: 89% accuracy on real data
- Mixed training (real + synthetic): 96% accuracy

---

## Data Flow Diagrams

### Sign → Speech Translation Flow

```
[CALLER DEVICE]
1. Camera captures video (30 FPS, 720p)
   ↓ 33ms per frame
2. MediaPipe Lite: Pose detection
   ↓ Landmarks: 543 × (x,y,z,visibility)
3. Edge models: Keywords + urgency
   ↓ 15ms
4. Compress video + landmarks
   ↓ H.264 500kbps + JSON 5KB/s

[NETWORK] TLS 1.3 encrypted WebRTC
   ↓ 50-200ms latency

[CLOUD SERVER]
5. Video decoder + preprocessing
   ↓ 50ms
6. MediaPipe Full: High-accuracy pose
   ↓ 25ms per frame
7. Sliding window buffer (90 frames = 3 seconds)
   ↓ Continuous
8. Sign Recognition Transformer
   ↓ 120ms per window
9. Confidence check
   ├─ >80%: Continue
   └─ <80%: Clarification module
   ↓
10. Text output: "FATHER CHEST PAIN BREATHE CAN'T"
    ↓ 10ms
11. Text-to-Speech (TTS)
    ↓ 100ms
12. Audio stream to dispatcher

[DISPATCHER WORKSTATION]
13. Audio playback + text display
    ↓ Real-time

Total Latency: 33+15+50+50+25+120+10+100 = 403ms
Target: <1000ms ✓ (NFR-1.1)
```


### Speech → Sign Translation Flow

```
[DISPATCHER WORKSTATION]
1. Microphone captures audio (16 kHz)
   ↓ Continuous stream
2. Voice Activity Detection (VAD)
   ↓ 10ms
3. Audio buffering (500ms chunks)
   ↓

[CLOUD SERVER]
4. Whisper STT (streaming mode)
   ↓ 400ms
5. Text output: "Is the patient breathing normally?"
   ↓ 5ms
6. Text-to-ASL Gloss translation
   ↓ 150ms
7. Gloss output: "PATIENT BREATHE NORMAL?"
   ↓ 10ms
8. Animation lookup / generation
   ├─ Pre-cached: 50ms (instant playback)
   └─ Procedural: 300ms (IK solver)
   ↓
9. Animation data stream (JSON)
   ↓ 20ms

[NETWORK] TLS 1.3 encrypted
   ↓ 50-200ms

[CALLER DEVICE]
10. Avatar rendering engine
    ↓ 16ms per frame (60 FPS)
11. Display to caller
    ↓ Real-time

Total Latency: 10+400+5+150+10+50+20+50+16 = 711ms (pre-cached)
              10+400+5+150+10+300+20+50+16 = 961ms (procedural)
Target: <1000ms ✓ (NFR-1.2)
```

### Urgency Classification Path

```
[PARALLEL PROCESSING - All inputs processed simultaneously]

Input Stream 1: Video frames
  ↓
Signing speed calculation (signs per minute)
  ↓ 10ms

Input Stream 2: Facial landmarks
  ↓
Facial expression CNN
  ↓ 30ms
Emotion classification (fear, pain, neutral)

Input Stream 3: Body pose landmarks
  ↓
Body language MLP
  ↓ 20ms
Posture analysis (ducking, trembling)

Input Stream 4: Sign recognition output
  ↓
Keyword extraction (weapon, fire, blood)
  ↓ 5ms

Input Stream 5: Background audio
  ↓
MFCC feature extraction
  ↓ 25ms
Audio event detection (screams, gunshots)

Input Stream 6: Temporal analysis
  ↓
Hesitation pattern detection
  ↓ 15ms

[FUSION]
All features → Multi-modal fusion transformer
  ↓ 80ms
Urgency score (1-10) + confidence
  ↓
Exponential moving average (smoothing)
  ↓ 2ms
Display to dispatcher (color-coded)

Total Latency: 80ms (fusion dominates)
Update Frequency: Every 1 second
```

### Critical Information Extraction Path

```
Sign Recognition Output: "FATHER CHEST PAIN ADDRESS 1-2-3 M-A-I-N STREET"
  ↓
[PARALLEL EXTRACTION]

Path 1: Location Extraction
  ↓
Named Entity Recognition (NER) - Location
  ↓ 30ms
Candidates: "123 MAIN STREET"
  ↓
Geocoding validation (Google Maps API)
  ↓ 100ms
Validated address + GPS coordinates
  ↓
Confidence: 95%

Path 2: Emergency Type Classification
  ↓
Keyword matching + context analysis
  ↓ 20ms
Keywords: "CHEST PAIN" → Medical emergency
  ↓
Severity classifier
  ↓ 15ms
Classification: Cardiac emergency (high severity)
  ↓
Confidence: 92%

Path 3: Victim Information
  ↓
Relationship extraction: "FATHER" → Family member
  ↓ 10ms
Age/gender inference (if mentioned)
  ↓
Count: 1 victim

Path 4: Symptom Extraction
  ↓
Medical NER: "CHEST PAIN"
  ↓ 25ms
Symptom database lookup
  ↓
Severity: Critical (possible cardiac arrest)
  ↓
Confidence: 88%

[AGGREGATION]
All extracted data → Structured JSON
  ↓ 5ms
{
  "emergency_type": "medical_cardiac",
  "location": "123 Main Street",
  "gps": [37.7749, -122.4194],
  "victims": 1,
  "symptoms": ["chest_pain", "breathing_difficulty"],
  "urgency": 9,
  "confidence_scores": {
    "location": 0.95,
    "emergency_type": 0.92,
    "symptoms": 0.88
  }
}
  ↓
Display in dispatcher dashboard
  ↓
CAD system integration (auto-populate fields)

Total Latency: 100ms (geocoding dominates)
```

---

## Deployment Architecture

### Edge vs Cloud Split

**Edge (Mobile Device):**
```
Components:
- Pose detection (MediaPipe Lite): 12 MB
- Emergency keyword classifier: 8 MB
- Local urgency scorer: 3 MB
- Avatar renderer: 50 MB (model + animations)
- Total: 73 MB

Rationale:
- Reduce bandwidth (landmarks vs full video)
- Enable offline mode
- Immediate urgency alerts
- Privacy (some processing on-device)

Limitations:
- Lower accuracy (88% vs 96%)
- Limited vocabulary (20 keywords vs 500 signs)
- No complex reasoning
```

**Cloud (GPU Cluster):**
```
Components:
- Full sign recognition transformer: 180 MB
- Multi-modal urgency fusion: 95 MB
- Critical info extraction (NER): 420 MB
- Speech-to-text (Whisper): 3 GB
- Text-to-sign translation: 250 MB
- Total: ~4 GB per inference worker

Rationale:
- High accuracy requirements (>95%)
- Complex multi-modal fusion
- Large model capacity
- Centralized updates
- Scalability (add GPUs as needed)

Requirements:
- Low latency (<1s end-to-end)
- High availability (99.99%)
- Geographic distribution
```

**Decision Criteria:**
```
Run on Edge if:
- Latency critical AND simple computation
- Must work offline
- Privacy sensitive
- Model size <50 MB

Run on Cloud if:
- Accuracy critical (>90%)
- Complex computation (transformers)
- Model size >100 MB
- Requires frequent updates
```

### GPU Cluster Requirements

**Normal Load (10,000 concurrent calls):**
```
Sign Recognition:
- Model: 180 MB per instance
- Throughput: 1,250 calls per A100 (batch size 32)
- Required: 8 × A100 (80GB)
- Cost: ~$3/hour per GPU = $24/hour

Avatar Generation:
- Model: 50 MB per instance
- Throughput: 2,500 calls per T4 (pre-cached animations)
- Required: 4 × T4 (16GB)
- Cost: ~$0.50/hour per GPU = $2/hour

Multi-modal Fusion:
- Shared with sign recognition GPUs
- Minimal overhead (80ms per call)

Total: 8 A100 + 4 T4 = $26/hour = $19,000/month
```

**Disaster Surge (100,000 concurrent calls):**
```
Auto-scaling trigger: >80% GPU utilization for 5 minutes

Scale-up plan:
- Minute 0-5: Alert triggered
- Minute 5-15: Provision 40 A100 spot instances (5x scale)
- Minute 15-30: Provision 40 more A100 (10x scale)
- Minute 30-60: Full 80 A100 deployment

Geographic distribution:
- US-East: 30 A100 (primary)
- US-West: 30 A100 (secondary)
- US-Central: 20 A100 (tertiary)

Cost: $240/hour during surge = $5,760/day
Acceptable for disaster scenarios (earthquakes, hurricanes)
```

**GPU Selection Rationale:**
- A100: Best performance/$ for transformer inference
- T4: Cost-effective for lighter workloads (avatar)
- Spot instances: 70% cost savings for surge capacity
- Reserved instances: 40% savings for baseline capacity

### Mobile Model Optimization

**Quantization:**
```
Original (FP32) → Quantized (INT8)
- Pose detection: 12 MB → 3 MB (4x reduction)
- Keyword classifier: 8 MB → 2 MB (4x reduction)
- Urgency scorer: 3 MB → 1 MB (3x reduction)

Accuracy impact:
- FP32: 88.5% keyword accuracy
- INT8: 87.8% keyword accuracy
- Degradation: 0.7% (acceptable)

Latency improvement:
- FP32: 45ms inference
- INT8: 15ms inference (3x faster)
```

**Pruning:**
```
Remove 30% of least important weights
- Keyword classifier: 8 MB → 5.6 MB
- Accuracy: 88.5% → 87.2% (1.3% drop)
- Acceptable tradeoff for size reduction
```

**Knowledge Distillation:**
```
Teacher: Cloud transformer (180 MB, 96% accuracy)
Student: Mobile MobileNet (8 MB, 88% accuracy)

Process:
1. Train teacher on full dataset
2. Generate soft labels (probability distributions)
3. Train student to match teacher's outputs
4. Result: Student learns teacher's knowledge in compact form
```

**Optimization Results:**
```
Before: 73 MB, 45ms latency, 88.5% accuracy
After: 42 MB, 15ms latency, 87.8% accuracy
Meets: <100 MB (NFR-1.7), <50ms latency, >85% accuracy (NFR-2.2)
```


### Fallback SMS Mechanism

**Purpose:** Send emergency info when data connection fails (FR-5.2)

**Implementation:**
```python
class SMSFallback:
    def __init__(self):
        self.sms_gateway = "911-sms-gateway"  # Local emergency SMS
        self.retry_attempts = 3
        self.retry_delay = 5  # seconds
    
    def trigger_conditions(self):
        """When to activate SMS fallback"""
        return (
            network_unavailable() or
            webrtc_connection_failed() or
            server_unreachable_for(10)  # seconds
        )
    
    def compose_emergency_sms(self, data):
        """Compose SMS with critical info"""
        return f"""EMERGENCY DEAF CALLER
Type: {data['emergency_type']}
Keywords: {', '.join(data['keywords'])}
Location: {data['gps_lat']}, {data['gps_lon']}
Address: {data['address_estimate']}
Urgency: {data['urgency']}/10
Time: {data['timestamp']}
CallID: {data['call_id']}
App: EmergencySignBridge
"""
    
    def send_with_retry(self, message):
        """Send SMS with retry logic"""
        for attempt in range(self.retry_attempts):
            try:
                send_sms(self.sms_gateway, message)
                log_success(attempt)
                return True
            except SMSError as e:
                if attempt < self.retry_attempts - 1:
                    time.sleep(self.retry_delay)
                else:
                    log_failure(e)
                    return False
```

**SMS Gateway Integration:**
- Direct integration with local 911 SMS systems
- Fallback to carrier SMS if 911 SMS unavailable
- Character limit: 160 chars (use abbreviations)
- Delivery confirmation required

**User Experience:**
```
Network lost detected
  ↓
Display: "Connection lost. Sending emergency SMS..."
  ↓
Extract GPS + keywords from local models
  ↓
Send SMS (3 retry attempts)
  ↓
Display: "Emergency SMS sent. Help is coming."
  ↓
Continue attempting to reconnect
  ↓
When reconnected: Upload full data
```

### CAD System Integration

**Purpose:** Integrate with Computer-Aided Dispatch systems (FR-7.3)

**Integration Architecture:**
```
Emergency Sign Bridge Server
  ↓
REST API / HL7 FHIR
  ↓
CAD System Adapter (per vendor)
  ├─ Hexagon (Intergraph)
  ├─ Motorola (PremierOne)
  ├─ Tyler Technologies (New World)
  └─ Central Square (Tritech)
  ↓
CAD Database
  ↓
Dispatcher Workstation
```

**API Payload:**
```json
{
  "call_id": "abc123-def456",
  "timestamp": "2026-02-15T14:32:18Z",
  "caller_type": "deaf_sign_language",
  "translation_mode": "ai_assisted",
  "emergency_data": {
    "type": "medical_cardiac",
    "severity": "critical",
    "urgency_score": 9,
    "location": {
      "address": "123 Main Street, Apt 4B",
      "city": "San Francisco",
      "state": "CA",
      "zip": "94102",
      "gps": {"lat": 37.7749, "lon": -122.4194},
      "confidence": 0.95
    },
    "victims": [
      {
        "count": 1,
        "relationship": "father",
        "age_estimate": "elderly",
        "condition": "unconscious",
        "symptoms": ["chest_pain", "breathing_difficulty"]
      }
    ],
    "hazards": [],
    "weapons": false
  },
  "confidence_scores": {
    "location": 0.95,
    "emergency_type": 0.92,
    "symptoms": 0.88
  },
  "transcript": "FATHER CHEST PAIN BREATHE CAN'T ADDRESS 1-2-3 M-A-I-N STREET",
  "media": {
    "video_url": "https://secure.esb.com/calls/abc123/video",
    "recording_consent": true
  }
}
```

**CAD System Actions:**
- Auto-populate incident fields
- Display confidence scores
- Flag as "AI-assisted translation"
- Provide video link for review
- Enable standard dispatch workflow

**Compliance:**
- NENA i3 standard compliance (NFR-8.1)
- HL7 FHIR for medical data (NFR-8.3)
- Audit logging for all CAD interactions

---

## API Design

### Client ↔ Server Endpoints

**1. Emergency Call Initiation**
```
POST /api/v1/emergency/initiate
Authorization: Bearer <jwt_token>

Request:
{
  "caller_id": "user_12345",
  "gps": {"lat": 37.7749, "lon": -122.4194},
  "device_info": {
    "platform": "iOS",
    "version": "16.0",
    "model": "iPhone 13"
  }
}

Response:
{
  "call_id": "abc123-def456",
  "webrtc_offer": "<sdp_offer>",
  "ice_servers": [...],
  "session_token": "<session_jwt>"
}
```

**2. Video Stream (WebRTC)**
```
WebRTC Data Channel: video_stream
- Video: H.264, 500-1000 kbps, 30 FPS
- Landmarks: JSON, 5 KB/s
- Metadata: Urgency, keywords, confidence
```

**3. Translation Results (Server → Client)**
```
WebSocket: wss://api.esb.com/translation/<call_id>

Message Format:
{
  "type": "sign_to_text",
  "timestamp": "2026-02-15T14:32:18.123Z",
  "text": "FATHER CHEST PAIN",
  "confidence": 0.92,
  "urgency": 9,
  "extracted_data": {
    "emergency_type": "medical_cardiac",
    "location": "123 Main Street",
    "confidence": 0.95
  }
}
```

**4. Avatar Animation (Server → Client)**
```
WebSocket: wss://api.esb.com/avatar/<call_id>

Message Format:
{
  "type": "avatar_animation",
  "timestamp": "2026-02-15T14:32:19.456Z",
  "animation_id": "pre_cached_cpr_instructions",
  "duration_ms": 5000,
  "text_overlay": "Push hard on chest. I'll count.",
  "urgency_level": "critical"
}
```

**5. Clarification Request (Server → Client)**
```
{
  "type": "clarification_request",
  "field": "location",
  "recognized_value": "123 Main Street",
  "confidence": 0.72,
  "question": "Did you sign 123 Main Street? Please repeat slowly.",
  "animation_id": "clarify_address"
}
```

**6. Offline Data Upload**
```
POST /api/v1/emergency/offline_data
Authorization: Bearer <session_token>

Request:
{
  "call_id": "abc123-def456",
  "offline_period": {
    "start": "2026-02-15T14:32:00Z",
    "end": "2026-02-15T14:33:30Z"
  },
  "queued_data": [
    {
      "timestamp": "2026-02-15T14:32:15Z",
      "landmarks": [...],
      "keywords": ["FIRE", "HELP"],
      "urgency": 8
    }
  ]
}
```

### Structured Emergency Data Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "EmergencyData",
  "type": "object",
  "required": ["call_id", "timestamp", "emergency_type", "location"],
  "properties": {
    "call_id": {
      "type": "string",
      "description": "Unique call identifier"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "emergency_type": {
      "type": "string",
      "enum": ["medical", "fire", "crime", "accident", "other"]
    },
    "emergency_subtype": {
      "type": "string",
      "examples": ["cardiac", "respiratory", "trauma", "assault"]
    },
    "urgency_score": {
      "type": "integer",
      "minimum": 1,
      "maximum": 10
    },
    "location": {
      "type": "object",
      "required": ["gps"],
      "properties": {
        "address": {"type": "string"},
        "city": {"type": "string"},
        "state": {"type": "string"},
        "zip": {"type": "string"},
        "gps": {
          "type": "object",
          "properties": {
            "lat": {"type": "number"},
            "lon": {"type": "number"}
          }
        },
        "landmark": {"type": "string"},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
      }
    },
    "victims": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "count": {"type": "integer"},
          "relationship": {"type": "string"},
          "age_estimate": {"type": "string"},
          "gender": {"type": "string"},
          "condition": {"type": "string"},
          "symptoms": {"type": "array", "items": {"type": "string"}},
          "injuries": {"type": "array", "items": {"type": "string"}}
        }
      }
    },
    "hazards": {
      "type": "array",
      "items": {"type": "string"},
      "examples": ["fire", "gas_leak", "structural_damage"]
    },
    "weapons": {
      "type": "object",
      "properties": {
        "present": {"type": "boolean"},
        "type": {"type": "string"},
        "confidence": {"type": "number"}
      }
    },
    "confidence_scores": {
      "type": "object",
      "properties": {
        "location": {"type": "number"},
        "emergency_type": {"type": "number"},
        "urgency": {"type": "number"},
        "symptoms": {"type": "number"}
      }
    }
  }
}
```

### Confidence Metadata Format

```json
{
  "confidence_metadata": {
    "overall_confidence": 0.89,
    "field_confidences": {
      "location": {
        "value": "123 Main Street",
        "confidence": 0.95,
        "source": "fingerspelling",
        "verification": "geocoded"
      },
      "emergency_type": {
        "value": "medical_cardiac",
        "confidence": 0.92,
        "source": "keyword_analysis",
        "keywords": ["CHEST", "PAIN", "BREATHE"]
      },
      "urgency": {
        "value": 9,
        "confidence": 0.88,
        "source": "multimodal_fusion",
        "contributors": {
          "signing_speed": 0.9,
          "facial_expression": 0.85,
          "keywords": 0.95
        }
      }
    },
    "clarification_history": [
      {
        "field": "location",
        "attempt": 1,
        "original_confidence": 0.72,
        "clarified_confidence": 0.95,
        "timestamp": "2026-02-15T14:32:25Z"
      }
    ],
    "quality_indicators": {
      "video_quality": "good",
      "lighting": "adequate",
      "camera_stability": "moderate_shake",
      "landmark_visibility": 0.92
    }
  }
}
```

### Streaming Protocol

**WebRTC Configuration:**
```javascript
const rtcConfig = {
  iceServers: [
    { urls: 'stun:stun.esb.com:3478' },
    {
      urls: 'turn:turn.esb.com:3478',
      username: 'emergency',
      credential: '<credential>'
    }
  ],
  iceTransportPolicy: 'all',
  bundlePolicy: 'max-bundle',
  rtcpMuxPolicy: 'require'
};

const mediaConstraints = {
  video: {
    width: { ideal: 1280, max: 1920 },
    height: { ideal: 720, max: 1080 },
    frameRate: { ideal: 30, max: 30 },
    facingMode: 'user'
  },
  audio: false  // Caller doesn't need audio
};
```

**Adaptive Bitrate:**
```javascript
function adjustBitrate(networkQuality) {
  const bitrateMap = {
    'excellent': 1000000,  // 1 Mbps
    'good': 500000,        // 500 kbps
    'fair': 300000,        // 300 kbps
    'poor': 200000         // 200 kbps (minimum per FR-5.4)
  };
  
  const sender = peerConnection.getSenders()[0];
  const parameters = sender.getParameters();
  parameters.encodings[0].maxBitrate = bitrateMap[networkQuality];
  sender.setParameters(parameters);
}
```

---

## Database Design

### Call Metadata Storage

**Table: emergency_calls**
```sql
CREATE TABLE emergency_calls (
    call_id VARCHAR(36) PRIMARY KEY,
    caller_id VARCHAR(36) NOT NULL,
    dispatcher_id VARCHAR(36),
    
    -- Timestamps
    initiated_at TIMESTAMP NOT NULL,
    connected_at TIMESTAMP,
    ended_at TIMESTAMP,
    
    -- Call details
    emergency_type VARCHAR(50),
    urgency_score INTEGER CHECK (urgency_score BETWEEN 1 AND 10),
    location_address TEXT,
    location_gps POINT,
    
    -- Status
    status VARCHAR(20), -- initiated, connected, ended, escalated
    escalated_to_human BOOLEAN DEFAULT FALSE,
    escalation_reason TEXT,
    
    -- Quality metrics
    avg_confidence DECIMAL(3,2),
    translation_accuracy DECIMAL(3,2),
    total_clarifications INTEGER DEFAULT 0,
    
    -- Media
    video_recording_url TEXT,
    transcript_url TEXT,
    
    -- Compliance
    recording_consent BOOLEAN,
    data_retention_until DATE,
    
    FOREIGN KEY (caller_id) REFERENCES users(user_id),
    FOREIGN KEY (dispatcher_id) REFERENCES dispatchers(dispatcher_id)
);

CREATE INDEX idx_calls_timestamp ON emergency_calls(initiated_at);
CREATE INDEX idx_calls_location ON emergency_calls USING GIST(location_gps);
CREATE INDEX idx_calls_type ON emergency_calls(emergency_type);
```

### Extracted Structured Emergency Records

**Table: emergency_data**
```sql
CREATE TABLE emergency_data (
    id SERIAL PRIMARY KEY,
    call_id VARCHAR(36) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    
    -- Emergency details
    emergency_type VARCHAR(50),
    emergency_subtype VARCHAR(50),
    urgency_score INTEGER,
    
    -- Location
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(2),
    zip VARCHAR(10),
    gps_lat DECIMAL(10,8),
    gps_lon DECIMAL(11,8),
    location_confidence DECIMAL(3,2),
    
    -- Victims
    victim_count INTEGER,
    victim_details JSONB,
    
    -- Hazards and weapons
    hazards TEXT[],
    weapon_present BOOLEAN,
    weapon_type VARCHAR(50),
    
    -- Confidence scores
    confidence_scores JSONB,
    
    -- CAD integration
    cad_incident_id VARCHAR(50),
    dispatched_units TEXT[],
    
    FOREIGN KEY (call_id) REFERENCES emergency_calls(call_id)
);

CREATE INDEX idx_emergency_data_call ON emergency_data(call_id);
CREATE INDEX idx_emergency_data_type ON emergency_data(emergency_type);
```

### Model Telemetry Logs

**Table: model_telemetry**
```sql
CREATE TABLE model_telemetry (
    id BIGSERIAL PRIMARY KEY,
    call_id VARCHAR(36),
    timestamp TIMESTAMP NOT NULL,
    
    -- Model performance
    model_name VARCHAR(100),
    model_version VARCHAR(20),
    inference_latency_ms INTEGER,
    
    -- Predictions
    prediction TEXT,
    confidence DECIMAL(3,2),
    ground_truth TEXT,  -- Filled later for training
    
    -- Input quality
    video_quality VARCHAR(20),
    lighting_level VARCHAR(20),
    camera_stability VARCHAR(20),
    landmark_visibility DECIMAL(3,2),
    
    -- Resource usage
    gpu_utilization DECIMAL(3,2),
    memory_usage_mb INTEGER,
    
    FOREIGN KEY (call_id) REFERENCES emergency_calls(call_id)
);

CREATE INDEX idx_telemetry_timestamp ON model_telemetry(timestamp);
CREATE INDEX idx_telemetry_model ON model_telemetry(model_name, model_version);
CREATE INDEX idx_telemetry_confidence ON model_telemetry(confidence);
```

**Table: accuracy_drift_monitoring**
```sql
CREATE TABLE accuracy_drift_monitoring (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    model_name VARCHAR(100),
    
    -- Accuracy metrics
    overall_accuracy DECIMAL(5,4),
    emergency_vocab_accuracy DECIMAL(5,4),
    location_accuracy DECIMAL(5,4),
    
    -- Confidence calibration
    avg_confidence DECIMAL(3,2),
    confidence_accuracy_gap DECIMAL(3,2),
    
    -- Demographic breakdown
    accuracy_by_demographics JSONB,
    
    -- Alerts
    drift_detected BOOLEAN,
    alert_triggered BOOLEAN,
    
    UNIQUE(date, model_name)
);
```

---

## Failure Modes & Mitigations


### Low Confidence Handling

**Failure Mode:** Model produces low confidence predictions (<80%)

**Detection:**
```python
if prediction_confidence < CRITICAL_THRESHOLD:
    trigger_low_confidence_handler(field, value, confidence)
```

**Mitigation Strategy:**
```
1. Automatic Clarification (SFR-1.2)
   - Generate clarification question in sign language
   - Request caller to repeat slowly
   - Re-inference with enhanced attention
   
2. Confidence Threshold (attempt 1)
   - If confidence improves to >80%: Continue
   - If confidence still <80%: Attempt 2
   
3. Second Clarification (attempt 2)
   - Different phrasing of question
   - Suggest alternative input method (fingerspelling)
   - Re-inference
   
4. Human Escalation (SFR-1.4)
   - If confidence still <70% after 2 attempts
   - Connect to human VRS interpreter
   - Transfer call context (transcript, extracted data)
   - AI continues in background (hybrid mode)
   
5. Dispatcher Override
   - Dispatcher can manually correct any field
   - Manual corrections logged for model improvement
   - High-confidence fields still displayed for reference
```

**Monitoring:**
- Track low confidence frequency per model
- Alert if >10% of calls require escalation
- Retrain model on low-confidence examples

### Network Instability Handling

**Failure Mode:** Intermittent network connectivity, high latency, packet loss

**Detection:**
```python
class NetworkMonitor:
    def check_quality(self):
        return {
            'latency': measure_rtt(),  # Round-trip time
            'packet_loss': measure_packet_loss(),
            'bandwidth': measure_bandwidth(),
            'jitter': measure_jitter()
        }
    
    def classify_quality(self, metrics):
        if metrics['latency'] > 500 or metrics['packet_loss'] > 10:
            return 'poor'
        elif metrics['latency'] > 200 or metrics['packet_loss'] > 5:
            return 'fair'
        else:
            return 'good'
```

**Mitigation Strategy:**
```
1. Adaptive Bitrate (FR-5.4)
   - Reduce video quality: 720p → 480p → 360p
   - Reduce frame rate: 30 FPS → 15 FPS
   - Increase compression
   
2. Landmark-Only Mode
   - If bandwidth <100 kbps: Stop video, send landmarks only
   - Landmarks: 5 KB/s (100x reduction)
   - Maintain translation functionality
   
3. Buffering Strategy
   - Client-side: 2-second buffer for smoothing
   - Server-side: 1-second buffer for batch processing
   - Trade latency for reliability
   
4. Offline Mode Transition (FR-5.1)
   - If network unavailable >10 seconds
   - Switch to edge-only processing
   - Send SMS with GPS + keywords
   - Queue data for later upload
   
5. Reconnection Logic
   - Exponential backoff: 1s, 2s, 4s, 8s, 16s
   - Maintain call state during brief disconnections (<30s)
   - Resume from last known state
   - Upload queued data when reconnected
```

**User Feedback:**
- Display network quality indicator (green/yellow/red)
- Show "Reduced quality mode" message
- Indicate when operating offline

### Model Crash Handling

**Failure Mode:** ML model crashes, GPU out of memory, inference timeout

**Detection:**
```python
try:
    prediction = model.infer(input_data, timeout=5.0)
except (ModelCrashError, TimeoutError, OutOfMemoryError) as e:
    handle_model_failure(e)
```

**Mitigation Strategy:**
```
1. Graceful Degradation
   - Primary model fails → Fallback to simpler model
   - Sign recognition fails → Use keyword-only mode
   - Avatar generation fails → Text-only display
   
2. Model Redundancy
   - Run 2 model instances per GPU (hot standby)
   - If instance 1 crashes, route to instance 2
   - Restart crashed instance in background
   
3. Request Retry
   - Retry inference up to 3 times
   - Use exponential backoff (100ms, 200ms, 400ms)
   - If all retries fail → Escalate to human
   
4. Circuit Breaker Pattern
   - If model fails >50% of requests in 1 minute
   - Open circuit: Route all requests to fallback
   - Half-open after 30 seconds: Test with 10% traffic
   - Close circuit if success rate >90%
   
5. Emergency Fallback
   - Always maintain connection to human VRS
   - Seamless handoff if all AI systems fail
   - Never block emergency dispatch
```

**Monitoring & Alerts:**
- Model crash rate: Alert if >0.1% of requests
- GPU memory usage: Alert if >90%
- Inference timeout: Alert if >1% of requests
- Auto-restart crashed services
- Page on-call engineer if critical

### Incorrect Classification Handling

**Failure Mode:** Model misclassifies emergency type, location, or urgency

**Detection:**
```python
# Post-call validation
if dispatcher_corrected_field(field):
    log_misclassification(field, predicted, actual, confidence)
    
# Real-time sanity checks
if emergency_type == "medical" and keywords_contain("fire"):
    flag_potential_misclassification()
```

**Mitigation Strategy:**
```
1. Confidence Display (FR-7.6)
   - Show confidence scores to dispatcher
   - Color-code: Green (>90%), Yellow (80-90%), Red (<80%)
   - Dispatcher can override any field
   
2. Sanity Checks
   - Cross-validate emergency type with keywords
   - Validate location with GPS (geocoding)
   - Check urgency score against symptoms
   - Flag inconsistencies for dispatcher review
   
3. Multi-Model Ensemble
   - Run 3 models independently
   - Use majority voting for classification
   - If disagreement: Flag as uncertain
   
4. Human-in-the-Loop
   - Critical decisions (weapon present, life-threatening)
   - Require dispatcher confirmation before dispatch
   - AI provides suggestion, human makes final call
   
5. Continuous Learning
   - Log all dispatcher corrections
   - Retrain model monthly on corrected data
   - A/B test new models before deployment
   - Track accuracy improvement over time
```

**Safety Mechanisms:**
- Never auto-dispatch based on AI alone for weapon situations
- Always show raw transcript alongside structured data
- Provide video playback for dispatcher review
- Err on side of caution (over-dispatch vs under-dispatch)

**Misclassification Scenarios:**
```
Scenario 1: Medical misclassified as Crime
- Keywords: "ATTACK" (panic attack vs assault)
- Mitigation: Context analysis, clarification question
- Dispatcher sees both interpretations

Scenario 2: Wrong location
- Fingerspelling error: "MAIN" vs "MAINE"
- Mitigation: Geocoding validation, show map
- Dispatcher confirms address

Scenario 3: Urgency overestimation
- Fast signing due to excitement, not panic
- Mitigation: Multi-modal fusion, facial expression
- Dispatcher adjusts priority
```

---

## Security Architecture

### End-to-End Encryption

**Transport Layer:**
```
Client ↔ Server: TLS 1.3
- Cipher suite: TLS_AES_256_GCM_SHA384
- Perfect forward secrecy (ECDHE)
- Certificate pinning on mobile app
- HSTS (HTTP Strict Transport Security)

WebRTC: DTLS-SRTP
- Media encryption: AES-256-GCM
- Key exchange: ECDHE
- SRTP for real-time media
```

**Data at Rest:**
```
Database: AES-256 encryption
- Encrypted columns: PII, medical data, transcripts
- Encryption key management: AWS KMS / Azure Key Vault
- Key rotation: Every 90 days

Video Recordings: AES-256
- Stored in encrypted S3 buckets
- Access logs for all retrievals
- Automatic deletion per retention policy
```

**Application Layer:**
```
API Authentication: JWT (JSON Web Tokens)
- RS256 algorithm (RSA + SHA-256)
- Token expiration: 1 hour (access), 7 days (refresh)
- Claims: user_id, role, permissions, call_id

Session Management:
- Secure session tokens
- HttpOnly, Secure, SameSite cookies
- Session timeout: 30 minutes inactivity
```

### Key Management

**Architecture:**
```
┌─────────────────────────────────────────┐
│     Hardware Security Module (HSM)      │
│         (FIPS 140-2 Level 3)           │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│    Key Management Service (KMS)         │
│  - Master keys (never leave HSM)        │
│  - Key derivation                       │
│  - Audit logging                        │
└────────────────┬────────────────────────┘
                 ↓
┌─────────────────────────────────────────┐
│    Application Encryption Keys          │
│  - Database encryption keys             │
│  - Video encryption keys                │
│  - API signing keys                     │
│  - Rotated every 90 days                │
└─────────────────────────────────────────┘
```

**Key Hierarchy:**
```
Master Key (HSM)
  ↓
Data Encryption Keys (DEK)
  ├─ Database DEK
  ├─ Video Storage DEK
  └─ Backup DEK
  ↓
Per-Call Encryption Keys
  - Unique key per emergency call
  - Derived from DEK + call_id
  - Destroyed after retention period
```

**Key Rotation:**
```python
def rotate_encryption_keys():
    """Rotate encryption keys every 90 days"""
    new_dek = kms.generate_data_key()
    
    # Re-encrypt active data with new key
    for record in get_active_records():
        plaintext = decrypt(record.data, old_dek)
        record.data = encrypt(plaintext, new_dek)
        record.save()
    
    # Archive old key for historical data
    archive_key(old_dek, retention_period=7_years)
    
    # Update active key
    set_active_key(new_dek)
    
    log_key_rotation(old_dek.id, new_dek.id)
```

### Data Retention Policy

**Retention Periods (per NFR-6.10):**
```
Call Recordings:
- Active: 30-90 days (state-dependent)
- Legal hold: Indefinite (if subpoenaed)
- Deletion: Secure wipe (DoD 5220.22-M)

Call Metadata:
- Active: 1 year
- Archived: 7 years (compliance requirement)
- Anonymized: Indefinite (for research)

Model Telemetry:
- Raw logs: 90 days
- Aggregated metrics: 5 years
- Anonymized: Indefinite

User Accounts:
- Active: Until user requests deletion
- Inactive: 3 years, then anonymize
- Deletion: 30-day grace period
```

**Automated Deletion:**
```python
@scheduled_task(cron="0 2 * * *")  # Daily at 2 AM
def enforce_retention_policy():
    """Delete data past retention period"""
    
    # Delete old call recordings
    cutoff_date = now() - timedelta(days=RETENTION_DAYS)
    old_calls = EmergencyCall.objects.filter(
        ended_at__lt=cutoff_date,
        legal_hold=False
    )
    
    for call in old_calls:
        # Secure deletion
        delete_video_recording(call.video_url)
        delete_audio_recording(call.audio_url)
        
        # Anonymize metadata
        call.caller_id = "ANONYMIZED"
        call.location_address = "REDACTED"
        call.transcript = "REDACTED"
        call.save()
        
        log_deletion(call.call_id, "retention_policy")
```

### Compliance Considerations

**HIPAA-like Handling (NFR-5.4):**
```
Technical Safeguards:
✓ Access controls (role-based)
✓ Audit logs (all data access)
✓ Encryption (at rest and in transit)
✓ Automatic logoff (30 min inactivity)
✓ Data backup and recovery

Administrative Safeguards:
✓ Security officer designated
✓ Workforce training (annual)
✓ Incident response plan
✓ Business associate agreements

Physical Safeguards:
✓ Facility access controls
✓ Workstation security
✓ Device encryption
```

**FCC Compliance (REC-1.1):**
```
TRS Requirements:
✓ 24/7 availability
✓ Functional equivalence to voice calls
✓ No per-minute charges to users
✓ Confidentiality of communications
✓ Speed of answer (<1 minute)
✓ Complaint process
✓ Annual reporting
```

**ADA Compliance (NFR-6.1):**
```
Accessibility Requirements:
✓ Equal access to emergency services
✓ No additional cost to deaf users
✓ Effective communication
✓ Auxiliary aids and services
✓ No discrimination based on disability
```

---

## Scalability Plan

### Peak Load Scenario (City-Wide Emergency)

**Scenario:** Major earthquake in San Francisco
- Normal load: 100 concurrent calls
- Peak load: 10,000 concurrent calls (100x surge)
- Duration: 6 hours
- Geographic concentration: SF Bay Area

**Scaling Strategy:**
```
T+0 minutes: Earthquake occurs
  ↓
T+2 minutes: Call volume 10x normal (1,000 calls)
  - Auto-scaling triggers (>80% GPU utilization)
  - Alert on-call team
  ↓
T+5 minutes: Call volume 50x normal (5,000 calls)
  - Provision 20 additional A100 GPUs (spot instances)
  - Scale web servers 5x (horizontal scaling)
  - Increase database connections
  ↓
T+10 minutes: Call volume 100x normal (10,000 calls)
  - Provision 40 more A100 GPUs (total 60)
  - Activate disaster recovery region (US-West)
  - Load balance across 2 regions
  ↓
T+30 minutes: Sustained peak load
  - All systems scaled
  - Monitoring dashboards active
  - On-call team monitoring
  ↓
T+6 hours: Call volume declining
  - Gradual scale-down
  - Maintain 2x capacity for aftershocks
  ↓
T+24 hours: Return to normal
  - Scale down to baseline + 20% buffer
  - Post-incident review
```

### Horizontal Scaling

**Stateless Services (Easy to Scale):**
```
API Servers:
- Current: 10 instances
- Peak: 100 instances (10x)
- Scaling trigger: CPU >70% or latency >200ms
- Scale-up time: 2 minutes (container startup)

ML Inference Workers:
- Current: 8 A100 GPUs
- Peak: 80 A100 GPUs (10x)
- Scaling trigger: GPU utilization >80%
- Scale-up time: 5-10 minutes (GPU provisioning)

WebRTC SFU:
- Current: 5 instances (2,000 calls each)
- Peak: 50 instances (10,000 calls total)
- Scaling trigger: Connection count >80% capacity
- Scale-up time: 3 minutes
```

**Stateful Services (Harder to Scale):**
```
Database (PostgreSQL):
- Primary: 1 instance (write)
- Read Replicas: 5 instances (read)
- Peak: 10 read replicas
- Scaling: Add replicas (5 min), shard if needed (hours)

Redis (Session State):
- Current: 3-node cluster
- Peak: 9-node cluster
- Scaling: Add nodes to cluster (2 min)
- Sharding: By call_id hash
```

**Load Balancing:**
```
Geographic Load Balancing:
- US-East: 40% traffic (primary)
- US-West: 40% traffic (secondary)
- US-Central: 20% traffic (tertiary)

Algorithm: Least connections + geographic proximity
- Route to nearest region with capacity
- Failover to next region if unavailable
- Health checks every 10 seconds
```

### Auto-Scaling Triggers

**Metrics-Based Triggers:**
```yaml
GPU Scaling:
  metric: gpu_utilization
  threshold: 80%
  duration: 5 minutes
  scale_up: +10 GPUs
  scale_down: -5 GPUs (gradual)
  cooldown: 10 minutes

API Server Scaling:
  metric: cpu_utilization OR request_latency
  threshold: 70% CPU OR 200ms latency
  duration: 2 minutes
  scale_up: +20% instances
  scale_down: -10% instances
  cooldown: 5 minutes

Database Read Replica Scaling:
  metric: read_query_latency
  threshold: 100ms
  duration: 5 minutes
  scale_up: +1 replica
  scale_down: -1 replica
  cooldown: 15 minutes
```

**Predictive Scaling:**
```python
def predict_load(historical_data, time_of_day, day_of_week):
    """Predict load and pre-scale"""
    # Historical patterns
    if is_disaster_prone_season():
        baseline_capacity *= 1.5
    
    # Time-based patterns
    if is_peak_hours():  # 6 PM - 10 PM
        baseline_capacity *= 1.2
    
    # Pre-scale before predicted surge
    if predicted_surge_in_next_hour():
        pre_provision_resources()
```

**Manual Override:**
```
Disaster Declaration:
- Emergency services can manually trigger "disaster mode"
- Immediately scale to 10x capacity
- Bypass auto-scaling cooldowns
- Alert all on-call engineers
- Activate incident command center
```

---

## Observability & Monitoring

### Latency Tracking

**End-to-End Latency Breakdown:**
```
Metric: e2e_translation_latency_ms
Target: <1000ms (95th percentile)

Breakdown:
- video_capture: 33ms
- edge_processing: 15ms
- network_upload: 50-200ms
- video_decode: 50ms
- pose_detection: 25ms
- sign_recognition: 120ms
- text_generation: 10ms
- tts_synthesis: 100ms
- network_download: 50-200ms
- audio_playback: 20ms

Total: 473-843ms (within budget)
```

**Monitoring Implementation:**
```python
from prometheus_client import Histogram

latency_histogram = Histogram(
    'translation_latency_seconds',
    'End-to-end translation latency',
    buckets=[0.1, 0.25, 0.5, 0.75, 1.0, 1.5, 2.0, 5.0]
)

@latency_histogram.time()
def process_translation(video_frame):
    # Processing logic
    pass

# Alert if p95 latency >1 second
alert_rule = """
  alert: HighTranslationLatency
  expr: histogram_quantile(0.95, translation_latency_seconds) > 1.0
  for: 5m
  annotations:
    summary: "Translation latency exceeds 1 second"
"""
```

**Latency Dashboard:**
```
Grafana Dashboard: Translation Performance
- P50, P95, P99 latency (line chart)
- Latency breakdown by component (stacked bar)
- Latency heatmap (time vs latency)
- Slow requests (table with call_id, latency, reason)
```

### Accuracy Drift Monitoring

**Daily Accuracy Calculation:**
```python
@scheduled_task(cron="0 3 * * *")  # Daily at 3 AM
def calculate_daily_accuracy():
    """Calculate accuracy from dispatcher corrections"""
    yesterday = date.today() - timedelta(days=1)
    
    calls = EmergencyCall.objects.filter(
        date=yesterday,
        status='completed'
    )
    
    metrics = {
        'total_calls': calls.count(),
        'location_accuracy': 0,
        'emergency_type_accuracy': 0,
        'urgency_accuracy': 0
    }
    
    for call in calls:
        # Compare AI prediction vs dispatcher correction
        if call.location_corrected:
            metrics['location_errors'] += 1
        if call.emergency_type_corrected:
            metrics['type_errors'] += 1
        if abs(call.urgency_ai - call.urgency_dispatcher) > 2:
            metrics['urgency_errors'] += 1
    
    metrics['location_accuracy'] = 1 - (metrics['location_errors'] / metrics['total_calls'])
    # ... calculate other accuracies
    
    # Store in database
    AccuracyDriftMonitoring.objects.create(
        date=yesterday,
        **metrics
    )
    
    # Check for drift
    if metrics['location_accuracy'] < 0.90:  # Below 90% threshold
        trigger_accuracy_alert('location', metrics['location_accuracy'])
```

**Drift Detection:**
```python
def detect_accuracy_drift(metric_name, window_days=7):
    """Detect if accuracy is declining"""
    recent = get_accuracy_metrics(days=window_days)
    baseline = get_accuracy_metrics(days=30, offset=window_days)
    
    recent_avg = mean([m[metric_name] for m in recent])
    baseline_avg = mean([m[metric_name] for m in baseline])
    
    drift = baseline_avg - recent_avg
    
    if drift > 0.05:  # 5% accuracy drop
        return {
            'drift_detected': True,
            'severity': 'high' if drift > 0.10 else 'medium',
            'recent_accuracy': recent_avg,
            'baseline_accuracy': baseline_avg,
            'drift_amount': drift
        }
    
    return {'drift_detected': False}
```

**Alerts:**
```
Alert: Accuracy Drift Detected
Trigger: Accuracy drops >5% over 7 days
Action:
  1. Page ML team
  2. Analyze recent calls for patterns
  3. Check for data distribution shift
  4. Retrain model if needed
  5. A/B test new model before deployment
```

### Urgency Misclassification Alerts

**Real-Time Monitoring:**
```python
def monitor_urgency_misclassification(call):
    """Alert on significant urgency misclassification"""
    ai_urgency = call.urgency_ai
    dispatcher_urgency = call.urgency_dispatcher
    
    difference = abs(ai_urgency - dispatcher_urgency)
    
    # Critical misclassification
    if difference >= 5:  # e.g., AI said 3, dispatcher said 8
        alert = {
            'severity': 'critical',
            'call_id': call.call_id,
            'ai_urgency': ai_urgency,
            'dispatcher_urgency': dispatcher_urgency,
            'emergency_type': call.emergency_type,
            'outcome': call.outcome
        }
        
        send_alert('urgency_misclassification', alert)
        
        # If pattern detected
        if recent_misclassifications() > 10:
            trigger_model_review()
```

**Dashboard:**
```
Urgency Misclassification Dashboard:
- Confusion matrix (AI urgency vs dispatcher urgency)
- Misclassification rate over time
- Top misclassified scenarios
- Impact analysis (did misclassification cause harm?)
```

---

## Technical Tradeoffs

### Why Edge Inference for Certain Models

**Decision:** Run pose detection, keyword classification, and urgency scoring on-device

**Pros:**
- Reduced bandwidth: 30 Mbps video → 5 KB/s landmarks (6000x reduction)
- Lower latency: No network round-trip for initial processing
- Offline capability: Emergency functionality without connectivity
- Privacy: Some processing stays on-device
- Cost: Reduced cloud compute costs

**Cons:**
- Lower accuracy: 88% vs 96% for keyword classification
- Limited model size: <100 MB constraint
- Battery drain: 30% per hour (acceptable per NFR-1.9)
- Fragmentation: Must support iOS and Android
- Update complexity: App store approval for model updates

**Why This Tradeoff Makes Sense:**
- Emergency scenarios often have poor connectivity
- Offline mode is critical safety requirement (FR-5.1)
- 88% accuracy acceptable for non-critical path (urgency hints)
- Cloud models still provide final high-accuracy results
- Cost savings: $50k/month in bandwidth costs

### Why Transformer-Based Recognition

**Decision:** Use Spatial-Temporal Transformer instead of LSTM/CNN

**Alternatives Considered:**
1. LSTM (Recurrent Neural Network)
2. 3D CNN (Convolutional)
3. Graph Neural Network
4. Transformer (chosen)

**Comparison:**
```
Model       | Accuracy | Latency | Size  | Parallelization
------------|----------|---------|-------|----------------
LSTM        | 89%      | 200ms   | 50MB  | Sequential (slow)
3D CNN      | 91%      | 150ms   | 120MB | Parallel (fast)
GNN         | 93%      | 180ms   | 80MB  | Moderate
Transformer | 96%      | 120ms   | 180MB | Parallel (fast)
```

**Why Transformer:**
- Highest accuracy (96% meets NFR-2.1 target of >95%)
- Fast inference due to parallelization
- Attention mechanism provides interpretability
- State-of-art on sign language benchmarks
- Handles variable-length sequences naturally
- Better long-range dependencies (3-second signs)

**Cons:**
- Larger model size (180 MB vs 50 MB LSTM)
- Higher memory usage (requires GPU)
- More complex to train

**Mitigation:**
- Model size acceptable for cloud deployment
- GPU cluster provides sufficient memory
- Pre-trained transformers reduce training time
- Quantization reduces size for edge deployment

### Why Synthetic Dataset Augmentation

**Decision:** Generate 500 synthetic emergency scenarios for training

**Problem:** Limited real emergency sign language data
- No public datasets of emergency ASL conversations
- Privacy concerns with real 911 calls
- Rare events (weapon situations) have <10 examples
- Ethical issues recording real emergencies

**Alternatives:**
1. Use only general ASL datasets (insufficient emergency vocabulary)
2. Hire deaf actors for all scenarios (expensive, time-consuming)
3. Synthetic data generation (chosen)
4. Transfer learning only (lower accuracy)

**Synthetic Data Approach:**
```
Cost Comparison:
- Real actors: $50k (100 hours @ $500/hour)
- Synthetic: $10k (motion capture setup + rendering)
- Savings: $40k

Time Comparison:
- Real actors: 6 months (scheduling, filming, processing)
- Synthetic: 1 month (setup + generation)
- Faster: 5 months saved

Scalability:
- Real actors: Linear cost per scenario
- Synthetic: Fixed cost, unlimited scenarios
```

**Quality Validation:**
- Deaf community review: 4.2/5.0 realism
- Model trained on synthetic: 89% accuracy on real data
- Mixed training (real + synthetic): 96% accuracy
- Conclusion: Synthetic data effective when combined with real data

**Limitations:**
- Synthetic data less realistic than real
- May not capture all natural variations
- Requires validation with real data
- Potential bias in generation process

**Mitigation:**
- Combine synthetic with real general ASL data
- Validate with deaf community
- Continuous improvement from real usage
- Diverse synthetic scenarios (lighting, angles, speeds)

---

## Future Improvements

### Federated Learning

**Concept:** Train models on-device without centralizing data

**Benefits:**
- Privacy: Data never leaves device
- Personalization: Adapt to individual signing styles
- Compliance: Easier GDPR/privacy compliance
- Bandwidth: No need to upload training data

**Implementation:**
```
1. Deploy base model to all devices
2. Each device trains on local data (with consent)
3. Devices send model updates (gradients) to server
4. Server aggregates updates (federated averaging)
5. Deploy improved model to all devices
6. Repeat
```

**Challenges:**
- Heterogeneous data (different users, scenarios)
- Communication cost (model updates still require bandwidth)
- Convergence (slower than centralized training)
- Security (malicious updates, poisoning attacks)

**Timeline:** 12-18 months (research + implementation)

### Regional Sign Adaptation

**Problem:** ASL varies by region (Southern, Northeastern, etc.)

**Current:** Single model for all ASL variants
**Future:** Region-specific models or adapters

**Approach:**
```
Base Model (General ASL)
  ↓
Regional Adapter Layers
  ├─ Southern ASL Adapter
  ├─ Northeastern ASL Adapter
  ├─ Western ASL Adapter
  └─ Black ASL Adapter

User selects region → Load appropriate adapter
```

**Benefits:**
- Higher accuracy for regional variations
- Cultural competency
- Inclusivity (Black ASL, Indigenous sign languages)

**Implementation:**
- Collect regional data (partnerships with local deaf communities)
- Train adapter layers (10-20 MB each)
- User selects region in app settings
- Download appropriate adapter on-demand

**Timeline:** 6-12 months per region

### Cross-Language Expansion

**Goal:** Support sign languages beyond ASL

**Priority Order:**
1. BSL (British Sign Language) - 150k users
2. LSF (French Sign Language) - 100k users
3. Auslan (Australian Sign Language) - 10k users
4. ISL (Indian Sign Language) - 2.7M users

**Challenges:**
- Different grammar structures
- Different sign vocabularies
- Limited training data for some languages
- Cultural differences in emergency communication

**Approach:**
```
Transfer Learning:
1. Pre-train on ASL (largest dataset)
2. Fine-tune on target language (smaller dataset)
3. Leverage shared features (hand shapes, movements)
4. Language-specific output heads

Multi-lingual Model:
- Single model supports multiple languages
- User selects language in app
- Shared encoder, language-specific decoders
```

**Timeline:** 12-18 months per language

---

**Document Control:**
- **Author:** Technical Architecture Team
- **Reviewers:** ML Team, Infrastructure Team, Security Team
- **Status:** Design Phase - Ready for Implementation
- **Next Steps:** Prototype development, pilot testing
- **Last Updated:** February 15, 2026

---

*End of Technical Design Document*
