# np-mcp Server Install & Maintenance Recipes

Agent-driven recipes for installing, updating, and reconfiguring the np-mcp server
(https://github.com/h0gballs/np-mcp). Paths assume the checkout lives at `~/git/np-mcp`;
if it's elsewhere, find it first (`systemctl cat np-mcp` shows the WorkingDirectory).

**Heads-up for the agent:** service operations (`make service-install`, `make service-restart`)
use sudo. Tell the user before running them so they know what they're approving.

## Fresh install

Use when the user asks to "set up np-mcp" and `systemctl status np-mcp` shows no unit.

1. Clone: `git clone https://github.com/h0gballs/np-mcp ~/git/np-mcp`
2. `cd ~/git/np-mcp && cp config.example.yaml config.yaml && cp .env.example .env`
3. Ask the user for each game's **game URL** (or game number) and **API code**
   (in-game menu → API Key), then edit `config.yaml`:

   ```yaml
   games:
     - game_number: "1234567890"   # from np.ironhelmet.com/game/<game_number>
       code: "AbCdEf"              # read-only API key
       label: "My Game"            # optional friendly name
   ```

4. If they want in-game mail reporting, set `NP_EMAIL` / `NP_PASSWORD` in `.env`
   (optional — see SKILL.md; don't push for it, it's their real NP login).
5. `make test` — suite must pass before installing the service.
6. `make service-install` (sudo) — installs, enables, and starts the systemd unit.
7. Verify: `make service-status`, then `make smoke` (peek mode, consumes no events).
8. If the Hermes gateway was already running before the MCP server existed:
   `hermes gateway restart`, then confirm tools with `hermes mcp list`.

## Add a game

Use when the user starts a new game ("add my new game ...").

1. Get the game number (from the URL) and the API code from the user.
   The API key must be generated in that game's menu — each game has its own.
2. Append the entry to the `games:` list in `~/git/np-mcp/config.yaml` (same shape as above).
   Give it a `label` so the user can refer to it by name.
3. `make service-restart` (sudo) — config is read at startup only.
4. Verify: call `mcp_np_mcp_list_games` and confirm the new game appears and is live.
   First `check_events` for the new game returns a `baseline` event — that's normal.

## Remove a game

Usually after a game ends. Delete its entry from `games:` in `config.yaml`,
then `make service-restart`. Verify with `mcp_np_mcp_list_games`.

## Update the server

Use when the user asks to "update np-mcp".

1. `cd ~/git/np-mcp && git pull`
2. `make test` — if tests fail, report the failure and **do not restart**; the
   running service keeps working on the old code.
3. `make service-restart` (sudo)
4. `make smoke` to confirm it serves.

## Health checks

- `make service-status` / `make service-logs` — systemd state and logs
- `make smoke` — one peek-mode MCP call against the running server
- `mcp_np_mcp_list_games` — end-to-end check through the Hermes gateway
- Tools missing in Hermes but server healthy → see [pitfalls.md](pitfalls.md)
  (gateway discovers MCP servers at startup only)

## Tuning

In `config.yaml` (restart after editing):
- `low_cash_threshold` — `low_cash` event fires below this after production (default 50)
- `turn_warning_minutes` — `turn_deadline_soon` lead time, turn-based games only (default 720)

In `.env` (restart after editing):
- `NP_MCP_HOST` / `NP_MCP_PORT` — where the server listens (default 127.0.0.1:8721).
  If changed, also update `mcp_servers.np-mcp.url` in the profile's `config.yaml`
  and restart the Hermes gateway.
- `NP_EMAIL` / `NP_PASSWORD` — optional, enables `get_messages` + `new_message` events.
