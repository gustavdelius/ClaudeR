# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**ClaudeR** is an R package that connects RStudio to MCP-configured LLM agents (Claude Code, Codex, Qwen Code, Gemini CLI). It enables interactive coding sessions where agents execute R code in a live RStudio environment and receive real-time feedback including plots and output.

The package consists of:
- **R addin** (Shiny-based UI in `R/`) that runs an HTTP server in RStudio
- **Python MCP server** (`inst/scripts/persistent_r_mcp.py`) that bridges CLI agents to the R addin
- **Built-in analysis protocols** (in `inst/prompts/`) for manuscript auditing, statistical best practices, and multi-agent coordination

Current version: 0.2.0 (MIT licensed)

## Architecture

### The Stack

1. **CLI Agent** (Claude Code, Codex, Qwen, Gemini) → initiates MCP connection
2. **Python MCP Server** (`persistent_r_mcp.py`) → translates MCP calls to HTTP POST requests
3. **HTTP Addin** (RStudio, running in `claudeAddin()`) → receives HTTP requests, executes R code
4. **R Session** (global environment) → where code actually runs and persists

### Key Flows

**Code Execution:**
- Agent calls `execute_r(code)` → MCP server POSTs to `http://127.0.0.1:{port}` → `claudeAddin()` evaluates in `.GlobalEnv` → response includes output + optional plot base64 data → agent sees results inline

**Session Discovery:**
- Each active RStudio session writes a JSON file to `~/.claude_r_sessions/{session_name}.json` (includes port, PID, timestamp)
- Python MCP server auto-discovers and connects to the "default" session (or falls back to lowest port)
- Agents can call `connect_session(name)` to bind to a specific named session instead

**Multi-Agent Coordination:**
- Each agent gets a unique ID on first tool call (`agent-{uuid}`)
- Execution history is tracked in-memory (`.claude_history_env`) and logged to file
- Agents can call `get_session_history(agent_filter = "all")` to see what others have done

**Plot Capture:**
- After code executes, ClaudeR detects if a new plot was created by comparing device state before/after
- PNG preferred (6x4" @ 100 dpi); JPEG fallback
- Stale plot detection prevents the last plot from re-appearing on subsequent executions
- HTML widgets (plotly, DT, leaflet) open in the browser instead of stealing the Shiny viewer pane

**Async Execution:**
- `execute_r_async` spawns a background R process via `callr::r_bg`
- Code runs in isolation with explicit input/output marshaling via `inputs`/`outputs` parameters (snapshots data via tempfile RDS)
- Agent polls with `get_async_result` until job completes
- Useful for long-running fits, simulations, or large data operations

**Code Security:**
- System commands (`system()`, `system2()`, `shell()`), terminal access (`rstudioapi::terminal`), and file deletion are blocked by regex checks in `validate_code_security()`
- These restrictions apply only to agent-executed code, not manual user code
- Agents can still read files, install packages, and create/modify R objects

**Data Annotation Workflow:**
- Two modes: interactive (`load_annotation_data` + `annotate` loop) or fully isolated (`run_annotation_job` spawns fresh subprocess per row)
- CSV schema defined in `_schema` column using type syntax: `choice[a,b,c]`, `float[min,max]`, `int[min,max]`, `bool`, `text`
- Working copy created on first load; original never modified; resumable on interruption
- Subprocess mode supports `claude`, `codex`, `gemini`, `qwen`, or local `ollama` HTTP calls (no context carryover between rows)

### File Structure

```
R/
  ui.R              # Main RStudio addin (Shiny UI, HTTP server, code execution engine)
  addin.R           # Legacy simple addin (claude_rstudio_addin); mostly superseded by ui.R
  setup.R           # install_clauder() / configure() for desktop apps (Claude Desktop, Cursor)
  install_cli.r     # install_cli() for CLI tools (Claude Code, Codex, Qwen, Gemini)

inst/
  rstudio/
    addins.dcf      # RStudio addin registration metadata
  scripts/
    persistent_r_mcp.py  # Python MCP server (2322 lines: 27 MCP tools, session discovery, annotation job runner)
  prompts/
    reviewer_zero.md           # 4-pass academic audit protocol
    r_best_practices.md        # Statistical analysis best practices
    multi_agent.md             # Multi-agent coordination protocol
    data_annotation.md         # Data annotation workflow

DESCRIPTION        # Package metadata (version 0.2.0, R >= 4.0, imports: httpuv, jsonlite, shiny, callr, etc.)
README.md          # Feature overview, quick start, demo videos, troubleshooting
LICENSE            # MIT
.Rbuildignore      # Excludes assets, .github, .claude, .positai from R build
```

## Building and Testing

### Development Setup

No special build steps required for development. Package uses standard R:

```bash
# Install dependencies (in R console)
install.packages(c("devtools", "roxygen2"))

# Load/test interactively
devtools::load_all()  # Simulates install_github()
```

### Documentation

Roxygen comments in `R/*.R` files define all exported functions. Regenerate `man/` after edits:

```bash
# In R console
roxygen2::roxygenise()
```

This updates function signatures in `NAMESPACE` and creates `.Rd` files in `man/`.

### Testing

There is no formal test suite. The package is tested via:
1. Manual integration tests using RStudio + Claude Desktop/CLI agents
2. Agent execution history logging (enabled in Shiny addin UI)
3. Session logs (`clauder_{session}_{port}_{timestamp}.R`) capture all executed code for review

To add formal tests in the future: create `tests/testthat/` with `test_*.R` files and add `devtools::test()` to CI.

### Build and Install

```bash
# Build the package for distribution
R CMD build .

# Install from built package
R CMD INSTALL ClaudeR_0.2.0.tar.gz

# Or install directly from source during development
devtools::install()
```

## Key Dependencies and Their Roles

- **httpuv**: Powers the HTTP server in the Shiny addin (receives MCP requests)
- **shiny**: Provides the Shiny UI for the addin viewer pane
- **miniUI**: Compact Shiny widgets for the addin (gadgetTitleBar, miniPage)
- **jsonlite**: JSON serialization for HTTP payloads and MCP communication
- **callr**: Spawns background R processes for `execute_r_async`
- **ggplot2**: Used for plot capture detection (checks `result$value` class)
- **base64enc**: Encodes PNG/JPEG images as base64 for MCP response payloads
- **rstudioapi**: Accesses RStudio context (navigate to files, show notifications)
- **grDevices**: Low-level plot device management (dev.copy, recordPlot)

Python MCP server dependencies (in `persistent_r_mcp.py`):
- **mcp**: Anthropic's Model Context Protocol library
- **httpx**: Async HTTP client for communicating with the R addin
- Standard library: asyncio, json, subprocess, threading, uuid, re, csv, tempfile, datetime

## Common Development Tasks

### Adding a New MCP Tool

All 27 tools are defined in `persistent_r_mcp.py`:

1. **Add the tool definition** in `list_tools()` (line ~579): define name, description, inputSchema (JSON Schema), and MCP annotations (readOnlyHint, destructiveHint, idempotentHint)
2. **Implement the handler** in `call_tool()` (line ~1222): parse arguments, call helper functions, return TextContent or ImageContent
3. **Helper functions**: write utility functions for subprocess execution, parsing, validation (examples: `_parse_annotation_schema`, `_validate_annotation`, `escape_r_string`)

Example: the `load_annotation_data` tool parses CSV, extracts schema from `_schema` column, saves a working copy, and returns the first unannotated row. The `annotate` tool validates input against schema, saves the row, and auto-loads the next.

### Extending the R Addin

The Shiny addin in `R/ui.R` handles:
- HTTP server startup/shutdown
- Request routing (POST for code execution, GET for status)
- Async job tracking via `callr` in `.claude_bg_jobs` environment
- Settings persistence (load/save to `~/.claude_r_settings.rds`)
- Log file management (header with sessionInfo, timestamp, working directory)

To add a new feature:
1. Add UI element in the `ui` definition (miniPage, wellPanel, actionButton, etc.)
2. Add observer in the `server` function (observeEvent, reactiveValues, renderText)
3. Add HTTP endpoint in the POST/GET handler if needed
4. Persist state to `.claude_server_env` if it should survive UI restart

### Updating Prompts

Built-in protocols live in `inst/prompts/*.md`. When editing:
- Keep sections clearly marked with `## Pass N:` headers (Reviewer Zero uses this structure)
- Include exact R code examples for agents to copy/paste
- Test with an actual agent before committing (protocols must be self-contained and runnable)

Examples:
- `reviewer_zero.md`: 4-pass manuscript audit (extract → verify → recompute → check references)
- `multi_agent.md`: task planning, claiming, handoffs, cross-checks
- `r_best_practices.md`: EDA → assumptions → model → diagnostics → reporting

## Common Pitfalls

1. **Plot capture timing**: After `eval()`, ClaudeR checks device state to detect *new* plots. If code calls a function that internally generates a plot and hides it, the agent won't see it. Advise agents to explicitly call `print()` on plot objects.

2. **htmlwidget viewer hijacking**: During agent execution, the viewer is redirected to the browser to prevent htmlwidgets from stealing the Shiny pane. This is controlled by `.claude_viewer_env$suppress`. Manual user code is unaffected.

3. **Agent context pollution**: When using `load_annotation_data` + `annotate` interactively, context accumulates across rows (useful for consistency but risky for anchoring). Use `run_annotation_job` instead for fresh subprocess per row.

4. **Session discovery**: If the R PID dies but its discovery file lingers (corrupted file, unclean shutdown), Python cleans it up on next discover call. To avoid stale sessions manually, delete `~/.claude_r_sessions/*.json`.

5. **Windows HOME vs USERPROFILE**: R's `path.expand("~")` can resolve to OneDrive-synced Documents on Windows (if HOME is set), but Python's `os.path.expanduser("~")` ignores HOME and uses USERPROFILE. ClaudeR explicitly uses USERPROFILE on Windows in both R and Python to stay in sync.

6. **Code security bypass attempts**: The validation regex in `validate_code_security()` is not foolproof (e.g., `sys\tem()` would slip through, as would comments hiding system calls). It's a speed bump, not a firewall. Always review agent output before trusting it in production.

## Key Constants and State

**Global Environments (R):**
- `.claude_server_env`: Persists server state across UI restarts (running, port, session_name, execution_count)
- `.claude_history_env`: Agent execution history (max 500 entries) with timestamps, agent IDs, code, success flag, has_plot
- `.claude_bg_jobs`: Background async job tracking (callr process handles, marshaling tempfile paths)
- `.claude_viewer_env`: Captures htmlwidget URLs and suppression flag for browser redirect

**Settings:**
- Stored in `~/.claude_r_settings.rds` (RDS, not JSON)
- Fields: print_to_console, log_to_file, log_file_path
- Loaded on addin startup, updated reactively in Shiny UI

**Session Discovery:**
- One JSON file per session in `~/.claude_r_sessions/{session_name}.json`
- Python prefers "default" session; agents can override with `connect_session(name)`

**Agent ID:**
- Generated as `agent-{uuid}` by Python MCP server on startup
- Passed to every R tool call so code execution is attributed

## Release and Distribution

ClaudeR is distributed via:
1. **GitHub**: `devtools::install_github("IMNMV/ClaudeR")`
2. **PyPI**: `clauder-mcp` package (the Python MCP server as a standalone CLI; installed via `uvx clauder-mcp` or legacy pip)

When bumping version:
- Update `Version: X.Y.Z` in `DESCRIPTION`
- Git commit and tag: `git tag v0.2.1 && git push origin v0.2.1`
- The R package build system (CRAN, GitHub Actions, etc.) picks this up automatically

For the PyPI `clauder-mcp` package: there's a separate `server.py` mirror that mirrors the exact tools from `persistent_r_mcp.py`. Keep them in sync.
