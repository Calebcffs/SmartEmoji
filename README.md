# SmartEmoji

Sync your Google Calendar to your Telegram emoji status, using a local LLM to pick the emoji and Telegram's community custom-emoji library to find a matching one.

> Status: planning. See [PLAN.md](PLAN.md) for the build plan.

## How it works

```
Google Calendar (secret iCal URL)
        │  every 5 min
        ▼
  Current event? ──no──► clear status (or let it expire)
        │ yes
        ▼
  Cache hit for this event? ──yes──► reuse chosen emoji
        │ no
        ▼
  Local LLM (Ollama) → one standard Unicode emoji, e.g. ⚖️
        │
        ▼
  Telegram: messages.SearchCustomEmojiRequest("⚖️")
        │  → list of community custom-emoji IDs
        ▼
  Pick one at random (seeded by event ID)
        │  (fallback: plain Unicode emoji)
        ▼
  account.UpdateEmojiStatusRequest(emoji_id, until=event_end)
```

## Stack

| Piece | Choice |
|---|---|
| Language | Python 3.11+ |
| Calendar | Google Calendar secret iCal URL (`icalendar` + `recurring_ical_events`), no OAuth |
| LLM | [Ollama](https://ollama.com) running `qwen2.5:3b` locally, JSON-constrained output |
| Telegram | [Telethon](https://docs.telethon.dev) userbot on your own account (Premium required for custom emoji status) |
| Scheduler | Windows Task Scheduler, every 5 min |
| Cache | Local JSON or SQLite file |

## Requirements

- Telegram Premium
- `api_id` / `api_hash` from https://my.telegram.org
- Google Calendar "Secret address in iCal format"
- Ollama installed with ~4 GB free RAM

## Security

The Telethon `.session` file gives **full access to your Telegram account**. Never commit it. The same goes for `.env` (API keys, iCal URL).
