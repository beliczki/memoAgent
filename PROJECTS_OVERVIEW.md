# memoAgent Ecosystem - Two-Project Architecture

## Overview

The memoAgent ecosystem consists of **two separate projects** that work together to provide intelligent meeting and conference transcription:

1. **Advanced Transcriber** (separate repo) - Core transcription service
2. **memoAgent** (this repo) - Audio source orchestrator and meeting integration

---

## Project 1: Advanced Transcriber

**Repository:** `beliczki/advanced-transcriber` (to be created)

**Purpose:** Headless transcription service that accepts audio streams and returns transcripts with confidence scores.

### Key Features:
- **Dual-Engine Processing**: Processes single audio stream through 2 STT engines (Google Cloud STT + OpenAI Whisper)
- **LLM Consolidation**: Uses GPT-4/Claude to merge transcripts and pick best interpretation
- **Word-Level Confidence**: Color-coded confidence scores showing engine agreement
- **WebSocket API**: Accepts audio streams, returns real-time transcripts
- **No Input Integration**: Pure service - doesn't care where audio comes from

### Audio Input:
- WebSocket streaming (PCM16, 16kHz, mono)
- Can be fed by ANY source (VLC player, memoAgent, direct mic, etc.)

### Output:
```json
{
  "text": "consolidated transcript",
  "confidence": 0.95,
  "words": [
    {"word": "hello", "confidence": 0.98, "engines_agree": true},
    {"word": "world", "confidence": 0.87, "engines_agree": false}
  ],
  "engine_1_text": "hello world",
  "engine_2_text": "hello word",
  "disagreements": [{"position": 1, "options": ["world", "word"]}]
}
```

---

## Project 2: memoAgent (This Repo)

**Repository:** `beliczki/memoAgent`

**Purpose:** Audio source orchestrator that captures audio from various sources and routes to Advanced Transcriber.

### Key Features:
- **Multi-Source Input**:
  1. **Conference Mode**: Mixer line out (mono) - digitalized conference audio
  2. **Local Mic Mode**: Direct microphone input
  3. **Meeting Bot Mode**: Join Google Meet / MS Teams as bot participant

- **Meeting SDK Integration**:
  - Google Meet SDK - join as bot, capture audio + speaker metadata
  - Microsoft Teams SDK - join as bot, capture audio + speaker metadata

- **Source Selection UI**: Choose which audio source to use
- **Audio Routing**: Streams selected source to Advanced Transcriber service
- **Transcript Display**: Receives transcripts from Transcriber and displays with UI

### Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────┐
│                        memoAgent                             │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Conference  │  │  Local Mic   │  │ Meeting Bot  │      │
│  │  (Line Out)  │  │              │  │ (Meet/Teams) │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │               │
│         └──────────────────┴──────────────────┘               │
│                            │                                  │
│                    ┌───────▼────────┐                        │
│                    │ Source Selector │                        │
│                    │   & Router      │                        │
│                    └───────┬────────┘                        │
│                            │                                  │
│                            │ WebSocket                        │
└────────────────────────────┼──────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  Advanced Transcriber                        │
│                                                               │
│         ┌────────────────────────────────────┐              │
│         │     Audio Distributor              │              │
│         └─────┬────────────────────┬─────────┘              │
│               │                    │                         │
│       ┌───────▼──────┐     ┌──────▼────────┐               │
│       │ Google Cloud │     │ OpenAI Whisper│               │
│       │     STT      │     │      API      │               │
│       └───────┬──────┘     └──────┬────────┘               │
│               │                    │                         │
│         ┌─────▼────────────────────▼─────┐                 │
│         │   LLM Consolidation Service    │                 │
│         │    (GPT-4 / Claude)            │                 │
│         └─────┬──────────────────────────┘                 │
│               │                                              │
│               ▼                                              │
│       Consolidated Transcript                               │
│                                                               │
└────────────────────────┬──────────────────────────────────────┘
                         │ WebSocket
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    memoAgent UI                              │
│                                                               │
│  - Display transcript with confidence colors                 │
│  - Show speaker names (from meeting bot metadata)           │
│  - Engine disagreement highlights                            │
│  - Session management                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## WebSocket API Contract

### memoAgent → Transcriber

**Connect:**
```javascript
ws://localhost:5001/transcribe
```

**Start Session:**
```json
{
  "action": "start_session",
  "session_id": "uuid-123",
  "source_type": "meeting_bot",  // or "conference", "local_mic"
  "source_metadata": {
    "meeting_platform": "google_meet",
    "meeting_id": "abc-defg-hij"
  }
}
```

**Stream Audio:**
```json
{
  "action": "audio_chunk",
  "session_id": "uuid-123",
  "audio_data": "base64-encoded-pcm16",
  "timestamp": 1234567890,
  "speaker_metadata": {  // Optional, from meeting bot
    "speaker_id": "user@example.com",
    "speaker_name": "John Doe"
  }
}
```

**Stop Session:**
```json
{
  "action": "stop_session",
  "session_id": "uuid-123"
}
```

### Transcriber → memoAgent

**Transcript Event:**
```json
{
  "event": "transcript",
  "session_id": "uuid-123",
  "text": "Hello world",
  "confidence": 0.95,
  "is_final": true,
  "words": [
    {
      "word": "Hello",
      "confidence": 0.98,
      "engines_agree": true,
      "start_time": 0.0,
      "end_time": 0.5
    },
    {
      "word": "world",
      "confidence": 0.92,
      "engines_agree": true,
      "start_time": 0.5,
      "end_time": 1.0
    }
  ],
  "engine_1": {
    "name": "google",
    "text": "Hello world",
    "confidence": 0.96
  },
  "engine_2": {
    "name": "whisper",
    "text": "Hello world",
    "confidence": 0.94
  },
  "disagreements": []
}
```

**Error Event:**
```json
{
  "event": "error",
  "session_id": "uuid-123",
  "error": "STT engine timeout"
}
```

---

## Meeting Bot Integration (memoAgent)

### Google Meet Bot

**SDK:** Google Meet REST API + LiveKit/Jitsi integration

**Flow:**
1. User selects "Join Google Meet" in memoAgent
2. Provides meeting URL or code
3. memoAgent authenticates via OAuth
4. Joins meeting as bot participant ("memoAgent Bot")
5. Captures audio stream from meeting
6. Captures speaker metadata (who's talking)
7. Streams audio + metadata to Transcriber
8. Displays transcript in real-time

**Required:**
- Google Cloud Project with Meet API enabled
- OAuth 2.0 credentials
- Bot participant permissions

### Microsoft Teams Bot

**SDK:** Microsoft Bot Framework + Teams SDK

**Flow:**
1. User selects "Join Teams Meeting"
2. Provides meeting link
3. memoAgent bot joins via Bot Framework
4. Captures audio stream
5. Captures participant metadata
6. Streams to Transcriber
7. Displays transcript

**Required:**
- Azure Bot Service registration
- Teams app manifest
- Bot permissions for meetings

---

## Development Roadmap

### Phase 1: Advanced Transcriber (Separate Repo)
- [ ] Single engine (Google STT) MVP
- [ ] WebSocket API for audio streaming
- [ ] Basic transcript output
- [ ] Database for session storage

### Phase 2: Transcriber - Dual Engine
- [ ] Add OpenAI Whisper integration
- [ ] Parallel processing of both engines
- [ ] Basic consolidation (choose higher confidence)

### Phase 3: Transcriber - LLM Consolidation
- [ ] GPT-4/Claude integration
- [ ] Smart merging of transcripts
- [ ] Word-level confidence tracking
- [ ] Disagreement detection

### Phase 4: memoAgent - Basic Sources
- [ ] Conference mode (line-in / WebSocket input)
- [ ] Local mic mode
- [ ] Source selection UI
- [ ] Connect to Transcriber service

### Phase 5: memoAgent - Google Meet Bot
- [ ] OAuth integration
- [ ] Join meeting as bot
- [ ] Audio stream capture
- [ ] Speaker metadata extraction

### Phase 6: memoAgent - Teams Bot
- [ ] Azure Bot Service setup
- [ ] Teams SDK integration
- [ ] Meeting join capability
- [ ] Audio + speaker capture

### Phase 7: Polish & Features
- [ ] Transcript export (txt, json, srt)
- [ ] Session replay
- [ ] Speaker name management
- [ ] Confidence visualization improvements

---

## Deployment

### Advanced Transcriber
- Standalone service
- Runs on port 5001
- Can be deployed separately on dedicated hardware
- Scalable (handle multiple concurrent sessions)

### memoAgent
- Runs on port 5000
- Requires Transcriber URL in config
- Can run on laptop/desktop
- Meeting bot components may require Azure/GCP deployment

---

## Tech Stack

### Advanced Transcriber
- **Backend:** Python, Flask, Flask-SocketIO
- **STT Engines:** Google Cloud Speech-to-Text, OpenAI Whisper API
- **LLM:** OpenAI GPT-4 / Anthropic Claude
- **Database:** SQLite (dev) / PostgreSQL (prod)
- **Audio Processing:** pydub, numpy

### memoAgent
- **Backend:** Python, Flask
- **Meeting SDKs:** Google Meet API, Microsoft Bot Framework
- **Audio Capture:** pyaudio, sounddevice
- **Frontend:** HTML/CSS/JS, Socket.IO client
- **Authentication:** OAuth 2.0 (Google, Microsoft)

---

## Configuration

### Advanced Transcriber (.env)
```env
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
TRANSCRIBER_PORT=5001
```

### memoAgent (.env)
```env
TRANSCRIBER_URL=ws://localhost:5001
GOOGLE_OAUTH_CLIENT_ID=...
GOOGLE_OAUTH_CLIENT_SECRET=...
MICROSOFT_APP_ID=...
MICROSOFT_APP_PASSWORD=...
MEMOAGENT_PORT=5000
```

---

## Testing with VLC

You can test the Advanced Transcriber by streaming audio from VLC:

```bash
# Stream audio file to WebSocket (requires VLC with HTTP module)
vlc audio.mp3 --sout '#transcode{acodec=s16l,channels=1,srate=16000}:http{dst=:8080/audio.raw}'

# Or use a simple Python script to read file and stream via WebSocket
python stream_audio.py --file test.wav --url ws://localhost:5001/transcribe
```

---

**Next Steps:**
1. Create `beliczki/advanced-transcriber` repository
2. Implement Phase 1 of Transcriber
3. Refactor this repo (memoAgent) to focus on source integration
4. Define complete WebSocket protocol
