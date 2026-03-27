# Codebase Concerns

**Analysis Date:** 2026-03-27

## Tech Debt

**Duplicated preset definitions:**
- Issue: The `presets` dictionary is defined identically in both `set_color_preset` and `set_group_color_preset` (lines 916-934 and 985-1003 in `hue_server.py`). Any change to presets must be made in two places.
- Files: `hue_server.py` (lines ~916 and ~985)
- Impact: Preset changes risk inconsistency between light and group versions. Adding a new preset requires editing two locations.
- Fix approach: Extract `presets` to a module-level constant (e.g., `COLOR_PRESETS`).

**Stale light cache:**
- Issue: `light_info` is populated once during server lifespan startup (line 128) and only updated when `refresh_lights` is explicitly called. All tool functions read from the cached `light_info` dict, which can be stale if lights are changed externally (via Hue app, physical switches, or other integrations).
- Files: `hue_server.py` (line 128, lines 256-268 for `get_all_lights`)
- Impact: The `get_all_lights` tool returns stale data. Capability checks (e.g., checking `'xy' in light_info[str(light_id)]['state']`) use stale state. Validation of light IDs misses newly added lights.
- Fix approach: Either auto-refresh the cache on read operations, add a TTL-based cache invalidation, or fetch live data for read tools while keeping the cache for validation only.

**Single-file architecture:**
- Issue: The entire server (1200 lines) lives in a single file `hue_server.py`. This includes configuration, context management, utility functions, 17 tool definitions, 3 prompt definitions, and the main entry point.
- Files: `hue_server.py`
- Impact: Harder to navigate, test in isolation, and extend. Adding new tools increases file length linearly.
- Fix approach: Split into modules: `config.py`, `context.py`, `utils.py`, `tools/lights.py`, `tools/groups.py`, `tools/scenes.py`, `prompts.py`.

**Color temperature presets use raw Kelvin, not mired:**
- Issue: Presets like `"warm": {"ct": 2500}` store color temperature in Kelvin, but the Hue API uses mired values. When `set_color_preset` applies these via `bridge.set_light(light_id, 'ct', value)` (line 959), it sends 2500 as mired (which maps to ~400K), not as 2500K. The `set_color_temperature` tool correctly converts K to mired (line 547), but the presets do not.
- Files: `hue_server.py` (lines 918-920 presets, line 959 application)
- Impact: Color temperature presets send incorrect values to the bridge. Warm (2500) as mired = ~400K, which is far below the supported range.
- Fix approach: Store presets in mired values directly (e.g., `"warm": {"ct": 400}` for 2500K) or convert K to mired before sending to the bridge in the preset application loop.

## Known Bugs

**Color temperature preset values sent as wrong unit:**
- Symptoms: Color temperature presets (warm, cool, daylight, concentration, relax, reading, energize) likely produce unexpected light colors.
- Files: `hue_server.py` (lines 918-926 define presets in Kelvin, line 959 sends them directly as `ct` without conversion)
- Trigger: Call `set_color_preset(1, "warm", ctx)` or any temperature-based preset.
- Workaround: Use `set_color_temperature` tool directly, which correctly converts K to mired.

## Security Considerations

**Bridge credentials stored in plaintext:**
- Risk: The Hue bridge username (API key) is stored in plaintext JSON at `~/.hue-mcp/config.json` (line 121-124).
- Files: `hue_server.py` (lines 120-125)
- Current mitigation: File is stored in user home directory with default OS permissions.
- Recommendations: Set restrictive file permissions (0600) on the config file after creation. The bridge username grants full control over all lights on the local network.

**No input sanitization on tool parameters:**
- Risk: While the Hue bridge API likely handles malformed inputs, the server does not sanitize string inputs (e.g., `name` in `create_group`, `name` in `quick_scene`, `name` in `find_light_by_name`).
- Files: `hue_server.py` (lines 761, 804, 726)
- Current mitigation: The phue library likely handles this at the HTTP level.
- Recommendations: Add basic input length validation for string parameters.

**SSE transport binds to localhost only:**
- Risk: Low risk currently since `--host` defaults to `127.0.0.1` (line 1170), but a user could pass `--host 0.0.0.0` to expose the MCP server to the network with no authentication.
- Files: `hue_server.py` (line 1170)
- Current mitigation: Localhost default.
- Recommendations: Add a warning if binding to non-localhost addresses.

## Performance Bottlenecks

**Repeated bridge API calls for group validation:**
- Problem: `validate_group_id` (line 222) calls `bridge.get_group()` to fetch all groups, then the calling tool function often calls `bridge.get_group(group_id)` again to get the group name. This means two HTTP roundtrips per group operation.
- Files: `hue_server.py` (lines 222-234, and each group tool function e.g., 556-581)
- Cause: Validation and data fetch are separate operations that both hit the bridge API.
- Improvement path: Cache groups similarly to lights, or combine validation and info retrieval into a single function that returns the group data.

**Sequential preset application:**
- Problem: When applying presets with multiple attributes (e.g., `{"ct": 4600, "bri": 254}`), each attribute is sent as a separate HTTP request in a loop (line 959).
- Files: `hue_server.py` (lines 958-960)
- Cause: The phue library's `set_light` takes a single attribute at a time in this usage.
- Improvement path: Use phue's ability to set multiple attributes at once by passing a dict: `bridge.set_light(light_id, presets[preset])`.

## Fragile Areas

**Lifespan context error handling:**
- Files: `hue_server.py` (lines 77-139)
- Why fragile: If bridge connection fails during startup, the server crashes with no recovery path. Auto-discovery (`Bridge()` with no args on line 109) can hang or fail silently on networks without UPnP/SSDP.
- Safe modification: Add timeout to bridge discovery, add retry logic, or allow the server to start in a degraded mode.
- Test coverage: No tests cover the lifespan context at all.

**Config file parsing:**
- Files: `hue_server.py` (lines 94-102)
- Why fragile: If `config.json` is malformed or contains unexpected types, the broad `except Exception` catches silently and falls through to default behavior. The username could be None or an empty string.
- Safe modification: Add schema validation for the config file.
- Test coverage: No tests cover config loading.

## Dependencies at Risk

**phue library:**
- Risk: The `phue` library (last PyPI release 2019, GitHub shows limited maintenance) wraps the Hue API v1. Philips has been transitioning to the Hue API v2 (CLIP v2) since 2021. API v1 may eventually be deprecated on newer bridge firmware.
- Impact: The entire server depends on `phue` for all bridge communication. If Hue API v1 is removed in a bridge firmware update, the server stops working entirely.
- Migration plan: Consider switching to `aiohue` (actively maintained, supports API v2) or the official Hue SDK. This would also enable proper async support.

**mcp library (>=1.5.0):**
- Risk: The MCP protocol and SDK are relatively new and evolving. Breaking changes between versions are possible.
- Impact: Server may need updates to match MCP SDK changes.
- Migration plan: Pin to a specific version range rather than open-ended `>=1.5.0`.

## Missing Critical Features

**No CI/CD pipeline:**
- Problem: No `.github/workflows/`, no CI configuration of any kind.
- Blocks: Automated testing, linting, and release processes.

**No async tool implementations:**
- Problem: All tool functions are synchronous (e.g., `def get_all_lights` not `async def`). The phue library makes synchronous HTTP calls to the bridge, which blocks the event loop in the async MCP server.
- Files: `hue_server.py` (all `@mcp.tool()` decorated functions)
- Blocks: Concurrent tool execution; the server handles one request at a time.

**No rate limiting or request throttling:**
- Problem: The Hue bridge has rate limits (~10 commands per second for individual lights, ~1 per second for groups). The server does not enforce any throttling.
- Blocks: Reliable operation when AI sends rapid successive commands (e.g., setting color for many lights in a loop).

## Test Coverage Gaps

**Only pure utility functions are tested:**
- What's not tested: All 17 tool functions, all 3 prompt functions, the lifespan context, config file handling, bridge error scenarios.
- Files: `tests/test_hue_server.py` (only tests `rgb_to_xy`, `validate_light_id`, `format_light_info`)
- Risk: Any change to tool functions, error handling, or bridge interaction logic is unvalidated. The color temperature preset bug (sending K as mired) would have been caught with basic unit tests.
- Priority: High

**No mocking of bridge interactions:**
- What's not tested: The comment at line 109 of `tests/test_hue_server.py` acknowledges "Integration tests would require a mock Hue bridge" but no mocking is implemented.
- Files: `tests/test_hue_server.py` (lines 109-121)
- Risk: Cannot test any tool that calls bridge methods without physical hardware.
- Priority: High - use `unittest.mock.patch` or `pytest-mock` to mock `phue.Bridge`.

**Skipped test stubs:**
- What's not tested: `test_bridge_connection` and `test_get_all_lights` are marked `@pytest.mark.skip` with empty `pass` bodies.
- Files: `tests/test_hue_server.py` (lines 112-121)
- Risk: These give a false sense of planned coverage without actual implementation.
- Priority: Medium

---

*Concerns audit: 2026-03-27*
