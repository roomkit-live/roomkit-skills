---
name: create-telegram-bot
description: Create a Telegram bot channel with RoomKit using the Telegram Bot API. Configure webhooks, parse updates, and handle text, media, and location messages. Use when the user wants to build a Telegram bot, parse Telegram webhooks, or integrate Telegram messaging.
license: MIT
compatibility: Requires Python 3.12+, roomkit package, and a Telegram Bot Token from BotFather.
metadata:
  author: roomkit
  version: "1.0"
---

# Telegram Bot Channel

## Quick Start

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    ChannelCategory,
    InboundMessage,
    MockAIProvider,
    RoomKit,
    TelegramBotProvider,
    TelegramChannel,
    TelegramConfig,
    TextContent,
    parse_telegram_webhook,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    telegram = TelegramChannel(
        "telegram-main",
        provider=TelegramBotProvider(
            TelegramConfig(
                bot_token=os.environ["TELEGRAM_BOT_TOKEN"],
            )
        ),
    )

    ai = AIChannel(
        "ai-main",
        provider=MockAIProvider(responses=["Hello from RoomKit!"]),
        system_prompt="You are a helpful Telegram bot.",
    )

    kit.register_channel(telegram)
    kit.register_channel(ai)

    await kit.create_room(room_id="telegram-room")
    await kit.attach_channel("telegram-room", "telegram-main")
    await kit.attach_channel("telegram-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # Simulate incoming Telegram webhook
    webhook_payload = {
        "update_id": 123456789,
        "message": {
            "message_id": 1,
            "from": {"id": 987654321, "first_name": "Alice", "is_bot": False},
            "chat": {"id": 987654321, "type": "private"},
            "date": 1700000000,
            "text": "Hello bot!",
        },
    }

    message = parse_telegram_webhook(webhook_payload)
    if message:
        await kit.process_inbound(message)


asyncio.run(main())
```

## Core Configuration

### Telegram Provider Setup

```python
from roomkit import TelegramBotProvider, TelegramConfig, TelegramChannel

provider = TelegramBotProvider(
    TelegramConfig(
        bot_token=os.environ["TELEGRAM_BOT_TOKEN"],
        webhook_secret="my-secret-token",    # Optional: webhook signature verification
        timeout=30,                           # Request timeout in seconds
    )
)

telegram = TelegramChannel("telegram-main", provider=provider)
```

### Webhook Setup

Set up a webhook endpoint in your web framework:

```python
# FastAPI example
from fastapi import FastAPI, Request
from roomkit import parse_telegram_webhook

app = FastAPI()

@app.post("/webhook/telegram")
async def telegram_webhook(request: Request):
    payload = await request.json()
    message = parse_telegram_webhook(payload)
    if message:
        await kit.process_inbound(message)
    return {"ok": True}
```

Register the webhook with Telegram:

```bash
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://your-domain.com/webhook/telegram", "secret_token": "my-secret-token"}'
```

### Webhook Signature Verification

```python
from roomkit import TelegramBotProvider

# Verify the X-Telegram-Bot-Api-Secret-Token header
is_valid = provider.verify_signature(
    headers={"X-Telegram-Bot-Api-Secret-Token": "my-secret-token"},
)
```

## Common Patterns

### Telegram Bot with AI

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
    TelegramBotProvider,
    TelegramChannel,
    TelegramConfig,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    telegram = TelegramChannel(
        "telegram-main",
        provider=TelegramBotProvider(
            TelegramConfig(bot_token=os.environ["TELEGRAM_BOT_TOKEN"])
        ),
    )

    ai = AIChannel(
        "ai-main",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
        ),
        system_prompt="You are a helpful Telegram bot. Use markdown formatting.",
    )

    kit.register_channel(telegram)
    kit.register_channel(ai)

    await kit.create_room(room_id="tg-room")
    await kit.attach_channel("tg-room", "telegram-main")
    await kit.attach_channel("tg-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    @kit.hook(HookTrigger.AFTER_BROADCAST)
    async def log_telegram(event, ctx):
        print(f"[Telegram] {event.source.direction}: {event.content}")


asyncio.run(main())
```

### Channel Capabilities

Telegram supports:
- Text with Markdown/HTML formatting
- Rich text with buttons and inline keyboards
- Media: images, videos, documents, audio
- Location messages
- Message edit and delete
- Reactions
