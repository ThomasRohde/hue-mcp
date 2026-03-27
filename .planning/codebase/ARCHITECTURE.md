# Architecture

**Analysis Date:** 2026-03-27

## Pattern Overview

**Overall:** Single-module MCP server (monolith)

The entire application is a single Python file (`hue_server.py`) that implements a Model Context Protocol (MCP) server for controlling Philips Hue smart lights. It uses the FastMCP framework to expose tools that an AI assistant (e.g., Claude) can invoke to interact with Hue hardware.

**Key Characteristics:**
- Single-file architecture with no internal module boundaries
- Declarative tool registration via `@mcp.tool()` decorators
- Lifespan-managed bridge connection with cached light state
- Dual transport support: stdio (default) and SSE over HTTP
- Synchronous tool functions (not async) despite async lifespan

## Layers

**Transport Layer:**
- Purpose: Handles communication between the AI client and the MCP server
- Location: Bottom of `hue_server.py` (lines 1163-1200, `__main__` block)
- Contains: Argument parsing, transport selection (stdio vs SSE/uvicorn)
- Depends on: `mcp` SDK, `uvicorn` (for SSE mode)
- Used by: External MCP clients (Claude Desktop, MCP Inspector)

**Server Framework Layer:**
- Purpose: MCP protocol handling, tool registration, request routing
- Location: `hue_server.py` line 142 (`mcp = FastMCP(...)`)
- Contains: The `FastMCP` instance and all `@mcp.tool()` / `@mcp.prompt()` decorated functions
- Depends on: `mcp.server.fastmcp.FastMCP`, `mcp.server.fastmcp.Context`
- Used by: Transport layer

**Lifespan / Context Layer:**
- Purpose: Manages the Hue bridge connection lifecycle and provides shared state
- Location: `hue_server.py` lines 71-146
- Contains: `HueContext` dataclass, `hue_lifespan()` async context manager
- Depends on: `phue.Bridge`, filesystem config (`~/.hue-mcp/config.json`)
- Used by: All tool functions via `get_bridge_ctx(ctx)`

**Tool Layer:**
- Purpose: Business logic for controlling lights, groups, scenes
- Location: `hue_server.py` lines 255-1096
- Contains: 18 tool functions registered with `@mcp.tool()`
- Depends on: Lifespan context (bridge + light_info cache)
- Used by: MCP framework (invoked by AI client requests)

**Utility Layer:**
- Purpose: Pure helper functions and validation
- Location: `hue_server.py` lines 148-251
- Contains: `get_bridge_ctx()`, `rgb_to_xy()`, `validate_light_id()`, `validate_group_id()`, `format_light_info()`
- Depends on: Nothing (pure functions, except `get_bridge_ctx` which accesses context)
- Used by: Tool layer

**Prompt Layer:**
- Purpose: Pre-defined prompt templates for common use cases
- Location: `hue_server.py` lines 1098-1159
- Contains: 3 prompt functions registered with `@mcp.prompt()`
- Depends on: Nothing
- Used by: MCP framework (offered to AI clients)

## Data Flow

**Tool Invocation Flow:**

1. MCP client (e.g., Claude Desktop) sends a tool call request over stdio or SSE
2. `FastMCP` framework routes the request to the matching `@mcp.tool()` function
3. Tool function calls `get_bridge_ctx(ctx)` to extract `Bridge` and `light_info` from the lifespan context
4. Tool validates inputs (light ID, group ID, value ranges)
5. Tool calls `phue.Bridge` methods (`set_light()`, `set_group()`, etc.) to control hardware
6. Tool returns a human-readable string result
7. FastMCP framework sends the result back to the MCP client

**Bridge Connection Flow (Startup):**

1. `hue_lifespan()` runs as an async context manager at server start
2. Loads saved config from `~/.hue-mcp/config.json` if it exists
3. Creates `phue.Bridge` instance (auto-discovery if no IP configured)
4. Calls `bridge.connect()` (may prompt user to press bridge link button on first run)
5. Saves bridge IP and username to config file
6. Caches all light info via `bridge.get_light()`
7. Yields `HueContext(bridge, light_info)` for the server's lifetime

**State Management:**
- Bridge connection: Held in `HueContext.bridge` for the server lifetime
- Light info cache: `HueContext.light_info` -- populated at startup, refreshable via `refresh_lights` tool
- Config persistence: `~/.hue-mcp/config.json` stores bridge IP and username across restarts
- No database; all state comes from the Hue bridge API or the local config file

## Key Abstractions

**HueContext:**
- Purpose: Bundles the bridge connection and light cache into a single lifespan context object
- Definition: `hue_server.py` line 72 (dataclass)
- Pattern: Passed through MCP's lifespan mechanism, accessed via `ctx.request_context.lifespan_context`

**FastMCP:**
- Purpose: The MCP server framework that handles protocol details
- Definition: `hue_server.py` line 142
- Pattern: Decorator-based registration (`@mcp.tool()`, `@mcp.prompt()`)

**phue.Bridge:**
- Purpose: Python client for the Philips Hue REST API
- Pattern: Used as a thin wrapper around Hue bridge HTTP calls; methods like `set_light()`, `set_group()`, `get_scene()`

## Entry Points

**stdio mode (default):**
- Location: `hue_server.py` line 1200
- Triggers: `uv run python hue_server.py` or `uv run mcp dev hue_server.py`
- Responsibilities: Runs MCP server reading/writing JSON-RPC over stdin/stdout

**SSE mode:**
- Location: `hue_server.py` lines 1189-1195
- Triggers: `uv run python hue_server.py --sse`
- Responsibilities: Runs MCP server as an HTTP SSE endpoint via uvicorn on configurable host:port (default `127.0.0.1:8080`)

**MCP Inspector (development):**
- Triggers: `uv run mcp dev hue_server.py`
- Responsibilities: Launches the MCP Inspector UI for interactive testing

## MCP Tools (18 total)

**Query tools (read-only):**
- `get_all_lights` - List all lights with formatted info
- `get_light` - Get details for a specific light
- `get_all_groups` - List all light groups
- `get_group` - Get details for a specific group
- `get_all_scenes` - List all scenes
- `find_light_by_name` - Search lights by name (case-insensitive)
- `refresh_lights` - Refresh the light info cache

**Light control tools:**
- `turn_on_light` / `turn_off_light` - Toggle individual lights
- `set_brightness` - Set light brightness (0-254)
- `set_color_rgb` - Set light color via RGB (converted to XY via `rgb_to_xy`)
- `set_color_temperature` - Set color temperature in Kelvin (converted to mired)
- `set_color_preset` - Apply named presets (warm, cool, relax, energize, etc.)
- `alert_light` - Flash a light for identification
- `set_light_effect` - Set dynamic effects (colorloop/none)

**Group control tools:**
- `turn_on_group` / `turn_off_group` - Toggle groups
- `set_group_brightness` - Set group brightness
- `set_group_color_rgb` - Set group color via RGB
- `set_group_color_preset` - Apply presets to groups
- `set_scene` - Apply a scene to a group
- `quick_scene` - Create ad-hoc lighting setup with multiple params
- `create_group` - Create a new light group

## MCP Prompts (3 total)

- `control_lights` - General natural language light control prompt
- `create_mood` - Mood lighting setup prompt
- `light_schedule` - Scheduling explanation prompt

## Error Handling

**Strategy:** Try/except with string error returns (no exceptions propagated to client)

**Patterns:**
- Every tool function wraps its body in `try/except Exception as e`
- Errors are logged via `logger.error()` and returned as `f"Error: {str(e)}"` strings
- Input validation happens before bridge calls (range checks, ID existence checks)
- Validation functions (`validate_light_id`, `validate_group_id`) return booleans; tools check and return error strings
- The lifespan function re-raises connection errors (these are fatal)

## Cross-Cutting Concerns

**Logging:** Python `logging` module, logger named `"hue-mcp"`, level configurable via `--log-level` CLI arg. Format: `%(asctime)s - %(name)s - %(levelname)s - %(message)s`

**Validation:** Input validation is inline in each tool function. Light IDs validated against the cached `light_info` dict. Group IDs validated by querying the bridge. Value ranges checked with simple conditionals.

**Authentication:** No application-level auth. The Hue bridge uses a username-based auth (press link button on first connect). Credentials stored in `~/.hue-mcp/config.json`.

**Configuration:** Module-level constants `BRIDGE_IP`, `CONFIG_DIR`, `CONFIG_FILE`. No environment variable support -- config is hardcoded or loaded from the JSON config file.

---

*Architecture analysis: 2026-03-27*
