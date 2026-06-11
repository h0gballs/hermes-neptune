# Neptune's Pride MCP Pitfalls

## MCP tools not available (cron or session)

**Symptom**: `mcp_np_mcp_*` tools don't appear in `hermes tools list`, and the agent
falls back to calling the NP API directly via `execute_code`.

**Check order**:
1. `systemctl status np-mcp` — is the server running?
2. Profile config has `mcp_servers.np-mcp` — NOT just root `~/.hermes/config.yaml`.
   The gateway reads `~/.hermes/profiles/neptune/config.yaml` (the active profile).
3. `hermes gateway restart` — MCP servers are discovered at gateway startup only.
4. `hermes mcp list` — verify the server is registered after restart.

## check_events consumption semantics

The first run of `check_events` always returns a `baseline` event (informational, not
a real alert). It writes a snapshot to `data/state.json`. Subsequent runs diff against
that snapshot.

- **Reading consumes events** — the snapshot is saved. Use `peek=true` for manual poking
  (e.g., `make smoke` does this).
- The baseline event has type `baseline` and message like "Now watching 'Game Name':
  baseline recorded at tick N".
- Cron prompt should account for this: on a baseline-only response, send nothing.

## YAML nesting in config.yaml

Agent config keys (`gateway_auto_continue_freshness`, `image_input_mode`,
`disabled_toolsets`) belong under `agent:`, not under `mcp_servers:`. A malformed
YAML block like:

```yaml
mcp_servers:
  np-mcp:
    url: "http://127.0.0.1:8721/mcp"
  gateway_auto_continue_freshness: 3600    # WRONG — this is an agent key
```

...will cause agent-level settings to be silently ignored.
