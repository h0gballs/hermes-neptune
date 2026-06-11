# Cron Alert Recipe

## Working cron config

This distribution ships the job below in `cron/jobs.json` (paused — resume it after np-mcp is up):

- **Job name:** Neptune's Pride Alerts
- **Schedule:** `*/10 * * * *`
- **Skills loaded:** `neptunes-pride`
- **Prompt:** "Using mcp_np_mcp, call check_events for each game returned by list_games. If any events come back, summarize them in one or two sentences each, most urgent first. For incoming_attack events, call get_threats and include attacker strength vs defending ships. If there are no events, do nothing and send nothing."

## First-run behavior

The first `check_events` call produces a `baseline` event (type "baseline", tick N) and
writes a snapshot to np-mcp's `data/state.json`. No alert message should fire on the first
run — the prompt correctly handles this with "do nothing, send nothing" for no events.

## Debugging

- `systemctl status np-mcp` — confirm MCP server is running
- `tail -f ~/.hermes/profiles/neptune/logs/agent.log | grep cron_` — follow cron agent output
- `make smoke` from the np-mcp checkout — peek-mode test that won't consume events
- `hermes -p neptune cron list` — check job status
- `hermes -p neptune cron run "Neptune's Pride Alerts"` — force a manual run
