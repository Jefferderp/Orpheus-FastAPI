
This is a fork of https://github.com/Lex-au/Orpheus-FastAPI, with changes incorporated from multiple branches. The goal of this fork is to "cut the fat" as much as possible, leaving only a lean OpenAI-compatible API for fast inference.

---

# Orpheus-FASTAPI API

High-performance Text-to-Speech API server with OpenAI-compatible endpoints, multilingual support with 24 voices, and emotion tags. Optimized for RTX GPUs.

## Features

- **OpenAI API Compatible**: Drop-in replacement for OpenAI's `/v1/audio/speech` endpoint
- **High Performance**: Optimized for RTX GPUs with parallel processing
- **Multilingual Support**: 24 different voices across 8 languages (English, French, German, Korean, Hindi, Mandarin, Spanish, Italian)
- **Emotion Tags**: Support for laughter, sighs, and other emotional expressions
- **Unlimited Audio Length**: Generate audio of any length through intelligent batching
- **Smooth Transitions**: Crossfaded audio segments for seamless listening experience

## Setup

### Prerequisites

- Python 3.8-3.11 (Python 3.12 is not supported due to removal of pkgutil.ImpImporter)
- CUDA-compatible GPU (recommended: RTX series for best performance)
- Using docker compose or separate LLM inference server running the Orpheus model (e.g., LM Studio or llama.cpp server)

### FastAPI Service Native Installation

1. Clone the repository:
```bash
git clone https://github.com/Jefferderp/Orpheus-FastAPI.git
cd Orpheus-FastAPI
```

2. Create a Python virtual environment:
```bash
# Using venv (Python's built-in virtual environment)
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

3. Install PyTorch with CUDA support:
```bash
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
```

4. Install other dependencies:
```bash
pip3 install -r requirements.txt
```

### Starting the Server

Run the FastAPI server:
```bash
python app.py
```

Or with specific host/port:
```bash
uvicorn app:app --host 0.0.0.0 --port 5005 --reload
```

### Streaming Endpoint

The server also provides a streaming endpoint at `/v1/audio/speech/stream` for real-time audio generation:

```bash
curl http://localhost:5005/v1/audio/speech/stream \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_api_key_here" \
  -d '{
    "input": "This is a streaming test of the Orpheus TTS system.",
    "voice": "tara",
    "response_format": "wav"
  }' \
  --output streaming_speech.wav
```

### Additional Streaming Endpoint

An additional streaming endpoint is available at `/api/tts/stream`:

```bash
curl -X POST http://localhost:5005/api/tts/stream \
  -H "Content-Type: application/json" \
  -d '{
    "text": "This is another streaming test.",
    "voice": "tara"
  }' \
  --output streaming_speech2.wav
```

### Available Voices

#### English
- `tara`: Female, conversational, clear
- `leah`: Female, warm, gentle
- `jess`: Female, energetic, youthful
- `leo`: Male, authoritative, deep
- `dan`: Male, friendly, casual
- `mia`: Female, professional, articulate
- `zac`: Male, enthusiastic, dynamic
- `zoe`: Female, calm, soothing

### Emotion Tags

You can insert emotion tags into your text to add expressiveness:

- `<laugh>`: Add laughter
- `<sigh>`: Add a sigh
- `<chuckle>`: Add a chuckle
- `<cough>`: Add a cough sound
- `<sniffle>`: Add a sniffle sound
- `<groan>`: Add a groan
- `<yawn>`: Add a yawning sound
- `<gasp>`: Add a gasping sound

## Technical Details

This server works as a frontend that connects to an external LLM inference server. It sends text prompts to the inference server, which generates tokens that are then converted to audio using the SNAC model. The system has been optimised for RTX 4090 GPUs with:

- Vectorised tensor operations
- Parallel processing with CUDA streams
- Efficient memory management
- Token and audio caching
- Optimised batch sizes

### Hardware Detection and Optimization

The system features intelligent hardware detection that automatically optimizes performance based on your hardware capabilities:

- **High-End GPU Mode** (dynamically detected based on capabilities):
  - Triggered by either: 16GB+ VRAM, compute capability 8.0+, or 12GB+ VRAM with 7.0+ compute capability
  - Advanced parallel processing with 4 workers
  - Optimized batch sizes (32 tokens)
  - High-throughput parallel file I/O
  - Full hardware details displayed (name, VRAM, compute capability)
  - GPU-specific optimizations automatically applied

- **Standard GPU Mode** (other CUDA-capable GPUs):
  - Efficient parallel processing
  - GPU-optimized parameters
  - CUDA acceleration where beneficial
  - Detailed GPU specifications

- **CPU Mode** (when no GPU is available):
  - Conservative processing with 2 workers
  - Optimized memory usage
  - Smaller batch sizes (16 tokens)
  - Sequential file I/O
  - Detailed CPU cores, threads, and RAM information

No manual configuration is needed - the system automatically detects hardware capabilities and adapts for optimal performance across different generations of GPUs and CPUs.

### Token Processing Optimization

The token processing system has been optimized with mathematically aligned parameters:
- Uses a context window of 49 tokens (7²)
- Processes in batches of 7 tokens (Orpheus model standard)
- This square relationship ensures complete token processing with no missed tokens
- Results in cleaner audio generation with proper token alignment
- Repetition penalty fixed at 1.1 for optimal quality generation (cannot be changed)

### Long Text Processing

The system features efficient batch processing for texts of any length:
- Automatically detects longer inputs (>1000 characters) 
- Splits text at logical points to create manageable chunks
- Processes each chunk independently for reliability
- Combines audio segments with smooth 50ms crossfades
- Intelligently stitches segments in-memory for consistent output
- Handles texts of unlimited length with no truncation
- Provides detailed progress reporting for each batch

**Note about long-form audio**: While the system now supports texts of unlimited length, there may be slight audio discontinuities between segments due to architectural constraints of the underlying model. The Orpheus model was designed for short to medium text segments, and our batching system works around this limitation by intelligently splitting and stitching content with minimal audible impact.

### Integration with OpenWebUI

You can easily integrate this TTS solution with [OpenWebUI](https://github.com/open-webui/open-webui) to add high-quality voice capabilities to your chatbot:

1. Start your Orpheus-FASTAPI server
2. In OpenWebUI, go to Admin Panel > Settings > Audio
3. Change TTS from Web API to OpenAI
4. Set APIBASE URL to your server address (e.g., `http://localhost:5005/v1`)
5. Set API Key to your configured API key or "not-needed" if API key authentication is not enabled
6. Set TTS Voice to one of the available voices: `tara`, `leah`, `jess`, `leo`, `dan`, `mia`, `zac`, or `zoe`
7. Set TTS Model to `tts-1`

### External Inference Server

This application requires a separate LLM inference server running the Orpheus model. For easy setup, use Docker Compose, which automatically handles this for you. Alternatively, you can use:

- [GPUStack](https://github.com/gpustack/gpustack) - GPU optimised LLM inference server (My pick) - supports LAN/WAN tensor split parallelisation
- [LM Studio](https://lmstudio.ai/) - Load the GGUF model and start the local server
- [llama.cpp server](https://github.com/ggerganov/llama.cpp) - Run with the appropriate model parameters
- Any compatible OpenAI API-compatible server

The inference server should be configured to expose an API endpoint that this FastAPI application will connect to.

### Environment Variables

Configure in docker compose, if using docker. Not using docker; create a `.env` file:

- `ORPHEUS_API_URL`: URL of the LLM inference API (default in Docker: http://llama-cpp-server:5006/v1/completions)
- `ORPHEUS_API_TIMEOUT`: Timeout in seconds for API requests (default: 120)
- `ORPHEUS_API_KEY`: API key for authentication with the OpenAI-compatible API (optional)
- `ORPHEUS_MAX_TOKENS`: Maximum tokens to generate (default: 8192)
- `ORPHEUS_TEMPERATURE`: Temperature for generation (default: 0.6)
- `ORPHEUS_TOP_P`: Top-p sampling parameter (default: 0.9)
- `ORPHEUS_SAMPLE_RATE`: Audio sample rate in Hz (default: 24000)
- `ORPHEUS_PORT`: Web server port (default: 5005)
- `ORPHEUS_HOST`: Web server host (default: 0.0.0.0)
- `ORPHEUS_MODEL_NAME`: Model name for inference server

The system now supports loading environment variables from a `.env` file in the project root, making it easier to configure without modifying system-wide environment settings. See `.env.example` for a template.

Note: Repetition penalty is hardcoded to 1.1 and cannot be changed through environment variables as this is the only value that produces stable, high-quality output.

Make sure the `ORPHEUS_API_URL` points to your running inference server.

## Development

### Project Components

- **app.py**: FastAPI server that handles HTTP requests and API endpoints
- **tts_engine/inference.py**: Handles token generation and API communication 
- **tts_engine/speechpipe.py**: Converts token sequences to audio using the SNAC model

### Adding New Voices

To add new voices, update the `AVAILABLE_VOICES` list in `tts_engine/inference.py`.

## Using with llama.cpp

When running the Orpheus model with llama.cpp, use these parameters to ensure optimal performance:

```bash
./llama-server -m models/Modelname.gguf \
  --ctx-size={{your ORPHEUS_MAX_TOKENS from .env}} \
  --n-predict={{your ORPHEUS_MAX_TOKENS from .env}} \
  --rope-scaling=linear
```

Important parameters:
- `--ctx-size`: Sets the context window size, should match your ORPHEUS_MAX_TOKENS setting
- `--n-predict`: Maximum tokens to generate, should match your ORPHEUS_MAX_TOKENS setting
- `--rope-scaling=linear`: Required for optimal positional encoding with the Orpheus model

For extended audio generation (books, long narrations), you may want to increase your token limits:
1. Set ORPHEUS_MAX_TOKENS to 32768 or higher in your .env file
2. Increase ORPHEUS_API_TIMEOUT to 1800 for longer processing times
3. Use the same values in your llama.cpp parameters (if you're using llama.cpp)

## License

This project is licensed under the Apache License 2.0 - see the LICENSE.txt file for details.
