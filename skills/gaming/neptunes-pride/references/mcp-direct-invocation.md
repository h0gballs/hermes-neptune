# NP MCP Tool Discovery

## Primary: MCP server config

If `mcp_np_mcp_*` tools aren't available, check two things:

1. **Profile config** — `mcp_servers` must be in the active profile's `config.yaml` (e.g.,
   `~/.hermes/profiles/neptune/config.yaml`), not just `~/.hermes/config.yaml`. The
   gateway reads the profile config on startup.

   ```yaml
   mcp_servers:
     np-mcp:
       url: "http://127.0.0.1:8721/mcp"
   ```

2. **Gateway restart** — MCP server changes require a gateway restart (no hot-reload).
   Run `hermes gateway restart` or `/reload-mcp` in an active session.

3. **Verify** — check `hermes mcp list` to confirm the server is registered, and
   `systemctl status np-mcp` to confirm the server process is running.

## Fallback: Direct HTTP invocation (last resort)

If the MCP client still can't connect, the server speaks plain JSON-RPC 2.0 over
streamable HTTP at `http://127.0.0.1:8721/mcp`. Use `execute_code` with:

```python
import requests, json, re

resp = requests.post(
    "http://127.0.0.1:8721/mcp",
    json={"jsonrpc":"2.0","method":"tools/call","params":{"name":"list_games","arguments":{}},"id":1},
    headers={"Content-Type": "application/json", "Accept": "application/json, text/event-stream"},
    timeout=15,
)
# Extract JSON from SSE data: lines
for line in resp.text.splitlines():
    if line.startswith("data: "):
        payload = json.loads(line[6:])
        # result is in payload["result"]["content"][0]["text"]
```

**Only use this fallback** if `mcp_np_mcp_*` tools aren't appearing after a gateway
restart with correct profile config. The raw HTTP path is fragile and error-prone.
