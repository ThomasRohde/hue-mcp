# Coding Conventions

**Analysis Date:** 2026-03-27

## Naming Patterns

**Files:**
- Single module architecture: all application code lives in `hue_server.py` (snake_case)
- Test files use `test_` prefix: `tests/test_hue_server.py`
- Configuration in `pyproject.toml`

**Functions:**
- Use `snake_case` for all functions: `rgb_to_xy`, `validate_light_id`, `format_light_info`
- MCP tool functions use verb-noun pattern: `turn_on_light`, `set_brightness`, `set_color_rgb`
- Getter functions use `get_` prefix: `get_all_lights`, `get_light`, `get_all_groups`
- Validation functions use `validate_` prefix: `validate_light_id`, `validate_group_id`
- Helper/utility functions use `format_` prefix: `format_light_info`

**Variables:**
- Use `snake_case`: `light_info`, `bridge_ip`, `group_name`, `light_id_str`
- Constants use `UPPER_SNAKE_CASE`: `BRIDGE_IP`, `CONFIG_DIR`, `CONFIG_FILE`
- Short-lived loop variables are acceptable: `r`, `g`, `b`, `X`, `Y`, `Z` (in color math)

**Types/Classes:**
- Use `PascalCase` for dataclasses: `HueContext`
- Type hints use modern Python syntax: `dict[str, Any]`, `list[int]`, `list[int] | None`

## Code Style

**Formatting:**
- Tool: Ruff (formatter and linter combined)
- Line length: 100 characters (configured in `pyproject.toml`)
- Target Python version: 3.10

**Linting:**
- Tool: Ruff with these rule sets enabled (configured in `pyproject.toml`):
  - `E` - pycodestyle errors
  - `W` - pycodestyle warnings
  - `F` - pyflakes
  - `I` - isort (import sorting)
  - `B` - flake8-bugbear
  - `C4` - flake8-comprehensions
  - `UP` - pyupgrade
- Ignored: `E501` (line too long, handled by formatter)

**Type Checking:**
- Tool: mypy with strict settings (configured in `pyproject.toml`)
- `disallow_untyped_defs = true` - all functions must have type annotations
- `disallow_incomplete_defs = true`
- `strict_optional = true`
- `warn_return_any = true`

**Indentation:**
- 4 spaces (Python standard)

## Import Organization

**Order:**
1. Standard library imports (`json`, `logging`, `os`, `collections.abc`, `contextlib`, `dataclasses`, `typing`)
2. Third-party imports (`mcp.server.fastmcp`, `phue`)
3. No local imports (single-module project)

**Style:**
- One import per line for standard library
- Specific names imported: `from typing import Any`, `from mcp.server.fastmcp import Context, FastMCP`
- Conditional imports in `if __name__ == "__main__"` block: `import argparse`, `import uvicorn`

**Path Aliases:**
- None used (single-module project)

## Error Handling

**Patterns:**
- MCP tool functions return error strings rather than raising exceptions: `return f"Error: Light with ID {light_id} not found."`
- All tool functions wrap bridge operations in `try/except Exception as e` blocks
- Errors are logged via `logger.error()` before returning error strings to the user
- Input validation happens BEFORE try/except blocks using early returns:
  ```python
  if not 0 <= brightness <= 254:
      return "Error: Brightness must be between 0 and 254."
  ```
- The lifespan function (`hue_lifespan`) re-raises exceptions after logging them
- Error messages always start with `"Error: "` prefix

**Validation pattern (use this for new tools):**
1. Validate simple input ranges first (before accessing bridge context)
2. Get bridge context via `get_bridge_ctx(ctx)`
3. Validate resource existence (`validate_light_id`, `validate_group_id`)
4. Check capability support (e.g., `'xy' not in light_info[str(light_id)]['state']`)
5. Auto-enable resource if needed (turn on light/group if off)
6. Perform the operation
7. Return human-readable confirmation string

## Logging

**Framework:** Python standard `logging` module

**Configuration (in `hue_server.py` module level):**
```python
logging.basicConfig(level=logging.INFO,
                   format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger("hue-mcp")
```

**Patterns:**
- Use module-level logger named `"hue-mcp"`
- Log errors in catch blocks: `logger.error(f"Error turning on light {light_id}: {e}")`
- Log info for lifecycle events: `logger.info(f"Loaded configuration from {CONFIG_FILE}")`
- Use f-strings for log messages (not lazy % formatting)
- Every `except` block logs the error before returning

## Comments

**When to Comment:**
- Section headers use `# --- Section Name ---` format (e.g., `# --- Configuration ---`, `# --- Utility Functions ---`, `# --- Tools ---`)
- Inline comments explain non-obvious logic: `# Convert temperature in Kelvin to mired (micro reciprocal degree)`
- Comments explain "why" not "what" where possible

**Docstrings:**
- Module-level docstring at top of `hue_server.py` with Features, Requirements, Setup, Usage sections
- All functions have docstrings using Google-style format:
  ```python
  def set_color_rgb(light_id: int, red: int, green: int, blue: int, ctx: Context) -> str:
      """
      Set light color using RGB values.

      Args:
          light_id: The ID of the light
          red: Red value (0-255)
          green: Green value (0-255)
          blue: Blue value (0-255)

      Returns:
          Confirmation message
      """
  ```
- Some docstrings include `Example:` sections with `>>>` notation
- Dataclass docstrings are single-line: `"""Context object holding the Hue bridge connection..."""`
- Test docstrings are single-line descriptions: `"""Test RGB to XY conversion for pure red."""`

## Function Design

**Size:** Functions are short, typically 10-30 lines. Each MCP tool does one thing.

**Parameters:**
- MCP tool functions always accept `ctx: Context` as a parameter
- Use type hints on all parameters: `light_id: int`, `name: str`, `rgb: list[int] | None = None`
- Optional parameters use `| None = None` syntax (modern Python union types)
- Default values for optional group_id: `group_id: int = 0`

**Return Values:**
- All MCP tool functions return `str` (JSON or human-readable messages)
- JSON output uses `json.dumps(data, indent=2)`
- Confirmation messages include both ID and name: `f"Light {light_id} ({light_info[str(light_id)]['name']}) turned on."`
- Utility functions return appropriate types: `list[float]`, `bool`, `dict[str, Any]`

## Module Design

**Exports:** No explicit `__all__`. Single-module project with all code in `hue_server.py`.

**Barrel Files:** `tests/__init__.py` exists with a single docstring: `"""Tests for hue-mcp server."""`

**Code Organization within `hue_server.py`:**
1. Module docstring
2. Standard library imports
3. Third-party imports
4. Configuration constants
5. Logging setup
6. Server context (dataclass + lifespan)
7. MCP server instantiation
8. Utility functions
9. Resource/getter tools
10. Action tools
11. Helper tools
12. Prompts
13. `if __name__ == "__main__"` block

## MCP Tool Pattern

**Every new MCP tool should follow this template:**
```python
@mcp.tool()
def tool_name(required_param: type, ctx: Context, optional_param: type | None = None) -> str:
    """
    Brief description of what the tool does.

    Args:
        required_param: Description
        optional_param: Description

    Returns:
        Confirmation message
    """
    # 1. Validate simple inputs first
    if not valid_range:
        return "Error: Description of invalid input."

    # 2. Get bridge context
    bridge, light_info = get_bridge_ctx(ctx)

    try:
        # 3. Validate resources exist
        if not validate_light_id(light_id, light_info):
            return f"Error: Light with ID {light_id} not found."

        # 4. Perform operation
        bridge.set_light(light_id, 'property', value)

        # 5. Return confirmation with names
        return f"Light {light_id} ({light_info[str(light_id)]['name']}) updated."
    except Exception as e:
        logger.error(f"Error doing X for light {light_id}: {e}")
        return f"Error: {str(e)}"
```

---

*Convention analysis: 2026-03-27*
