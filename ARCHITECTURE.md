# memoAgent - Multi-Stream Real-Time Transcription System

## Overview

memoAgent is an advanced real-time audio transcription service that supports dual-stream audio processing, LLM-powered transcript consolidation, confidence scoring, and multi-engine STT support.

## Core Capabilities

1. **Dual-Stream Transcription** - Process 2 separate audio streams simultaneously
2. **Multi-Engine Support** - Switch between Google Speech-to-Text, OpenAI Whisper, Deepgram
3. **LLM Consolidation** - Merge dual transcripts intelligently using language models
4. **Confidence Scores** - Word-level probability scores for transcript accuracy
5. **Speaker Name Management** - Match and manage speaker identities across streams
6. **Admin Panel** - Configure engines, manage settings, view analytics

---

## Architecture Diagram

```
┌─────────────────┐                    ┌─────────────────┐
│  Audio Stream 1 │                    │  Audio Stream 2 │
│  (Mic/WebRTC)   │                    │  (Mic/WebRTC)   │
└────────┬────────┘                    └────────┬────────┘
         │ WebSocket A                          │ WebSocket B
         ▼                                      ▼
┌──────────────────────────────────────────────────────────┐
│             Flask + SocketIO Web Server                  │
│  ┌────────────────────────────────────────────────────┐  │
│  │         Stream Manager (Session Controller)        │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
         │                                      │
         ▼                                      ▼
┌──────────────────┐              ┌──────────────────┐
│  STT Engine 1    │              │  STT Engine 2    │
│  (Configurable)  │              │  (Configurable)  │
└────────┬─────────┘              └────────┬─────────┘
         │                                  │
         │  Transcript A + Confidence       │  Transcript B + Confidence
         └──────────────┬───────────────────┘
                        ▼
         ┌──────────────────────────────┐
         │  LLM Consolidation Service   │
         │  (GPT-4 / Claude / Gemini)   │
         │  - Merge overlapping text    │
         │  - Resolve conflicts         │
         │  - Match speaker names       │
         └──────────────┬───────────────┘
                        ▼
         ┌──────────────────────────────┐
         │   Unified Transcript Output  │
         │   + Confidence Heat Map      │
         │   + Speaker Attribution      │
         │   + Timestamps               │
         └──────────────┬───────────────┘
                        ▼
         ┌──────────────────────────────┐
         │     WebSocket → Clients      │
         │     SQLite Storage           │
         └──────────────────────────────┘
```

---

## Technology Stack

### Backend
- **Python 3.10+** - Core language
- **Flask** - Web framework
- **Flask-SocketIO** - WebSocket communication
- **Google Cloud Speech-to-Text** - Primary STT engine
- **OpenAI Whisper API** - Secondary STT option
- **Deepgram API** - Third STT option
- **OpenAI GPT-4** / **Anthropic Claude** - LLM for consolidation
- **SQLite** - Database for transcripts, settings, sessions
- **Redis** (optional) - Session/state management for production

### Frontend
- **HTML5 + CSS3** - UI
- **Socket.IO Client** - Real-time communication
- **MediaRecorder API** - Audio capture
- **Vanilla JavaScript** - Keep dependencies minimal

### Audio Processing
- **pydub** - Audio manipulation
- **webrtcvad** (optional) - Voice activity detection

---

## Directory Structure

```
memoAgent/
├── app/
│   ├── __init__.py                  # Flask app factory
│   ├── config.py                    # Configuration management
│   ├── models.py                    # Database models
│   ├── sockets.py                   # SocketIO event handlers
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── main.py                  # Main transcription UI
│   │   ├── admin.py                 # Admin panel
│   │   └── api.py                   # REST API endpoints
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── audio_processor.py       # Audio stream handling
│   │   ├── stream_manager.py        # Dual-stream coordination
│   │   ├── stt/
│   │   │   ├── __init__.py
│   │   │   ├── base.py              # Abstract STT interface
│   │   │   ├── google_stt.py        # Google Cloud STT
│   │   │   ├── whisper_stt.py       # OpenAI Whisper
│   │   │   └── deepgram_stt.py      # Deepgram STT
│   │   ├── consolidation_service.py # LLM-based transcript merging
│   │   ├── confidence_tracker.py    # Confidence score management
│   │   └── speaker_manager.py       # Speaker name matching
│   │
│   ├── static/
│   │   ├── css/
│   │   │   ├── main.css             # Main UI styles
│   │   │   └── admin.css            # Admin panel styles
│   │   └── js/
│   │       ├── recorder.js          # Dual audio stream capture
│   │       ├── transcript-ui.js     # Real-time transcript display
│   │       └── admin.js             # Admin panel logic
│   │
│   └── templates/
│       ├── base.html                # Base template
│       ├── index.html               # Main transcription UI
│       ├── admin.html               # Admin settings panel
│       └── components/
│           ├── confidence-heatmap.html  # Word confidence visualization
│           └── stream-selector.html     # Audio source selector
│
├── data/
│   ├── memoagent.db                 # SQLite database
│   └── sessions/                    # Temp audio storage
│
├── tests/
│   ├── test_stt_engines.py
│   ├── test_consolidation.py
│   └── test_confidence.py
│
├── .env.example                     # Environment variable template
├── .gitignore
├── requirements.txt
├── run.py                           # Application entry point
├── README.md
└── ARCHITECTURE.md                  # This file
```

---

## Core Components

### 1. Stream Manager

Manages dual audio streams and coordinates transcription.

```python
class StreamManager:
    """
    Coordinates dual-stream transcription.
    """
    def __init__(self, session_id):
        self.session_id = session_id
        self.stream_a = AudioStream(stream_id='A')
        self.stream_b = AudioStream(stream_id='B')
        self.consolidator = ConsolidationService()

    async def process_chunk(self, stream_id, audio_data):
        """Process audio chunk from either stream."""
        stream = self.stream_a if stream_id == 'A' else self.stream_b
        transcript = await stream.transcribe(audio_data)

        # Consolidate if both streams have recent data
        if self.should_consolidate():
            unified = await self.consolidate_transcripts()
            return unified

        return transcript
```

### 2. STT Engine Interface

Abstract interface for multiple STT engines.

```python
class BaseSTTEngine(ABC):
    """Base class for all STT engines."""

    @abstractmethod
    async def transcribe(self, audio_data: bytes) -> TranscriptResult:
        """
        Transcribe audio data.

        Returns:
            TranscriptResult with:
            - text: str
            - confidence: float (0.0-1.0)
            - words: List[WordResult]  # Word-level confidence
            - is_final: bool
        """
        pass

    @abstractmethod
    def configure(self, config: dict):
        """Configure engine-specific settings."""
        pass
```

**Google Cloud STT Implementation:**

```python
class GoogleSTTEngine(BaseSTTEngine):
    def __init__(self):
        self.client = speech.SpeechClient()
        self.streaming_config = speech.StreamingRecognitionConfig(
            config=speech.RecognitionConfig(
                encoding=speech.RecognitionConfig.AudioEncoding.LINEAR16,
                sample_rate_hertz=16000,
                language_code="en-US",
                enable_word_confidence=True,  # ← Get word-level confidence
                enable_automatic_punctuation=True,
            ),
            interim_results=True,
        )

    async def transcribe(self, audio_data: bytes) -> TranscriptResult:
        responses = self.client.streaming_recognize(
            self.streaming_config,
            [speech.StreamingRecognizeRequest(audio_content=audio_data)]
        )

        for response in responses:
            for result in response.results:
                alternative = result.alternatives[0]

                words = [
                    WordResult(
                        word=word_info.word,
                        confidence=word_info.confidence,
                        start_time=word_info.start_time.total_seconds(),
                        end_time=word_info.end_time.total_seconds()
                    )
                    for word_info in alternative.words
                ]

                return TranscriptResult(
                    text=alternative.transcript,
                    confidence=alternative.confidence,
                    words=words,
                    is_final=result.is_final
                )
```

### 3. Consolidation Service

LLM-powered transcript merging.

```python
class ConsolidationService:
    """
    Uses LLM to intelligently merge dual transcripts.
    """
    def __init__(self):
        self.llm = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
        self.speaker_manager = SpeakerManager()

    async def consolidate(
        self,
        transcript_a: TranscriptResult,
        transcript_b: TranscriptResult,
        context_window: int = 5
    ) -> ConsolidatedResult:
        """
        Merge two transcripts considering:
        - Overlapping content
        - Speaker attribution
        - Confidence scores
        - Previous context
        """

        # Get last N sentences for context
        context_a = self.get_recent_sentences(transcript_a, context_window)
        context_b = self.get_recent_sentences(transcript_b, context_window)

        prompt = f"""
        You are merging two real-time transcripts from different audio sources.

        Stream A (last {context_window} sentences):
        {context_a}

        New from Stream A: {transcript_a.text} (confidence: {transcript_a.confidence})

        Stream B (last {context_window} sentences):
        {context_b}

        New from Stream B: {transcript_b.text} (confidence: {transcript_b.confidence})

        Speaker names: {self.speaker_manager.get_known_speakers()}

        Task:
        1. Merge the new text if they represent the same speech
        2. Identify if they're from different speakers (separate)
        3. Match speaker names if mentioned
        4. Return consolidated text with speaker attribution

        Return JSON:
        {{
          "merged": true/false,
          "text": "consolidated text",
          "speaker": "name or null",
          "confidence_adjustment": 0.0-1.0
        }}
        """

        response = await self.llm.chat.completions.create(
            model="gpt-4-turbo-preview",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        result = json.loads(response.choices[0].message.content)

        return ConsolidatedResult(
            text=result['text'],
            speaker=result['speaker'],
            merged=result['merged'],
            confidence=self.calculate_consolidated_confidence(
                transcript_a,
                transcript_b,
                result['confidence_adjustment']
            )
        )
```

### 4. Confidence Tracker

Manages and visualizes word-level confidence scores.

```python
class ConfidenceTracker:
    """
    Track confidence scores for visualization.
    """
    def __init__(self):
        self.word_confidences = []

    def add_words(self, words: List[WordResult]):
        """Add word results with confidence scores."""
        self.word_confidences.extend(words)

    def get_heatmap_data(self) -> List[dict]:
        """
        Generate data for confidence heat map visualization.

        Returns list of:
        {
          "word": "hello",
          "confidence": 0.95,
          "color": "rgb(0, 255, 0)"  # Green for high confidence
        }
        """
        return [
            {
                "word": word.word,
                "confidence": word.confidence,
                "color": self.confidence_to_color(word.confidence)
            }
            for word in self.word_confidences
        ]

    def confidence_to_color(self, confidence: float) -> str:
        """
        Map confidence to color:
        - 1.0 = Green (100% certain)
        - 0.7-1.0 = Yellow to Green
        - 0.0-0.7 = Red to Yellow (low confidence / guessed)
        """
        if confidence >= 0.9:
            return "rgb(0, 200, 0)"  # Green
        elif confidence >= 0.7:
            return "rgb(200, 200, 0)"  # Yellow
        else:
            return "rgb(200, 0, 0)"  # Red
```

### 5. Speaker Manager

Match and manage speaker names across streams.

```python
class SpeakerManager:
    """
    Manage speaker identification and name matching.
    """
    def __init__(self):
        self.known_speakers = {}  # {speaker_id: name}
        self.voice_profiles = {}  # For future voice matching

    def register_speaker(self, speaker_id: str, name: str):
        """Register a speaker name."""
        self.known_speakers[speaker_id] = name

    def match_speaker(self, text: str, stream_id: str) -> Optional[str]:
        """
        Attempt to identify speaker from context or voice.

        Uses:
        - Name mentions in text ("This is John speaking")
        - Stream assignment (Stream A = Speaker 1)
        - Voice profile matching (future)
        """
        # Check for self-identification
        patterns = [
            r"(?:this is|i'm|i am|my name is)\s+(\w+)",
            r"(\w+)\s+speaking",
        ]

        for pattern in patterns:
            match = re.search(pattern, text, re.IGNORECASE)
            if match:
                name = match.group(1)
                self.register_speaker(stream_id, name)
                return name

        # Return known speaker for this stream
        return self.known_speakers.get(stream_id)

    def get_known_speakers(self) -> List[str]:
        """Get list of known speaker names."""
        return list(self.known_speakers.values())
```

---

## Admin Panel Features

### Engine Configuration

```
┌─────────────────────────────────────────┐
│  STT Engine Settings                    │
├─────────────────────────────────────────┤
│  Active Engine: ● Google Cloud STT      │
│                 ○ OpenAI Whisper        │
│                 ○ Deepgram              │
│                                         │
│  Stream A Engine: [Google Cloud ▼]     │
│  Stream B Engine: [Google Cloud ▼]     │
│                                         │
│  ☑ Enable word-level confidence        │
│  ☑ Enable automatic punctuation        │
│  ☑ Use LLM consolidation               │
│                                         │
│  [Save Settings]                        │
└─────────────────────────────────────────┘
```

### LLM Consolidation Settings

```
┌─────────────────────────────────────────┐
│  Consolidation Settings                 │
├─────────────────────────────────────────┤
│  LLM Model: [GPT-4 Turbo ▼]            │
│                                         │
│  Context Window: [5] sentences          │
│  Merge Threshold: [0.7] similarity      │
│                                         │
│  ☑ Enable speaker identification       │
│  ☑ Auto-match speaker names            │
│                                         │
│  [Test Consolidation]                  │
└─────────────────────────────────────────┘
```

### Speaker Name Management

```
┌─────────────────────────────────────────┐
│  Speaker Management                     │
├─────────────────────────────────────────┤
│  Stream A Speaker: [John ___________]   │
│  Stream B Speaker: [Sarah __________]   │
│                                         │
│  Known Speakers:                        │
│  • John (Stream A) - 145 segments       │
│  • Sarah (Stream B) - 132 segments      │
│  • Unknown - 12 segments                │
│                                         │
│  [Auto-detect Names] [Clear All]       │
└─────────────────────────────────────────┘
```

---

## Data Models

### Database Schema

```sql
-- Sessions
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME,
    stt_engine_a TEXT,
    stt_engine_b TEXT,
    consolidation_enabled BOOLEAN DEFAULT TRUE
);

-- Transcripts (raw from each stream)
CREATE TABLE transcripts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    stream_id TEXT CHECK(stream_id IN ('A', 'B')),
    text TEXT,
    confidence REAL,
    is_final BOOLEAN,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Words (word-level confidence)
CREATE TABLE words (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    transcript_id INTEGER,
    word TEXT,
    confidence REAL,
    start_time REAL,
    end_time REAL,
    FOREIGN KEY (transcript_id) REFERENCES transcripts(id)
);

-- Consolidated Transcripts
CREATE TABLE consolidated_transcripts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    text TEXT,
    speaker_name TEXT,
    confidence REAL,
    merged_from_streams TEXT,  -- 'A', 'B', or 'A+B'
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Speakers
CREATE TABLE speakers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    name TEXT,
    stream_id TEXT,
    first_mentioned_at DATETIME,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Settings
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## WebSocket Events

### Client → Server

```javascript
// Start new session with dual streams
socket.emit('start_session', {
  stt_engine_a: 'google',
  stt_engine_b: 'google',
  enable_consolidation: true
});

// Send audio chunk from stream A or B
socket.emit('audio_chunk', {
  session_id: 'uuid',
  stream_id: 'A',  // or 'B'
  audio_data: base64EncodedAudio,
  timestamp: Date.now()
});

// End session
socket.emit('end_session', {
  session_id: 'uuid'
});
```

### Server → Client

```javascript
// Raw transcript from Stream A
socket.on('transcript_raw_A', (data) => {
  // {
  //   text: "Hello world",
  //   confidence: 0.95,
  //   is_final: true,
  //   words: [{word: "Hello", confidence: 0.98}, ...]
  // }
});

// Raw transcript from Stream B
socket.on('transcript_raw_B', (data) => { ... });

// Consolidated transcript (merged by LLM)
socket.on('transcript_consolidated', (data) => {
  // {
  //   text: "John: Hello everyone, good morning!",
  //   speaker: "John",
  //   confidence: 0.93,
  //   merged_from: ['A', 'B'],
  //   timestamp: 1699123456789
  // }
});

// Confidence heatmap update
socket.on('confidence_update', (data) => {
  // {
  //   words: [
  //     {word: "Hello", confidence: 0.98, color: "rgb(0,200,0)"},
  //     {word: "world", confidence: 0.65, color: "rgb(200,200,0)"}
  //   ]
  // }
});
```

---

## Environment Variables

```env
# Flask
FLASK_SECRET_KEY=your-secret-key
FLASK_ENV=development
FLASK_DEBUG=True

# Google Cloud STT
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
GOOGLE_CLOUD_PROJECT=your-project-id

# OpenAI (for Whisper + GPT-4 consolidation)
OPENAI_API_KEY=sk-...

# Deepgram (optional)
DEEPGRAM_API_KEY=your-deepgram-key

# LLM for Consolidation
CONSOLIDATION_LLM=openai  # or 'anthropic', 'google'
ANTHROPIC_API_KEY=sk-ant-...  # if using Claude

# Audio Settings
MAX_AUDIO_BUFFER_SIZE=10485760  # 10MB
CHUNK_DURATION_MS=1000
SAMPLE_RATE=16000

# Consolidation Settings
CONSOLIDATION_CONTEXT_WINDOW=5
CONSOLIDATION_MERGE_THRESHOLD=0.7
ENABLE_SPEAKER_DETECTION=true

# Admin
ADMIN_PASSWORD=your-admin-password
```

---

## Implementation Phases

### Phase 1: Foundation (Week 1)
- [ ] Project setup & structure
- [ ] Flask + SocketIO base
- [ ] Single stream audio capture
- [ ] Google Cloud STT integration
- [ ] Basic UI with one stream

### Phase 2: Dual Stream (Week 2)
- [ ] Dual audio stream capture
- [ ] Stream manager implementation
- [ ] Parallel STT processing
- [ ] Display both transcripts side-by-side

### Phase 3: Confidence Scores (Week 3)
- [ ] Word-level confidence extraction
- [ ] Confidence tracker service
- [ ] Heatmap visualization UI
- [ ] Color-coded transcript display

### Phase 4: LLM Consolidation (Week 4)
- [ ] Consolidation service with GPT-4
- [ ] Context window management
- [ ] Merge logic implementation
- [ ] Speaker attribution

### Phase 5: Speaker Management (Week 5)
- [ ] Speaker name extraction
- [ ] Manual speaker assignment
- [ ] Speaker matching across streams
- [ ] Speaker history tracking

### Phase 6: Admin Panel (Week 6)
- [ ] Engine configuration UI
- [ ] LLM settings management
- [ ] Speaker management interface
- [ ] Analytics & session history

### Phase 7: Multi-Engine Support (Week 7)
- [ ] Abstract STT interface
- [ ] OpenAI Whisper implementation
- [ ] Deepgram implementation
- [ ] Engine switching logic

---

## API Endpoints

```
GET  /                      # Main transcription UI
GET  /admin                 # Admin panel
POST /api/sessions          # Create new session
GET  /api/sessions/:id      # Get session details
POST /api/sessions/:id/end  # End session
GET  /api/transcripts/:session_id  # Get all transcripts for session
POST /api/speakers          # Register speaker name
GET  /api/settings          # Get all settings
POST /api/settings          # Update settings
```

---

## Next Steps

1. **Initialize git repository**
2. **Set up Google Cloud STT credentials**
3. **Create virtual environment & install dependencies**
4. **Implement Phase 1: Foundation**

Ready to start implementation?
