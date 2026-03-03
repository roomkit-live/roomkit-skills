# Voice Provider Reference

## Voice Backends

Voice backends are pure audio transports — they handle sending/receiving audio, not speech detection.

### FastRTC (WebRTC)

```python
from roomkit.voice.backends import FastRTCBackend

backend = FastRTCBackend(
    host="0.0.0.0",
    port=8080,
)
```

Capabilities: `INTERRUPTION`, `BARGE_IN`

### RTP Backend

```python
from roomkit.voice.backends import RTPBackend

backend = RTPBackend(
    host="0.0.0.0",
    port=5004,
)
```

### SIP Backend

```python
from roomkit.voice.backends import SIPBackend

backend = SIPBackend(
    host="0.0.0.0",
    port=5060,
)
```

### Local Audio (Mic/Speaker)

```bash
uv add "roomkit[local-audio]"
```

```python
from roomkit.voice import get_local_audio_backend

backend = get_local_audio_backend()
```

Requires `sounddevice` + `numpy`. Captures mic audio and plays through speakers.

Capabilities: `INTERRUPTION`

### Mock Backend (Testing)

```python
from roomkit import MockVoiceBackend

backend = MockVoiceBackend()

# Simulate audio input
backend.simulate_audio_received(audio_bytes, sample_rate=16000)
```

## STT Providers

### Deepgram

```python
from roomkit.voice.stt import DeepgramSTTProvider, DeepgramSTTConfig

stt = DeepgramSTTProvider(
    DeepgramSTTConfig(
        api_key="...",          # DEEPGRAM_API_KEY
        model="nova-2",        # Model name
        language="en",         # Language code
    )
)
```

**Environment variable:** `DEEPGRAM_API_KEY`

### SherpaOnnx (Local)

```python
from roomkit.voice.stt import SherpaOnnxSTTProvider, SherpaOnnxSTTConfig

stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        model_path="./models/stt",
    )
)
```

No API key required — runs locally with ONNX runtime.

### Gradium

```python
from roomkit.voice.stt import GradiumSTTProvider

stt = GradiumSTTProvider()
```

### Mock STT (Testing)

```python
from roomkit import MockSTTProvider

stt = MockSTTProvider(
    transcriptions=["Hello", "How are you?"],
)
```

## TTS Providers

### ElevenLabs

```python
from roomkit.voice.tts import ElevenLabsTTSProvider, ElevenLabsTTSConfig

tts = ElevenLabsTTSProvider(
    ElevenLabsTTSConfig(
        api_key="...",                      # ELEVENLABS_API_KEY
        voice_id="21m00Tcm4TlvDq8ikWAM",  # Voice ID
        model_id="eleven_turbo_v2_5",      # Model
    )
)
```

**Environment variable:** `ELEVENLABS_API_KEY`

### SherpaOnnx (Local)

```python
from roomkit.voice.tts import SherpaOnnxTTSProvider, SherpaOnnxTTSConfig

tts = SherpaOnnxTTSProvider(
    SherpaOnnxTTSConfig(
        model_path="./models/tts",
    )
)
```

### Qwen TTS

```python
from roomkit.voice.tts import QwenTTSProvider

tts = QwenTTSProvider()
```

### Gradium

```python
from roomkit.voice.tts import GradiumTTSProvider

tts = GradiumTTSProvider()
```

### Mock TTS (Testing)

```python
from roomkit import MockTTSProvider

tts = MockTTSProvider()
```

## Realtime Voice Providers (Speech-to-Speech)

For direct speech-to-speech AI without separate STT/TTS:

### OpenAI Realtime

```python
from roomkit import RealtimeVoiceChannel
from roomkit.voice.realtime import OpenAIRealtimeProvider

realtime = RealtimeVoiceChannel(
    "realtime-main",
    provider=OpenAIRealtimeProvider(api_key=os.environ["OPENAI_API_KEY"]),
    backend=backend,
)
```

### Gemini Live

```python
from roomkit.voice.realtime import GeminiLiveProvider

realtime = RealtimeVoiceChannel(
    "realtime-main",
    provider=GeminiLiveProvider(api_key=os.environ["GEMINI_API_KEY"]),
    backend=backend,
)
```

### Mock Realtime (Testing)

```python
from roomkit import MockRealtimeProvider

provider = MockRealtimeProvider()
```

## Voice Capabilities

Backends declare capabilities via `VoiceCapability` flags:

| Capability | Description |
|-----------|-------------|
| `INTERRUPTION` | Can cancel playback |
| `BARGE_IN` | Supports user interrupting TTS |
| `NATIVE_AEC` | Backend has built-in echo cancellation |
| `NATIVE_AGC` | Backend has built-in gain control |
| `DTMF_INBAND` | In-band DTMF tone detection |
| `DTMF_SIGNALING` | Out-of-band DTMF signaling |

When a backend has `NATIVE_AEC` or `NATIVE_AGC`, the pipeline automatically skips the corresponding software stage.
