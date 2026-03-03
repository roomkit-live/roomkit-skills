# RoomKit Skills

Skills for building multi-channel AI conversation agents with [RoomKit](https://www.roomkit.live). These skills teach AI coding assistants (Claude Code, Cursor, VS Code Copilot, Gemini CLI) how to build with RoomKit's room-based messaging framework.

RoomKit is a pure async Python library that unifies SMS, RCS, Email, WhatsApp, Messenger, Teams, Telegram, Voice, WebSocket, HTTP, and AI channels through a single room abstraction. Messages flow into rooms, get processed by hooks, and broadcast to all attached channels with automatic content transcoding.

## Installation

### Claude Code Plugin (recommended)

```bash
claude plugin install roomkit-skills
```

### Manual Copy

Clone this repo next to your project and reference it in your Claude Code settings:

```bash
git clone https://github.com/roomkit-live/roomkit-skills.git
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [setup-project](./setup-project/) | Install RoomKit, create your first room, run the quickstart |
| [create-room](./create-room/) | Room lifecycle, participants, channel binding, permissions |
| [create-ai-channel](./create-ai-channel/) | AI providers (Anthropic, OpenAI, Gemini, Mistral), tools, system prompts |
| [create-voice-agent](./create-voice-agent/) | Voice channel with STT/TTS, audio pipeline, interruption handling |
| [create-sms-channel](./create-sms-channel/) | SMS/RCS via Twilio, Sinch, Telnyx with delivery status |
| [add-hooks](./add-hooks/) | Hook system with 40+ triggers, sync/async execution, filtering |
| [create-multichannel](./create-multichannel/) | Bridge multiple channel types in one room with content transcoding |
| [setup-orchestration](./setup-orchestration/) | ConversationRouter, phases, handoffs, delegation, pipeline stages |
| [create-telegram-bot](./create-telegram-bot/) | Telegram Bot API channel with webhook parsing |
| [create-whatsapp-agent](./create-whatsapp-agent/) | WhatsApp Cloud API and Personal (neonize) channels |
| [setup-postgres](./setup-postgres/) | PostgresStore for production persistence |
| [setup-telemetry](./setup-telemetry/) | OpenTelemetry, console telemetry, protocol tracing |

## Configuration

**Requirements:** Python 3.12+

```bash
uv add roomkit        # recommended
pip install roomkit   # alternative
```

For voice features:

```bash
uv add "roomkit[voice]"       # STT/TTS/pipeline
uv add "roomkit[local-audio]" # local mic/speaker
```

## Quick Start

### 1. Install RoomKit

```bash
uv init my-agent && cd my-agent
uv add roomkit
```

### 2. Create your first room

```python
from __future__ import annotations

import asyncio
from roomkit import RoomKit, WebSocketChannel, InboundMessage, TextContent

async def main() -> None:
    kit = RoomKit()

    ws = WebSocketChannel("ws-user")
    kit.register_channel(ws)

    await kit.create_room(room_id="my-room")
    await kit.attach_channel("my-room", "ws-user")

    await kit.process_inbound(
        InboundMessage(channel_id="ws-user", sender_id="user", content=TextContent(body="Hello!"))
    )

asyncio.run(main())
```

### 3. Add an AI channel

```python
from roomkit import AnthropicAIProvider, AnthropicConfig, ChannelCategory
from roomkit.channels.ai import AIChannel

ai = AIChannel(
    "ai-assistant",
    provider=AnthropicAIProvider(AnthropicConfig(api_key="sk-...")),
    system_prompt="You are a helpful assistant.",
)
kit.register_channel(ai)
await kit.attach_channel("my-room", "ai-assistant", category=ChannelCategory.INTELLIGENCE)
```

### 4. Add hooks for control

```python
from roomkit import HookTrigger, HookResult, RoomEvent, RoomContext

@kit.hook(HookTrigger.BEFORE_BROADCAST)
async def log_messages(event: RoomEvent, ctx: RoomContext) -> HookResult:
    print(f"[{event.source.channel_id}] {event.content}")
    return HookResult.allow()
```

## API Reference

- **Full documentation:** https://www.roomkit.live
- **RFC specification:** Available in the `roomkit-specs` repository

## License

MIT
