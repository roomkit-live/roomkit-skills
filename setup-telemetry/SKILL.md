---
name: setup-telemetry
description: Set up telemetry and observability for RoomKit with OpenTelemetry, console logging, and protocol tracing. Monitor LLM latency, voice pipeline performance, hook execution, and delivery status. Use when the user wants to add tracing, metrics, or observability to their RoomKit application.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package. OpenTelemetry requires opentelemetry-api and opentelemetry-sdk.
metadata:
  author: roomkit
  version: "1.0"
---

# Telemetry and Observability Setup

## Quick Start

```python
from __future__ import annotations

import asyncio
import logging

from roomkit import (
    ConsoleTelemetryProvider,
    InboundMessage,
    MockAIProvider,
    RoomKit,
    TextContent,
    WebSocketChannel,
    ChannelCategory,
)
from roomkit.channels.ai import AIChannel

logging.basicConfig(level=logging.INFO)


async def main() -> None:
    # Console telemetry — logs spans to stdout
    telemetry = ConsoleTelemetryProvider()

    kit = RoomKit(telemetry=telemetry)

    ws = WebSocketChannel("ws-user")
    ai = AIChannel("ai-main", provider=MockAIProvider(responses=["Hello!"]))

    kit.register_channel(ws)
    kit.register_channel(ai)

    await kit.create_room(room_id="traced-room")
    await kit.attach_channel("traced-room", "ws-user")
    await kit.attach_channel("traced-room", "ai-main", category=ChannelCategory.INTELLIGENCE)

    # This will produce telemetry spans for:
    # - Inbound pipeline processing
    # - Hook execution
    # - LLM generation
    # - Event broadcast
    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-user",
            sender_id="user",
            content=TextContent(body="Hello with tracing!"),
        )
    )


asyncio.run(main())
```

## Core Configuration

### Telemetry Providers

RoomKit includes three telemetry providers:

```python
# 1. Noop (default) — zero overhead
from roomkit import NoopTelemetryProvider
kit = RoomKit(telemetry=NoopTelemetryProvider())

# 2. Console — logs to stdout via logging module
from roomkit import ConsoleTelemetryProvider
kit = RoomKit(telemetry=ConsoleTelemetryProvider())

# 3. OpenTelemetry — full OTel SDK integration
from roomkit import OpenTelemetryProvider
kit = RoomKit(telemetry=OpenTelemetryProvider(service_name="my-agent"))
```

### TelemetryConfig

Fine-grained telemetry control:

```python
from roomkit import TelemetryConfig, OpenTelemetryProvider

config = TelemetryConfig(
    provider=OpenTelemetryProvider(service_name="my-agent"),
    sample_rate=1.0,                          # 0.0-1.0 (default: 1.0)
    metadata={"environment": "production"},   # Global metadata
    suppressed_hook_triggers={                 # High-frequency hooks to suppress
        "on_input_audio_level",
        "on_output_audio_level",
        "on_vad_audio_level",
    },
)

kit = RoomKit(telemetry=config)
```

### Span Kinds

RoomKit emits these span types:

| Span Kind | Description |
|-----------|-------------|
| `PIPELINE_INBOUND` | Inbound audio pipeline processing |
| `PIPELINE_OUTBOUND` | Outbound audio pipeline processing |
| `STT_TRANSCRIBE` | Speech-to-text transcription |
| `STT_STREAM` | Streaming STT session |
| `TTS_SYNTHESIZE` | Text-to-speech synthesis |
| `LLM_GENERATE` | AI model generation |
| `LLM_TOOL_CALL` | AI tool call execution |
| `HOOK_SYNC` | Synchronous hook execution |
| `HOOK_ASYNC` | Asynchronous hook execution |
| `INBOUND_PIPELINE` | Full inbound message processing |
| `BROADCAST` | Event broadcast to channels |
| `DELIVERY` | Message delivery to a channel |
| `VOICE_SESSION` | Voice session lifecycle |
| `STORE_QUERY` | Database query |
| `BACKEND_CONNECT` | Backend connection |
| `REALTIME_SESSION` | Realtime voice session |
| `REALTIME_TURN` | Realtime voice turn |
| `REALTIME_TOOL_CALL` | Realtime tool invocation |
| `PIPELINE_SPEECH_SEGMENT` | Speech segment processing |
| `CUSTOM` | Custom application spans |

### Span Attributes

Standard attributes on spans:

```python
from roomkit.telemetry import Attr

# Common
Attr.PROVIDER         # Provider name
Attr.MODEL            # Model name
Attr.ROOM_ID          # Room ID
Attr.SESSION_ID       # Voice session ID
Attr.CHANNEL_ID       # Channel ID

# Timing
Attr.TTFB_MS          # Time to first byte
Attr.DURATION_MS      # Total duration

# STT
Attr.STT_TEXT          # Transcribed text
Attr.STT_CONFIDENCE    # Confidence score

# TTS
Attr.TTS_VOICE         # Voice ID
Attr.TTS_CHAR_COUNT    # Character count

# LLM
Attr.LLM_INPUT_TOKENS  # Input token count
Attr.LLM_OUTPUT_TOKENS # Output token count
Attr.LLM_TOOL_COUNT    # Tool calls made

# Delivery
Attr.DELIVERY_SUCCESS   # Delivery succeeded
Attr.DELIVERY_ERROR     # Error message
```

## Common Patterns

### OpenTelemetry with Jaeger

```python
from __future__ import annotations

import asyncio

from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

from roomkit import OpenTelemetryProvider, RoomKit, TelemetryConfig


async def main() -> None:
    # Configure OTel SDK
    jaeger = JaegerExporter(agent_host_name="localhost", agent_port=6831)
    tracer_provider = TracerProvider()
    tracer_provider.add_span_processor(BatchSpanProcessor(jaeger))

    telemetry = TelemetryConfig(
        provider=OpenTelemetryProvider(
            service_name="my-voice-agent",
            tracer_provider=tracer_provider,
        ),
    )

    kit = RoomKit(telemetry=telemetry)


asyncio.run(main())
```

### Protocol Tracing

Monitor raw protocol-level events:

```python
from roomkit import HookTrigger

@kit.hook(HookTrigger.ON_PROTOCOL_TRACE)
async def trace_protocol(event, ctx):
    trace = event.metadata
    print(f"[{trace['level']}] {trace['message']}")
```

### Custom Spans

Create custom telemetry spans:

```python
from roomkit.telemetry import SpanKind

# Using context manager
with kit._telemetry.span(SpanKind.CUSTOM, "my_operation", room_id="room-1") as span_id:
    # ... your operation
    await kit._telemetry.set_attribute(span_id, "custom.key", "value")
```
