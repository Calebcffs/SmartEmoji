# SmartEmoji build plan

**Target:** Firebase Cloud Functions + Gemini API. Runs 24/7, about $0/month, and the status updates within 1 minute of an event starting.

Total estimate: **about 4–5 hours** for phases 0–5. Phase 6 is optional.

---

## Architecture

Two scheduled functions share state in Firestore.

```
┌───────────────────────── plan_ahead (every 60 min) ─────────────────────────┐
│ fetch iCal → events in next 24 h → for each unplanned event:                │
│   Gemini → Unicode emoji → SearchCustomEmojiRequest → seeded random pick    │
│   → write events/{key} to Firestore                                         │
└─────────────────────────────────────────────────────────────────────────────┘

┌───────────────────────── apply_status (every 1 min) ────────────────────────┐
│ fetch iCal → current event?                                                 │
│   none          → do nothing (previous status expires on its own)           │
│   planned       → read emoji from Firestore                                 │
│   not planned   → plan it inline (last-minute event, +1–2 s)                │
│ if emoji ≠ state/current → UpdateEmojiStatusRequest(id, until=event.end)    │
│                           → write state/current                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why two functions:** the LLM call happens ahead of time, so the every-minute run only does a quick lookup. Telegram is called only when the status actually changes.

---

## Cost check

| Service | Usage | Free allowance | Expected |
|---|---|---|---|
| Cloud Functions | ~44k runs/month | 2M runs/month | $0 |
| Cloud Scheduler | 2 jobs | 3 jobs per billing account | $0 |
| Firestore | ~3k reads/day | 50k reads/day | $0 |
| Secret Manager | 5 secrets, accessed on cold starts | 6 versions + 10k accesses | ~$0–0.15 |
| Gemini API | ~10–30 calls/day | free tier | $0 |
| Artifact Registry (deploy images) | ~few hundred MB | 0.5 GB | ~$0–0.10 |

Requires the **Blaze plan** (card on file). Set a **$2 budget alert**.

⚠️ **Privacy:** on the Gemini free tier, Google may use prompts to improve its products. Event titles and descriptions go to Gemini. If that matters, either send titles only (no descriptions), or switch to paid Gemini or Claude Haiku, which cost cents at this volume.

---

## Phase 0: Firebase setup (~30 min)

1. Create a Firebase project and upgrade to Blaze. Set a $2 budget alert in Google Cloud Billing
2. Enable Firestore (Native mode, region `asia-southeast1`)
3. `npm i -g firebase-tools`, then `firebase login`, then `firebase init functions` → **Python**
4. Get a Gemini API key from Google AI Studio
5. `.gitignore`: `*.session`, `.env`, `.secret.local`, `venv/`, `__pycache__/`, `functions/venv/`

**Done when:** `firebase deploy --only functions` deploys the hello-world function.

---

## Phase 1: Telegram login + emoji search spike (~30 min, local)

The project depends on this step, so test it first.

- `scripts/make_session.py`: logs in to Telethon interactively on your PC and prints a **StringSession**. Copy it straight into Secret Manager in Phase 4. Never save it to a file in the repo
- `scripts/search_test.py`: for ⚖️ 📚 🏋️ 💼 🍻, call `messages.SearchCustomEmojiRequest(emoticon=..., hash=0)` and print the number of IDs returned
- Manually set one result with `account.UpdateEmojiStatusRequest` and check it shows in the app

**Done when:** you know how many results a typical emoji returns. If coverage is poor, switch to the `SearchEmojiStickerSetsRequest(q=...)` approach described in Phase 2.

---

## Phase 2: Core library (~1.5 h, local)

Plain Python with no Firebase code, so it can be tested on your PC.

`functions/smartemoji/calendar.py`
- `get_events(start, end) -> list[Event]`: fetch iCal, expand recurring events with `recurring_ical_events`
- `get_current_event(now) -> Event | None`: skip all-day events (configurable); if events overlap, take the one that started most recently
- `Event` = `uid`, `title`, `description`, `start`, `end`, `key` (`f"{uid}:{start.isoformat()}"`, since recurring events share a UID)

`functions/smartemoji/llm.py`
- `pick_unicode(title, description) -> str`
- `google-genai` SDK, current Flash-Lite model (model name in config), `temperature=0`
- Structured output: `response_mime_type="application/json"`, schema `{"emoji": string}`
- Validate that the output is a single emoji. If it isn't, or the call errors or takes over 10 s, fall back to 📅

`functions/smartemoji/telegram.py`
- `connect()`: Telethon client from the StringSession (`async`, wrapped with `asyncio.run` in the function)
- `find_custom_emoji(client, unicode) -> list[int]` via `SearchCustomEmojiRequest`
- Alternative mode (config flag): Gemini returns 1–2 search words → `SearchEmojiStickerSetsRequest(q)` → random set → random emoji in that set
- `choose(ids, event_key) -> int | None`: `random.Random(event_key).choice(ids)`, so one event always gets the same emoji
- `set_status(client, emoji_id, until)`: always pass `until=event.end`

`scripts/run_local.py`: runs the full pipeline once against your real calendar

**Done when:** `run_local.py` during a real event sets your Telegram status.

---

## Phase 3: Firestore state + two functions (~45 min)

**Firestore schema**

```
events/{key}
  title, start, end, unicode, emoji_id (nullable), planned_at

state/current
  key, emoji_id, until, applied_at
```

**`functions/main.py`**

```python
from firebase_functions import scheduler_fn, options
from firebase_functions.params import SecretParam

SECRETS = [SecretParam(n) for n in
           ["TG_API_ID", "TG_API_HASH", "TG_SESSION", "ICAL_URL", "GEMINI_API_KEY"]]

@scheduler_fn.on_schedule(schedule="every 60 minutes", timezone="Asia/Singapore",
                          region="asia-southeast1", secrets=SECRETS, timeout_sec=300)
def plan_ahead(event): ...

@scheduler_fn.on_schedule(schedule="every 1 minutes", timezone="Asia/Singapore",
                          region="asia-southeast1", secrets=SECRETS, timeout_sec=60,
                          memory=options.MemoryOption.MB_512)
def apply_status(event): ...
```

- `plan_ahead`: skip events already in `events/`, and delete `events/` docs older than 7 days
- `apply_status`: compare against `state/current` before calling Telegram, so it makes no API call if nothing changed
- Log each decision: event title → unicode → emoji ID → applied or skipped

**Done when:** both functions run cleanly against real Firestore from your PC (call the handlers directly).

---

## Phase 4: Deploy (~45 min)

1. Set the secrets: `firebase functions:secrets:set TG_SESSION` (repeat for each of the 5)
2. `firebase deploy --only functions`
3. Check that the Cloud Scheduler console shows 2 jobs with the correct timezone
4. Approve the "new login" notice Telegram sends (from a Google data centre IP)

---

## Phase 5: Verify end to end (~15 min)

- [ ] Create a calendar event starting in 3 minutes → the status changes within 1 minute of its start
- [ ] The status clears at the event's end
- [ ] Create an event starting in 2 minutes (not yet planned) → the inline planning path works
- [ ] Logs show `skipped` on unchanged minutes (no spare Telegram calls)
- [ ] After 24 h, check billing: still $0.00

---

## Phase 6: Later / optional

- **Blocklist:** a Firestore `blocklist` collection of emoji IDs or pack names never to use
- **Override:** an event description containing `emoji: 🎯` skips Gemini
- **Telegram control bot:** `/now`, `/reroll`, `/pause` commands
- **Vision filter:** Gemini (it can read images) checks candidate emoji thumbnails and rejects off-topic or NSFW ones
- **Smarter timing:** replace the 1-minute poll with Cloud Tasks scheduled at each event's exact start time

---

## Risks

| Risk | Mitigation |
|---|---|
| StringSession leak = account takeover | Only ever in Secret Manager; never in code, logs or the repo |
| Search returns few or no results for some emoji | Unicode fallback; sticker-set search mode |
| Random pick is ugly or NSFW | Seeded choice + blocklist; vision filter later |
| Telegram flood limits | Only call the API when the status changes |
| Telegram flags the cloud login | Approve once; keep one long-lived session |
| Gemini free tier limits change | Model name in config; Claude Haiku as a drop-in swap |
| Unexpected bill | $2 budget alert; Phase 5 billing check |

---

## Proposed layout

```
SmartEmoji/
├── functions/
│   ├── main.py              # plan_ahead + apply_status
│   ├── requirements.txt     # firebase-functions, firebase-admin, telethon,
│   │                        # icalendar, recurring_ical_events, google-genai, requests
│   └── smartemoji/
│       ├── __init__.py
│       ├── calendar.py
│       ├── llm.py
│       ├── telegram.py
│       └── store.py         # Firestore reads/writes
├── scripts/
│   ├── make_session.py
│   ├── search_test.py
│   └── run_local.py
├── firebase.json
├── .firebaserc
├── .gitignore
├── README.md
└── PLAN.md
```
