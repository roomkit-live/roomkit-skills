# RoomKit Skills

RoomKit is a pure async Python library for multi-channel conversations. It provides a unified abstraction for orchestrating messages across SMS, RCS, Email, WhatsApp, Messenger, Teams, Telegram, Voice, WebSocket, HTTP, and AI channels — all connected through rooms with hooks and pluggable backends.

## Available Skills

| Skill | Description |
|-------|-------------|
| `setup-project` | Install RoomKit, create your first room, run the quickstart |
| `create-room` | Room lifecycle, participants, channel binding, permissions |
| `create-ai-channel` | AI providers (Anthropic, OpenAI, Gemini, Mistral), tools, system prompts |
| `create-voice-agent` | Voice channel with STT/TTS, audio pipeline, interruption handling |
| `create-sms-channel` | SMS/RCS via Twilio, Sinch, Telnyx with delivery status |
| `add-hooks` | Hook system with 40+ triggers, sync/async execution, filtering |
| `create-multichannel` | Bridge multiple channel types in one room with content transcoding |
| `setup-orchestration` | ConversationRouter, phases, handoffs, delegation, pipeline stages |
| `create-telegram-bot` | Telegram Bot API channel with webhook parsing |
| `create-whatsapp-agent` | WhatsApp Cloud API and Personal (neonize) channels |
| `setup-postgres` | PostgresStore for production persistence |
| `setup-telemetry` | OpenTelemetry, console telemetry, protocol tracing |

## Configuration

```bash
uv add roomkit        # recommended
pip install roomkit   # alternative
```

Python 3.12+ required. All examples use `async`/`await`.

## Links

- Docs: https://www.roomkit.live
- RFC: `../roomkit-specs/roomkit-rfc.md`
