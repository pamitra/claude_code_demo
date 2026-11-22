# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A Python package of document-related tools (format conversion and processing) exposed to AI assistants through an MCP server built with `FastMCP` (`mcp[cli]==1.8.0`). Document conversion is backed by `markitdown[docx,pdf]`.

## Commands

Uses `uv`. A `.venv` already exists at the repo root; `uv run` picks it up automatically, so activation is optional (on Windows PowerShell: `.\.venv\Scripts\Activate.ps1`).

```bash
uv pip install -e .                 # install in development mode
uv run main.py                      # start the MCP server (stdio)
uv run pytest                       # run all tests
uv run pytest tests/test_document.py::TestBinaryDocumentToMarkdown::test_binary_document_to_markdown_with_pdf   # single test
```

No linter or formatter is configured.

## Architecture

- `main.py` — creates the `FastMCP("docs")` server and registers tools. **Only functions registered here are exposed over MCP.**
- `tools/` — plain Python functions, one module per area (`math.py`, `document.py`). They have no MCP dependency and can be unit-tested directly.
- `tests/` — pytest tests calling tool functions directly; binary fixtures (`mcp_docs.docx`, `mcp_docs.pdf`) live in `tests/fixtures/`.

Current state: only `tools.math.add` is registered. `tools.document.binary_document_to_markdown` (bytes + file extension → markdown via `MarkItDown` with `StreamInfo(extension=...)`) is implemented and tested but **not yet registered** and lacks the `Field`-annotated parameters/full docstring the conventions below require.

## Defining MCP tools

1. Write the tool as a function in the appropriate `tools/` module.
2. Register it in `main.py`:
   ```python
   from tools.document import my_function
   mcp.tool()(my_function)
   ```
3. Describe **every parameter** with pydantic `Field(description=...)` as its default value — FastMCP uses these to build the tool's input schema, which is what the AI client sees:
   ```python
   from pydantic import Field

   def my_tool(
       param1: str = Field(description="Detailed description of this parameter"),
       param2: int = Field(description="Explain what this parameter does"),
   ) -> ReturnType:
       """Comprehensive docstring here"""
   ```
   Include type hints on all parameters and the return value.
4. The docstring becomes the tool description sent to the model. It must:
   - Begin with a one-line summary
   - Give a detailed explanation of what the tool does
   - Explain when to use (and when not to use) the tool
   - Include usage examples with expected input/output

`tools/math.py::add` is the reference example of a fully compliant tool definition.

Note: because parameters default to `Field(...)` objects, calling a tool function in tests with arguments omitted will pass the `FieldInfo` object rather than a value — always pass all arguments explicitly in tests.
