---
name: create-multichannel
description: Bridge multiple channel types (SMS, WhatsApp, Voice, WebSocket, Email) in a single RoomKit room with automatic content transcoding. Use when the user wants to connect different messaging platforms together, create omnichannel experiences, or route messages across channel types.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package with relevant provider extras.
metadata:
  author: roomkit
  version: "1.0"
---

# Multi-Channel Room Setup

## Quick Start

```python
from __future__ import annotations

import asyncio

from roomkit import (
    ChannelCategory,
    InboundMessage,
    MockAIProvider,
    MockSMSProvider,
    RoomEvent,
    RoomKit,
    SMSChannel,
    TextContent,
    WebSocketChannel,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    # Multiple channel types in one room
    ws = WebSocketChannel("ws-agent")
    sms = SMSChannel("sms-customer", provider=MockSMSProvider())
    ai = AIChannel("ai-main", provider=MockAIProvider(responses=["I'll help!"]))

    kit.register_channel(ws)
    kit.register_channel(sms)
    kit.register_channel(ai)

    inbox: list[RoomEvent] = []
    ws.register_connection("agent-conn", lambda _c, ev: inbox.append(ev))

    await kit.create_room(room_id="support-room")
    await kit.attach_channel("support-room", "ws-agent")
    await kit.attach_channel("support-room", "sms-customer")
    await kit.attach_channel("support-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # Customer sends SMS -> broadcasts to WebSocket agent + AI
    await kit.process_inbound(
        InboundMessage(
            channel_id="sms-customer",
            sender_id="+15559876543",
            content=TextContent(body="I need help with my order"),
        )
    )

    print(f"Agent received {len(inbox)} messages via WebSocket")
    for ev in inbox:
        print(f"  [{ev.source.channel_type}] {ev.content.body}")


asyncio.run(main())
```

## Core Configuration

### Attaching Multiple Channel Types

```python
# Transport channels — deliver messages
await kit.attach_channel("room", "sms-main")
await kit.attach_channel("room", "ws-agent")
await kit.attach_channel("room", "email-main")
await kit.attach_channel("room", "whatsapp-main")
await kit.attach_channel("room", "telegram-main")

# Intelligence channels — generate AI responses
await kit.attach_channel("room", "ai-main", category=ChannelCategory.INTELLIGENCE)
```

### Content Transcoding

RoomKit automatically transcodes content between channels based on their capabilities. For example:

- **Rich text -> SMS**: Strips HTML, keeps plain text
- **Media -> Email**: Embeds media as attachments
- **Buttons -> SMS**: Converts to numbered list text
- **Audio -> Text channel**: Uses transcription if available

Channel capabilities determine what content types are supported:

```python
# SMS: text + media (MMS)
# Email: text + rich + media + threading
# WhatsApp: text + rich + media + templates + buttons + reactions
# Telegram: text + rich + media + location + edit/delete
# WebSocket: text + rich + media + audio + video
```

### Channel Bindings

Bindings map external identifiers (phone numbers, emails, chat IDs) to channels within rooms:

```python
from roomkit import ChannelBinding, ChannelType

# SMS binding
await kit.store.add_binding(
    "room",
    ChannelBinding(
        channel_id="sms-main",
        channel_type=ChannelType.SMS,
        external_id="+15559876543",
        provider="twilio",
    ),
)

# WhatsApp binding
await kit.store.add_binding(
    "room",
    ChannelBinding(
        channel_id="whatsapp-main",
        channel_type=ChannelType.WHATSAPP,
        external_id="+15559876543",
        provider="meta",
    ),
)
```

## Common Patterns

### Support Room: SMS + WebSocket + AI

A customer contacts via SMS, a human agent monitors via WebSocket, and AI assists:

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    AnthropicAIProvider,
    AnthropicConfig,
    ChannelCategory,
    HookResult,
    HookTrigger,
    RoomContext,
    RoomEvent,
    RoomKit,
    SMSChannel,
    TextContent,
    TwilioConfig,
    TwilioSMSProvider,
    WebSocketChannel,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    sms = SMSChannel(
        "sms-customer",
        provider=TwilioSMSProvider(TwilioConfig(
            account_sid=os.environ["TWILIO_ACCOUNT_SID"],
            auth_token=os.environ["TWILIO_AUTH_TOKEN"],
            from_number="+15551234567",
        )),
    )

    ws_agent = WebSocketChannel("ws-agent")

    ai = AIChannel(
        "ai-assist",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
        ),
        system_prompt=(
            "You are a support assistant. Help the customer. "
            "Keep SMS replies under 160 characters."
        ),
    )

    kit.register_channel(sms)
    kit.register_channel(ws_agent)
    kit.register_channel(ai)

    await kit.create_room(room_id="support-123")
    await kit.attach_channel("support-123", "sms-customer")
    await kit.attach_channel("support-123", "ws-agent")
    await kit.attach_channel("support-123", "ai-assist", category=ChannelCategory.INTELLIGENCE)

    @kit.hook(HookTrigger.AFTER_BROADCAST, name="notify_agent")
    async def notify_agent(event: RoomEvent, ctx: RoomContext) -> None:
        print(f"[{event.source.channel_type}] {event.content}")


asyncio.run(main())
```

### Five-Channel Bridge

```python
ws = WebSocketChannel("ws-main")
sms = SMSChannel("sms-main", provider=sms_provider)
email = EmailChannel("email-main", provider=email_provider)
http = HTTPChannel("http-main", provider=http_provider)
ai = AIChannel("ai-main", provider=ai_provider, system_prompt="You bridge conversations.")

for ch in [ws, sms, email, http, ai]:
    kit.register_channel(ch)

await kit.create_room(room_id="bridge-room")
await kit.attach_channel("bridge-room", "ws-main")
await kit.attach_channel("bridge-room", "sms-main")
await kit.attach_channel("bridge-room", "email-main")
await kit.attach_channel("bridge-room", "http-main")
await kit.attach_channel("bridge-room", "ai-main", category=ChannelCategory.INTELLIGENCE)
```

### Room Timeline

```python
events = await kit.store.list_events("bridge-room")
for ev in events:
    direction = ev.source.direction
    channel = ev.source.channel_type
    print(f"[{direction}] {channel}: {ev.content}")
```
