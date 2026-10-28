# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A Python package implementing document-related tools (format conversion/processing), exposed through an MCP (Model Context Protocol) server for use by AI assistants. Built with `mcp[cli]` (FastMCP) and `markitdown` for document-to-markdown conversion.

## Commands

```bash
# Create and activate a virtual env (Windows PowerShell: .venv\Scripts\Activate.ps1)
uv venv
source .venv/bin/activate

# Install the package in development mode
uv pip install -e .

# Start the MCP server
uv run main.py

# Run all tests
uv run pytest

# Run a single test file / test
uv run pytest tests/test_document.py
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf
```

## Architecture

- `main.py` is the MCP server entry point. It creates a `FastMCP("docs")` instance and registers each tool function explicitly via `mcp.tool()(function_name)`. **A tool implemented in `tools/` is not exposed to the MCP server until it's registered here** — e.g. `tools/document.py` currently defines `binary_document_to_markdown` but it is not yet wired into `main.py`.
- `tools/` holds tool implementations as plain Python functions, one concern per module (`math.py`, `document.py`). These functions are the actual logic; `main.py` is purely wiring.
- `tests/` mirrors `tools/` (e.g. `tests/test_document.py` tests `tools/document.py`). Binary fixtures for document-conversion tests live in `tests/fixtures/`.

### Defining a new MCP tool

Tools are plain Python functions, registered with the server via `mcp.tool()(fn)` in `main.py`. Each tool function must:

1. Live in `tools/` as a standalone function (not a method), grouped into a module by concern (e.g. `tools/document.py` for document operations).
2. Annotate every argument and the return value with its concrete type (e.g. `bytes`, `str`, `float`) — never leave a parameter or return type unannotated. This is required for the MCP schema generation to work correctly, not just style.
3. Use `pydantic.Field` for every parameter to supply a detailed description — this is what the MCP client/AI assistant sees to understand how to call the tool correctly:
   ```python
   from pydantic import Field

   def my_tool(
       param1: str = Field(description="Detailed description of this parameter"),
       param2: int = Field(description="Explain what this parameter does"),
   ) -> ReturnType:
       """Comprehensive docstring here."""
       ...
   ```
4. Have a docstring that:
   - Begins with a one-line summary (this becomes the tool's short description to the model).
   - Follows with a detailed explanation of functionality.
   - Explains when to use (and when *not* to use) the tool.
   - Includes usage examples with expected input/output (see `tools/math.py::add` for the reference format, including `>>>`-style doctest examples).
5. Be registered in `main.py`:
   ```python
   from tools.my_module import my_tool
   mcp.tool()(my_tool)
   ```
   Forgetting this step means the function exists and may be tested, but is invisible to the MCP server/client.
6. Have a corresponding test module under `tests/` (e.g. `tests/test_my_module.py`) that exercises the function directly (not through the MCP layer). For tools that take binary file data, add fixture files under `tests/fixtures/` rather than generating binary content inline in the test.
