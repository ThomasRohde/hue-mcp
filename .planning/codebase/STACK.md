# Technology Stack

**Analysis Date:** 2026-03-27

## Languages

**Primary:**
- Python 3.10+ - All application code in single-file server (`hue_server.py`)

**Secondary:**
- None

## Runtime

**Environment:**
- Python 3.10 (pinned in `.python-version`)
- Supports 3.10, 3.11, 3.12, 3.13 per `pyproject.toml` classifiers

**Package Manager:**
- uv (Astral)
- Lockfile: `uv.lock` (present, committed)

## Frameworks

**Core:**
- `mcp` 1.5.0 - Model Context Protocol SDK; uses `FastMCP` from `mcp.server.fastmcp`
- `phue` 1.1 - Philips Hue bridge communication library

**Testing:**
- `pytest` 8.4.2 - Test runner
- `pytest-asyncio` 1.2.0 - Async test support (asyncio_mode = "auto")

**Build/Dev:**
- `hatchling` - Build backend (`pyproject.toml` [build-system])
- `ruff` 0.14.2 - Linting and formatting
- `mypy` 1.18.2 - Static type checking
- `uvicorn` 0.34.0 - ASGI server for SSE transport mode

## Key Dependencies

**Critical (runtime):**
- `mcp` >=1.5.0 (locked 1.5.0) - MCP protocol server SDK. Provides `FastMCP`, `Context`, tool/prompt decorators.
- `phue` >=1.1 (locked 1.1) - Philips Hue REST API client. Provides `Bridge` class for light/group/scene control.

**Infrastructure (transitive, notable):**
- `uvicorn` 0.34.0 - Used in `__main__` block for SSE transport via `mcp.sse_app()`
- `anyio` 4.9.0 - Async I/O (MCP dependency)
- `pydantic` - Data validation (MCP dependency)
- `httpx` - HTTP client (MCP dependency)
- `starlette` - ASGI framework (MCP SSE dependency)

**Dev-only:**
- `ruff` >=0.8.0 (locked 0.14.2) - Linter + formatter
- `mypy` >=1.13.0 (locked 1.18.2) - Type checker
- `pytest` >=8.3.0 (locked 8.4.2) - Test runner
- `pytest-asyncio` >=0.24.0 (locked 1.2.0) - Async test support

## Configuration

**Build Configuration:**
- `pyproject.toml` - Single config file for project metadata, build system, and all tool configs
- Build backend: hatchling
- Wheel target includes only `hue_server.py`

**Tool Configuration (all in `pyproject.toml`):**
- Ruff: line-length 100, target py310, selects E/W/F/I/B/C4/UP rules, ignores E501
- Mypy: strict settings - disallow_untyped_defs, strict_optional, check_untyped_defs
- Pytest: testpaths = ["tests"], asyncio_mode = "auto"

**Runtime Configuration:**
- Bridge config stored at `~/.hue-mcp/config.json` (auto-created)
- No `.env` files used; bridge IP is hardcoded as `None` (auto-discovery) in `hue_server.py` line 58

## Run Commands

```bash
# Install dependencies
uv sync

# Run server (stdio mode, default)
uv run python hue_server.py

# Run server (SSE mode)
uv run python hue_server.py --sse --host 127.0.0.1 --port 8080

# Development with MCP Inspector
uv run mcp dev hue_server.py

# Install in Claude Desktop
uv run mcp install hue_server.py --name "Philips Hue"

# Run tests
uv run pytest

# Lint
uv run ruff check hue_server.py

# Type check
uv run mypy hue_server.py
```

## Platform Requirements

**Development:**
- Python 3.10+
- uv package manager
- Access to a Philips Hue bridge on the local network (or tests will skip bridge interaction)

**Production:**
- Python 3.10+
- Network access to Philips Hue bridge
- Physical access to bridge for initial pairing (press link button)

---

*Stack analysis: 2026-03-27*
