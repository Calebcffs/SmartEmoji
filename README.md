# smartemoji 🤖✨🫠

ok so basically 👉 ur google calendar 📅 tells a tiny lil robot brain 🧠🤏 what ur doing and then it goes digging thru the entire telegram community emoji dumpster 🗑️🔍 and slaps something random on ur emoji status 💅🎲 so everyone knows ur in a lecture 📚😴 or at the gym 🏋️💦 or crying about contract law ⚖️😭

no keywords no fixed menu just vibes 🌈🦄🍄

still in the planning stage btw 🚧👷 the actual plan is in [PLAN.md](PLAN.md) 📜🤓

## how it works 🛠️🐒

```
google calendar 📅 (secret ical link 🤫)
        │  every 5 min ⏰
        ▼
  r u doing something rn 🤔 ──nah──► do nothing 😴 status expires by itself
        │ ye
        ▼
  already picked one for this event 🧐 ──ye──► keep it 👍
        │ nah
        ▼
  local llm 🦙 (ollama) picks one normal emoji like ⚖️
        │
        ▼
  ask telegram for every community emoji tagged ⚖️ 📡
        │  → a big pile of custom emoji ids 🗻
        ▼
  pick one at random 🎰 (same event always gets the same one so it doesnt flicker 🪩)
        │  (if nothing found just use the plain emoji 🤷)
        ▼
  set it as ur status until the event ends ⏳ then it vanishes 👻
```

## what its built with 🧱🔩

| thing 🧩 | what 🍕 |
|---|---|
| language 🐍 | python 3.11 or newer |
| calendar 📅 | google calendar secret ical link with `icalendar` and `recurring_ical_events` so no oauth pain 🙅‍♂️🔐 |
| brain 🧠 | [ollama](https://ollama.com) running `qwen2.5:3b` on ur own pc 🖥️🔥 |
| telegram 📨 | [telethon](https://docs.telethon.dev) logged in as u 🕵️ (premium needed for custom emoji status 💎) |
| alarm clock ⏰ | windows task scheduler every 5 min 🪟 |
| memory 🐘 | a lil json or sqlite file 🗃️ |

## stuff u need 🛒🧺

- telegram premium 💎💸
- `api_id` and `api_hash` from https://my.telegram.org 🔑🗝️
- ur google calendar secret address in ical format 🤫📅
- ollama installed and like 4 gb of free ram 🐏🐏🐏🐏

## pls read this one 🚨🚨🚨

the telethon `.session` file is literally a key to ur whole telegram account 🔓😱 if u commit it anyone can log in as u and post cringe 💀🤡 never ever push it and same goes for `.env` 🙈🙉🙊

ok bye 👋🐸🌮
