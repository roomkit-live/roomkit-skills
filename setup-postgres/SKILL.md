---
name: setup-postgres
description: Configure PostgresStore for production RoomKit deployments with connection pooling, schema migration, and telemetry integration. Use when the user wants to persist rooms and events in PostgreSQL, migrate from InMemoryStore, or set up production storage.
license: MIT
compatibility: Requires Python 3.12+, roomkit[postgres] extras, and a PostgreSQL database.
metadata:
  author: roomkit
  version: "1.0"
---

# PostgreSQL Store Setup

## Quick Start

```bash
uv add "roomkit[postgres]"
```

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    InboundMessage,
    PostgresStore,
    RoomKit,
    TextContent,
    WebSocketChannel,
)


async def main() -> None:
    # Connect to PostgreSQL
    store = PostgresStore(
        dsn=os.environ.get("DATABASE_URL", "postgresql://localhost:5432/roomkit"),
    )

    async with store:
        kit = RoomKit(store=store)

        ws = WebSocketChannel("ws-user")
        kit.register_channel(ws)

        await kit.create_room(room_id="persistent-room")
        await kit.attach_channel("persistent-room", "ws-user")

        await kit.process_inbound(
            InboundMessage(
                channel_id="ws-user",
                sender_id="user-1",
                content=TextContent(body="This is persisted in PostgreSQL!"),
            )
        )

        # Query events
        events = await store.list_events("persistent-room")
        print(f"Stored {len(events)} events")

        # Paginated query
        page = await store.list_events("persistent-room", offset=0, limit=10)
        print(f"Page: {len(page)} events")

        count = await store.get_event_count("persistent-room")
        print(f"Total events: {count}")


asyncio.run(main())
```

## Core Configuration

### PostgresStore Constructor

```python
from roomkit import PostgresStore

store = PostgresStore(
    dsn="postgresql://user:pass@localhost:5432/roomkit",
    min_pool_size=2,     # Minimum connection pool size (default: 2)
    max_pool_size=10,    # Maximum connection pool size (default: 10)
)
```

### Connection as Context Manager

Always use `async with` for proper connection lifecycle:

```python
async with PostgresStore(dsn=os.environ["DATABASE_URL"]) as store:
    kit = RoomKit(store=store)
    # ... use kit
```

Or manage manually:

```python
store = PostgresStore(dsn=os.environ["DATABASE_URL"])
await store.connect()
try:
    kit = RoomKit(store=store)
    # ... use kit
finally:
    await store.close()
```

### Schema

PostgresStore creates the following tables automatically on first connect:

- `rooms` — room metadata, status, timers
- `events` — all room events (messages, system, voice, etc.)
- `bindings` — channel bindings per room
- `participants` — room participants with roles
- `identities` — user identity records
- `identity_addresses` — channel_type + address -> identity mapping
- `tasks` — background tasks
- `observations` — agent observations
- `read_markers` — per-channel read tracking
- `schema_version` — migration tracking

All tables use JSONB columns for metadata. Events and rooms support GIN indexes for efficient JSON queries.

### Environment Variables

```bash
# Connection string
export DATABASE_URL=postgresql://user:password@localhost:5432/roomkit

# Or individual components
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=roomkit
export PGUSER=user
export PGPASSWORD=password
```

## Common Patterns

### Production Setup with Telemetry

```python
from __future__ import annotations

import asyncio
import os

from roomkit import (
    OpenTelemetryProvider,
    PostgresStore,
    RoomKit,
    TelemetryConfig,
)


async def main() -> None:
    telemetry = TelemetryConfig(
        provider=OpenTelemetryProvider(service_name="my-agent"),
    )

    async with PostgresStore(dsn=os.environ["DATABASE_URL"]) as store:
        kit = RoomKit(
            store=store,
            telemetry=telemetry,
        )
        # PostgresStore automatically wraps queries with telemetry spans


asyncio.run(main())
```

### Fallback to InMemoryStore

```python
from roomkit import InMemoryStore, PostgresStore, RoomKit

try:
    store = PostgresStore(dsn=os.environ.get("DATABASE_URL", ""))
    await store.connect()
except Exception:
    print("PostgreSQL unavailable, falling back to InMemoryStore")
    store = InMemoryStore()

kit = RoomKit(store=store)
```

### Querying Rooms and Events

```python
# Find rooms by status
rooms = await store.list_rooms(status="active")

# Find rooms by metadata
rooms = await store.find_rooms(metadata={"customer_id": "cust-123"})

# Find latest room for a participant
room = await store.find_latest_room(
    participant_id="user-1",
    channel_type=ChannelType.SMS,
)

# Get room participants
participants = await store.list_participants("room-id")

# Get event timeline
events = await store.list_events("room-id", offset=0, limit=100)

# Check idempotency (prevent duplicate events)
exists = await store.check_idempotency("room-id", "idempotency-key-123")
```
