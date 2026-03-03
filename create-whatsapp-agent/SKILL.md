---
name: create-whatsapp-agent
description: Create WhatsApp channels using the Cloud API or Personal (neonize) backend. Handle text, media, location, and template messages. Use when the user wants to integrate WhatsApp messaging, set up WhatsApp Business API, or use WhatsApp Personal with neonize.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package. Cloud API requires Meta Business credentials. Personal requires neonize package.
metadata:
  author: roomkit
  version: "1.0"
---

# WhatsApp Channel Setup

## Quick Start — Cloud API

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    ChannelCategory,
    InboundMessage,
    MockAIProvider,
    RoomKit,
    TextContent,
    WhatsAppChannel,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    whatsapp = WhatsAppChannel("whatsapp-main")
    ai = AIChannel(
        "ai-main",
        provider=MockAIProvider(responses=["Hello from WhatsApp!"]),
        system_prompt="You are a WhatsApp assistant.",
    )

    kit.register_channel(whatsapp)
    kit.register_channel(ai)

    await kit.create_room(room_id="wa-room")
    await kit.attach_channel("wa-room", "whatsapp-main")
    await kit.attach_channel("wa-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # Process inbound WhatsApp message
    await kit.process_inbound(
        InboundMessage(
            channel_id="whatsapp-main",
            sender_id="+15559876543",
            content=TextContent(body="Hi, I need help!"),
        )
    )


asyncio.run(main())
```

## Core Configuration

### WhatsApp Cloud API

```python
from roomkit import WhatsAppChannel

whatsapp = WhatsAppChannel(
    "whatsapp-cloud",
    provider=whatsapp_provider,  # WhatsApp Cloud API provider
)
```

### WhatsApp Personal (neonize)

For personal WhatsApp accounts using the neonize library:

```python
from roomkit import WhatsAppPersonalChannel, WhatsAppPersonalProvider

provider = WhatsAppPersonalProvider()
whatsapp = WhatsAppPersonalChannel("whatsapp-personal", provider=provider)
```

The personal backend uses QR code authentication:

```python
from roomkit.sources import WhatsAppPersonalSourceProvider

source = WhatsAppPersonalSourceProvider()

# QR code callback for pairing
@source.on_qr_code
async def on_qr(qr_data: str):
    print(f"Scan QR code: {qr_data}")

@source.on_authenticated
async def on_auth():
    print("WhatsApp authenticated!")

@source.on_message
async def on_message(message):
    await kit.process_inbound(message)
```

### Webhook Parsing (Cloud API)

```python
# FastAPI example
from fastapi import FastAPI, Request

app = FastAPI()

@app.post("/webhook/whatsapp")
async def whatsapp_webhook(request: Request):
    payload = await request.json()
    # Parse Cloud API webhook format
    for entry in payload.get("entry", []):
        for change in entry.get("changes", []):
            messages = change.get("value", {}).get("messages", [])
            for msg in messages:
                message = InboundMessage(
                    channel_id="whatsapp-main",
                    sender_id=msg["from"],
                    content=TextContent(body=msg.get("text", {}).get("body", "")),
                )
                await kit.process_inbound(message)
    return {"status": "ok"}

# Webhook verification (GET request)
@app.get("/webhook/whatsapp")
async def verify_webhook(request: Request):
    mode = request.query_params.get("hub.mode")
    token = request.query_params.get("hub.verify_token")
    challenge = request.query_params.get("hub.challenge")
    if mode == "subscribe" and token == os.environ["WHATSAPP_VERIFY_TOKEN"]:
        return int(challenge)
    return {"error": "Forbidden"}, 403
```

## Common Patterns

### WhatsApp with AI and Templates

```python
from roomkit import TemplateContent

# Send a template message (required for initiating conversations)
template = TemplateContent(
    template_name="hello_world",
    language_code="en",
    components=[],
)
```

### Content Types

WhatsApp supports multiple content types:

```python
from roomkit import TextContent, MediaContent, LocationContent, CompositeContent

# Text
text = TextContent(body="Hello!")

# Image
image = MediaContent(
    url="https://example.com/photo.jpg",
    mime_type="image/jpeg",
    caption="Check this out",
)

# Location
location = LocationContent(
    latitude=37.7749,
    longitude=-122.4194,
    label="San Francisco",
    address="San Francisco, CA",
)

# Composite (multi-part)
composite = CompositeContent(
    parts=[
        TextContent(body="Here's the location:"),
        LocationContent(latitude=37.7749, longitude=-122.4194),
    ]
)
```

### WhatsApp Personal with Events

```python
from roomkit.sources import WhatsAppPersonalSourceProvider

source = WhatsAppPersonalSourceProvider()

@source.on_presence
async def on_presence(jid: str, status: str):
    print(f"{jid} is {status}")

@source.on_receipt
async def on_receipt(jid: str, receipt_type: str):
    print(f"Receipt from {jid}: {receipt_type}")

# Attach source to kit
kit.attach_source("whatsapp-source", source)
```

### Channel Capabilities

WhatsApp supports:
- Text messages
- Rich text with buttons and quick replies
- Media: images, videos, audio, documents
- Location messages
- Template messages (Business API)
- Reactions
