# Neptune — a Neptune's Pride co-pilot for Hermes Agent

**Neptune** is a [Hermes Agent](https://hermes-agent.nousresearch.com) profile distribution that watches your [Neptune's Pride](https://np.ironhelmet.com) games and messages you when something needs attention: incoming attacks, stars lost or captured, research completions, production cycles, low cash, diplomacy changes, new in-game messages, and turn deadlines.

It talks to the game through [np-mcp](https://github.com/h0gballs/np-mcp), a read-only MCP server for the Neptune's Pride API, and ships with:

- a `neptunes-pride` skill that teaches the agent every np-mcp tool plus game-specific lore (tech names, war states, threat math)
- a **cron job that checks your games every 10 minutes** and alerts you only when something actually changed
- a tactical SOUL — terse fleet-officer reports, most urgent first, silence when all is quiet

The API key Neptune uses is read-only: it can observe and advise, but only you can issue orders in the game client.

---

## Prerequisites

- [Hermes Agent](https://hermes-agent.nousresearch.com) ≥ 0.16.0 installed (`hermes --version`)
- `git`, Python 3.11+
- A Neptune's Pride game you're playing, and its API key (in-game menu → **API Key**)
- An LLM API key — the profile defaults to [OpenRouter](https://openrouter.ai), but any provider Hermes supports works

## Step 1 — Set up the np-mcp server

Neptune gets all game data from a locally running np-mcp server.

```bash
git clone https://github.com/h0gballs/np-mcp ~/git/np-mcp
cd ~/git/np-mcp

cp config.example.yaml config.yaml   # add each game's game_number + API code
cp .env.example .env                 # host/port; NP creds only for messages

make test     # run the test suite
make run      # serve http://127.0.0.1:8721/mcp (foreground, good for a first test)
```

For each game in `config.yaml`:
- **`game_number`** is in the game URL: `https://np.ironhelmet.com/game/<game_number>`
- **`code`** is the read-only API key from the game menu → API Key

### Optional: message reporting (NP_EMAIL / NP_PASSWORD)

If you want Neptune to read your **in-game mail and events feed** (the `get_messages` tool and `new_message` alerts), set `NP_EMAIL` and `NP_PASSWORD` in `~/git/np-mcp/.env` — your normal Neptune's Pride login. These use the *unofficial* login API because the read-only key can't see your inbox. **Everything else works without them**; skip this if you'd rather not store your NP credentials.

### Run it for real (systemd)

```bash
make service-install     # install + enable + start (sudo)
make service-status
make smoke               # call the running server once via MCP (peek mode, consumes nothing)
```

## Step 2 — Install the Neptune profile

```bash
hermes profile install github.com/h0gballs/hermes-neptune --alias
```

The installer shows the manifest, tells you which env vars you still need, and creates the profile at `~/.hermes/profiles/neptune/`. With `--alias` you can use `neptune chat` instead of `hermes -p neptune chat`.

## Step 3 — Fill in your keys

```bash
cp ~/.hermes/profiles/neptune/.env.EXAMPLE ~/.hermes/profiles/neptune/.env
# edit .env:
```

| Variable | Required? | What for |
|---|---|---|
| `OPENROUTER_API_KEY` | yes (by default config) | LLM access. The profile defaults to OpenRouter; run `hermes -p neptune setup` or edit `config.yaml` to use a different provider/model. |
| `TELEGRAM_BOT_TOKEN` | optional | Telegram gateway (Step 6) |
| `TELEGRAM_ALLOWED_USERS` | optional | Who may talk to your bot |
| `TELEGRAM_HOME_CHANNEL` | optional | Where unprompted cron alerts go |
| `BRAVE_SEARCH_API_KEY` | optional | Web search toolset |

(`NP_EMAIL`/`NP_PASSWORD` do **not** go here — they live in np-mcp's own `.env`, see Step 1.)

## Step 4 — Verify the MCP wiring

The profile's `config.yaml` already points at the np-mcp server:

```yaml
mcp_servers:
  np-mcp:
    url: http://127.0.0.1:8721/mcp
```

If your np-mcp runs elsewhere (another port/host), edit that URL. Then check it's alive:

```bash
hermes -p neptune mcp list        # np-mcp should be listed
neptune chat                      # ask: "What's my NP game status?"
```

> MCP servers are discovered at gateway startup only — if you change the URL later, run `hermes -p neptune gateway restart`.

## Step 5 — Enable the 10-minute game watch (cron)

The distribution ships a cron job, **Neptune's Pride Alerts**, that runs every 10 minutes: it calls `check_events` on each configured game and messages you only if something changed. It is installed **paused** (Hermes never auto-runs cron jobs from a distribution). Enable it once np-mcp is up:

```bash
hermes -p neptune cron list --all     # paused jobs are hidden without --all
hermes -p neptune cron resume "Neptune's Pride Alerts"
```

Cron jobs are executed by the gateway, so the gateway must be running (Step 6, or `hermes -p neptune gateway run` for foreground). Two things to know:

- **First run is a baseline.** The first `check_events` call records a snapshot and fires no alerts; real alerts start on the next tick.
- **No news is no message.** The job is prompted to stay silent when nothing happened — an alert always means something changed.

If you'd rather hand-roll the job, the equivalent is:

```bash
hermes -p neptune cron create \
  --name "Neptune's Pride Alerts" \
  --schedule "*/10 * * * *" \
  --skill neptunes-pride \
  --prompt "Using mcp_np_mcp, call check_events for each game returned by list_games. If any events come back, summarize them most urgent first; for incoming_attack events call get_threats and include attacker strength vs defenders. If there are no events, do nothing and send nothing."
```

## Step 6 — Hook it up to Telegram

This is where the 10-minute alerts land, and how you talk to Neptune from your phone.

1. **Create a bot:** message [@BotFather](https://t.me/BotFather) on Telegram, send `/newbot`, follow the prompts, and copy the bot token.
2. **Find your user ID:** message [@userinfobot](https://t.me/userinfobot) (or any ID bot) — it replies with your numeric ID.
3. **Configure the profile** in `~/.hermes/profiles/neptune/.env`:

   ```bash
   TELEGRAM_BOT_TOKEN=123456789:AA...your-token...
   TELEGRAM_ALLOWED_USERS=<your numeric user id>
   TELEGRAM_HOME_CHANNEL=<your numeric user id>
   ```

   `TELEGRAM_ALLOWED_USERS` keeps strangers out of your bot; `TELEGRAM_HOME_CHANNEL` is where unprompted messages (the cron alerts) are delivered — for DMs it's the same as your user ID.

4. **Start the gateway:**

   ```bash
   hermes -p neptune gateway install   # systemd/launchd background service (recommended)
   # or, in a terminal / WSL / Docker / Termux:
   hermes -p neptune gateway run
   ```

5. **Open Telegram and message your bot.** Try "any threats?" — and from now on, the cron job pings you within 10 minutes of anything happening in your games.

You can also walk through this interactively with `hermes -p neptune gateway setup`.

### Other gateways: Discord, Email, and more

Telegram is just one front-end. The same gateway can serve Neptune over **Discord**, **Email**, **Slack**, **WhatsApp**, **Signal**, **Matrix**, **SMS**, and more — run `hermes -p neptune gateway setup` to configure any of them, or see the [Hermes gateway docs](https://hermes-agent.nousresearch.com/docs/) for per-platform setup (bot tokens, allowed users, home channels work analogously to Telegram). Several platforms can run simultaneously on one profile.

---

## Using Neptune

Ask it things like:

- *"What's my game status?"* — cash, rank, totals, next production, research ETA
- *"Any incoming attacks?"* — attacker, ship counts, ETA, and whether your defenders hold
- *"How's my empire?"* — per-star breakdown, weakest stars first
- *"Who's winning?"* — leaderboard with diplomatic relations
- *"What am I researching, and what should I queue next?"*
- *"Read my diplomacy inbox"* — needs NP_EMAIL/NP_PASSWORD in np-mcp's `.env`

## Troubleshooting

| Symptom | Fix |
|---|---|
| `mcp_np_mcp_*` tools missing | `systemctl status np-mcp`; confirm `mcp_servers.np-mcp` is in `~/.hermes/profiles/neptune/config.yaml`; `hermes -p neptune gateway restart`; `hermes -p neptune mcp list` |
| Cron job missing from `cron list` / never fires | It ships paused (hidden without `--all`) — `hermes -p neptune cron resume "Neptune's Pride Alerts"`; gateway must be running |
| First cron run alerted nothing | Expected — first run records a baseline snapshot only |
| No Telegram messages from cron | Check `TELEGRAM_HOME_CHANNEL` is set and the gateway log: `~/.hermes/profiles/neptune/logs/` |
| `get_messages` errors | Set `NP_EMAIL`/`NP_PASSWORD` in np-mcp's `.env` and `sudo systemctl restart np-mcp` |
| Changed np-mcp `config.yaml` (new game etc.) | `sudo systemctl restart np-mcp` |

## Updating

```bash
hermes profile update neptune
```

Pulls the latest version of this distribution. Your `.env`, memories, sessions, and `config.yaml` tweaks are preserved; the SOUL, skill, and cron definitions are refreshed.

## What's in the box

```
neptune/
├── distribution.yaml          # manifest: name, version, required env vars
├── SOUL.md                    # the fleet-officer persona
├── config.yaml                # model defaults + np-mcp MCP wiring
├── cron/jobs.json             # the 10-minute alert job (ships paused)
└── skills/gaming/neptunes-pride/
    ├── SKILL.md               # tool guide + game lore
    └── references/            # pitfalls, cron recipe, API notes, HTTP fallback
```

Good hunting, and may your weakest star never be the one they jump to. 🔭
