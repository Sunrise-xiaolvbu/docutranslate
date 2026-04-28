# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DocuTranslate is a lightweight local file translation tool based on Large Language Models. It supports multiple input formats (PDF, DOCX, XLSX, MD, TXT, JSON, EPUB, SRT, ASS, etc.) and provides auto-glossary generation, PDF table/formula recognition, and format preservation.

## Key Commands

### Development Setup
```bash
# Install dependencies
uv sync --group dev

# Install with optional features
uv sync --group dev --extra mcp  # For MCP support
uv sync --group dev --extra docling  # For docling PDF parsing

# Run tests
uv run pytest tests/ -v

# Run with coverage
uv run pytest tests/ -v --cov=docutranslate --cov-report=term --cov-report=html:htmlcov
```

### Running the Application
```bash
# Start web UI (default port 8010)
docutranslate -i

# Start with LAN access
docutranslate -i --host 0.0.0.0

# Start with specific port
docutranslate -i -p 8081

# Start with CORS enabled
docutranslate -i --cors

# Start with MCP server
docutranslate --mcp

# Start with both Web UI and MCP
docutranslate -i --with-mcp
```

### Testing Specific Components
```bash
# Run specific test module
uv run pytest tests/test_glossary/ -v

# Run single test file
uv run pytest tests/test_glossary/test_glossary.py -v

# Run single test case
uv run pytest tests/test_glossary/test_glossary.py::test_glossary_generation -v
```

## Architecture Overview

### Core Components

1. **Workflow System** ([docutranslate/workflow/](docutranslate/workflow/))
   - Each file format has its own workflow (PDF/MD images → MarkdownBasedWorkflow, DOCX → DocxWorkflow, etc.)
   - Workflows follow a common pattern: read → translate → export
   - All workflows implement base interfaces from [workflow/interfaces.py](docutranslate/workflow/interfaces.py)

2. **Translation Pipeline**
   ```
   Input File → Converter → Translator → Exporter → Output
   ```
   - **Converter**: Translates input to intermediate format (Markdown for PDFs, preserves format for DOCX/XLSX)
   - **Translator**: Uses LLM API to translate text content
   - **Exporter**: Converts translated content to desired output format

3. **Client SDK** ([docutranslate/sdk.py](docutranslate/sdk.py))
   - Main entry point for programmatic use
   - `Client` class simplifies workflow creation and execution
   - `TranslationResult` handles output operations

4. **Web Server** ([docutranslate/app.py](docutranslate/app.py))
   - FastAPI-based REST API
   - Interactive web UI with static files
   - CORS support for cross-origin requests

5. **MCP Server** ([docutranslate/mcp/](docutranslate/mcp/))
   - Model Context Protocol implementation
   - Supports multiple transport modes (stdio, SSE, HTTP)
   - Shared task queue with Web UI when combined mode is used

### Key Design Patterns

1. **Factory Pattern** ([core/factory.py](docutranslate/core/factory.py))
   - Creates appropriate workflow based on input parameters
   - Handles configuration mapping from payload to workflow-specific configs

2. **Strategy Pattern**
   - Different translation strategies per file format
   - Pluggable converter engines (MinerU, Docling)

3. **Protocol-based Interfaces**
   - Type-safe interfaces for different export formats
   - Protocol classes ensure consistent implementation across workflows

### Configuration

Environment variables are used throughout:
- `DOCUTRANSLATE_API_KEY`: AI platform API key
- `DOCUTRANSLATE_BASE_URL`: AI platform base URL
- `DOCUTRANSLATE_MODEL_ID`: Model ID to use
- `DOCUTRANSLATE_TO_LANG`: Target language
- `DOCUTRANSLATE_CONCURRENT`: Number of concurrent requests
- `DOCUTRANSLATE_CONVERT_ENGINE`: PDF conversion engine
- `DOCUTRANSLATE_MINERU_TOKEN`: MinerU API token

## File Format Support

### Input Formats
- PDF (requires MinerU or Docling)
- DOCX/DOC (with formatting preservation)
- XLSX/XLS/CSV (with selective region translation)
- Markdown/TXT
- JSON (with JSONPath selection)
- EPUB (with metadata preservation)
- SRT/ASS (subtitles)
- PPTX/PPT
- HTML/HTM
- Images (PNG/JPG → OCR to text)

### Output Formats
- PDF/MD Images: HTML, Markdown (with/without embedded images), DOCX
- DOCX: DOCX, HTML
- XLSX: XLSX, HTML
- Text formats: Original format + HTML
- JSON: JSON, HTML
- Subtitles: Original format + HTML
- EPUB: EPUB, HTML

## Important Implementation Details

1. **PDF Processing**: PDFs are first converted to Markdown using MinerU (online) or local Docling. This loses original layout but preserves text, tables, and formulas.

2. **Async Support**: All operations are async-first. Use `await` for translation operations.

3. **Glossary System**: Supports both manual glossary dictionaries and auto-generated glossaries for term consistency.

4. **Caching**: Markdown-based conversions are cached in memory (last 10 operations).

5. **Error Handling**: All external API calls have retry logic and timeout handling.

6. **Testing**: Tests use pytest with mock isolation for external APIs. Test data should be placed in `tests/test_data/`.

## Adding New File Formats

To add support for a new file format:

1. Create new workflow files in `docutranslate/workflow/`
2. Implement required protocols from `workflow/interfaces.py`
3. Add converter if needed (in `converter/`)
4. Add exporter (in `exporter/`)
5. Update mappings in `sdk.py`
6. Add tests in `tests/`
7. Update core schemas in `core/schemas.py`

## Code Style

- Follow PEP 8 with line length 100
- Type hints required for all public interfaces
- Use Pydantic for configuration schemas
- Async methods use `async def` and `await`
- Logging via Python's logging module