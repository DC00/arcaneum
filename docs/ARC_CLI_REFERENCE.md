# Arc CLI Reference

Complete reference for the Arcaneum CLI tool - semantic and full-text search across Qdrant and MeiliSearch vector databases.

## Quick Reference for AI Agents

### Most Common Commands

```bash
# Search operations (use these most often)
arc search semantic "your query" --collection CollectionName
arc search text "your query" --index IndexName

# List available resources
arc collection list
arc models list

# Check system health
arc doctor
arc container status
```

**IMPORTANT**: The subcommand (`semantic` or `text`) must come **BEFORE** the query.
❌ Incorrect: `arc search --collection MyCol "query"`
✅ Correct: `arc search semantic "query" --collection MyCol`

## Command Categories

- [Search](#search-commands) - Query collections
- [Collection](#collection-management) - Manage Qdrant collections
- [Index](#indexing-commands) - Add content to collections
- [Corpus](#corpus-management) - Dual-index management
- [Container](#container-management) - Docker services
- [Models](#model-management) - Embedding models
- [Store](#store-command) - Agent memory storage
- [Config](#configuration) - Cache and settings
- [Doctor](#doctor-command) - System diagnostics

---

## Search Commands

### `arc search semantic`

Vector-based semantic search using embeddings.

**Usage:**

```bash
arc search semantic QUERY [OPTIONS]
```

**Required Options:**

- `--collection TEXT` - Collection to search

**Optional:**

- `--vector-name TEXT` - Vector name to use (auto-detects if not specified)
- `--filter TEXT` - Metadata filter (key=value or JSON)
- `--limit INTEGER` - Number of results (default varies by collection)
- `--offset INTEGER` - Number of results to skip (for pagination)
- `--score-threshold FLOAT` - Minimum score threshold
- `--json` - Output JSON format
- `-v, --verbose` - Verbose output

**Examples:**

```bash
# Basic semantic search
arc search semantic "authentication middleware" --collection MyCode

# With filters and limits
arc search semantic "error handling" --collection MyCode --limit 10 --filter "language=python"

# JSON output for programmatic use
arc search semantic "database queries" --collection MyCode --json

# With score threshold
arc search semantic "security" --collection Docs --score-threshold 0.8
```

### `arc search text`

Keyword-based full-text search using MeiliSearch.

**Usage:**

```bash
arc search text QUERY [OPTIONS]
```

**Required Options:**

- `--index TEXT` - MeiliSearch index to search

**Optional:**

- `--filter TEXT` - Metadata filter (key=value or JSON)
- `--limit INTEGER` - Number of results
- `--offset INTEGER` - Number of results to skip (for pagination)
- `--json` - Output JSON format
- `-v, --verbose` - Verbose output

**Examples:**

```bash
# Basic text search
arc search text "error handling" --index MyCodeIndex

# With pagination
arc search text "authentication" --index Docs --limit 20 --offset 40
```

---

## Collection Management

### `arc collection list`

List all collections.

**Usage:**

```bash
arc collection list [OPTIONS]
```

**Options:**

- `-v, --verbose` - Verbose output
- `--json` - Output JSON format

**Example:**

```bash
arc collection list --json
```

### `arc collection create`

Create a new collection.

**Usage:**

```bash
arc collection create NAME [OPTIONS]
```

**Options:**

- `--model TEXT` - Embedding model (stella, modernbert, bge, jina-code). If not specified, inferred from --type
- `--type [pdf|code|markdown]` - Collection type. Model will be inferred from type if not specified
- `--hnsw-m INTEGER` - HNSW index parameter m
- `--hnsw-ef INTEGER` - HNSW index parameter ef_construct
- `--on-disk` - Store vectors on disk
- `--json` - Output JSON format

**Examples:**

```bash
# Create code collection (uses jina-code model automatically)
arc collection create MyCode --type code

# Create PDF collection with specific model
arc collection create Papers --type pdf --model stella

# Create with custom HNSW parameters
arc collection create LargeDocs --type markdown --hnsw-m 32 --hnsw-ef 200

# On-disk storage for large collections
arc collection create BigData --type pdf --on-disk
```

### `arc collection info`

Show collection details.

**Usage:**

```bash
arc collection info NAME [OPTIONS]
```

**Options:**

- `--json` - Output JSON format

**Example:**

```bash
arc collection info MyCode --json
```

### `arc collection items`

List all indexed files/repos in collection.

**Usage:**

```bash
arc collection items NAME [OPTIONS]
```

**Options:**

- `--json` - Output JSON format

**Example:**

```bash
arc collection items MyCode --json
```

### `arc collection verify`

Verify collection integrity (fsck-like check). Scans the collection to detect items with incomplete chunk sets.

**Usage:**

```bash
arc collection verify NAME [OPTIONS]
```

**Options:**

- `--project TEXT` - Verify specific project identifier only (code collections)
- `-v, --verbose` - Show detailed file-level results
- `--json` - Output JSON format

**Examples:**

```bash
# Verify entire collection
arc collection verify MyCode

# Verify specific project
arc collection verify MyCode --project myrepo#main

# Detailed verification
arc collection verify MyPDFs --verbose
```

### `arc collection delete`

Delete a collection.

**Usage:**

```bash
arc collection delete NAME [OPTIONS]
```

**Options:**

- `--confirm` - Skip confirmation prompt
- `--json` - Output JSON format

**Example:**

```bash
arc collection delete OldCollection --confirm
```

---

## Indexing Commands

### `arc index code`

Index source code repositories.

**Usage:**

```bash
arc index code [PATH] [OPTIONS]
```

**Required Options:**

- `--collection TEXT` - Target collection name

**Input Options:**

- `PATH` - Directory or file to index
- `--from-file TEXT` - Read file paths from list (one per line, or "-" for stdin)

**Performance Options:**

- `--embedding-batch-size INTEGER` - Batch size for embedding generation. Auto-tuned for GPU memory if not
  specified. Larger batches (300-500) improve throughput 10-20%
- `--process-priority [low|normal|high]` - Process scheduling priority (default: normal). Use low for background indexing
- `--not-nice` - Disable process priority reduction for worker processes
- `--no-gpu` - Disable GPU acceleration (use CPU only, 2-3x slower)
- `--no-streaming` - Disable streaming mode (accumulate all embeddings before upload, uses more memory)

**Chunking Options:**

- `--chunk-size INTEGER` - Target chunk size in tokens (default: 400)
- `--chunk-overlap INTEGER` - Overlap between chunks in tokens (default: 20)

**Git Options:**

- `--depth INTEGER` - Git discovery depth

**Other Options:**

- `--force` - Force reindex all projects
- `--verify` - Verify and repair incomplete items after indexing (fsck-like check)
- `-v, --verbose` - Verbose output
- `--debug` - Debug mode (show all library warnings)
- `--profile` - Show pipeline performance profiling (stage breakdown, throughput)
- `--json` - Output JSON format

**Examples:**

```bash
# Index current directory
arc index code . --collection MyCode

# Index with custom chunk size
arc index code ~/projects/myapp --collection MyCode --chunk-size 600

# Background indexing with low priority
arc index code . --collection MyCode --process-priority low

# Force reindex
arc index code . --collection MyCode --force

# Index from file list
find ~/projects -name "*.py" | arc index code --from-file - --collection MyCode
```

### `arc index markdown`

Index markdown files.

**Usage:**

```bash
arc index markdown [PATH] [OPTIONS]
```

**Required Options:**

- `--collection TEXT` - Target collection name

**Input Options:**

- `PATH` - Directory or file to index
- `--from-file TEXT` - Read file paths from list (one per line, or "-" for stdin)
- `--recursive / --no-recursive` - Search subdirectories recursively
- `--exclude TEXT` - Patterns to exclude (e.g., node_modules, .obsidian)

**Performance Options:**

- `--embedding-batch-size INTEGER` - Batch size for embedding generation
- `--process-priority [low|normal|high]` - Process scheduling priority (default: normal)
- `--not-nice` - Disable process priority reduction
- `--no-gpu` - Disable GPU acceleration
- `--offline` - Offline mode (use cached models only, no network)
- `--randomize` - Randomize file processing order (useful for parallel indexing)
- `--no-streaming` - Disable streaming mode

**Chunking Options:**

- `--chunk-size INTEGER` - Target chunk size in tokens
- `--chunk-overlap INTEGER` - Overlap between chunks in tokens

**Other Options:**

- `--qdrant-url TEXT` - Qdrant server URL
- `--force` - Force reindex all files
- `--verify` - Verify collection integrity after indexing
- `-v, --verbose` - Verbose output
- `--debug` - Debug mode
- `--json` - Output JSON format

**Examples:**

```bash
# Index markdown directory
arc index markdown ~/notes --collection MyNotes --recursive

# Exclude patterns
arc index markdown ~/docs --collection Docs --exclude ".obsidian" --exclude "node_modules"

# Offline mode
arc index markdown ~/notes --collection MyNotes --offline

# Random order for parallel indexing
arc index markdown ~/docs --collection Docs --randomize
```

### `arc index pdf`

Index PDF files.

**Usage:**

```bash
arc index pdf [PATH] [OPTIONS]
```

**Required Options:**

- `--collection TEXT` - Target collection name

**Input Options:**

- `PATH` - Directory or file to index
- `--from-file TEXT` - Read file paths from list (one per line, or "-" for stdin)

**PDF Options:**

- `--no-ocr` - Disable OCR (enabled by default for scanned PDFs)
- `--ocr-language TEXT` - OCR language code
- `--ocr-workers INTEGER` - Parallel OCR workers for page processing (default: cpu_count, effective for scanned PDFs only)
- `--normalize-only` - Skip markdown conversion, only normalize whitespace (max 47% token savings)
- `--preserve-images` - Extract images for future multimodal search (slower processing)

**Performance Options:**

- `--embedding-batch-size INTEGER` - Batch size for embedding generation
- `--process-priority [low|normal|high]` - Process scheduling priority
- `--not-nice` - Disable process priority reduction
- `--no-gpu` - Disable GPU acceleration
- `--offline` - Offline mode
- `--randomize` - Randomize file processing order
- `--no-streaming` - Disable streaming mode

**Other Options:**

- `--force` - Force reindex all files
- `--verify` - Verify collection integrity after indexing
- `-v, --verbose` - Verbose output
- `--debug` - Debug mode
- `--json` - Output JSON format

**Examples:**

```bash
# Index PDF directory
arc index pdf ~/papers --collection Research

# With OCR for scanned documents
arc index pdf ~/scanned --collection Archives --ocr-language eng --ocr-workers 8

# Normalize only (faster, saves tokens)
arc index pdf ~/papers --collection Research --normalize-only

# Preserve images for future use
arc index pdf ~/papers --collection Research --preserve-images
```

---

## Corpus Management

Manage dual-index corpora (Qdrant + MeiliSearch).

### `arc corpus create`

Create both collection and index.

**Usage:**

```bash
arc corpus create NAME [OPTIONS]
```

**Required Options:**

- `--type [pdf|code|markdown]` - Corpus type

**Options:**

- `--models TEXT` - Embedding models (comma-separated)
- `--json` - Output JSON format

**Example:**

```bash
arc corpus create MyCorpus --type code --models jina-code
```

### `arc corpus sync`

Index to both vector and full-text.

**Usage:**

```bash
arc corpus sync DIRECTORY [OPTIONS]
```

**Required Options:**

- `--corpus TEXT` - Corpus name

**Options:**

- `--models TEXT` - Embedding models (comma-separated)
- `--file-types TEXT` - File extensions to index (e.g., .py,.md)
- `--json` - Output JSON format

**Example:**

```bash
arc corpus sync ~/project --corpus MyCorpus --file-types .py,.md
```

---

## Container Management

Manage Docker container services (Qdrant, MeiliSearch).

### `arc container start`

Start container services.

**Usage:**

```bash
arc container start
```

### `arc container stop`

Stop container services.

**Usage:**

```bash
arc container stop
```

### `arc container status`

Show container services status.

**Usage:**

```bash
arc container status
```

### `arc container restart`

Restart container services.

**Usage:**

```bash
arc container restart
```

### `arc container logs`

Show container services logs.

**Usage:**

```bash
arc container logs
```

### `arc container reset`

Reset all container data (WARNING: deletes all collections).

**Usage:**

```bash
arc container reset
```

---

## Model Management

### `arc models list`

List available embedding models.

**Usage:**

```bash
arc models list [OPTIONS]
```

**Options:**

- `--json` - Output JSON format

**Example:**

```bash
arc models list --json
```

**Available Models:**

- `stella` - Best for general documents and PDFs
- `jina-code` - Optimized for source code
- `modernbert` - Modern BERT variant
- `bge` - BGE embeddings

---

## Store Command

Store agent-generated content for long-term memory.

**Usage:**

```bash
arc store FILE [OPTIONS]
```

Designed for AI agents to store research, analysis, and synthesized information. Content is persisted to disk for
re-indexing and full-text retrieval, then indexed to Qdrant for semantic search.

**Storage location:** `~/.arcaneum/agent-memory/{collection}/`

**Required Options:**

- `--collection TEXT` - Target collection name

**Options:**

- `--model TEXT` - Embedding model (default: stella for documents)
- `--title TEXT` - Document title
- `--category TEXT` - Document category
- `--tags TEXT` - Comma-separated tags
- `--metadata TEXT` - Additional metadata as JSON
- `--chunk-size INTEGER` - Target chunk size in tokens
- `--chunk-overlap INTEGER` - Overlap between chunks in tokens
- `-v, --verbose` - Verbose output
- `--json` - Output JSON format

**Examples:**

```bash
# Store from stdin (agent workflow)
echo "# Research\n\nFindings..." | arc store - --collection knowledge

# Store from file with metadata
arc store analysis.md --collection research \
    --title "Security Findings" \
    --category security \
    --tags "audit,critical"

# Store with custom chunking
arc store research.md --collection knowledge \
    --chunk-size 800 \
    --chunk-overlap 100
```

**Note:** For indexing existing markdown directories, use `arc index markdown` instead.

---

## Configuration

### `arc config show-cache-dir`

Show cache directory location and sizes.

**Usage:**

```bash
arc config show-cache-dir
```

### `arc config clear-cache`

Clear model cache directory.

**Usage:**

```bash
arc config clear-cache [OPTIONS]
```

**Options:**

- `--confirm` - Confirm deletion

**Example:**

```bash
arc config clear-cache --confirm
```

---

## Doctor Command

Verify Arcaneum setup and prerequisites.

**Usage:**

```bash
arc doctor [OPTIONS]
```

**Options:**

- `-v, --verbose` - Show detailed diagnostic information
- `--json` - Output JSON format

**Example:**

```bash
arc doctor --verbose
```

Checks:

- Container services status
- Model availability
- System dependencies
- Configuration validity

---

## Common Workflows

### Initial Setup

```bash
# Check system health
arc doctor --verbose

# Start services
arc container start

# List available models
arc models list
```

### Creating and Indexing a Collection

```bash
# Create collection
arc collection create MyCode --type code

# Index code
arc index code ~/projects/myapp --collection MyCode

# Verify indexing
arc collection verify MyCode --verbose

# Check collection info
arc collection info MyCode
```

### Searching

```bash
# Semantic search
arc search semantic "authentication logic" --collection MyCode --limit 10

# Full-text search
arc search text "def authenticate" --index MyCodeIndex

# With JSON output for scripts
arc search semantic "error handling" --collection MyCode --json
```

### Agent Memory Storage

```bash
# Store research findings
echo "# Analysis\n\nKey findings..." | arc store - \
    --collection knowledge \
    --title "Analysis Results" \
    --category research

# Later, search stored knowledge
arc search semantic "findings" --collection knowledge
```

### Maintenance

```bash
# Verify collection integrity
arc collection verify MyCode

# Check what's indexed
arc collection items MyCode

# Clear model cache
arc config clear-cache --confirm

# Reset everything (careful!)
arc container reset
```

---

## Output Formats

Most commands support `--json` flag for structured output suitable for programmatic use:

```bash
arc collection list --json
arc search semantic "query" --collection MyCol --json
arc models list --json
```

JSON output includes structured data with consistent schemas for easy parsing by scripts and AI agents.

---

## Performance Tips

1. **GPU Acceleration**: Enabled by default. Use `--no-gpu` only if needed
2. **Batch Size**: Auto-tuned by default. Increase to 300-500 for 10-20% throughput improvement
3. **Process Priority**: Use `--process-priority low` for background indexing
4. **Streaming**: Enabled by default. Disable with `--no-streaming` only if memory is abundant
5. **Randomization**: Use `--randomize` for parallel indexing workflows
6. **Verification**: Run `--verify` after large indexing operations to ensure integrity

---

## Error Handling

- Most commands return non-zero exit codes on failure
- Use `--verbose` for detailed error information
- Use `--debug` to see full library warnings and stack traces
- Run `arc doctor` to diagnose system issues
- Check container status with `arc container status`

---

## Version Information

```bash
arc --version
```
