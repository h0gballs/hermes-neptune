---
name: neptunes-pride
description: Monitor and analyze Neptune's Pride game state via the np-mcp MCP server.
---

# Neptune's Pride

Monitor and analyze Neptune's Pride games through the `np-mcp` MCP server (https://github.com/h0gballs/np-mcp).

## Triggers
- User asks for game status, empire overview, leaderboard, threats, or research.
- User wants periodic alerts about game events (attacks, stars lost/captured, diplo changes, messages).
- User asks "what's happening in my NP game" or similar.

## Primary Interface: MCP Tools

All NP data comes through the `np-mcp` MCP server (typically a systemd service `np-mcp`, listening at `http://127.0.0.1:8721/mcp`). Tools are registered as `mcp_np_mcp_*`.

### Quick Status Check
Call `mcp_np_mcp_get_game_status` (empty `game` param defaults to first configured game). Returns cash, totals (stars/econ/ind/sci/ships/fleets), rank, incoming attack count, research ETA, and production/turn countdown.

### Deeper Inspection
- `mcp_np_mcp_get_my_empire` — per-star breakdown, weakest 5 stars first
- `mcp_np_mcp_get_threats` — enemy fleets with waypoints on your stars: attacker, ships, target, ETA, defending ships, relation
- `mcp_np_mcp_get_research_status` — current research progress/ETA, queued tech, all tech levels
- `mcp_np_mcp_get_players` — full leaderboard with relation to you, conceded/AI flags
- `mcp_np_mcp_get_messages` — in-game mail (group="diplomacy") or events feed (group="events"); needs NP_EMAIL/NP_PASSWORD in np-mcp's .env

### Alerting / Periodic Checks
`mcp_np_mcp_check_events` is the alerting tool:
- Returns only what changed since the last call (diffs against np-mcp's `data/state.json`)
- **Reading consumes events** — snapshot is saved on read. Use `peek=true` for manual poking.
- Event types: `incoming_attack`, `star_lost`, `star_captured`, `research_complete`, `production_occurred`, `low_cash`, `war_declared`, `relation_changed`, `player_conceded`, `player_eliminated`, `new_message`, `turn_deadline_soon`, `baseline`
- Every event has a stable `id` for dedup

- **Pitfalls & Workarounds:** See [references/pitfalls.md](references/pitfalls.md) for MCP discovery, check_events consumption, and config YAML traps. See [references/mcp-direct-invocation.md](references/mcp-direct-invocation.md) for the raw HTTP fallback path. See [references/cron-alert-recipe.md](references/cron-alert-recipe.md) for the working cron alert setup pattern.

## Setup & Troubleshooting

### MCP Tools Not Available
If `mcp_np_mcp_*` tools don't appear in the tool list:
1. Confirm `mcp_servers.np-mcp` is in the **active profile's** `config.yaml` (e.g., `~/.hermes/profiles/neptune/config.yaml`), not just root `~/.hermes/config.yaml`. The gateway reads the profile config.
2. Run `hermes gateway restart` — MCP servers are discovered at gateway startup only (no hot-reload).
3. Verify with `hermes mcp list` and `systemctl status np-mcp`.

### check_events First Run
The first `check_events` call returns a `baseline` event (informational). It writes a snapshot. Subsequent calls diff against that snapshot. No real alerts fire until the second tick. Use `peek=true` for manual poking without consuming events.

## Server Details
(Paths below assume np-mcp is cloned at `~/git/np-mcp`; adjust to the actual checkout.)
- **Project:** `~/git/np-mcp`
- **Service:** `systemctl status np-mcp`
- **Config:** `~/git/np-mcp/config.yaml` (games, thresholds)
- **State:** `~/git/np-mcp/data/state.json` (snapshots for `check_events` diffing)
- **NP API creds:** `~/git/np-mcp/.env` (NP_EMAIL, NP_PASSWORD — optional, only for messages)
- **Restart after config changes:** `sudo systemctl restart np-mcp`
- **Smoke test:** `cd ~/git/np-mcp && make smoke`

### Config flags in np-mcp's `config.yaml`:
- `low_cash_threshold`: fires `low_cash` event when cash drops below this after production (default 50)
- `turn_warning_minutes`: fires `turn_deadline_soon` when turn deadline is within this many minutes and you're not ready (turn-based games only, default 720)

## Tech Reference
- Range: Tech kind 3 (UI label "Range"). Determines Jump Range and Fleet Speed. Always call it "Range", not "Propulsion".
- Weapons: Level 10+ is common in late-game; Level 11 ETA can be 50+ hours.
- Experimentation: Adds raw points/exp to a random tech each production cycle. Check `mcp_np_mcp_get_messages(group="events")` to see which tech "rolled" on experience.
- War labels: 0=war, 1/2=peace_pending, 3=peace (default)
- Threat detection: traces fleet waypoint queues; only detects fleets inside your scan range

## Common Requests

**Status check:** "What's my NP game status?" → call `get_game_status`

**Threat check:** "Any incoming attacks?" → call `get_threats`; if threats exist, highlight attacker, ship count, ETA, and whether your defending ships can hold

**Production timing:** "When's the next production?" → `get_game_status` returns `next_production_minutes` (RT) or `turn_deadline_minutes` (TB)

**Research check:** "What am I researching?" → call `get_research_status`

**Empire health:** "How's my empire?" → call `get_my_empire`, highlight weakest stars that need reinforcement
