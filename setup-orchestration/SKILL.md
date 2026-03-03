---
name: setup-orchestration
description: Set up multi-agent orchestration with ConversationRouter, phases, handoffs, delegation, and pipeline stages. Use when the user wants to route messages between AI agents, implement conversation phases, hand off between agents, or delegate background tasks.
license: MIT
compatibility: Requires Python 3.12+ and roomkit package.
metadata:
  author: roomkit
  version: "1.0"
---

# Orchestration Setup

## Quick Start

```python
from __future__ import annotations

import asyncio

from roomkit import (
    ChannelCategory,
    ConversationRouter,
    HandoffRequest,
    HookTrigger,
    InboundMessage,
    MockAIProvider,
    RoomKit,
    RoutingConditions,
    RoutingRule,
    TextContent,
    WebSocketChannel,
    build_handoff_tool,
    setup_handoff,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    ws = WebSocketChannel("ws-user")

    # Two AI agents with different roles
    triage = AIChannel(
        "ai-triage",
        provider=MockAIProvider(responses=["Let me transfer you to billing."]),
        system_prompt="You are a triage agent. Route customers to the right department.",
    )
    billing = AIChannel(
        "ai-billing",
        provider=MockAIProvider(responses=["I can help with your bill."]),
        system_prompt="You are a billing specialist.",
    )

    kit.register_channel(ws)
    kit.register_channel(triage)
    kit.register_channel(billing)

    await kit.create_room(room_id="support")
    await kit.attach_channel("support", "ws-user")
    await kit.attach_channel("support", "ai-triage", category=ChannelCategory.INTELLIGENCE)
    await kit.attach_channel("support", "ai-billing", category=ChannelCategory.INTELLIGENCE)

    @kit.hook(HookTrigger.ON_HANDOFF)
    async def log_handoff(event, ctx):
        print(f"Handoff: {event.metadata}")

    await kit.process_inbound(
        InboundMessage(
            channel_id="ws-user",
            sender_id="user",
            content=TextContent(body="I have a billing question"),
        )
    )


asyncio.run(main())
```

## Core Configuration

### ConversationRouter

Route messages to different AI agents based on conditions:

```python
from roomkit import ConversationRouter, RoutingRule, RoutingConditions

router = ConversationRouter(
    rules=[
        RoutingRule(
            name="billing",
            conditions=RoutingConditions(phase="billing"),
            target_agent="ai-billing",
            priority=10,
        ),
        RoutingRule(
            name="tech-support",
            conditions=RoutingConditions(phase="technical"),
            target_agent="ai-tech",
            priority=10,
        ),
        RoutingRule(
            name="default",
            conditions=RoutingConditions(),
            target_agent="ai-triage",
            priority=0,
        ),
    ],
)
```

### Conversation Phases

Track conversation state through defined phases:

```python
from roomkit import (
    ConversationPhase,
    ConversationState,
    get_conversation_state,
    set_conversation_state,
)

# Define phases
class Phase(ConversationPhase):
    TRIAGE = "triage"
    BILLING = "billing"
    TECHNICAL = "technical"
    RESOLVED = "resolved"

# Set initial phase
set_conversation_state("room-id", ConversationState(phase=Phase.TRIAGE))

# Transition phases
@kit.hook(HookTrigger.ON_PHASE_TRANSITION)
async def log_phase(event, ctx):
    transition = event.metadata
    print(f"Phase: {transition['from_phase']} -> {transition['to_phase']}")
```

### Handoffs

Transfer conversations between AI agents:

```python
from roomkit import (
    build_handoff_tool,
    setup_handoff,
    HandoffHandler,
    HandoffRequest,
    HandoffResult,
    HANDOFF_TOOL,
)

# Provide the handoff tool to AI
ai = AIChannel(
    "ai-triage",
    provider=provider,
    system_prompt="Route to billing or tech support using the handoff tool.",
    tool_handler=handle_tools,
)

# The built-in handoff tool
handoff_tool = build_handoff_tool()
# Or use the pre-built constant:
# HANDOFF_TOOL
```

### Delegation (Background Tasks)

Delegate long-running tasks to background agents:

```python
from roomkit import (
    build_delegate_tool,
    setup_delegation,
    DELEGATE_TOOL,
    DelegateHandler,
    DelegatedTask,
    DelegatedTaskResult,
    TaskRunner,
    ImmediateDelivery,
)

# Provide the delegate tool to AI
delegate_tool = build_delegate_tool()

# Configure task delivery strategy
kit = RoomKit(
    task_runner=my_task_runner,
    delivery_strategy=ImmediateDelivery(),
)

@kit.hook(HookTrigger.ON_TASK_DELEGATED)
async def on_delegated(event, ctx):
    print(f"Task delegated: {event.metadata['task_id']}")

@kit.hook(HookTrigger.ON_TASK_COMPLETED)
async def on_completed(event, ctx):
    print(f"Task completed: {event.metadata['task_id']}")
```

### Pipeline Stages

Multi-stage message processing before AI generation:

```python
from roomkit import ConversationPipeline, PipelineStage

pipeline = ConversationPipeline(
    stages=[
        PipelineStage(name="classify", handler=classify_intent),
        PipelineStage(name="enrich", handler=enrich_context),
        PipelineStage(name="route", handler=route_to_agent),
    ],
)
```

## Common Patterns

### Multi-Department Support Bot

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
    WebSocketChannel,
    build_handoff_tool,
)
from roomkit.channels.ai import AIChannel


async def main() -> None:
    kit = RoomKit()

    provider = AnthropicAIProvider(
        AnthropicConfig(api_key=os.environ["ANTHROPIC_API_KEY"])
    )

    triage = AIChannel(
        "ai-triage",
        provider=provider,
        system_prompt=(
            "You are a triage agent. Determine if the customer needs billing, "
            "technical, or general support. Use the handoff tool to transfer."
        ),
    )

    billing = AIChannel(
        "ai-billing",
        provider=provider,
        system_prompt="You are a billing specialist. Help with invoices and payments.",
    )

    tech = AIChannel(
        "ai-tech",
        provider=provider,
        system_prompt="You are a technical support engineer. Help with product issues.",
    )

    ws = WebSocketChannel("ws-user")

    for ch in [ws, triage, billing, tech]:
        kit.register_channel(ch)

    await kit.create_room(room_id="support")
    await kit.attach_channel("support", "ws-user")
    await kit.attach_channel("support", "ai-triage", category=ChannelCategory.INTELLIGENCE)
    await kit.attach_channel("support", "ai-billing", category=ChannelCategory.INTELLIGENCE)
    await kit.attach_channel("support", "ai-tech", category=ChannelCategory.INTELLIGENCE)

    @kit.hook(HookTrigger.ON_HANDOFF)
    async def log_handoff(event, ctx):
        print(f"Handoff: {event.metadata}")


asyncio.run(main())
```

### Steering Directives

Control AI behavior mid-conversation:

```python
from roomkit import SteeringDirective, InjectMessage, UpdateSystemPrompt, Cancel

# Inject a system message
inject = InjectMessage(content=TextContent(body="[System: VIP customer detected]"))

# Update system prompt dynamically
update = UpdateSystemPrompt(new_prompt="You are now in escalation mode.")

# Cancel current generation
cancel = Cancel(reason="User disconnected")
```

## References

- [Orchestration API Reference](./references/orchestration-api.md)
