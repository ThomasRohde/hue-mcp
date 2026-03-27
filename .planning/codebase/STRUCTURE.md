# Codebase Structure

**Analysis Date:** 2026-03-27

## Directory Layout

```
hue-mcp/
├── .gitignore              # Git ignore rules
├── .python-version         # Python version pin
├── .vscode/
│   └── mcp.json            # VS Code MCP configuration (empty)
├── hue_server.py           # Entire application (single file)
├── pyproject.toml           # Project metadata, dependencies, tool config
├── uv.lock                 # Dependency lockfile (uv)
├── tests/
│   ├── __init__.py         # Makes tests a package
│   └── test_hue_server.py  # Unit tests for utility functions
├── CHANGELOG.md            # Version history
├── CONTRIBUTING.md         # Contribution guidelines
├── LICENSE                 # MIT license
├── README.md               # Project documentation
└── mcp.md                  # MCP protocol reference/notes
```

## Directory Purposes

**Root (`/`):**
- Purpose: Contains the entire application as a single-file Python module
- Contains: The main server file, project config, documentation
- Key files: `hue_server.py`, `pyproject.toml`

**`tests/`:**
- Purpose: Unit tests for pure utility functions
- Contains: Test files using pytest
- Key files: `tests/test_hue_server.py`

**`.vscode/`:**
- Purpose: VS Code workspace configuration
- Contains: MCP configuration (currently empty)
- Key files: `.vscode/mcp.json`

## Key File Locations

**Entry Points:**
- `hue_server.py`: The entire application -- server setup, tools, prompts, utilities, CLI entry point

**Configuration:**
- `pyproject.toml`: Project metadata, dependencies, ruff/mypy/pytest config
- `.python-version`: Python version (for pyenv/uv)
- `~/.hue-mcp/config.json`: Runtime config (bridge IP, username) -- generated at first run, not in repo

**Core Logic:**
- `hue_server.py` lines 71-146: Lifespan management and bridge connection
- `hue_server.py` lines 148-251: Utility functions (rgb_to_xy, validation, formatting)
- `hue_server.py` lines 255-1096: All 18 MCP tool definitions
- `hue_server.py` lines 1098-1159: All 3 MCP prompt definitions
- `hue_server.py` lines 1163-1200: CLI argument parsing and server startup

**Testing:**
- `tests/test_hue_server.py`: Tests for `rgb_to_xy`, `validate_light_id`, `format_light_info`

**Documentation:**
- `README.md`: Setup instructions, usage, configuration
- `CHANGELOG.md`: Version history
- `CONTRIBUTING.md`: How to contribute
- `mcp.md`: MCP protocol reference

## Naming Conventions

**Files:**
- `snake_case.py` for Python source files: `hue_server.py`
- `test_` prefix for test files: `test_hue_server.py`
- `UPPERCASE.md` for standard docs: `README.md`, `LICENSE`, `CHANGELOG.md`, `CONTRIBUTING.md`

**Functions:**
- `snake_case` for all functions: `get_all_lights`, `set_color_rgb`, `rgb_to_xy`
- Tool functions named as verb phrases: `turn_on_light`, `set_brightness`, `find_light_by_name`
- Validation functions prefixed with `validate_`: `validate_light_id`, `validate_group_id`
- Formatting functions prefixed with `format_`: `format_light_info`

**Variables:**
- `snake_case` for all variables: `light_info`, `bridge_ip`, `group_name`
- `UPPER_SNAKE_CASE` for module-level constants: `BRIDGE_IP`, `CONFIG_DIR`, `CONFIG_FILE`

**Classes:**
- `PascalCase`: `HueContext`

## Import Organization

**Order observed in `hue_server.py`:**
1. Standard library imports (`json`, `logging`, `os`, `collections.abc`, `contextlib`, `dataclasses`, `typing`)
2. Third-party imports (`mcp.server.fastmcp`, `phue`)
3. No internal imports (single-file application)

**In tests:**
- Imports are done inside test functions: `from hue_server import rgb_to_xy`

**Path Aliases:**
- None used. Direct imports only.

## Where to Add New Code

**New MCP Tool:**
- Add to `hue_server.py` in the appropriate section (query tools ~lines 255-380, light control ~lines 384-1096)
- Use `@mcp.tool()` decorator
- Follow the pattern: validate inputs, call `get_bridge_ctx(ctx)`, wrap in try/except, return string
- Add corresponding tests in `tests/test_hue_server.py` for any pure helper functions

**New MCP Prompt:**
- Add to `hue_server.py` after line 1159 (prompt section)
- Use `@mcp.prompt()` decorator
- Return a multi-line string with instructions for the AI

**New Utility Function:**
- Add to `hue_server.py` in the utility section (lines 148-251)
- Keep functions pure when possible (no side effects)
- Add unit tests in `tests/test_hue_server.py`

**New Tests:**
- Add to `tests/test_hue_server.py`
- Use `def test_function_name():` pattern (no class-based tests currently)
- Import the function under test inside the test function body
- Tests requiring hardware should use `@pytest.mark.skip(reason="Requires Hue bridge hardware")`

**New Configuration:**
- Add to module-level constants in `hue_server.py` (near line 54)
- If CLI-configurable, add to the `argparse` block at the bottom

## Special Directories

**`~/.hue-mcp/` (runtime, not in repo):**
- Purpose: Stores bridge connection configuration
- Generated: Yes, at first server run
- Committed: No (lives in user home directory)
- Contains: `config.json` with `bridge_ip` and `username`

**`.venv/` (not in repo):**
- Purpose: Python virtual environment managed by `uv`
- Generated: Yes, via `uv sync`
- Committed: No (in `.gitignore`)

## Build and Distribution

**Build system:** Hatchling (`pyproject.toml` `[build-system]`)
- Only `hue_server.py` is included in wheel builds (`[tool.hatch.build.targets.wheel]`)
- No `src/` layout; the module is at the project root

**Package manager:** `uv`
- Lockfile: `uv.lock` (committed)
- Install: `uv sync`
- Run: `uv run python hue_server.py`

---

*Structure analysis: 2026-03-27*
