# memoAgent

Real-time audio transcription with **dual-engine processing** for improved accuracy through AI-powered consolidation.

## Concept

Process **one audio stream** through **two different STT engines** (e.g., Google + Whisper) simultaneously, then use an LLM to consolidate both transcripts into the best possible interpretation. Think of it as getting a "second opinion" on your transcription.

## Features

✨ **Dual-Engine Processing** - One mic, two STT engines for redundancy
🎯 **Multi-Engine Support** - Google Cloud STT, OpenAI Whisper, Deepgram
🤖 **LLM Consolidation** - GPT-4/Claude picks the best interpretation
📊 **Confidence Comparison** - See where engines agree/disagree
👤 **Speaker Detection** - Auto-identify speakers from context
⚙️ **Admin Panel** - Switch engines and configure settings

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

### Basic Transcription

1. Open app in browser
2. Click "Start Session"
3. Select two STT engines (e.g., Google + Whisper)
4. Click "Start Recording" (single microphone)
5. Watch both engines transcribe in real-time
6. See consolidated result with confidence heatmap

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

- [ ] Phase 1: Single engine MVP
- [ ] Phase 2: Dual engine processing
- [ ] Phase 3: LLM consolidation
- [ ] Phase 4: Confidence visualization & heatmap
- [ ] Phase 5: Speaker detection
- [ ] Phase 6: Admin panel
- [ ] Phase 7: Advanced features (3+ engines, export, analytics)

---

**Built with ❤️ by [beliczki](https://github.com/beliczki)**
