# Hook Triggers Reference

All hook triggers available in RoomKit, organized by category.

## Event Pipeline

| Trigger | Execution | Description |
|---------|-----------|-------------|
| `BEFORE_BROADCAST` | SYNC | Block or modify events before routing to channels. Can return `HookResult.block()`, `HookResult.modify()`, or `HookResult.allow()`. |
| `AFTER_BROADCAST` | ASYNC | Fire-and-forget side effects after broadcast. Cannot block events. |

## Room Lifecycle

| Trigger | Description |
|---------|-------------|
| `ON_ROOM_CREATED` | Room initialized |
| `ON_ROOM_PAUSED` | Room paused |
| `ON_ROOM_CLOSED` | Room closed |

## Channel Lifecycle

| Trigger | Description |
|---------|-------------|
| `ON_CHANNEL_ATTACHED` | Channel bound to room |
| `ON_CHANNEL_DETACHED` | Channel removed from room |
| `ON_CHANNEL_MUTED` | Channel silenced |
| `ON_CHANNEL_UNMUTED` | Channel restored |

## Identity

| Trigger | Description |
|---------|-------------|
| `ON_IDENTITY_AMBIGUOUS` | Sender identification uncertain |
| `ON_IDENTITY_UNKNOWN` | Sender unrecognized |
| `ON_PARTICIPANT_IDENTIFIED` | Sender confirmed |

## Side Effects & Delivery

| Trigger | Description |
|---------|-------------|
| `ON_TASK_CREATED` | Task created |
| `ON_DELIVERY_STATUS` | Message delivery status updated |
| `ON_ERROR` | Error occurred |

## Voice Pipeline

| Trigger | Description |
|---------|-------------|
| `ON_SESSION_STARTED` | Voice session initialized |
| `ON_SPEECH_START` | Speech begins (VAD detected voice) |
| `ON_SPEECH_END` | Speech ends (VAD detected silence) |
| `ON_TRANSCRIPTION` | STT transcription complete |
| `ON_PARTIAL_TRANSCRIPTION` | Interim STT result (streaming) |
| `BEFORE_TTS` | Before TTS synthesis — can modify text |
| `AFTER_TTS` | After TTS synthesis complete |
| `ON_BARGE_IN` | User interrupted TTS playback |
| `ON_TTS_CANCELLED` | TTS playback was cancelled |

## Voice Audio Levels

| Trigger | Description |
|---------|-------------|
| `ON_VAD_SILENCE` | VAD detects prolonged silence |
| `ON_VAD_AUDIO_LEVEL` | VAD reports audio level |
| `ON_INPUT_AUDIO_LEVEL` | Inbound audio level measurement |
| `ON_OUTPUT_AUDIO_LEVEL` | Outbound audio level measurement |

## Voice Advanced

| Trigger | Description |
|---------|-------------|
| `ON_SPEAKER_CHANGE` | Speaker diarization detected new speaker |
| `ON_DTMF` | DTMF tone detected |
| `ON_TURN_COMPLETE` | Turn detection: AI should respond |
| `ON_TURN_INCOMPLETE` | Turn detection: user still speaking |
| `ON_BACKCHANNEL` | Backchannel detected (uh-huh, mm-hmm) |
| `ON_RECORDING_STARTED` | Audio recording started |
| `ON_RECORDING_STOPPED` | Audio recording stopped |

## Realtime Voice

| Trigger | Description |
|---------|-------------|
| `ON_REALTIME_TOOL_CALL` | Realtime AI tool invoked |
| `ON_REALTIME_TEXT_INJECTED` | Text injected into realtime session |

## AI / Orchestration

| Trigger | Description |
|---------|-------------|
| `ON_AI_THINKING` | AI thinking/reasoning event (extended thinking) |
| `ON_PROTOCOL_TRACE` | Protocol-level observability event |
| `ON_PHASE_TRANSITION` | Orchestration phase changed |
| `ON_HANDOFF` | Agent handoff initiated |
| `ON_HANDOFF_REJECTED` | Agent handoff rejected |
| `ON_TASK_DELEGATED` | Background task delegated to agent |
| `ON_TASK_COMPLETED` | Background task completed |

## Usage Examples

### Sync Hook (BEFORE_BROADCAST)

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST, execution=HookExecution.SYNC)
async def filter_messages(event: RoomEvent, ctx: RoomContext) -> HookResult:
    if should_block(event):
        return HookResult.block("Blocked by filter")
    if should_modify(event):
        modified = event.model_copy(update={"content": TextContent(body="[modified]")})
        return HookResult.modify(modified)
    return HookResult.allow()
```

### Async Hook (AFTER_BROADCAST)

```python
@kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC)
async def log_event(event: RoomEvent, ctx: RoomContext) -> None:
    # Fire-and-forget — cannot block
    await analytics.log(event)
```

### Filtered Hook

```python
@kit.hook(
    HookTrigger.BEFORE_BROADCAST,
    channel_types=[ChannelType.SMS, ChannelType.WHATSAPP],
    directions=[ChannelDirection.INBOUND],
    priority=100,
)
async def sms_filter(event, ctx) -> HookResult:
    return HookResult.allow()
```

### Voice Hooks

```python
@kit.hook(HookTrigger.ON_TRANSCRIPTION)
async def on_transcription(event, ctx):
    print(f"User said: {event.content.body}")

@kit.hook(HookTrigger.BEFORE_TTS)
async def before_tts(event, ctx) -> HookResult:
    # Modify text before synthesis
    text = event.content.body
    text = text.replace("*", "").replace("_", "")  # Strip markdown
    modified = event.model_copy(update={"content": TextContent(body=text)})
    return HookResult.modify(modified)

@kit.hook(HookTrigger.ON_BARGE_IN)
async def on_barge_in(event, ctx):
    print("User interrupted AI")
```

## HookResult API

```python
from roomkit import HookResult

HookResult.allow()                          # Let event proceed
HookResult.block("reason")                  # Block event with reason
HookResult.modify(modified_event)           # Replace event with modified version
```

## Execution Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `HookExecution.SYNC` | Sequential, can block/modify | Filtering, validation, content modification |
| `HookExecution.ASYNC` | Fire-and-forget, non-blocking | Logging, analytics, notifications |
