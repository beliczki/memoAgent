# memoAgent

Real-time dual-stream audio transcription with AI-powered consolidation and confidence scoring.

## Features

✨ **Dual-Stream Processing** - Transcribe two audio sources simultaneously
🎯 **Multi-Engine Support** - Google Cloud STT, OpenAI Whisper, Deepgram
🤖 **LLM Consolidation** - Intelligently merge transcripts using GPT-4/Claude
📊 **Confidence Scores** - Word-level probability visualization
👤 **Speaker Management** - Auto-detect and manage speaker names
⚙️ **Admin Panel** - Configure engines and settings in real-time

## Quick Start

### Prerequisites

- Python 3.10+
- Google Cloud account with Speech-to-Text API enabled
- OpenAI API key (for LLM consolidation)

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
# Edit .env with your API keys

# Initialize database
python -c "from app.models import init_db; init_db()"

# Run application
python run.py
```

Visit `http://localhost:5000` to start transcribing!

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for detailed technical documentation.

## Configuration

### Google Cloud STT

1. Create service account at [Google Cloud Console](https://console.cloud.google.com)
2. Download JSON credentials
3. Set environment variable:
   ```bash
   export GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
   ```

### OpenAI (for consolidation)

```bash
export OPENAI_API_KEY=sk-...
```

## Usage

### Basic Dual-Stream Transcription

1. Open app in browser
2. Click "Start Session"
3. Select audio sources for Stream A and Stream B
4. Click "Start Recording"
5. Watch real-time transcription with confidence scores

### Admin Panel

Access at `/admin` to:
- Switch between STT engines
- Configure LLM consolidation settings
- Manage speaker names
- View session history

## API

### WebSocket Events

**Start Session:**
```javascript
socket.emit('start_session', {
  stt_engine_a: 'google',
  stt_engine_b: 'google',
  enable_consolidation: true
});
```

**Send Audio:**
```javascript
socket.emit('audio_chunk', {
  session_id: 'uuid',
  stream_id: 'A',  // or 'B'
  audio_data: base64Audio
});
```

**Receive Consolidated Transcript:**
```javascript
socket.on('transcript_consolidated', (data) => {
  console.log(data.text, data.speaker, data.confidence);
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

Words are color-coded by confidence:

- 🟢 **Green** (0.9-1.0): High confidence, certain
- 🟡 **Yellow** (0.7-0.9): Medium confidence
- 🔴 **Red** (0.0-0.7): Low confidence, likely guessed

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

- [ ] Phase 1: Foundation (single stream, basic UI)
- [ ] Phase 2: Dual stream support
- [ ] Phase 3: Confidence scores & heatmap
- [ ] Phase 4: LLM consolidation
- [ ] Phase 5: Speaker management
- [ ] Phase 6: Admin panel
- [ ] Phase 7: Multi-engine support

---

**Built with ❤️ by [beliczki](https://github.com/beliczki)**
