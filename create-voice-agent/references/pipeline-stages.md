# Audio Pipeline Stage Reference

The audio pipeline processes audio between the voice backend and STT/TTS. Stages execute in a fixed order per the RFC.

## Pipeline Order

```
Inbound:  Backend -> [Resampler] -> [Recorder] -> [AEC] -> [AGC] -> [Denoiser] -> VAD -> [Diarization] + [DTMF]
Outbound: TTS -> [PostProcessors] -> [Recorder] -> AEC.feed_reference -> [Resampler] -> Backend
```

Stages in brackets are optional. VAD is the minimum required stage for voice activity detection.

## AudioPipelineConfig

```python
from roomkit.voice.pipeline import AudioPipelineConfig

config = AudioPipelineConfig(
    vad=vad_provider,                    # Voice Activity Detection
    denoiser=denoiser_provider,          # Noise reduction
    diarization=diarization_provider,    # Speaker identification
    aec=aec_provider,                    # Acoustic Echo Cancellation
    agc=agc_provider,                    # Automatic Gain Control
    agc_config=agc_config,              # AGC settings
    dtmf=dtmf_provider,                 # DTMF tone detection
    turn_detector=turn_provider,         # Turn-taking detection
    backchannel_detector=bc_provider,    # Backchannel detection
    recorder=recorder_provider,          # Audio recording
    recording_config=recording_config,   # Recording settings
    postprocessors=[post1, post2],       # Outbound post-processing
    resampler=resampler_provider,        # Sample rate conversion
    contract=pipeline_contract,          # Input/output format spec
    interruption=interruption_config,    # Interruption settings
    debug_taps=debug_taps,              # Diagnostic audio capture
    telemetry=telemetry_provider,        # Telemetry
)
```

## Stage Details

### 1. VAD (Voice Activity Detection)

Detects speech vs silence. Required for speech segmentation.

```python
from roomkit.voice.pipeline import SherpaOnnxVADProvider, SherpaOnnxVADConfig

vad = SherpaOnnxVADProvider(
    config=SherpaOnnxVADConfig(
        threshold=0.5,                    # Detection threshold (0.0-1.0)
        min_speech_duration_ms=200,       # Minimum speech to trigger
        min_silence_duration_ms=500,      # Silence before speech-end
    )
)
```

**Energy VAD** (simple, no model):

```python
from roomkit.voice.pipeline import EnergyVADProvider

vad = EnergyVADProvider()
```

**Events:** `SPEECH_START`, `SPEECH_END`, `SILENCE`, `AUDIO_LEVEL`

### 2. AEC (Acoustic Echo Cancellation)

Removes speaker output from mic input to prevent echo.

```python
from roomkit.voice.pipeline import SpeexAECProvider

aec = SpeexAECProvider()
```

Skipped automatically when backend has `NATIVE_AEC` capability.

### 3. AGC (Automatic Gain Control)

Normalizes audio volume levels.

```python
from roomkit.voice.pipeline import AGCProvider, AGCConfig

agc_config = AGCConfig(
    target_level_dbfs=-20.0,
    max_gain_db=30.0,
)
```

Skipped automatically when backend has `NATIVE_AGC` capability.

### 4. Denoiser

Removes background noise from audio.

```python
# RNNoise
from roomkit.voice.pipeline import RNNoiseDenoiserProvider
denoiser = RNNoiseDenoiserProvider()

# SherpaOnnx
from roomkit.voice.pipeline import SherpaOnnxDenoiserProvider, SherpaOnnxDenoiserConfig
denoiser = SherpaOnnxDenoiserProvider(
    config=SherpaOnnxDenoiserConfig(
        context_frames=3,           # Warmup frames (60ms)
        silence_threshold=0.005,    # Skip inference for silent frames
    )
)
```

### 5. Diarization (Speaker Identification)

Identifies which speaker is talking.

```python
from roomkit.voice.pipeline import DiarizationProvider

# Returns DiarizationResult with speaker_id, start_time, end_time, confidence
```

### 6. DTMF Detection

Detects touch-tone signals.

```python
from roomkit.voice.pipeline import DTMFDetector

dtmf = DTMFDetector()
# Emits DTMFEvent with digit, duration
```

### 7. Recorder

Records audio to WAV files.

```python
from roomkit.voice.pipeline import WavFileRecorder, RecordingConfig, RecordingMode

recorder = WavFileRecorder()
recording_config = RecordingConfig(
    path="./recordings",
    mode=RecordingMode.BOTH,        # INBOUND, OUTBOUND, or BOTH
)
```

### 8. Turn Detector

Determines when the user has finished speaking and the AI should respond.

```python
from roomkit.voice.pipeline import TurnDetector, TurnContext, TurnDecision

# Custom turn detector
class MyTurnDetector(TurnDetector):
    async def detect(self, context: TurnContext) -> TurnDecision:
        # Analyze silence duration, text content, etc.
        ...
```

### 9. Backchannel Detector

Detects conversational backchannels ("uh-huh", "mm-hmm").

```python
from roomkit.voice.pipeline import BackchannelDetector, BackchannelContext, BackchannelDecision
```

### 10. PostProcessor

Outbound audio post-processing (e.g., volume normalization, effects).

```python
from roomkit.voice.pipeline import AudioPostProcessor
```

## Audio Formats

```python
from roomkit.voice.pipeline import AudioFormat, AudioPipelineContract

contract = AudioPipelineContract(
    transport_inbound_format=AudioFormat(
        sample_rate=16000,      # Sample rate in Hz
        channels=1,             # Mono
        sample_width=2,         # 16-bit
        codec="pcm_s16le",     # PCM signed 16-bit little-endian
    ),
    transport_outbound_format=AudioFormat(sample_rate=16000),
    internal_format=AudioFormat(sample_rate=16000),
)
```

The resampler automatically converts between transport and internal formats.

## Interruption Handling

```python
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy

config = InterruptionConfig(
    strategy=InterruptionStrategy.IMMEDIATE,   # Cancel TTS immediately
    # CONFIRMED: Wait for confirmed speech before cancelling
    # SEMANTIC: Analyze content to decide
    # DISABLED: Never interrupt
)
```

## Mock Providers (Testing)

Every stage has a mock for testing:

```python
from roomkit.voice.pipeline import (
    MockVADProvider,
    MockAECProvider,
    MockAGCProvider,
    MockDenoiserProvider,
    MockDiarizationProvider,
)
```
