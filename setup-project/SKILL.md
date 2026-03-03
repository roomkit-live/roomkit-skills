---
name: setup-project
description: Set up a new RoomKit project with installation, configuration, and a working quickstart. Use when the user wants to start building with RoomKit, needs help installing it, or wants to create their first multi-channel room.
license: MIT
compatibility: Requires Python 3.12+ and internet access for package installation.
metadata:
  author: roomkit
  version: "1.0"
---

# RoomKit Project Setup

## Quick Start

```bash
uv init my-agent && cd my-agent
uv add roomkit
```

```python
from __future__ import annotations

import asyncio

from roomkit import (
    InboundMessage,
    RoomKit,
    TextContent,
    WebSocketChannel,
)


async def main() -> None:
    kit = RoomKit()

    ws = WebSocketChannel("ws-user")
    kit.register_channel(ws)

    inbox: list = []
    ws.register_connection("conn", lambda _c, ev: inbox.append(ev))

    await kit.create_room(room_id="my-room")
    await kit.attach_channel("my-room", "ws-user")

    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-user",
            sender_id="user-1",
            content=TextContent(body="Hello RoomKit!"),
        )
    )

    events = await kit.store.list_events("my-room")
    for ev in events:
        print(f"[{ev.source.channel_id}] {ev.content.body}")


asyncio.run(main())
```

## Installation Options

### Using uv (recommended)

```bash
uv add roomkit
```

### Using pip

```bash
pip install roomkit
```

### Optional extras

```bash
uv add "roomkit[voice]"        # STT/TTS/audio pipeline
uv add "roomkit[local-audio]"  # Local mic/speaker backend
uv add "roomkit[postgres]"     # PostgreSQL persistence
```

## Core Configuration

### RoomKit Constructor

```python
from roomkit import RoomKit, InMemoryStore

kit = RoomKit(
    store=InMemoryStore(),           # Default — use PostgresStore for production
    max_chain_depth=5,               # AI-to-AI reentry limit
    identity_timeout=10.0,           # Identity resolution timeout (seconds)
    process_timeout=30.0,            # Processing phase timeout (seconds)
)
```

### Environment Variables

Set API keys as environment variables — never hardcode secrets:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export OPENAI_API_KEY=sk-...
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export DEEPGRAM_API_KEY=...
export ELEVENLABS_API_KEY=...
```

### Project Structure

A typical RoomKit project:

```
my-agent/
├── pyproject.toml
├── .env                # API keys (gitignored)
├── main.py             # Entry point
├── channels.py         # Channel setup
├── hooks.py            # Hook definitions
└── tools.py            # AI tool handlers
```

## Common Patterns

### Quickstart with AI

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    AnthropicAIProvider,
    AnthropicConfig,
    ChannelCategory,
    InboundMessage,
    RoomKit,
    TextContent,
    WebSocketChannel,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    ws = WebSocketChannel("ws-user")
    ai = AIChannel(
        "ai-assistant",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
        ),
        system_prompt="You are a helpful assistant. Keep answers concise.",
    )

    kit.register_channel(ws)
    kit.register_channel(ai)

    inbox: list = []
    ws.register_connection("conn", lambda _c, ev: inbox.append(ev))

    await kit.create_room(room_id="demo")
    await kit.attach_channel("demo", "ws-user")
    await kit.attach_channel("demo", "ai-assistant", category=ChannelCategory.INTELLIGENCE)

    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-user",
            sender_id="user",
            content=TextContent(body="What is RoomKit?"),
        )
    )

    for ev in inbox:
        print(f"AI: {ev.content.body}")


asyncio.run(main())
```

### Run with uv

```bash
uv run python main.py
```

## References

- Full documentation: https://www.roomkit.live
- Examples: https://github.com/pabloquintana/roomkit/tree/main/examples
