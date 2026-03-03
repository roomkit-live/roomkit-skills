---
name: add-hooks
description: Add hooks to RoomKit rooms for event filtering, modification, logging, and side effects. Covers all 40+ hook triggers with sync and async execution modes. Use when the user wants to filter messages, add logging, modify events before broadcast, or react to voice/AI events.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package.
metadata:
  author: roomkit
  version: "1.0"
---

# Hook System

## Quick Start

```python
from __future__ import annotations

import asyncio

from roomkit import (
    HookExecution,
    HookResult,
    HookTrigger,
    InboundMessage,
    MockAIProvider,
    RoomContext,
    RoomEvent,
    RoomKit,
    TextContent,
    WebSocketChannel,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    ws = WebSocketChannel("ws-user")
    ai = AIChannel("ai-main", provider=MockAIProvider(responses=["OK!"]))
    kit.register_channel(ws)
    kit.register_channel(ai)

    # SYNC hook: can block or modify events before broadcast
    @kit.hook(HookTrigger.BEFORE_BROADCAST, name="profanity_filter")
    async def profanity_filter(event: RoomEvent, ctx: RoomContext) -> HookResult:
        if isinstance(event.content, TextContent) and "spam" in event.content.body.lower():
            return HookResult.block("Blocked by profanity filter")
        return HookResult.allow()

    # ASYNC hook: fire-and-forget side effects
    @kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC, name="logger")
    async def log_event(event: RoomEvent, ctx: RoomContext) -> None:
        print(f"[LOG] {event.source.channel_id}: {event.content}")

    from roomkit import ChannelCategory

    await kit.create_room(room_id="demo")
    await kit.attach_channel("demo", "ws-user")
    await kit.attach_channel("demo", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # This gets through
    await kit.process_inbound(
        InboundMessage(channel_id="ws-user", sender_id="user", content=TextContent(body="Hello!"))
    )

    # This gets blocked
    result = await kit.process_inbound(
        InboundMessage(channel_id="ws-user", sender_id="user", content=TextContent(body="Buy spam now"))
    )
    print(f"Blocked: {result.blocked}, reason: {result.reason}")


asyncio.run(main())
```

## Core Configuration

### Hook Registration

```python
# Decorator style
@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def my_hook(event: RoomEvent, ctx: RoomContext) -> HookResult:
    return HookResult.allow()

# With options
@kit.hook(
    HookTrigger.BEFORE_BROADCAST,
    name="my_filter",                    # Hook name (for debugging)
    execution=HookExecution.SYNC,        # SYNC (default) or ASYNC
    priority=10,                         # Higher = runs first
    channel_types=[ChannelType.SMS],     # Filter by channel type
    directions=[ChannelDirection.INBOUND],  # Filter by direction
)
async def filtered_hook(event: RoomEvent, ctx: RoomContext) -> HookResult:
    return HookResult.allow()
```

### HookResult Actions

```python
# Allow the event to proceed
return HookResult.allow()

# Block the event
return HookResult.block("Reason for blocking")

# Modify the event
modified = event.model_copy(update={"content": TextContent(body="[redacted]")})
return HookResult.modify(modified)
```

### Execution Modes

- **SYNC** (default): Runs sequentially. Can block or modify events. Use for filtering, validation, content modification.
- **ASYNC**: Fire-and-forget. Cannot block events. Use for logging, analytics, notifications.

```python
# Sync — blocks pipeline until complete
@kit.hook(HookTrigger.BEFORE_BROADCAST, execution=HookExecution.SYNC)
async def validate(event, ctx) -> HookResult:
    return HookResult.allow()

# Async — runs in background, never blocks
@kit.hook(HookTrigger.AFTER_BROADCAST, execution=HookExecution.ASYNC)
async def log(event, ctx) -> None:
    await send_to_analytics(event)
```

### Hook Trigger Categories

**Event Pipeline:**
- `BEFORE_BROADCAST` — Block/modify before routing (SYNC)
- `AFTER_BROADCAST` — Side effects after broadcast (ASYNC)

**Room Lifecycle:**
- `ON_ROOM_CREATED`, `ON_ROOM_PAUSED`, `ON_ROOM_CLOSED`

**Channel Lifecycle:**
- `ON_CHANNEL_ATTACHED`, `ON_CHANNEL_DETACHED`, `ON_CHANNEL_MUTED`, `ON_CHANNEL_UNMUTED`

**Identity:**
- `ON_IDENTITY_AMBIGUOUS`, `ON_IDENTITY_UNKNOWN`, `ON_PARTICIPANT_IDENTIFIED`

**Voice:**
- `ON_SPEECH_START`, `ON_SPEECH_END`, `ON_TRANSCRIPTION`
- `BEFORE_TTS`, `AFTER_TTS`, `ON_BARGE_IN`, `ON_TTS_CANCELLED`
- `ON_PARTIAL_TRANSCRIPTION`, `ON_SESSION_STARTED`
- `ON_VAD_SILENCE`, `ON_VAD_AUDIO_LEVEL`
- `ON_INPUT_AUDIO_LEVEL`, `ON_OUTPUT_AUDIO_LEVEL`
- `ON_SPEAKER_CHANGE`, `ON_DTMF`
- `ON_TURN_COMPLETE`, `ON_TURN_INCOMPLETE`, `ON_BACKCHANNEL`
- `ON_RECORDING_STARTED`, `ON_RECORDING_STOPPED`

**AI/Orchestration:**
- `ON_AI_THINKING`, `ON_PROTOCOL_TRACE`
- `ON_PHASE_TRANSITION`, `ON_HANDOFF`, `ON_HANDOFF_REJECTED`
- `ON_TASK_DELEGATED`, `ON_TASK_COMPLETED`

**Realtime Voice:**
- `ON_REALTIME_TOOL_CALL`, `ON_REALTIME_TEXT_INJECTED`

**Side Effects:**
- `ON_TASK_CREATED`, `ON_DELIVERY_STATUS`, `ON_ERROR`

## Common Patterns

### Content Moderation

```python
@kit.hook(HookTrigger.BEFORE_BROADCAST, name="content_moderation")
async def moderate(event: RoomEvent, ctx: RoomContext) -> HookResult:
    if not isinstance(event.content, TextContent):
        return HookResult.allow()

    # Check against blocklist
    blocked_words = {"spam", "scam", "phishing"}
    text = event.content.body.lower()
    for word in blocked_words:
        if word in text:
            return HookResult.block(f"Blocked: contains '{word}'")

    return HookResult.allow()
```

### Analytics Logging

```python
@kit.hook(
    HookTrigger.AFTER_BROADCAST,
    execution=HookExecution.ASYNC,
    name="analytics",
)
async def log_analytics(event: RoomEvent, ctx: RoomContext) -> None:
    await analytics.track(
        event="message_sent",
        properties={
            "room_id": event.room_id,
            "channel_type": event.source.channel_type,
            "direction": event.source.direction,
        },
    )
```

### Voice Event Logging

```python
@kit.hook(HookTrigger.ON_TRANSCRIPTION)
async def log_transcription(event, ctx):
    print(f"User said: {event.content.body}")

@kit.hook(HookTrigger.ON_BARGE_IN)
async def log_barge_in(event, ctx):
    print("User interrupted the AI")

@kit.hook(HookTrigger.BEFORE_TTS)
async def before_tts(event, ctx) -> HookResult:
    # Modify text before synthesis
    return HookResult.allow()
```

## References

- [All Hook Triggers Reference](./references/hook-triggers.md)
