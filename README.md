# RAG Anything MCP Server

An MCP (Model Context Protocol) server that provides RAG (Retrieval-Augmented Generation) capabilities over a **single shared workspace** of documents, using the `raganything` library with full multimodal support.

All ingested documents land in one shared workspace on disk (default `~/.rag_anything/shared_workspace`). Queries run against that workspace as a whole — there is no per-directory RAG instance.

## Features

- **End-to-End Document Processing**: Document parsing with multimodal content extraction
- **Multimodal RAG**: Images, tables, equations, and text
- **Batch Processing**: Process a directory recursively with parallel workers
- **Advanced Querying**: Pure text and multimodal-enhanced queries
- **Multiple Query Modes**: `hybrid`, `local`, `global`, `naive`, `mix`, `bypass`
- **Vision Processing**: Image analysis via an OpenAI vision model
- **Persistent Storage**: Workspace is maintained on disk and reused across runs

## Available Tools

### `process_directory`
Process all matching files in a directory and ingest them into the shared workspace.

**Parameters:**
- `directory_path` (required): Directory containing files to process
- `file_extensions`: Extensions to include. Default: `[".pdf", ".docx", ".pptx", ".txt", ".md", ".ppt", ".rtf"]`
- `recursive`: Recurse into subdirectories. Default: `True`
- `max_workers`: Concurrent processing workers. Default: `4`

Files already present in the workspace are skipped.

### `process_single_document`
Ingest a single document into the shared workspace.

**Parameters:**
- `file_path` (required): Path to the document
- `parse_method`: MinerU parse method — `"auto"`, `"ocr"`, or `"txt"`. Default: `"auto"`

### `check_doc_ingested`
Check whether a given file has already been ingested into the workspace.

**Parameters:**
- `file_path` (required): Path to the document

### `query_workspace`
Pure text query against the indexed workspace.

**Parameters:**
- `query` (required): The question to ask
- `mode`: Query mode — `"hybrid"`, `"local"`, `"global"`, `"naive"`, `"mix"`, or `"bypass"`. Default: `"hybrid"`

### `query_with_multimodal`
Query the workspace with additional multimodal context (tables, equations, etc.) supplied at query time.

**Parameters:**
- `query` (required): The question to ask
- `multimodal_content` (required): List of multimodal content dictionaries
- `mode`: Query mode. Default: `"hybrid"`

**Example `multimodal_content`:**
```json
[
  {
    "type": "table",
    "table_data": "Method,Accuracy\nRAGAnything,95.2%\nBaseline,87.3%",
    "table_caption": "Performance comparison"
  },
  {
    "type": "equation",
    "latex": "P(d|q) = \\frac{P(q|d) \\cdot P(d)}{P(q)}",
    "equation_caption": "Document relevance probability"
  }
]
```

### `get_workspace_info`
Return the active RAGAnything configuration (working directory, output directory, processing flags).

### `clear_all_data`
**Destructive.** Permanently deletes everything under the working directory and output directory, then recreates them empty.

**Parameters:**
- `confirm`: Must be set to `True` to actually perform the wipe. Default: `False`.

## Usage Examples

### 1. Basic Directory Processing
```
process_directory(directory_path="/path/to/documents")
```

### 2. Advanced Directory Processing
```
process_directory(
  directory_path="/path/to/research_papers",
  file_extensions=[".pdf", ".docx"],
  recursive=true,
  max_workers=6
)
```

### 3. Pure Text Query
```
query_workspace(
  query="What are the main findings in these research papers?",
  mode="hybrid"
)
```

### 4. Multimodal Query with Table Data
```
query_with_multimodal(
  query="Compare these results with the document findings",
  multimodal_content=[{
    "type": "table",
    "table_data": "Method,Accuracy,Speed\nRAGAnything,95.2%,120ms\nBaseline,87.3%,180ms",
    "table_caption": "Performance comparison"
  }],
  mode="hybrid"
)
```

### 5. Single Document Processing
```
process_single_document(file_path="/path/to/important_paper.pdf")
```

## Requirements

- **Python 3.12+**
- **NVIDIA GPU with CUDA support.** `pyproject.toml` pins to CUDA 12.8 PyTorch wheels, and document ingestion calls MinerU with `device="cuda:0"` hard-coded. There is no CPU fallback at present.
- **OpenAI API access** (or an OpenAI-compatible endpoint).

## Setup

### 1. Environment Variables

Required:
```bash
export OPENAI_API_KEY="your-openai-api-key-here"
```

Optional:
- `RAG_ANYTHING_WORKING_DIR` — workspace directory (default `~/.rag_anything/shared_workspace`)
- `RAG_ANYTHING_OUTPUT_DIR` — parsed-output directory (default `~/.rag_anything/output`)
- `RAG_ANYTHING_LLM_MODEL` — LLM model (default `gpt-4o-mini`)
- `RAG_ANYTHING_IMAGE_MODEL` — vision model (default `gpt-4.1`)
- `RAG_ANYTHING_EMBEDDING_MODEL` — embedding model (default `text-embedding-3-large`)
- `RAG_ANYTHING_IMAGE_PROCESSING_PROMPT` — system prompt used for image analysis

### 2. Install Dependencies
```bash
uv sync
```

### 3. Run the MCP Server
```bash
python main.py run
```

### 4. Wire It Into an MCP Client

See `mcp_config.example.json` for the shape of the configuration. Copy it into your MCP client's config file (e.g. Claude Desktop's `claude_desktop_config.json`) and replace the placeholder paths and API key with real values. Note that `args` includes the `run` subcommand — without it, the entrypoint prints Typer help and exits.

## CLI Mode

The same operations are exposed as a Typer CLI under the `cli` subcommand:

```bash
python main.py cli process-directory /path/to/documents
python main.py cli process-single-document /path/to/file.pdf --parse-method auto
python main.py cli check-doc /path/to/file.pdf
python main.py cli query "What are the main findings?" --mode hybrid
python main.py cli query-mm "Compare these results" --content @multimodal.json
python main.py cli workspace-info
python main.py cli clear-all-data --yes
```

For `query-mm`, `--content` accepts either a raw JSON string or `@/path/to/file.json` to read from disk.

## Query Modes Explained

- **hybrid**: Combines local and global search (recommended for most use cases)
- **local**: Focuses on local context and entity relationships
- **global**: Provides broader, document-level insights and summaries
- **naive**: Simple keyword-based search without graph reasoning
- **mix**: Combines multiple approaches for comprehensive results
- **bypass**: Direct access without RAG processing

## Multimodal Content Types

The server supports processing and querying with:

- **Images**: Automatic caption generation and visual analysis
- **Tables**: Structure extraction and content analysis
- **Equations**: LaTeX parsing and mathematical reasoning
- **Charts/Graphs**: Visual data interpretation
- **Mixed Content**: Combined analysis of multiple content types

Image, table, and equation processing are enabled by default and configured globally inside the server (not per-call).

## API Configuration

The server uses OpenAI's APIs by default:
- **LLM**: `gpt-4o-mini`
- **Vision**: `gpt-4.1`
- **Embeddings**: `text-embedding-3-large` (3072 dimensions)

Models can be swapped via the `RAG_ANYTHING_*_MODEL` environment variables above. The server does not expose a `base_url` option — pointing at an OpenAI-compatible endpoint depends on whatever the underlying `lightrag` OpenAI client picks up from the environment (typically `OPENAI_BASE_URL`), and is not verified here.

## File Support

Supported file formats include:
- PDF documents
- Microsoft Word (`.docx`)
- PowerPoint presentations (`.pptx`, `.ppt`)
- Text files (`.txt`)
- Markdown files (`.md`)
- Rich text (`.rtf`)
- And more via the raganything library

## Performance Notes

- **Concurrent Processing**: Use `max_workers` on `process_directory` to control parallel document processing
- **Memory Usage**: Large documents with many images may require significant memory
- **API Costs**: Vision processing is more expensive than text processing
- **Storage**: Processed data is stored locally in the shared workspace for efficient re-querying

## Caveats

- **Filename-only deduplication.** The "already ingested" check compares files by base filename (case-insensitive), not by absolute path or content hash. Two distinct files named `paper.pdf` in different directories will be treated as the same document, and the second one will be silently skipped.
- **English-only parsing.** Document ingestion calls MinerU with `lang="en"` hard-coded. Non-English documents will still be ingested, but parsing accuracy on non-Latin scripts may be reduced.
- **Standalone example script.** `examples/test.py` is a self-contained pipeline demo that does not import the MCP server. It uses its own hardcoded paths and is intended as a reference, not a test.

## License

MIT — see [LICENSE](LICENSE).
