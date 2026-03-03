# Orchestration API Reference

## ConversationRouter

Routes messages to different AI agents based on conditions.

```python
from roomkit import ConversationRouter, RoutingRule, RoutingConditions

router = ConversationRouter(
    rules=[
        RoutingRule(
            name="billing",
            conditions=RoutingConditions(
                phase="billing",                      # Match conversation phase
                participant_role=ParticipantRole.MEMBER,  # Match participant role
                custom={"intent": "billing"},         # Custom condition
            ),
            target_agent="ai-billing",
            priority=10,                              # Higher = matched first
        ),
    ],
)
```

### RoutingRule

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Rule identifier |
| `conditions` | `RoutingConditions` | Match criteria |
| `target_agent` | `str` | AI channel ID to route to |
| `priority` | `int` | Priority (higher = first, default: 0) |

### RoutingConditions

| Field | Type | Description |
|-------|------|-------------|
| `phase` | `ConversationPhase \| None` | Match conversation phase |
| `participant_role` | `ParticipantRole \| None` | Match sender role |
| `custom` | `dict[str, Any]` | Custom key-value conditions |

## Conversation Phases

Track conversation state through user-defined phases.

```python
from roomkit import ConversationPhase, ConversationState

# Define phases as a StrEnum
class Phase(ConversationPhase):
    TRIAGE = "triage"
    BILLING = "billing"
    RESOLVED = "resolved"

# Get/set state
from roomkit import get_conversation_state, set_conversation_state

state = get_conversation_state("room-id")
set_conversation_state("room-id", ConversationState(
    phase=Phase.BILLING,
    metadata={"escalated": True},
))
```

### PhaseTransition

| Field | Type | Description |
|-------|------|-------------|
| `from_phase` | `ConversationPhase` | Previous phase |
| `to_phase` | `ConversationPhase` | New phase |
| `trigger` | `str` | What caused the transition |
| `metadata` | `dict[str, Any]` | Transition metadata |

## Handoffs

Transfer conversations between AI agents.

### Setup

```python
from roomkit import setup_handoff, build_handoff_tool, HANDOFF_TOOL

# Pre-built handoff tool (add to AI tools)
handoff_tool = build_handoff_tool()
# Or use the constant: HANDOFF_TOOL
```

### HandoffRequest

| Field | Type | Description |
|-------|------|-------------|
| `from_agent` | `str` | Source agent channel ID |
| `to_agent` | `str` | Target agent channel ID |
| `reason` | `str` | Why the handoff is happening |
| `metadata` | `dict[str, Any]` | Context to pass |

### HandoffResult

| Field | Type | Description |
|-------|------|-------------|
| `success` | `bool` | Whether handoff succeeded |
| `error` | `str \| None` | Error message if failed |

### HandoffHandler (ABC)

```python
from roomkit import HandoffHandler, HandoffRequest, HandoffResult

class MyHandler(HandoffHandler):
    async def handle(self, request: HandoffRequest) -> HandoffResult:
        # Validate and execute handoff
        return HandoffResult(success=True)

setup_handoff(kit, MyHandler())
```

### HandoffMemoryProvider

Summarizes conversation context for the receiving agent:

```python
from roomkit import HandoffMemoryProvider
```

## Delegation (Background Tasks)

Delegate long-running tasks to background agents.

### Setup

```python
from roomkit import (
    setup_delegation,
    build_delegate_tool,
    DELEGATE_TOOL,
    TaskRunner,
    ImmediateDelivery,
)

delegate_tool = build_delegate_tool()
# Or use the constant: DELEGATE_TOOL

kit = RoomKit(
    task_runner=my_runner,
    delivery_strategy=ImmediateDelivery(),
)
```

### Task Model

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Task identifier |
| `room_id` | `str` | Room context |
| `description` | `str` | What the task should do |
| `status` | `TaskStatus` | PENDING, IN_PROGRESS, COMPLETED, FAILED, CANCELLED |
| `created_at` | `datetime` | Creation timestamp |

### DelegatedTask

| Field | Type | Description |
|-------|------|-------------|
| `task_id` | `str` | Task identifier |
| `agent_id` | `str` | Agent to execute |
| `instructions` | `str` | Task instructions |
| `context` | `dict[str, Any]` | Additional context |

### Delivery Strategies

| Strategy | Description |
|----------|-------------|
| `ImmediateDelivery` | Deliver result as soon as task completes |
| `ContextOnlyDelivery` | Add to context only, don't send message |
| `BackgroundTaskDeliveryStrategy` | Custom delivery logic |
| `WaitForIdleDelivery` | Wait for conversation idle before delivering |

### TaskRunner (ABC)

```python
from roomkit import TaskRunner, DelegatedTask, DelegatedTaskResult

class MyRunner(TaskRunner):
    async def run(self, task: DelegatedTask) -> DelegatedTaskResult:
        # Execute the task
        return DelegatedTaskResult(
            task_id=task.task_id,
            status=TaskStatus.COMPLETED,
            result={"data": "result"},
        )
```

## ConversationPipeline

Multi-stage message processing:

```python
from roomkit import ConversationPipeline, PipelineStage

pipeline = ConversationPipeline(
    stages=[
        PipelineStage(name="classify", handler=classify_handler),
        PipelineStage(name="enrich", handler=enrich_handler),
        PipelineStage(name="route", handler=route_handler),
    ],
)
```

## Steering Directives

Control AI behavior mid-conversation:

```python
from roomkit import SteeringDirective, InjectMessage, UpdateSystemPrompt, Cancel

# Inject a message into the conversation
InjectMessage(content=TextContent(body="[System notice]"))

# Update the AI's system prompt
UpdateSystemPrompt(new_prompt="New instructions...")

# Cancel current AI generation
Cancel(reason="User disconnected")
```

## Orchestration Hooks

| Hook Trigger | Description |
|-------------|-------------|
| `ON_PHASE_TRANSITION` | Phase changed (metadata has from/to) |
| `ON_HANDOFF` | Handoff initiated |
| `ON_HANDOFF_REJECTED` | Handoff was rejected |
| `ON_TASK_DELEGATED` | Task delegated to background agent |
| `ON_TASK_COMPLETED` | Background task completed |
