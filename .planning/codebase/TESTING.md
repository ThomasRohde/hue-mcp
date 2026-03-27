# Testing Patterns

**Analysis Date:** 2026-03-27

## Test Framework

**Runner:**
- pytest >= 8.3.0
- Config: `pyproject.toml` (`[tool.pytest.ini_options]`)

**Async Support:**
- pytest-asyncio >= 0.24.0
- Mode: `asyncio_mode = "auto"` (async tests are detected automatically, no `@pytest.mark.asyncio` needed)

**Assertion Library:**
- Built-in pytest assertions (no additional assertion library)

**Run Commands:**
```bash
uv run pytest              # Run all tests
uv run pytest -v           # Run with verbose output
uv run pytest tests/       # Run tests in specific directory
python -m pytest           # Alternative: run via python module
```

Note: No coverage tool is currently configured. No watch mode is configured.

## Test File Organization

**Location:**
- Separate `tests/` directory (not co-located with source)
- Configured via `testpaths = ["tests"]` in `pyproject.toml`

**Naming:**
- Test files: `test_*.py` (e.g., `tests/test_hue_server.py`)
- Test functions: `test_<description>` using snake_case (e.g., `test_rgb_to_xy_pure_red`)

**Structure:**
```
tests/
├── __init__.py              # Package init with docstring
└── test_hue_server.py       # Tests for hue_server module
```

**Init file:** `tests/__init__.py` contains only a module docstring: `"""Tests for hue-mcp server."""`

## Test Structure

**Suite Organization:**
- Tests are flat functions (no test classes)
- Tests are grouped by function being tested
- Each test function tests one specific case
- No setup/teardown fixtures currently used

```python
def test_rgb_to_xy_pure_red():
    """Test RGB to XY conversion for pure red."""
    from hue_server import rgb_to_xy

    result = rgb_to_xy(255, 0, 0)
    assert isinstance(result, list)
    assert len(result) == 2
    assert result[0] > 0.6
    assert 0 <= result[1] <= 1
```

**Patterns observed:**
- Import the function under test inside the test function body (not at module level)
- Single docstring describing the test case
- Arrange-Act-Assert pattern (implicit, not commented)
- Use plain `assert` statements with comparison operators
- Test boundary values and type checking together

## Mocking

**Framework:** None currently used

**Current approach:**
- Tests only cover pure utility functions that do not require the Hue bridge
- Hardware-dependent tests are skipped with `@pytest.mark.skip(reason="Requires Hue bridge hardware")`
- No mock objects for `Bridge`, `Context`, or MCP server

**What IS tested (no mocking needed):**
- `rgb_to_xy()` - pure function, color math
- `validate_light_id()` - takes a plain dict, no bridge needed
- `format_light_info()` - takes a plain dict, no bridge needed

**What is NOT tested (would need mocking):**
- All `@mcp.tool()` functions (require `Context` and `Bridge`)
- `hue_lifespan()` context manager
- `get_bridge_ctx()` helper
- Bridge connection and discovery logic

**Recommended mocking approach for future tests:**
```python
from unittest.mock import MagicMock, patch

# Mock the Bridge class
mock_bridge = MagicMock()
mock_bridge.get_light.return_value = {
    "1": {"name": "Test Light", "state": {"on": True, "bri": 254}}
}

# Mock the Context
mock_ctx = MagicMock()
mock_ctx.request_context.lifespan_context = HueContext(
    bridge=mock_bridge,
    light_info=mock_bridge.get_light()
)
```

## Fixtures and Factories

**Test Data:**
- Inline test data within each test function
- Light info dicts are constructed manually:
```python
light_info = {
    "1": {
        "name": "Living Room",
        "type": "Extended color light",
        "state": {
            "on": True,
            "bri": 254,
            "colormode": "xy",
            "reachable": True,
        },
        "modelid": "LCT015",
        "manufacturername": "Signify",
    }
}
```

**Location:**
- No shared fixtures file
- No `conftest.py` file exists
- All test data is defined inline within individual test functions

## Coverage

**Requirements:** None enforced. No coverage tool configured.

**To add coverage:**
```bash
# Install
uv add --dev pytest-cov

# Run with coverage
uv run pytest --cov=hue_server --cov-report=term-missing

# Generate HTML report
uv run pytest --cov=hue_server --cov-report=html
```

## Test Types

**Unit Tests:**
- Only type currently present
- Test pure utility functions: `rgb_to_xy`, `validate_light_id`, `format_light_info`
- Located in `tests/test_hue_server.py`
- 7 active test functions, 2 skipped stubs

**Integration Tests:**
- Not implemented
- Placeholder stubs exist with `@pytest.mark.skip(reason="Requires Hue bridge hardware")`:
  - `test_bridge_connection` (empty)
  - `test_get_all_lights` (empty)

**E2E Tests:**
- Not used

## Common Patterns

**Testing pure functions:**
```python
def test_rgb_to_xy_black():
    """Test RGB to XY conversion for black (should return [0, 0])."""
    from hue_server import rgb_to_xy

    result = rgb_to_xy(0, 0, 0)
    assert result == [0.0, 0.0]
```

**Testing with range assertions (for float results):**
```python
def test_rgb_to_xy_white():
    """Test RGB to XY conversion for white."""
    from hue_server import rgb_to_xy

    result = rgb_to_xy(255, 255, 255)
    assert isinstance(result, list)
    assert len(result) == 2
    assert 0.2 <= result[0] <= 0.4
    assert 0.2 <= result[1] <= 0.4
```

**Testing validation functions:**
```python
def test_validate_light_id():
    """Test light ID validation."""
    from hue_server import validate_light_id

    light_info = {
        "1": {"name": "Light 1"},
        "2": {"name": "Light 2"},
    }

    assert validate_light_id(1, light_info) is True
    assert validate_light_id(3, light_info) is False
```

**Skipping hardware-dependent tests:**
```python
@pytest.mark.skip(reason="Requires Hue bridge hardware")
def test_bridge_connection():
    """Test connection to Hue bridge."""
    pass
```

**Running tests directly:**
```python
if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

## Test Gaps

**Untested areas (all in `hue_server.py`):**
- All 16 `@mcp.tool()` functions (lines 255-1096)
- `hue_lifespan()` context manager (line 78)
- `get_bridge_ctx()` helper (line 150)
- `format_light_info()` edge cases (empty dict, missing keys)
- Input validation branches in tool functions
- Error handling paths in tool functions
- The 3 `@mcp.prompt()` functions (lines 1100-1159)
- CLI argument parsing (lines 1163-1200)
- Config file load/save logic (lines 88-125)

**Priority for adding tests:**
1. Mock-based tests for core tool functions (`turn_on_light`, `set_brightness`, `set_color_rgb`)
2. Validation edge cases (invalid IDs, out-of-range values)
3. Error handling paths (bridge connection failures)
4. Config persistence logic

---

*Testing analysis: 2026-03-27*
