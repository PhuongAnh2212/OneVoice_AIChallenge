# OneVoice_AIChallenge

### Offline Vietnamese ↔ Mandarin Edge AI Translator for Industrial Workplaces

## Overview

OneVoice Factory is an Edge AI-powered multilingual communication device designed to bridge language barriers between Vietnamese workers and Mandarin-speaking supervisors in factories, warehouses, logistics hubs, and construction sites.

Unlike cloud-based translation applications, OneVoice Factory performs all speech recognition, translation, and speech synthesis directly on-device, enabling reliable operation in environments with limited or unstable Internet connectivity.

The project is developed as a submission for the OneVoice AI Challenge 2026 organized by Saigon AI Hub and Qualcomm.

---

## Problem Statement

Many manufacturing facilities in Vietnam employ Chinese-speaking managers while the workforce primarily communicates in Vietnamese.

Common communication challenges include:

* Language barriers during daily operations
* Delays in task execution due to translation needs
* Safety risks caused by misunderstood instructions
* Poor Internet connectivity in industrial environments
* Hands-busy workflows where smartphones are impractical

OneVoice Factory aims to provide real-time bilingual communication with minimal latency while remaining fully offline.

---

## Key Features

### Real-Time Speech Translation

* Vietnamese → Mandarin Chinese
* Mandarin Chinese → Vietnamese

### Fully Offline

* No cloud dependency
* No Internet connection required
* All AI models run locally on-device

### Industrial Vocabulary Optimization

Built-in domain glossary for:

* Manufacturing
* Logistics
* Warehousing
* Construction
* Workplace safety

### Low Latency Pipeline

Target performance:

| Metric                 | Target      |
| ---------------------- | ----------- |
| Real-Time Factor (RTF) | < 1.0       |
| Translation Latency    | < 2 seconds |
| Offline Operation      | 100%        |
| Device Dependency      | Edge Only   |

### Portable Form Factor

Supports:

* Handheld device
* Wearable badge
* Earbud integration
* Embedded industrial hardware

---

## System Architecture

```text
Microphone
    │
    ▼
Voice Activity Detection
    │
    ▼
Automatic Speech Recognition
(Vietnamese / Mandarin)
    │
    ▼
Translation Engine
    │
    ▼
Domain Glossary Correction
    │
    ▼
Text-To-Speech
    │
    ▼
Speaker / Earbud
```

---

## Technical Stack

### Speech Recognition (ASR)

Options:

* Faster-Whisper
* Distil-Whisper
* Whisper Small (Quantized)

### Translation Engine

Options:

* NLLB Distilled
* MarianMT
* Quantized Qwen
* Quantized Gemma

### Text-To-Speech (TTS)

Options:

* Piper TTS
* Kokoro TTS
* Edge-compatible Vietnamese and Mandarin TTS models

### Runtime

* Python
* ONNX Runtime
* Qualcomm AI Engine
* Qualcomm AI Hub Optimized Models

### Hardware

Prototype:

* Raspberry Pi 5
* USB Microphone
* Portable Speaker

Production Target:

* Qualcomm RB3 Gen 2
* Qualcomm QCS6490
* Snapdragon Edge AI Platforms

---

## Example Workflow

### Mandarin → Vietnamese

Input:

```text
请检查三号生产线
```

Output:

```text
Vui lòng kiểm tra dây chuyền sản xuất số 3.
```

---

### Vietnamese → Mandarin

Input:

```text
Máy ép đang bị kẹt.
```

Output:

```text
压机卡住了。
```

---

## Project Structure

```text
onevoice-factory/
│
├── app/
│   ├── asr/
│   ├── translator/
│   ├── glossary/
│   ├── tts/
│   └── ui/
│
├── models/
│   ├── asr/
│   ├── translation/
│   └── tts/
│
├── data/
│   └── terminology/
│
├── docs/
│   ├── architecture/
│   ├── benchmark/
│   └── challenge_submission/
│
├── tests/
│
├── requirements.txt
└── README.md
```

---

## Future Enhancements

* Speaker diarization
* Multi-person conversations
* Camera-assisted translation
* OCR for industrial signs
* Safety incident reporting
* Additional language support:

  * Korean
  * English
  * Japanese

---

## Business Applications

### Manufacturing

Communication between workers and supervisors.

### Logistics

Cross-border warehouse operations.

### Construction

Multilingual project coordination.

### Industrial Safety

Rapid delivery of emergency instructions.

---

## Competition Alignment

This solution directly addresses the objectives of the OneVoice AI Challenge:

* Fully offline Edge AI processing
* Real-time multilingual communication
* Low latency operation
* Practical industrial deployment
* Strong business viability
* Portable device architecture

---

## Team

Built for the OneVoice AI Challenge 2026.

Empowering communication without connectivity.
