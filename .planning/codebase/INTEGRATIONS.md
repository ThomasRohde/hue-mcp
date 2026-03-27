# External Integrations

**Analysis Date:** 2026-03-27

## APIs & External Services

**Philips Hue Bridge (Local REST API):**
- The primary and only external integration
- SDK/Client: `phue` library (`Bridge` class from `phue`)
- Protocol: Local HTTP REST API on the Hue Bridge (not cloud-based)
- Auth: Bridge link-button pairing; username stored in `~/.hue-mcp/config.json`
- Discovery: Auto-discovery via UPnP/mDNS when `BRIDGE_IP` is `None` (default)
- API operations used:
  - `bridge.connect()` - Initial connection and auth
  - `bridge.get_light()` - Get all light states
  - `bridge.set_light(id, param, value)` - Control individual lights (on/off, brightness, color xy, color temp ct, alert, effect)
  - `bridge.get_group()` / `bridge.set_group()` - Group operations
  - `bridge.get_scene()` - Scene listing
  - `bridge.create_group()` - Group creation
  - `bridge.get_api()` - Full API state refresh

**Model Context Protocol (MCP):**
- Server role: This application IS an MCP server (not a client)
- SDK: `mcp` package, specifically `mcp.server.fastmcp.FastMCP`
- Transport modes:
  - **stdio** (default): `mcp.run(transport='stdio')` - for Claude Desktop integration
  - **SSE**: `mcp.sse_app()` served via uvicorn - for HTTP-based MCP clients
- Exposed MCP tools (20 total): `get_all_lights`, `get_light`, `get_all_groups`, `get_group`, `get_all_scenes`, `turn_on_light`, `turn_off_light`, `set_brightness`, `set_color_rgb`, `set_color_temperature`, `turn_on_group`, `turn_off_group`, `set_group_brightness`, `set_group_color_rgb`, `set_scene`, `find_light_by_name`, `create_group`, `quick_scene`, `refresh_lights`, `set_color_preset`, `set_group_color_preset`, `alert_light`, `set_light_effect`
- Exposed MCP prompts (3): `control_lights`, `create_mood`, `light_schedule`

## Data Storage

**Databases:**
- None - no database used

**File Storage:**
- Local filesystem only
- `~/.hue-mcp/config.json` - Stores bridge IP and username after initial pairing
- Created automatically by `hue_lifespan()` in `hue_server.py` (lines 88-124)

**Caching:**
- In-memory only: `HueContext.light_info` dict caches light state at startup
- Refreshable via `refresh_lights()` MCP tool
- No external cache service

## Authentication & Identity

**Hue Bridge Auth:**
- First-run: User must physically press the link button on the Hue bridge
- `phue.Bridge` handles the pairing handshake automatically
- Credentials (bridge IP + username) persisted to `~/.hue-mcp/config.json`
- No API keys, tokens, or environment variables required

**MCP Auth:**
- No authentication on the MCP server itself
- stdio mode: relies on process-level access control
- SSE mode: binds to `127.0.0.1` by default (localhost only)

## Monitoring & Observability

**Error Tracking:**
- None (no external error tracking service)

**Logs:**
- Python stdlib `logging` module
- Logger name: `"hue-mcp"`
- Default level: `INFO`
- Configurable via `--log-level` CLI argument
- Format: `%(asctime)s - %(name)s - %(levelname)s - %(message)s`
- All tool functions log errors via `logger.error()`

## CI/CD & Deployment

**Hosting:**
- Local execution only (designed for desktop use with Claude Desktop)
- SSE mode available for network-accessible deployment

**CI Pipeline:**
- None detected (no `.github/workflows/`, no CI config files)

## Environment Configuration

**Required env vars:**
- None - the server uses no environment variables

**CLI arguments (optional):**
- `--port` (default: 8080) - SSE server port
- `--host` (default: 127.0.0.1) - SSE server bind address
- `--log-level` (default: info) - Logging verbosity
- `--sse` (flag) - Use SSE transport instead of stdio

**Hardcoded config in `hue_server.py`:**
- `BRIDGE_IP = None` (line 58) - Set to an IP string to skip auto-discovery
- `CONFIG_DIR = ~/.hue-mcp` (line 61) - Bridge config storage location
- `CONFIG_FILE = ~/.hue-mcp/config.json` (line 62) - Bridge credentials file

## Webhooks & Callbacks

**Incoming:**
- None (MCP tools are request/response, not webhook-based)

**Outgoing:**
- None

## Network Requirements

- Local network access to Philips Hue Bridge (typically on same subnet)
- No internet/cloud access required at runtime
- Bridge discovery uses UPnP/SSDP on the local network when `BRIDGE_IP` is `None`

---

*Integration audit: 2026-03-27*
