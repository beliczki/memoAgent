# memoAgent - Dual-Engine Real-Time Transcription System

## Overview

memoAgent is an advanced real-time audio transcription service that processes a **single audio stream through TWO different STT engines simultaneously** for improved accuracy through redundancy and LLM-powered consolidation.

## Core Concept

Think of it as getting a "second opinion" on your transcription:
- **One microphone** captures your voice
- **Two STT engines** (e.g., Google + Whisper) transcribe the same audio independently
- **LLM consolidates** both transcripts, using confidence scores to pick the best interpretation
- **Result:** Higher accuracy, fewer mistakes, confidence visualization

## Core Capabilities

1. **Dual-Engine Processing** - Process single audio through 2 STT engines (Google, Whisper, Deepgram)
2. **LLM Consolidation** - Intelligently merge two interpretations of the same audio
3. **Confidence Comparison** - See which engine is more confident for each word
4. **Speaker Detection** - Identify and label speakers automatically
5. **Admin Panel** - Switch engines, configure consolidation, manage speakers

---

## Architecture Diagram

```
┌─────────────────────┐
│   Single Audio      │
│   Source (Mic)      │
└──────────┬──────────┘
           │ WebSocket (Audio Chunks)
           ▼
┌──────────────────────────────────────────────────────────┐
│             Flask + SocketIO Web Server                  │
│  ┌────────────────────────────────────────────────────┐  │
│  │     Audio Distributor (Duplicate to both engines)  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
           │                        │
           │ (same audio)          │ (same audio)
           ▼                        ▼
    ┌─────────────┐          ┌─────────────┐
    │ STT Engine 1│          │ STT Engine 2│
    │  (Google)   │          │  (Whisper)  │
    └─────┬───────┘          └─────┬───────┘
          │                         │
          │ Transcript A            │ Transcript B
          │ + Confidence scores     │ + Confidence scores
          │                         │
          └──────────┬──────────────┘
                     ▼
          ┌──────────────────────────┐
          │  LLM Consolidation       │
          │  (GPT-4 / Claude)        │
          │                          │
          │  Compares:               │
          │  - Word-level confidence │
          │  - Contextual fit        │
          │  - Previous sentences    │
          │                          │
          │  Outputs:                │
          │  - Best transcript       │
          │  - Disagreement markers  │
          │  - Speaker labels        │
          └──────────┬───────────────┘
                     ▼
          ┌──────────────────────────┐
          │  Unified Transcript      │
          │  + Confidence Heatmap    │
          │  + Engine Comparison     │
          │  + Speaker Attribution   │
          └──────────┬───────────────┘
                     ▼
          ┌──────────────────────────┐
          │  WebSocket → Client      │
          │  SQLite Storage          │
          └──────────────────────────┘
```

---

## Technology Stack

### Backend
- **Python 3.10+** - Core language
- **Flask** - Web framework
- **Flask-SocketIO** - Real-time WebSocket
- **Google Cloud Speech-to-Text** - Primary STT engine
- **OpenAI Whisper API** - Secondary STT engine
- **Deepgram API** - Optional third engine
- **OpenAI GPT-4 / Anthropic Claude** - Consolidation LLM
- **SQLite** - Transcript storage

### Frontend
- **HTML5 + CSS3** - UI
- **Socket.IO Client** - Real-time communication
- **MediaRecorder API** - Single audio capture
- **Vanilla JavaScript** - Minimal dependencies

### Audio Processing
- **pydub** - Audio manipulation
- **webrtcvad** - Voice activity detection

---

## How It Works

### 1. Audio Capture

```javascript
// Single microphone stream
navigator.mediaDevices.getUserMedia({ audio: true })
  .then(stream => {
    const recorder = new MediaRecorder(stream);

    recorder.ondataavailable = (event) => {
      // Send audio chunk to server
      socket.emit('audio_chunk', {
        session_id: sessionId,
        audio_data: base64Audio,
        timestamp: Date.now()
      });
    };
  });
```

### 2. Server Distributes to Both Engines

```python
class AudioDistributor:
    """Sends same audio to multiple STT engines."""

    async def process_chunk(self, audio_data: bytes, session_id: str):
        # Send to both engines in parallel
        results = await asyncio.gather(
            self.engine_1.transcribe(audio_data),  # Google
            self.engine_2.transcribe(audio_data),  # Whisper
            return_exceptions=True
        )

        google_result, whisper_result = results

        # Consolidate
        final = await self.consolidate(google_result, whisper_result)

        return final
```

### 3. LLM Consolidation

The LLM compares both transcripts and chooses the best interpretation:

```python
class ConsolidationService:
    async def consolidate(
        self,
        transcript_google: TranscriptResult,
        transcript_whisper: TranscriptResult,
        context: str = ""
    ) -> ConsolidatedResult:
        """
        Compare two transcriptions of the SAME audio.
        """

        prompt = f"""
        You are consolidating two transcriptions of the SAME audio.

        Google STT (confidence: {transcript_google.confidence}):
        "{transcript_google.text}"

        Word-level confidence:
        {self.format_word_confidence(transcript_google.words)}

        Whisper STT (confidence: {transcript_whisper.confidence}):
        "{transcript_whisper.text}"

        Context (previous sentences):
        "{context}"

        Task:
        1. Compare both transcriptions
        2. Where they agree, use that text
        3. Where they disagree, choose based on:
           - Word confidence scores
           - Contextual fit
           - Language model probability
        4. Identify speaker if mentioned
        5. Mark low-confidence words

        Return JSON:
        {{
          "text": "best consolidated transcript",
          "speaker": "name or null",
          "disagreements": [
            {{"word": "X", "google": "hello", "whisper": "yellow", "chosen": "hello", "reason": "higher confidence"}}
          ],
          "low_confidence_spans": [
            {{"text": "might be wrong", "confidence": 0.65}}
          ]
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
            speaker=result.get('speaker'),
            disagreements=result['disagreements'],
            low_confidence_spans=result['low_confidence_spans']
        )
```

---

## Benefits of Dual-Engine Approach

### 1. **Higher Accuracy**
- Two engines rarely make the same mistake
- LLM picks the correct interpretation
- Reduces transcription errors by ~30-40%

### 2. **Confidence Validation**
- If both engines agree → high confidence
- If engines disagree → LLM arbitrates
- User sees where engines disagreed

### 3. **Engine-Specific Strengths**
- **Google:** Better with technical terms, proper nouns
- **Whisper:** Better with accents, background noise
- **LLM:** Picks the best of both

### 4. **Redundancy**
- If one engine fails, you still have the other
- Graceful degradation

---

## Directory Structure

```
memoAgent/
├── app/
│   ├── __init__.py                  # Flask app factory
│   ├── config.py                    # Configuration
│   ├── models.py                    # Database models
│   ├── sockets.py                   # SocketIO handlers
│   │
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── main.py                  # Main transcription UI
│   │   ├── admin.py                 # Admin panel
│   │   └── api.py                   # REST API endpoints
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── audio_distributor.py     # Sends audio to both engines
│   │   ├── stt/
│   │   │   ├── __init__.py
│   │   │   ├── base.py              # Abstract STT interface
│   │   │   ├── google_stt.py        # Google Cloud STT
│   │   │   ├── whisper_stt.py       # OpenAI Whisper
│   │   │   └── deepgram_stt.py      # Deepgram STT
│   │   ├── consolidation_service.py # LLM-powered merging
│   │   ├── confidence_tracker.py    # Confidence visualization
│   │   └── speaker_manager.py       # Speaker detection
│   │
│   ├── static/
│   │   ├── css/
│   │   │   ├── main.css
│   │   │   └── admin.css
│   │   └── js/
│   │       ├── recorder.js          # Audio capture
│   │       ├── transcript-ui.js     # Display with heatmap
│   │       └── admin.js             # Admin panel
│   │
│   └── templates/
│       ├── base.html
│       ├── index.html               # Main UI
│       ├── admin.html               # Admin panel
│       └── components/
│           ├── confidence-heatmap.html
│           └── engine-comparison.html
│
├── data/
│   ├── memoagent.db                 # SQLite database
│   └── sessions/                    # Temp audio storage
│
├── tests/
│   ├── test_consolidation.py
│   ├── test_engines.py
│   └── test_confidence.py
│
├── .env.example
├── .gitignore
├── requirements.txt
├── run.py
├── README.md
└── ARCHITECTURE.md                  # This file
```

---

## Core Components

### 1. Audio Distributor

```python
class AudioDistributor:
    """
    Takes single audio stream and sends to multiple STT engines.
    """
    def __init__(self, engines: List[BaseSTTEngine]):
        self.engines = engines
        self.consolidator = ConsolidationService()

    async def process(self, audio_data: bytes, session_id: str):
        """Process audio through all engines in parallel."""

        # Transcribe with all engines simultaneously
        tasks = [engine.transcribe(audio_data) for engine in self.engines]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Filter out failures
        valid_results = [r for r in results if isinstance(r, TranscriptResult)]

        if len(valid_results) == 0:
            raise Exception("All STT engines failed")

        if len(valid_results) == 1:
            # Only one engine succeeded, return it
            return valid_results[0]

        # Consolidate multiple results
        consolidated = await self.consolidator.consolidate(
            *valid_results,
            context=self.get_context(session_id)
        )

        return consolidated
```

### 2. STT Engine Interface

```python
class BaseSTTEngine(ABC):
    """Abstract base for all STT engines."""

    @abstractmethod
    async def transcribe(self, audio_data: bytes) -> TranscriptResult:
        """
        Transcribe audio.

        Returns:
            TranscriptResult with:
            - text: str
            - confidence: float (0.0-1.0)
            - words: List[WordResult]  # Word-level confidence
            - is_final: bool
            - engine_name: str
        """
        pass
```

### 3. Consolidation Service

```python
class ConsolidationService:
    """
    LLM-powered consolidation of multiple transcripts from SAME audio.
    """

    def __init__(self):
        self.llm = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))

    async def consolidate(
        self,
        *transcripts: TranscriptResult,
        context: str = ""
    ) -> ConsolidatedResult:
        """
        Compare transcripts and pick best interpretation.

        For each word/phrase:
        - If engines agree → use that
        - If engines disagree → LLM picks based on:
          * Confidence scores
          * Context
          * Language model probability
        """

        if len(transcripts) == 1:
            return self.single_to_consolidated(transcripts[0])

        # Build comparison prompt
        comparison = self.build_comparison(transcripts)

        prompt = f"""
        Consolidate {len(transcripts)} transcriptions of the SAME audio.

        {comparison}

        Previous context: "{context}"

        Rules:
        1. Where engines agree, use that text
        2. Where engines disagree, choose based on confidence + context
        3. Identify speaker names if mentioned
        4. Mark disagreements for user transparency

        Return JSON with best consolidated transcript.
        """

        # LLM processing...
        # (implementation details as shown earlier)

        return consolidated_result
```

### 4. Confidence Tracker

```python
class ConfidenceTracker:
    """
    Visualize confidence scores and engine comparisons.
    """

    def generate_heatmap(self, consolidated: ConsolidatedResult):
        """
        Create heatmap data showing:
        - Green: Both engines agree + high confidence
        - Yellow: Engines agree but lower confidence
        - Orange: Engines disagree, moderate confidence
        - Red: Engines disagree, low confidence
        """

        heatmap = []

        for word in consolidated.words:
            color = self.calculate_color(
                google_conf=word.google_confidence,
                whisper_conf=word.whisper_confidence,
                engines_agree=word.google_text == word.whisper_text
            )

            heatmap.append({
                "word": word.chosen_text,
                "color": color,
                "google": {"text": word.google_text, "conf": word.google_confidence},
                "whisper": {"text": word.whisper_text, "conf": word.whisper_confidence},
                "disagreement": word.google_text != word.whisper_text
            })

        return heatmap

    def calculate_color(self, google_conf, whisper_conf, engines_agree):
        avg_conf = (google_conf + whisper_conf) / 2

        if engines_agree and avg_conf >= 0.9:
            return "rgb(0, 200, 0)"  # Green - high confidence, agree
        elif engines_agree and avg_conf >= 0.7:
            return "rgb(150, 200, 0)"  # Yellow-green - medium, agree
        elif not engines_agree and avg_conf >= 0.7:
            return "rgb(255, 165, 0)"  # Orange - disagree, picked best
        else:
            return "rgb(200, 0, 0)"  # Red - low confidence or disagree
```

---

## Database Schema

```sql
-- Sessions
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME,
    engine_1 TEXT,  -- e.g., 'google'
    engine_2 TEXT,  -- e.g., 'whisper'
    consolidation_enabled BOOLEAN DEFAULT TRUE
);

-- Raw transcripts (one per engine)
CREATE TABLE raw_transcripts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    engine_name TEXT,  -- 'google', 'whisper', etc.
    text TEXT,
    confidence REAL,
    is_final BOOLEAN,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Consolidated transcripts (merged by LLM)
CREATE TABLE consolidated_transcripts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    text TEXT,
    speaker_name TEXT,
    average_confidence REAL,
    disagreement_count INTEGER,  -- How many words engines disagreed on
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (session_id) REFERENCES sessions(id)
);

-- Word-level details
CREATE TABLE transcript_words (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    consolidated_id INTEGER,
    word TEXT,
    chosen_text TEXT,  -- The final chosen version
    engine_1_text TEXT,  -- Google's version
    engine_1_confidence REAL,
    engine_2_text TEXT,  -- Whisper's version
    engine_2_confidence REAL,
    engines_agreed BOOLEAN,
    start_time REAL,
    end_time REAL,
    FOREIGN KEY (consolidated_id) REFERENCES consolidated_transcripts(id)
);

-- Disagreements (for analysis)
CREATE TABLE disagreements (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    word_id INTEGER,
    engine_1_version TEXT,
    engine_2_version TEXT,
    chosen_version TEXT,
    reason TEXT,  -- Why LLM chose this version
    FOREIGN KEY (word_id) REFERENCES transcript_words(id)
);

-- Speakers
CREATE TABLE speakers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    name TEXT,
    first_detected_at DATETIME,
    detection_method TEXT,  -- 'self_identification', 'context', 'manual'
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
// Start session with selected engines
socket.emit('start_session', {
  engine_1: 'google',
  engine_2: 'whisper',
  enable_consolidation: true
});

// Send audio chunk (goes to BOTH engines)
socket.emit('audio_chunk', {
  session_id: 'uuid',
  audio_data: base64Audio,
  timestamp: Date.now()
});

// End session
socket.emit('end_session', {
  session_id: 'uuid'
});
```

### Server → Client

```javascript
// Raw transcript from Engine 1
socket.on('transcript_engine1', (data) => {
  // { text: "Hello world", confidence: 0.95, engine: "google" }
});

// Raw transcript from Engine 2
socket.on('transcript_engine2', (data) => {
  // { text: "Hello world", confidence: 0.89, engine: "whisper" }
});

// Consolidated transcript (best of both)
socket.on('transcript_consolidated', (data) => {
  // {
  //   text: "Hello world",
  //   confidence: 0.95,
  //   speaker: null,
  //   disagreements: [],
  //   words: [
  //     {
  //       word: "Hello",
  //       google: {text: "Hello", conf: 0.98},
  //       whisper: {text: "Hello", conf: 0.96},
  //       agreed: true
  //     },
  //     {
  //       word: "world",
  //       google: {text: "world", conf: 0.92},
  //       whisper: {text: "word", conf: 0.82},
  //       agreed: false,
  //       reason: "Google had higher confidence"
  //     }
  //   ]
  // }
});

// Engine comparison update
socket.on('engine_comparison', (data) => {
  // {
  //   total_words: 150,
  //   agreements: 142,
  //   disagreements: 8,
  //   agreement_rate: 0.947
  // }
});
```

---

## Admin Panel Features

### Engine Configuration

```
┌─────────────────────────────────────────┐
│  STT Engine Configuration               │
├─────────────────────────────────────────┤
│  Engine 1: [Google Cloud STT ▼]        │
│  Engine 2: [OpenAI Whisper ▼]          │
│                                         │
│  Available engines:                     │
│  ☑ Google Cloud Speech-to-Text         │
│  ☑ OpenAI Whisper                      │
│  ☐ Deepgram (not configured)           │
│                                         │
│  Consolidation:                         │
│  ☑ Enable LLM consolidation            │
│  LLM Model: [GPT-4 Turbo ▼]            │
│                                         │
│  [Test Engines] [Save Settings]        │
└─────────────────────────────────────────┘
```

### Engine Performance Comparison

```
┌─────────────────────────────────────────┐
│  Engine Performance (Last 100 words)    │
├─────────────────────────────────────────┤
│  Google STT:                            │
│    Avg Confidence: 0.93 ████████████▓░  │
│    Words transcribed: 100               │
│                                         │
│  Whisper:                               │
│    Avg Confidence: 0.88 ███████████░░░  │
│    Words transcribed: 100               │
│                                         │
│  Agreement Rate: 94.5% █████████████▓░  │
│  Disagreements: 5                       │
│                                         │
│  [View Disagreements] [Export Stats]   │
└─────────────────────────────────────────┘
```

---

## Environment Variables

```env
# Flask
FLASK_SECRET_KEY=your-secret-key
FLASK_ENV=development

# Google Cloud STT (Engine 1)
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
GOOGLE_CLOUD_PROJECT=your-project-id

# OpenAI (Whisper + GPT-4 consolidation)
OPENAI_API_KEY=sk-...

# Deepgram (Optional Engine 2/3)
DEEPGRAM_API_KEY=your-key

# Consolidation Settings
CONSOLIDATION_LLM=openai  # 'openai', 'anthropic', 'google'
CONSOLIDATION_MODEL=gpt-4-turbo-preview
ENABLE_CONSOLIDATION=true
CONSOLIDATION_CONTEXT_WINDOW=5

# Audio Settings
MAX_AUDIO_BUFFER_SIZE=10485760
CHUNK_DURATION_MS=1000
SAMPLE_RATE=16000

# Admin
ADMIN_PASSWORD=your-admin-password
```

---

## Implementation Phases

### Phase 1: Single Engine MVP (Week 1)
- [ ] Flask + SocketIO setup
- [ ] Single audio stream capture
- [ ] Google STT integration
- [ ] Basic UI with transcript display

### Phase 2: Dual Engine Processing (Week 2)
- [ ] Audio distributor service
- [ ] Add Whisper as second engine
- [ ] Display both transcripts side-by-side
- [ ] Show disagreements

### Phase 3: LLM Consolidation (Week 3)
- [ ] Consolidation service with GPT-4
- [ ] Merge logic based on confidence
- [ ] Context window management
- [ ] Disagreement tracking

### Phase 4: Confidence Visualization (Week 4)
- [ ] Word-level confidence heatmap
- [ ] Color-coded transcript
- [ ] Engine comparison view
- [ ] Agreement rate statistics

### Phase 5: Speaker Detection (Week 5)
- [ ] Speaker name extraction
- [ ] Context-based speaker matching
- [ ] Manual speaker assignment
- [ ] Speaker history

### Phase 6: Admin Panel (Week 6)
- [ ] Engine configuration UI
- [ ] Performance comparison dashboard
- [ ] Settings management
- [ ] Session history

### Phase 7: Advanced Features (Week 7)
- [ ] Add Deepgram as optional 3rd engine
- [ ] Export transcripts (TXT, JSON, SRT)
- [ ] Search functionality
- [ ] Analytics dashboard

---

## Next Steps

1. **Set up Google Cloud STT credentials**
2. **Get OpenAI API key** (for Whisper + GPT-4)
3. **Create Python virtual environment**
4. **Start Phase 1 implementation**

Ready to begin?
