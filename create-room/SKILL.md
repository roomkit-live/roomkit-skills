---
name: create-room
description: Create and manage RoomKit rooms with participants, channel bindings, permissions, and lifecycle management. Use when the user wants to create rooms, attach channels, manage participants, or configure room behavior.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package.
metadata:
  author: roomkit
  version: "1.0"
---

# Room Creation and Management

## Quick Start

```python
from __future__ import annotations

import asyncio

from roomkit import (
    ChannelCategory,
    InboundMessage,
    RoomKit,
    TextContent,
    WebSocketChannel,
)
from roomkit.channels.ai import AIChannel
from roomkit import MockAIProvider


async def main() -> None:
    kit = RoomKit()

    ws_alice = WebSocketChannel("ws-alice")
    ws_bob = WebSocketChannel("ws-bob")
    ai = AIChannel("ai-main", provider=MockAIProvider(responses=["Hello!"]))

    kit.register_channel(ws_alice)
    kit.register_channel(ws_bob)
    kit.register_channel(ai)

    # Create a room
    await kit.create_room(room_id="chat-room", metadata={"topic": "general"})

    # Attach channels
    await kit.attach_channel("chat-room", "ws-alice")
    await kit.attach_channel("chat-room", "ws-bob")
    await kit.attach_channel("chat-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # Send a message — broadcasts to all attached channels
    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-alice",
            sender_id="alice",
            content=TextContent(body="Hello everyone!"),
        )
    )

    # List participants
    participants = await kit.store.list_participants("chat-room")
    for p in participants:
        print(f"  {p.display_name or p.id} ({p.role})")


asyncio.run(main())
```

## Core Configuration

### Room Creation

```python
# Simple room
await kit.create_room(room_id="my-room")

# Room with metadata
await kit.create_room(
    room_id="support-123",
    metadata={"customer_id": "cust-456", "priority": "high"},
)

# Room with organization scope
await kit.create_room(
    room_id="org-room",
    organization_id="org-001",
)
```

### Channel Attachment

Channels are registered globally on the RoomKit instance, then attached to specific rooms:

```python
# Register channel once
kit.register_channel(ws)

# Attach to room (transport channel — delivers messages)
await kit.attach_channel("my-room", "ws-user")

# Attach as intelligence (AI channel — generates responses)
await kit.attach_channel("my-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

# Detach channel
await kit.detach_channel("my-room", "ws-user")
```

### Channel Bindings

Bindings connect external identifiers to channels within rooms:

```python
from roomkit import ChannelBinding, ChannelType

binding = ChannelBinding(
    channel_id="sms-main",
    channel_type=ChannelType.SMS,
    external_id="+15551234567",
    provider="twilio",
)

await kit.store.add_binding("my-room", binding)
```

### Room Lifecycle

```python
from roomkit import RoomStatus

# Pause a room (stops processing)
await kit.update_room("my-room", status=RoomStatus.PAUSED)

# Resume
await kit.update_room("my-room", status=RoomStatus.ACTIVE)

# Close a room
await kit.close_room("my-room")
```

### Participants

Participants are auto-created from inbound messages. They have roles and statuses:

```python
from roomkit import ParticipantRole, ParticipantStatus

# Roles: OWNER, AGENT, MEMBER, OBSERVER, BOT
# Statuses: ACTIVE, INACTIVE, LEFT, BANNED
```

## Common Patterns

### Multi-user Chat Room

```python
from __future__ import annotations

import asyncio

from roomkit import (
    InboundMessage,
    RoomEvent,
    RoomKit,
    TextContent,
    WebSocketChannel,
)


async def main() -> None:
    kit = RoomKit()

    users = {}
    for name in ["alice", "bob", "carol"]:
        ws = WebSocketChannel(f"ws-{name}")
        kit.register_channel(ws)
        inbox: list[RoomEvent] = []
        ws.register_connection(f"{name}-conn", lambda _c, ev, i=inbox: i.append(ev))
        users[name] = inbox

    await kit.create_room(room_id="group-chat")
    for name in users:
        await kit.attach_channel("group-chat", f"ws-{name}")

    # Alice sends a message — Bob and Carol receive it
    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-alice",
            sender_id="alice",
            content=TextContent(body="Hey team!"),
        )
    )

    for name, inbox in users.items():
        print(f"{name} received {len(inbox)} messages")


asyncio.run(main())
```

### Room with Timers

```python
await kit.create_room(
    room_id="timed-room",
    metadata={
        "inactive_after_seconds": 300,   # Auto-pause after 5 min inactivity
        "closed_after_seconds": 3600,    # Auto-close after 1 hour
    },
)
```

### Querying Room History

```python
# List all events in a room
events = await kit.store.list_events("my-room")

# Paginated query
events = await kit.store.list_events("my-room", offset=0, limit=50)

# Get event count
count = await kit.store.get_event_count("my-room")

# Find rooms by metadata
rooms = await kit.store.find_rooms(metadata={"customer_id": "cust-456"})
```
