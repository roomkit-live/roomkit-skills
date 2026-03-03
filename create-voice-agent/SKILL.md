---
name: create-voice-agent
description: Create voice AI agents with RoomKit's VoiceChannel, STT/TTS providers, audio pipeline, and interruption handling. Use when the user wants to build a voice bot, add speech-to-text, text-to-speech, or configure audio processing pipelines with VAD, AEC, denoising, or DTMF detection.
license: MIT
compatibility: Requires Python 3.12+, roomkit[voice] extras, and API keys for STT/TTS providers.
metadata:
  author: roomkit
  version: "1.0"
---

# Voice Agent Creation

## Quick Start

```bash
uv add "roomkit[voice]"
```

```python
from __future__ import annotations

import asyncio

from roomkit import (
    ChannelCategory,
    HookTrigger,
    RoomKit,
    VoiceChannel,
)
from roomkit.channels.ai import AIChannel
from roomkit import MockAIProvider, MockSTTProvider, MockTTSProvider, MockVoiceBackend
from roomkit.voice.pipeline import AudioPipelineConfig, MockVADProvider, VADEvent, VADEventType


async def main() -> None:
    kit = RoomKit()

    # Create voice channel with STT, TTS, and audio pipeline
    backend = MockVoiceBackend()
    voice = VoiceChannel(
        "voice-main",
        stt=MockSTTProvider(transcriptions=["Hello, how can I help?"]),
        tts=MockTTSProvider(),
        backend=backend,
        pipeline=AudioPipelineConfig(
            vad=MockVADProvider(events=[
                VADEvent(type=VADEventType.SPEECH_START),
                VADEvent(type=VADEventType.SPEECH_END),
            ]),
        ),
    )

    ai = AIChannel("ai-main", provider=MockAIProvider(responses=["I can help with that!"]))

    kit.register_channel(voice)
    kit.register_channel(ai)

    await kit.create_room(room_id="voice-room")
    await kit.attach_channel("voice-room", "voice-main")
    await kit.attach_channel("voice-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # Connect voice session
    session = await kit.connect_voice("voice-room", "voice-main", participant_id="caller")

    @kit.hook(HookTrigger.ON_TRANSCRIPTION)
    async def on_transcription(event, ctx):
        print(f"Transcribed: {event.content.body}")

    print(f"Voice session started: {session.session_id}")


asyncio.run(main())
```

## Core Configuration

### VoiceChannel Constructor

```python
from roomkit import VoiceChannel

voice = VoiceChannel(
    "voice-main",
    stt=stt_provider,               # Speech-to-text provider
    tts=tts_provider,               # Text-to-speech provider
    backend=voice_backend,          # Audio transport (FastRTC, RTP, SIP, Local)
    pipeline=pipeline_config,       # Audio processing pipeline
    streaming=True,                 # Stream STT (default True)
    enable_barge_in=True,           # Allow user to interrupt TTS (default True)
    barge_in_threshold_ms=200,      # Min speech duration to trigger barge-in
    batch_mode=False,               # Manual flush STT mode
    voice_map={"agent-1": "alloy"}, # Per-agent voice overrides
    tts_filter=strip_markdown,      # Text filter before synthesis
)
```

### Audio Pipeline

The pipeline processes audio between the voice backend and STT/TTS:

```
Inbound:  Backend -> [Resampler] -> [Recorder] -> [AEC] -> [AGC] -> [Denoiser] -> VAD -> [Diarization] + [DTMF]
Outbound: TTS -> [PostProcessors] -> [Recorder] -> AEC.feed_reference -> [Resampler] -> Backend
```

```python
from roomkit.voice.pipeline import (
    AudioPipelineConfig,
    AudioPipelineContract,
    AudioFormat,
    SherpaOnnxVADProvider,
    SherpaOnnxVADConfig,
    SpeexAECProvider,
    RNNoiseDenoiserProvider,
    WavFileRecorder,
    RecordingConfig,
)

pipeline = AudioPipelineConfig(
    vad=SherpaOnnxVADProvider(config=SherpaOnnxVADConfig(
        threshold=0.5,
        min_speech_duration_ms=200,
        min_silence_duration_ms=500,
    )),
    aec=SpeexAECProvider(),
    denoiser=RNNoiseDenoiserProvider(),
    recorder=WavFileRecorder(),
    recording_config=RecordingConfig(path="./recordings"),
    contract=AudioPipelineContract(
        transport_inbound_format=AudioFormat(sample_rate=16000),
        transport_outbound_format=AudioFormat(sample_rate=16000),
        internal_format=AudioFormat(sample_rate=16000),
    ),
)
```

### Interruption Handling

4 strategies for handling user interruptions during TTS playback:

```python
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy

voice = VoiceChannel(
    "voice-main",
    stt=stt,
    tts=tts,
    backend=backend,
    pipeline=AudioPipelineConfig(
        interruption=InterruptionConfig(
            strategy=InterruptionStrategy.IMMEDIATE,  # Cancel TTS instantly
            # Other options: CONFIRMED, SEMANTIC, DISABLED
        ),
    ),
)
```

### Voice Backends

```python
# FastRTC (WebRTC)
from roomkit.voice.backends import FastRTCBackend
backend = FastRTCBackend(host="0.0.0.0", port=8080)

# Local mic/speaker (for development)
from roomkit.voice import get_local_audio_backend
backend = get_local_audio_backend()

# Mock (for testing)
from roomkit import MockVoiceBackend
backend = MockVoiceBackend()
```

### STT Providers

```python
# Deepgram
from roomkit.voice.stt import DeepgramSTTProvider, DeepgramSTTConfig
stt = DeepgramSTTProvider(DeepgramSTTConfig(api_key=os.environ["DEEPGRAM_API_KEY"]))

# SherpaOnnx (local, no API key)
from roomkit.voice.stt import SherpaOnnxSTTProvider, SherpaOnnxSTTConfig
stt = SherpaOnnxSTTProvider(SherpaOnnxSTTConfig(model_path="./models/stt"))
```

### TTS Providers

```python
# ElevenLabs
from roomkit.voice.tts import ElevenLabsTTSProvider, ElevenLabsTTSConfig
tts = ElevenLabsTTSProvider(ElevenLabsTTSConfig(
    api_key=os.environ["ELEVENLABS_API_KEY"],
    voice_id="21m00Tcm4TlvDq8ikWAM",
))

# SherpaOnnx (local, no API key)
from roomkit.voice.tts import SherpaOnnxTTSProvider, SherpaOnnxTTSConfig
tts = SherpaOnnxTTSProvider(SherpaOnnxTTSConfig(model_path="./models/tts"))
```

## Common Patterns

### Production Voice Agent

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    AnthropicAIProvider,
    AnthropicConfig,
    ChannelCategory,
    HookTrigger,
    RoomKit,
    VoiceChannel,
)
from roomkit.channels.ai import AIChannel
from roomkit.voice.backends import FastRTCBackend
from roomkit.voice.stt import DeepgramSTTProvider, DeepgramSTTConfig
from roomkit.voice.tts import ElevenLabsTTSProvider, ElevenLabsTTSConfig
from roomkit.voice.pipeline import (
    AudioPipelineConfig,
    SherpaOnnxVADProvider,
    SherpaOnnxVADConfig,
    SpeexAECProvider,
    RNNoiseDenoiserProvider,
)
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy


async def main() -> None:
    kit = RoomKit()

    voice = VoiceChannel(
        "voice-main",
        stt=DeepgramSTTProvider(DeepgramSTTConfig(
            api_key=os.environ["DEEPGRAM_API_KEY"],
        )),
        tts=ElevenLabsTTSProvider(ElevenLabsTTSConfig(
            api_key=os.environ["ELEVENLABS_API_KEY"],
            voice_id="21m00Tcm4TlvDq8ikWAM",
        )),
        backend=FastRTCBackend(host="0.0.0.0", port=8080),
        pipeline=AudioPipelineConfig(
            vad=SherpaOnnxVADProvider(config=SherpaOnnxVADConfig()),
            aec=SpeexAECProvider(),
            denoiser=RNNoiseDenoiserProvider(),
            interruption=InterruptionConfig(strategy=InterruptionStrategy.IMMEDIATE),
        ),
        enable_barge_in=True,
    )

    ai = AIChannel(
        "ai-main",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
        ),
        system_prompt="You are a helpful voice assistant. Be concise and conversational.",
    )

    kit.register_channel(voice)
    kit.register_channel(ai)

    await kit.create_room(room_id="voice-room")
    await kit.attach_channel("voice-room", "voice-main")
    await kit.attach_channel("voice-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    @kit.hook(HookTrigger.ON_TRANSCRIPTION)
    async def log_transcription(event, ctx):
        print(f"User said: {event.content.body}")

    @kit.hook(HookTrigger.ON_BARGE_IN)
    async def log_barge_in(event, ctx):
        print("User interrupted AI speech")

    session = await kit.connect_voice("voice-room", "voice-main", participant_id="caller")
    print(f"Listening on session: {session.session_id}")


asyncio.run(main())
```

### Voice with Greeting

```python
session = await kit.connect_voice("voice-room", "voice-main", participant_id="caller")
await kit.send_greeting("voice-room", "Welcome! How can I help you today?")
```

## References

- [Voice Provider Configurations](./references/voice-providers.md)
- [Pipeline Stage Reference](./references/pipeline-stages.md)
