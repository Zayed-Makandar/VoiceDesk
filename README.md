# VoiceDesk: AI Front-Desk Voice Assistant

A self-hosted, voice-driven receptionist kiosk. Visitors speak a question, and the assistant answers out loud from the organisation's own knowledge base, using local speech recognition, retrieval-augmented generation (RAG) and a locally served LLM.

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![Ollama](https://img.shields.io/badge/Ollama-local_LLM-000000?logo=ollama&logoColor=white)
![ChromaDB](https://img.shields.io/badge/ChromaDB-RAG-FF6F61)
![Whisper](https://img.shields.io/badge/faster--whisper-STT-412991)
![TTS](https://img.shields.io/badge/Edge--TTS-speech-0078D4)

> **Note:** The source code is private. This repository documents the architecture, design decisions and results. I'm happy to walk through the implementation in an interview.

![Kiosk](screenshots/01-kiosk-idle.png)

---

## Overview

Front desks at institutes and offices spend much of the day answering the same questions: courses, fees, timings, directions, who to meet. VoiceDesk handles these at a walk-up kiosk. A visitor says the wake word, asks a question in plain speech, and hears a spoken answer within seconds, while an animated face shows whether the assistant is listening, thinking or speaking.

Everything that touches visitor data runs on the kiosk itself: speech recognition, retrieval and the language model. Staff keep the answers current through an admin console, and their edits are searchable immediately, without a restart.

## Key Features

- **Wake-word voice conversation** with fuzzy phonetic matching, tuned on real transcription errors from the deployment site.
- **Grounded answers (RAG):** questions about the organisation are answered from an admin-editable knowledge base instead of the model's memory.
- **Streaming speech output:** the reply is spoken sentence by sentence while the LLM is still generating, so the visitor hears the start of the answer sooner.
- **Smart routing:** time, weather and arithmetic questions bypass the LLM entirely for instant answers. Organisation questions trigger retrieval, and everything else goes straight to the LLM.
- **Offline fallback:** if the internet drops, cloud text-to-speech automatically switches to an on-device voice.
- **Admin console:** edit the knowledge base, upload documents and monitor system health. Changes are re-embedded live.

## Screenshots

| | |
|---|---|
| ![Listening](screenshots/02-kiosk-listening.png) | ![Speaking](screenshots/03-kiosk-speaking.png) |
| *Listening to a visitor* | *Answering out loud* |
| ![Knowledge base](screenshots/04-admin-knowledge-base.png) | ![System health](screenshots/05-admin-health.png) |
| *Admin knowledge-base editor (demo data)* | *System health dashboard* |

**Demo video:** [link to a 60-second screen recording]

## Architecture

```mermaid
flowchart LR
    subgraph Kiosk["Kiosk browser"]
        Mic["Mic capture + voice activity detection"]
        Face["Animated face (idle / thinking / speaking)"]
    end
    subgraph Server["FastAPI backend (on-prem)"]
        STT["Speech-to-text: faster-whisper INT8"]
        Wake["Wake-word + correction layer"]
        Route{"Query router"}
        Quick["Instant handlers: time, weather, maths"]
        Retrieve["Retriever: MiniLM embeddings"]
        LLM["Local LLM via Ollama"]
        TTS["Streaming TTS (cloud, offline fallback)"]
    end
    VDB[("ChromaDB knowledge base")]
    Admin["Admin console"] -->|"edit / upload"| VDB
    Mic -->|audio| STT --> Wake --> Route
    Route -->|"simple"| Quick --> TTS
    Route -->|"organisation question"| Retrieve <--> VDB
    Retrieve --> LLM
    Route -->|"general"| LLM
    LLM -->|"sentence chunks"| TTS -->|"audio stream"| Face
```

## How It Works

1. The kiosk records the visitor and detects the end of speech using audio energy (voice activity detection).
2. The audio is transcribed locally with faster-whisper (INT8). A correction layer fixes known mis-hearings of the wake word and of domain terms.
3. A router decides how to answer: an instant handler, retrieval plus the LLM, or the LLM alone.
4. For organisation questions, the most relevant knowledge-base entries are retrieved from ChromaDB. A similarity threshold filters out weak matches so the model isn't fed irrelevant context.
5. The local LLM streams its answer, and each finished sentence is sent to text-to-speech straight away, so speech starts before generation ends.
6. The animated face changes state in step with the pipeline.

## Results

| Metric | Value |
|---|---|
| Time to first spoken word | [X] s |
| Speech-to-text latency (3 s clip, CPU) | [X] ms |
| Retrieval hit rate on a [X]-question test set | [X]% |
| Answer faithfulness (RAGAS) | [X] |
| Deployment | On-prem kiosk PC, CPU only |

## Tech Stack

| Category | Tools |
|---|---|
| Speech-to-text | faster-whisper (CTranslate2, INT8), noise reduction, Butterworth filtering |
| Retrieval | sentence-transformers (all-MiniLM-L6-v2), ChromaDB (cosine HNSW) |
| LLM | Ollama, small instruction-tuned model (~1B parameters) |
| Text-to-speech | Edge-TTS with automatic pyttsx3/eSpeak offline fallback |
| Backend | FastAPI, Uvicorn, NDJSON streaming |
| Frontend | HTML, CSS, JavaScript, Web Audio (WAV encoding, voice activity detection) |

## Engineering Decisions

- **Self-hosted models:** visitor speech never leaves the building, and there are no per-request API costs for a kiosk that runs all day.
- **Fixes driven by production logs:** the wake-word matcher and correction table were built from real failed transcripts at the deployment site, not guessed in advance. That made the biggest difference to reliability in a noisy lobby.
- **Streaming at sentence level:** waiting for the full LLM answer before speaking felt slow. Speaking each sentence as soon as it's finished cuts the perceived wait sharply without changing the model.
- **Bypassing the LLM where possible:** time, weather and arithmetic don't need a language model. Handling them directly is faster and avoids hallucinated answers.

## My Role

I built and deployed the full system: the speech pipeline and its field tuning, the RAG pipeline and knowledge-base admin tools, LLM integration and prompt design, streaming text-to-speech, and the kiosk interface.

## Contact

**Zayed Makandar** | [zayedmakandar03@gmail.com](mailto:zayedmakandar03@gmail.com) | [LinkedIn](https://www.linkedin.com/in/zayed-makandar-642536422/)
