# smartemoji 🤖✨🫠

ok so basically 👉 ur google calendar 📅 tells a robot brain in the cloud ☁️🧠 what ur doing and then it goes digging thru the entire telegram community emoji dumpster 🗑️🔍 and slaps something random on ur emoji status 💅🎲 so everyone knows ur in a lecture 📚😴 or at the gym 🏋️💦 or crying about ur studies ⚖️😭

no keywords no fixed menu just vibes lmao 🌈🦄🍄

runs 24/7 on firebase 🔥 so ur pc can be off 😴💤 costs like 0 dollars 🆓 and updates within a minute ⚡

still in the planning stage btw 🚧👷 the actual plan is in [PLAN.md](PLAN.md) 📜🤓

## how it works 🛠️🐒

```
every hour ⏰ firebase looks at the next 24 h 🔮
  gemini 🤖 picks emojis for upcoming events early
  and stashes them in firestore 🗃️

then every 1 min ⏱️
google calendar 📅 (secret ical link 🤫)
        │
        ▼
  r u doing something rn 🤔 ──nah──► do nothing 😴 status expires by itself
        │ ye
        ▼
  already picked one for this event 🧐 ──ye──► use it 👍
        │ nah (last minute event 🏃)
        ▼
  gemini 🤖 picks one normal emoji like ⚖️
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
| runs on ☁️ | firebase cloud functions 🔥 two scheduled ones |
| brain 🧠 | gemini api free tier 🤖 |
| telegram 📨 | [telethon](https://docs.telethon.dev) logged in as u 🕵️ (premium needed for custom emoji status 💎) |
| alarm clock ⏰ | cloud scheduler every 1 min and every 60 min |
| memory 🐘 | firestore 🗃️ |
| secrets 🤫 | secret manager 🔒 |

## stuff u need 🛒🧺

- telegram premium 💎💸
- `api_id` and `api_hash` from https://my.telegram.org 🔑🗝️
- ur google calendar secret address in ical format 🤫📅
- a firebase project on the blaze plan 💳 (it stays free just set a 2 dollar budget alert 🚨)
- a gemini api key from google ai studio 🔑

heads up 👀 on the free gemini tier google might use ur event titles to train stuff 🕵️ so dont name events anything spicy 🌶️

## pls read this one 🚨🚨🚨

the telethon session string is literally a key to ur whole telegram account 🔓😱 if it leaks anyone can log in as u and post cringe 💀🤡 it only ever lives in secret manager 🔒 never in code never in the repo never in a `.session` file u push 🙈🙉🙊

ok bye 👋🐸🌮
