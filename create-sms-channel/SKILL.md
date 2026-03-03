---
name: create-sms-channel
description: Create SMS and RCS channels with Twilio, Sinch, or Telnyx providers. Configure webhooks, delivery status tracking, and phone number validation. Use when the user wants to send or receive SMS/RCS messages, set up Twilio/Sinch/Telnyx, or handle inbound text messages.
license: MIT
compatibility: Requires Python 3.12+, roomkit package, and an account with Twilio, Sinch, or Telnyx.
metadata:
  author: roomkit
  version: "1.0"
---

# SMS/RCS Channel Creation

## Quick Start

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    ChannelType,
    InboundMessage,
    RoomKit,
    SMSChannel,
    TextContent,
    TwilioConfig,
    TwilioSMSProvider,
)


async def main() -> None:
    kit = RoomKit()

    sms = SMSChannel(
        "sms-main",
        provider=TwilioSMSProvider(
            TwilioConfig(
                account_sid=os.environ["TWILIO_ACCOUNT_SID"],
                auth_token=os.environ["TWILIO_AUTH_TOKEN"],
                from_number="+15551234567",
            )
        ),
    )
    kit.register_channel(sms)

    await kit.create_room(room_id="sms-room")
    await kit.attach_channel("sms-room", "sms-main")

    # Process inbound SMS (from webhook)
    await kit.process_inbound(
        InboundMessage(
            channel_id="sms-main",
            sender_id="+15559876543",
            content=TextContent(body="Hi there!"),
        )
    )


asyncio.run(main())
```

## Core Configuration

### Twilio SMS

```python
from roomkit import TwilioSMSProvider, TwilioConfig, SMSChannel

provider = TwilioSMSProvider(
    TwilioConfig(
        account_sid=os.environ["TWILIO_ACCOUNT_SID"],
        auth_token=os.environ["TWILIO_AUTH_TOKEN"],
        from_number="+15551234567",
    )
)
sms = SMSChannel("sms-twilio", provider=provider)
```

### Sinch SMS

```python
from roomkit import SinchSMSProvider, SinchConfig, SMSChannel

provider = SinchSMSProvider(
    SinchConfig(
        api_token=os.environ["SINCH_API_TOKEN"],
        service_plan_id=os.environ["SINCH_SERVICE_PLAN_ID"],
        from_number="+15551234567",
    )
)
sms = SMSChannel("sms-sinch", provider=provider)
```

### Telnyx SMS

```python
from roomkit import TelnyxSMSProvider, TelnyxConfig, SMSChannel

provider = TelnyxSMSProvider(
    TelnyxConfig(
        api_key=os.environ["TELNYX_API_KEY"],
        from_number="+15551234567",
    )
)
sms = SMSChannel("sms-telnyx", provider=provider)
```

### RCS Channels

```python
from roomkit import RCSChannel, TwilioRCSProvider, TwilioRCSConfig

rcs = RCSChannel(
    "rcs-main",
    provider=TwilioRCSProvider(
        TwilioRCSConfig(
            account_sid=os.environ["TWILIO_ACCOUNT_SID"],
            auth_token=os.environ["TWILIO_AUTH_TOKEN"],
        )
    ),
    fallback=True,  # Fall back to SMS if RCS unavailable
)
```

### Webhook Parsing

Each provider has a webhook parser for inbound messages:

```python
from roomkit import parse_twilio_webhook, parse_sinch_webhook, parse_telnyx_webhook

# In your web framework (FastAPI example):
@app.post("/webhook/twilio")
async def twilio_webhook(request: Request):
    body = await request.form()
    message = parse_twilio_webhook(dict(body))
    await kit.process_inbound(message)
    return Response(status_code=200)
```

### Phone Number Utilities

```python
from roomkit import is_valid_phone, normalize_phone

is_valid_phone("+15551234567")   # True
normalize_phone("(555) 123-4567", country="US")  # "+15551234567"
```

## Common Patterns

### SMS with AI Assistant

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    AnthropicAIProvider,
    AnthropicConfig,
    ChannelCategory,
    RoomKit,
    SMSChannel,
    TwilioConfig,
    TwilioSMSProvider,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    sms = SMSChannel(
        "sms-main",
        provider=TwilioSMSProvider(TwilioConfig(
            account_sid=os.environ["TWILIO_ACCOUNT_SID"],
            auth_token=os.environ["TWILIO_AUTH_TOKEN"],
            from_number="+15551234567",
        )),
    )

    ai = AIChannel(
        "ai-main",
        provider=AnthropicAIProvider(
            AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
        ),
        system_prompt="You are an SMS assistant. Keep replies under 160 characters.",
    )

    kit.register_channel(sms)
    kit.register_channel(ai)

    await kit.create_room(room_id="sms-ai-room")
    await kit.attach_channel("sms-ai-room", "sms-main")
    await kit.attach_channel("sms-ai-room", "ai-main", category=ChannelCategory.INTELLIGENCE)


asyncio.run(main())
```

### Delivery Status Tracking

```python
from roomkit import HookTrigger

@kit.hook(HookTrigger.ON_DELIVERY_STATUS)
async def track_delivery(event, ctx):
    status = event.metadata.get("delivery_status")
    recipient = event.metadata.get("recipient")
    print(f"SMS to {recipient}: {status}")  # delivered, failed, etc.
```

### Media Messages (MMS)

```python
from roomkit import MediaContent

await kit.process_inbound(
    InboundMessage(
        channel_id="sms-main",
        sender_id="+15559876543",
        content=MediaContent(
            url="https://example.com/photo.jpg",
            mime_type="image/jpeg",
            caption="Check this out!",
        ),
    )
)
```
