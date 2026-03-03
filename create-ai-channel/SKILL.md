---
name: create-ai-channel
description: Create AI channels with providers like Anthropic Claude, OpenAI GPT, Google Gemini, and Mistral. Configure system prompts, tools, temperature, and tool policies. Use when the user wants to add AI intelligence to a room, configure AI providers, or set up tool calling.
license: MIT
compatibility: Requires Python 3.12+, roomkit package, and an API key for the chosen AI provider.
metadata:
  author: roomkit
  version: "1.0"
---

# AI Channel Creation

## Quick Start

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
        temperature=0.7,
        max_tokens=1024,
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
            content=TextContent(body="Explain async/await in Python"),
        )
    )

    for ev in inbox:
        print(f"AI: {ev.content.body}")


asyncio.run(main())
```

## Core Configuration

### AIChannel Constructor

```python
from roomkit.channels.ai import AIChannel

ai = AIChannel(
    "ai-main",
    provider=provider,                    # Required: AIProvider instance
    system_prompt="You are...",           # System prompt
    temperature=0.7,                      # Sampling temperature (0.0-1.0)
    max_tokens=1024,                      # Max response tokens
    max_context_events=50,                # Conversation history window
    tool_handler=my_tool_handler,         # Tool execution callback
    max_tool_rounds=200,                  # Tool call loop limit
    tool_loop_timeout_seconds=300.0,      # Total tool loop timeout
    retry_policy=RetryPolicy(max_retries=3),
    fallback_provider=backup_provider,    # Fallback on primary failure
    thinking_budget=4096,                 # Extended thinking token budget
)
```

### Provider Setup

All providers follow the same pattern: Config + Provider.

```python
# Anthropic Claude
from roomkit import AnthropicAIProvider, AnthropicConfig

provider = AnthropicAIProvider(
    AnthropicConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
    )
)

# OpenAI GPT
from roomkit import OpenAIAIProvider, OpenAIConfig

provider = OpenAIAIProvider(
    OpenAIConfig(
        api_key=os.environ["OPENAI_API_KEY"],
        model="gpt-4o",
    )
)

# Google Gemini
from roomkit import GeminiAIProvider, GeminiConfig

provider = GeminiAIProvider(
    GeminiConfig(
        api_key=os.environ["GEMINI_API_KEY"],
        model="gemini-2.0-flash",
    )
)

# Mistral
from roomkit import MistralAIProvider, MistralConfig

provider = MistralAIProvider(
    MistralConfig(
        api_key=os.environ["MISTRAL_API_KEY"],
        model="mistral-large-latest",
    )
)

# Azure OpenAI
from roomkit import AzureAIProvider, AzureAIConfig

provider = AzureAIProvider(
    AzureAIConfig(
        api_key=os.environ["AZURE_OPENAI_API_KEY"],
        endpoint="https://my-resource.openai.azure.com",
        deployment="gpt-4o",
        api_version="2024-02-15-preview",
    )
)

# Local vLLM
from roomkit import create_vllm_provider, VLLMConfig

provider = create_vllm_provider(
    VLLMConfig(base_url="http://localhost:8000/v1", model="meta-llama/Llama-3-8B")
)
```

### Tool Calling

```python
from roomkit import AITool

# Define tools
tools = [
    AITool(
        name="get_weather",
        description="Get current weather for a city",
        input_schema={
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "City name"},
            },
            "required": ["city"],
        },
    ),
]

# Tool handler
async def handle_tool(name: str, input: dict) -> str:
    if name == "get_weather":
        return f"Weather in {input['city']}: 72F, sunny"
    return "Unknown tool"

ai = AIChannel(
    "ai-tools",
    provider=provider,
    system_prompt="You are a weather assistant.",
    tool_handler=handle_tool,
)
```

### Streaming Responses

AI responses are streamed by default to WebSocket channels. The streaming is handled automatically — no extra configuration needed.

## Common Patterns

### AI with Fallback Provider

```python
primary = AnthropicAIProvider(AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"]))
fallback = OpenAIAIProvider(OpenAIConfig(api_key=os.environ["OPENAI_API_KEY"]))

ai = AIChannel(
    "ai-resilient",
    provider=primary,
    fallback_provider=fallback,
    retry_policy=RetryPolicy(max_retries=2),
)
```

### Extended Thinking

```python
ai = AIChannel(
    "ai-thinker",
    provider=AnthropicAIProvider(
        AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"], model="claude-sonnet-4-20250514")
    ),
    thinking_budget=4096,  # Token budget for extended reasoning
)
```

### Multiple AI Channels in One Room

```python
analyst = AIChannel("ai-analyst", provider=provider, system_prompt="You analyze data.")
writer = AIChannel("ai-writer", provider=provider, system_prompt="You write content.")

kit.register_channel(analyst)
kit.register_channel(writer)

await kit.attach_channel("room", "ai-analyst", category=ChannelCategory.INTELLIGENCE)
await kit.attach_channel("room", "ai-writer", category=ChannelCategory.INTELLIGENCE)
```

## References

- [AI Provider Configurations](./references/providers.md)
