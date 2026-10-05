# SmartEmoji build plan

Total estimate: **about 3–4 hours** for phases 0–5. Phase 6 is optional.

---

## Phase 0: Setup (~20 min)

1. Create `.gitignore` with `*.session`, `.env`, `cache.*`, `__pycache__/`, `.venv/`
2. Create venv, add `requirements.txt`: `telethon`, `icalendar`, `recurring_ical_events`, `requests`, `ollama`, `python-dotenv`
3. Create `.env.example`:
   ```
   TG_API_ID=
   TG_API_HASH=
   ICAL_URL=
   OLLAMA_MODEL=qwen2.5:3b
   TIMEZONE=Asia/Singapore
   ```
4. Install Ollama: `winget install Ollama.Ollama`, then `ollama pull qwen2.5:3b`

**Done when:** `ollama run qwen2.5:3b "hi"` replies and `.env` is filled in.

---

## Phase 1: Telegram login + emoji search spike (~30 min)

The project depends on this step, so test it first.

- `scripts/login.py`: one-time interactive Telethon login, which creates `smartemoji.session`
- `scripts/search_test.py`: for ⚖️ 📚 🏋️ 💼 🍻, call `messages.SearchCustomEmojiRequest(emoticon=..., hash=0)` and print the number of IDs returned
- Manually set one result with `account.UpdateEmojiStatusRequest` and check it shows in the app

**Done when:** you know how many results a typical emoji returns. If coverage is poor, switch to the `SearchEmojiStickerSetsRequest(q=...)` approach described in Phase 3.

---

## Phase 2: Calendar reader (~30 min)

`smartemoji/calendar.py`

- `get_current_event(now) -> Event | None`
- Fetch the iCal URL and expand recurring events with `recurring_ical_events.at(...)`
- Skip all-day events (configurable)
- If events overlap, take the one that started most recently
- `Event` = `uid`, `title`, `description`, `start`, `end`

**Done when:** running it during a known event prints that event.

---

## Phase 3: Emoji picker (~45 min)

`smartemoji/llm.py`

- `pick_unicode(title, description) -> str`
- Ollama chat at `temperature=0`, JSON schema output `{"emoji": string}`
- Validate that the output is a single emoji (use the `emoji` package or a regex). If it isn't, fall back to 📅
- Timeout ~10 s; on failure, fall back to 📅

`smartemoji/telegram.py`

- `find_custom_emoji(client, unicode) -> list[int]` via `SearchCustomEmojiRequest`
- Alternative mode (config flag): the LLM returns 1–2 search words → `SearchEmojiStickerSetsRequest(q)` → random set → random emoji in that set
- `choose(ids, event_uid) -> int`: `random.Random(event_uid).choice(ids)`, so one event always gets the same emoji

**Done when:** the event "Contract Law tutorial" resolves to a custom emoji ID.

---

## Phase 4: Status setter + cache (~30 min)

`smartemoji/telegram.py`

- `set_status(client, emoji_id | None, until)`: custom emoji if an ID was found, otherwise `EmojiStatusEmpty` / Unicode fallback
- Always pass `until=event.end` so the status clears itself when the event finishes

`smartemoji/cache.py`

- Key: `f"{uid}:{start.isoformat()}"` (recurring events share a UID)
- Value: `{unicode, emoji_id, until}`
- Skip the API call when the status already set matches the cache (avoids flooding Telegram)

---

## Phase 5: Main loop + scheduling (~30 min)

`smartemoji/main.py`: one run per call:

```
event = get_current_event(now)
if not event: exit
if cached: exit
unicode = pick_unicode(event)
ids     = find_custom_emoji(unicode)
emoji   = choose(ids, event.uid) or None
set_status(emoji, until=event.end)
save cache
```

- Log each run to `smartemoji.log` (event title → unicode → emoji ID)
- `run.bat` that activates the venv and runs `python -m smartemoji.main`
- Windows Task Scheduler: every 5 min, "run whether user is logged on or not"

**Done when:** a test event created 5 minutes ahead changes your status on its own and clears at its end.

---

## Phase 6: Later / optional

- **Vision filter:** a local `qwen2.5vl` model downloads candidate emoji thumbnails and rejects off-topic/NSFW ones (~1–2 h)
- **Blocklist:** `blocklist.json` of emoji IDs or pack names never to use
- **Override:** an event description containing `emoji: 🎯` skips the LLM
- **Telegram control bot:** `/now`, `/reroll`, `/pause` commands
- **Cloud mode:** move to an always-on VM (needs ~4 GB RAM for the LLM, or swap to a hosted LLM)

---

## Risks

| Risk | Mitigation |
|---|---|
| Search returns few or no results for some emoji | Unicode fallback; sticker-set search mode |
| Random pick is ugly or NSFW | Seeded choice + blocklist; vision filter later |
| `.session` leak = account takeover | `.gitignore`, never sync to cloud drives |
| Telegram flood limits | Cache; only call the API when the status changes |
| PC off = no updates | Statuses expire on their own via `until`, so nothing gets stuck |

## Proposed layout

```
SmartEmoji/
├── smartemoji/
│   ├── __init__.py
│   ├── calendar.py
│   ├── llm.py
│   ├── telegram.py
│   ├── cache.py
│   └── main.py
├── scripts/
│   ├── login.py
│   └── search_test.py
├── .env.example
├── .gitignore
├── requirements.txt
├── run.bat
├── README.md
└── PLAN.md
```
