# memoAgent

Intelligent meeting and conference transcription orchestrator with **multi-source audio capture** and **meeting bot integration**.

## Concept

memoAgent captures audio from multiple sources (conferences, local mic, online meetings) and routes it to the [Advanced Transcriber](https://github.com/beliczki/advanced-transcriber) service for dual-engine transcription with LLM consolidation.

## Architecture

memoAgent is **part of a two-project ecosystem**:

1. **[Advanced Transcriber](https://github.com/beliczki/advanced-transcriber)** - Headless transcription service (dual STT engines + LLM)
2. **memoAgent** (this project) - Audio source orchestrator and meeting integration

See [PROJECTS_OVERVIEW.md](./PROJECTS_OVERVIEW.md) for complete architecture.

## Features

🎤 **Multi-Source Audio Input**:
- **Conference Mode** - Mixer line out (mono) for live events
- **Local Microphone** - Direct mic capture
- **Meeting Bot** - Join Google Meet / MS Teams as participant

🤖 **Meeting Bot Integration**:
- Join Google Meet meetings as bot
- Join Microsoft Teams meetings as bot
- Capture audio + speaker metadata
- Real-time transcription

📡 **Transcription Service Client**:
- Connects to Advanced Transcriber via WebSocket
- Streams audio from selected source
- Receives transcripts with confidence scores
- Displays engine agreement/disagreement

👤 **Smart Speaker Detection** - From meeting metadata and content
📊 **Confidence Visualization** - See word-level confidence
⚙️ **Source Selection UI** - Choose audio input easily

## Quick Start

### Prerequisites

- Python 3.10+
- **[Advanced Transcriber](https://github.com/beliczki/advanced-transcriber)** running (see separate repo)
- For Meeting Bots:
  - Google Cloud project (for Google Meet)
  - Azure Bot Service (for MS Teams)

### Installation

```bash
# Clone repository
git clone https://github.com/beliczki/memoAgent.git
cd memoAgent

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with Transcriber URL and OAuth credentials

# Initialize database
python -c "from app.models import init_db; init_db()"

# Run application
python run.py
```

Visit `http://localhost:5000` to select audio source and start transcribing!

## Architecture

See [PROJECTS_OVERVIEW.md](./PROJECTS_OVERVIEW.md) for complete two-project architecture.

## Configuration

### Advanced Transcriber Connection

```env
TRANSCRIBER_URL=ws://localhost:5001/transcribe
```

### Google Meet Bot (Optional)

1. Create project at [Google Cloud Console](https://console.cloud.google.com)
2. Enable Google Meet API
3. Create OAuth 2.0 credentials
4. Add to .env:
   ```bash
   GOOGLE_OAUTH_CLIENT_ID=...
   GOOGLE_OAUTH_CLIENT_SECRET=...
   ```

### Microsoft Teams Bot (Optional)

1. Create bot at [Azure Bot Service](https://portal.azure.com)
2. Register Teams app
3. Add to .env:
   ```bash
   MICROSOFT_APP_ID=...
   MICROSOFT_APP_PASSWORD=...
   ```

## Usage

### Conference Mode

1. Connect mixer line out to computer audio interface
2. Open memoAgent at `http://localhost:5000`
3. Select "Conference Mode"
4. Start transcription
5. Audio streams to Advanced Transcriber
6. View real-time transcript with confidence scores

### Local Microphone Mode

1. Open memoAgent
2. Select "Local Microphone"
3. Choose your microphone device
4. Start recording
5. View transcription in real-time

### Meeting Bot Mode (Google Meet)

1. Open memoAgent
2. Select "Join Google Meet"
3. Paste meeting link or code
4. Authenticate with Google OAuth
5. Bot joins meeting as "memoAgent Bot"
6. Real-time transcription with speaker names
7. Leave meeting when done

### Meeting Bot Mode (MS Teams)

1. Open memoAgent
2. Select "Join Teams Meeting"
3. Paste meeting link
4. Bot joins via Bot Framework
5. Real-time transcription with participants
6. Leave meeting when done

## API

### WebSocket Events

**Start Session:**
```javascript
socket.emit('start_session', {
  engine_1: 'google',
  engine_2: 'whisper',
  enable_consolidation: true
});
```

**Send Audio** (goes to both engines):
```javascript
socket.emit('audio_chunk', {
  session_id: 'uuid',
  audio_data: base64Audio
});
```

**Receive Consolidated Transcript:**
```javascript
socket.on('transcript_consolidated', (data) => {
  console.log(data.text, data.disagreements, data.confidence);
});
```

### REST Endpoints

```
GET  /                      # Main UI
GET  /admin                 # Admin panel
POST /api/sessions          # Create session
GET  /api/transcripts/:id   # Get transcripts
POST /api/speakers          # Register speaker
```

## Development

### Project Structure

```
memoAgent/
├── app/                    # Application code
│   ├── routes/            # HTTP routes
│   ├── services/          # Business logic
│   │   └── stt/          # STT engine implementations
│   ├── static/           # Frontend assets
│   └── templates/        # HTML templates
├── data/                  # Database & sessions
├── tests/                 # Unit tests
└── run.py                # Entry point
```

### Running Tests

```bash
pytest tests/
```

### Adding a New STT Engine

1. Create `app/services/stt/your_engine.py`
2. Extend `BaseSTTEngine` interface
3. Implement `transcribe()` and `configure()`
4. Register in admin panel

## Confidence Score Visualization

Words are color-coded by engine agreement and confidence:

- 🟢 **Green**: Both engines agree + high confidence
- 🟡 **Yellow**: Engines agree but lower confidence
- 🟠 **Orange**: Engines disagree, LLM picked best
- 🔴 **Red**: Engines disagree + low confidence

## Troubleshooting

**No audio detected:**
- Check microphone permissions in browser
- Verify sample rate (16kHz recommended)
- Check browser console for errors

**Low confidence scores:**
- Ensure quiet environment
- Speak clearly and at moderate pace
- Check microphone quality

**Consolidation not working:**
- Verify OpenAI API key is set
- Check that both streams have recent data
- Review consolidation logs in console

## Contributing

Contributions welcome! Please read [CONTRIBUTING.md](./CONTRIBUTING.md) first.

## License

MIT License - see [LICENSE](./LICENSE)

## Roadmap

### memoAgent (This Project)
- [ ] Phase 1: Basic UI and source selection
- [ ] Phase 2: WebSocket client for Transcriber
- [ ] Phase 3: Conference mode (line-in audio capture)
- [ ] Phase 4: Local microphone mode
- [ ] Phase 5: Google Meet bot integration
- [ ] Phase 6: Microsoft Teams bot integration
- [ ] Phase 7: Advanced features (session recording, export, analytics)

### Advanced Transcriber (Separate Project)
See [beliczki/advanced-transcriber](https://github.com/beliczki/advanced-transcriber) repository for transcription service roadmap.

---

**Built with ❤️ by [beliczki](https://github.com/beliczki)**
