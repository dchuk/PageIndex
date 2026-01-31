# PageIndex + Repomix + qmd Integration Exploration

## Executive Summary

This document explores a comprehensive dependency knowledge system for AI coding agents, combining three tools:

1. **Repomix** - Generates structured representations of code repositories
2. **PageIndex** - Creates hierarchical tree structures for reasoning-based code navigation
3. **qmd** - Provides semantic and keyword search over documentation, guides, and tutorials

The vision: For each dependency, agents have access to both the **source code** (via PageIndex + Repomix) and **documentation/guides** (via qmd), enabling complete understanding of how to use any library.

## Problem Statement

When AI coding agents work on projects with external dependencies, they often need to understand:
- How to properly use a library's API
- Best practices and common patterns
- Available functions, classes, and their relationships
- Implementation details for complex integrations

Currently, agents either:
1. Query web search (slow, may find outdated info)
2. Read raw source files (overwhelming, lacks structure)
3. Use vector RAG (loses context, can't navigate hierarchically)

**Proposed Solution**: Pre-process dependencies with Repomix, then index with an adapted PageIndex to enable structured, reasoning-based navigation through dependency codebases.

---

## System Overview

### Repomix: What It Does

Repomix consolidates entire repositories into single, AI-optimized files with:

| Feature | Description |
|---------|-------------|
| **File Aggregation** | Combines all repo files into one document |
| **Directory Structure** | Preserves and exposes folder hierarchy |
| **Token Counting** | Calculates tokens for LLM context management |
| **Code Compression** | Optional Tree-sitter extraction (reduces ~70% tokens) |
| **Security Scanning** | Secretlint integration to exclude sensitive data |
| **Multiple Formats** | XML (default), Markdown, JSON, Plain text |

**Repomix Output Structure (XML - Default)**:
```xml
<repomix-output>
  <file_summary>
    <!-- Metadata, token counts, AI usage instructions -->
  </file_summary>

  <directory_structure>
    <!-- Repository tree layout -->
    src/
      components/
        Button.tsx
        Modal.tsx
      utils/
        helpers.ts
  </directory_structure>

  <files>
    <file path="src/components/Button.tsx">
      <!-- Full file content -->
    </file>
    <file path="src/utils/helpers.ts">
      <!-- Full file content -->
    </file>
  </files>

  <instruction>
    <!-- Custom project-specific instructions -->
  </instruction>
</repomix-output>
```

**Repomix Markdown Output**:
```markdown
# File Summary
...metadata...

# Directory Structure
```
src/
  components/
    Button.tsx
```

# Files

## src/components/Button.tsx
```tsx
// file content
```
```

### PageIndex: What It Does

PageIndex creates hierarchical tree structures from documents for reasoning-based RAG:

| Feature | Description |
|---------|-------------|
| **No Vectors** | Uses LLM reasoning instead of embeddings |
| **No Chunking** | Preserves natural document structure |
| **Tree Navigation** | Human-like exploration through hierarchy |
| **Explainable** | Results are traceable to exact locations |
| **Summaries** | Auto-generated summaries at each level |

**PageIndex Output Structure**:
```json
{
  "doc_name": "react-library",
  "doc_description": "A React component library for...",
  "structure": [
    {
      "title": "Components",
      "node_id": "0001",
      "summary": "React UI components including buttons, modals...",
      "start_index": 1,
      "end_index": 50,
      "nodes": [
        {
          "title": "Button",
          "node_id": "0002",
          "summary": "Customizable button component with variants...",
          "start_index": 1,
          "end_index": 15
        }
      ]
    }
  ]
}
```

### qmd: What It Does

qmd is an on-device search engine that indexes markdown documents with hybrid search capabilities:

| Feature | Description |
|---------|-------------|
| **BM25 Full-Text** | Fast keyword-based search |
| **Vector Semantic** | Cosine similarity for concept matching |
| **LLM Re-ranking** | Hybrid query with confidence scoring |
| **Local Processing** | All search via local GGUF models |
| **Collections** | Organize docs by project/topic |
| **MCP Server** | Native AI agent integration |

**Search Modes**:
| Mode | Mechanism | Best For |
|------|-----------|----------|
| `search` | BM25 only | Fast keyword lookup |
| `vsearch` | Vector similarity | Conceptual questions |
| `query` | Hybrid + re-ranking | Highest quality results |

**Key CLI Commands**:
```bash
# Collection management
qmd collection add ./docs --name axios-docs

# Search modes
qmd search "interceptors" -c axios-docs
qmd vsearch "how to handle errors" -c axios-docs
qmd query "retry failed requests" -c axios-docs --json

# Document retrieval
qmd get docs/guide.md:50 -l 100  # Get lines 50-150
qmd multi-get "docs/**/*.md"     # Get multiple files
```

**Why qmd for Documentation**:
- Handles unstructured content (tutorials, guides, blog posts)
- Semantic search finds conceptually related content
- Local models = fast, private, works offline
- MCP server enables direct agent integration
- Score-based relevance (0.0-1.0) helps agents prioritize

---

## Combined Architecture: The Full Picture

The system provides **two complementary search layers**:

| Layer | Tool | Content | Search Type | Best For |
|-------|------|---------|-------------|----------|
| **Code** | PageIndex + Repomix | Source code | Hierarchical navigation | "Where is X implemented?" |
| **Docs** | qmd | Docs, guides, tutorials | Semantic + keyword | "How do I use X?" |

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INDEXING PHASE                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐                                                               │
│  │ Dependency   │                                                               │
│  │ (e.g. axios) │                                                               │
│  └──────┬───────┘                                                               │
│         │                                                                        │
│         ├─────────────────────────────────┐                                     │
│         │                                 │                                     │
│         ▼                                 ▼                                     │
│  ┌─────────────────┐              ┌─────────────────┐                          │
│  │  SOURCE CODE    │              │  DOCUMENTATION  │                          │
│  │                 │              │                 │                          │
│  │  Repository     │              │  - README.md    │                          │
│  │  Source Files   │              │  - /docs/*.md   │                          │
│  └────────┬────────┘              │  - Tutorials    │                          │
│           │                       │  - API guides   │                          │
│           ▼                       │  - Blog posts   │                          │
│  ┌─────────────────┐              └────────┬────────┘                          │
│  │    Repomix      │                       │                                    │
│  │   (Generate)    │                       ▼                                    │
│  └────────┬────────┘              ┌─────────────────┐                          │
│           │                       │      qmd        │                          │
│           ▼                       │  (Index Docs)   │                          │
│  ┌─────────────────┐              └────────┬────────┘                          │
│  │   PageIndex     │                       │                                    │
│  │  (Build Tree)   │                       │                                    │
│  └────────┬────────┘                       │                                    │
│           │                                │                                    │
│           ▼                                ▼                                    │
│  ┌─────────────────┐              ┌─────────────────┐                          │
│  │ pageindex.json  │              │  qmd collection │                          │
│  │ (Code Tree)     │              │  (Doc Index)    │                          │
│  └─────────────────┘              └─────────────────┘                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              RETRIEVAL PHASE                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────┐                                                           │
│  │   Coding Agent   │                                                           │
│  │                  │                                                           │
│  │ "How do I add    │                                                           │
│  │  retry logic     │                                                           │
│  │  with axios?"    │                                                           │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ├──────────────────────────────────┐                                  │
│           │                                  │                                  │
│           ▼                                  ▼                                  │
│  ┌─────────────────────────┐    ┌─────────────────────────┐                    │
│  │   PageIndex Navigator   │    │      qmd Search         │                    │
│  │                         │    │                         │                    │
│  │  - Navigate code tree   │    │  - Semantic doc search  │                    │
│  │  - Find implementations │    │  - Find tutorials       │                    │
│  │  - Get function sigs    │    │  - Get usage examples   │                    │
│  └───────────┬─────────────┘    └───────────┬─────────────┘                    │
│              │                              │                                   │
│              ▼                              ▼                                   │
│  ┌─────────────────────────┐    ┌─────────────────────────┐                    │
│  │  Code Context           │    │  Doc Context            │                    │
│  │                         │    │                         │                    │
│  │  - lib/core/Axios.js    │    │  - "Retry Guide" (0.92) │                    │
│  │  - InterceptorManager   │    │  - "Error Handling"     │                    │
│  │  - Function signatures  │    │  - Code examples        │                    │
│  └───────────┬─────────────┘    └───────────┬─────────────┘                    │
│              │                              │                                   │
│              └──────────────┬───────────────┘                                   │
│                             ▼                                                   │
│              ┌─────────────────────────┐                                        │
│              │   Combined Response     │                                        │
│              │                         │                                        │
│              │  Agent understands:     │                                        │
│              │  - WHERE (code paths)   │                                        │
│              │  - HOW (docs/examples)  │                                        │
│              │  - WHY (guides)         │                                        │
│              └─────────────────────────┘                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Directory Structure for Dependencies

```
project/
├── src/                              # Your project code
├── .dependencies/                    # Dependency knowledge base
│   ├── axios/
│   │   ├── code/
│   │   │   ├── repomix-output.xml    # Raw Repomix output
│   │   │   └── pageindex.json        # Indexed code tree
│   │   └── docs/                     # Documentation collection (qmd indexed)
│   │       ├── README.md             # Main readme
│   │       ├── api-reference.md      # API docs
│   │       ├── guides/
│   │       │   ├── getting-started.md
│   │       │   ├── interceptors.md
│   │       │   └── error-handling.md
│   │       └── tutorials/
│   │           ├── retry-logic.md
│   │           └── authentication.md
│   │
│   ├── react-query/
│   │   ├── code/
│   │   │   ├── repomix-output.xml
│   │   │   └── pageindex.json
│   │   └── docs/
│   │       ├── overview.md
│   │       ├── queries.md
│   │       └── mutations.md
│   │
│   └── zod/
│       ├── code/
│       │   └── ...
│       └── docs/
│           └── ...
│
├── depindex.config.yaml              # Combined configuration
└── .qmd/                             # qmd index cache (auto-generated)
```

---

## Repomix-Specific PageIndex Adapter

### Key Differences from PDF/Markdown Processing

| Aspect | PDF/Markdown | Repomix Output |
|--------|--------------|----------------|
| **Structure Source** | TOC / Headers | Directory tree + file organization |
| **Hierarchy** | Document sections | Folders → Files → Code elements |
| **Content Units** | Pages / Sections | Files / Functions / Classes |
| **Index Reference** | Page numbers | File paths + line numbers |
| **Natural Grouping** | Chapters | Modules / Packages |

### Proposed Tree Structure for Repomix

```json
{
  "doc_name": "axios",
  "doc_type": "repomix",
  "doc_description": "Promise-based HTTP client for browser and node.js",
  "metadata": {
    "version": "1.6.0",
    "total_files": 45,
    "total_tokens": 125000,
    "language": "typescript"
  },
  "structure": [
    {
      "title": "lib",
      "node_id": "0001",
      "type": "directory",
      "summary": "Core library implementation including Axios class, request/response handling, and adapters",
      "path": "lib/",
      "nodes": [
        {
          "title": "axios.js",
          "node_id": "0002",
          "type": "file",
          "path": "lib/axios.js",
          "summary": "Main entry point, creates and exports default Axios instance",
          "line_start": 1,
          "line_end": 85,
          "nodes": [
            {
              "title": "createInstance",
              "node_id": "0003",
              "type": "function",
              "path": "lib/axios.js",
              "line_start": 15,
              "line_end": 45,
              "summary": "Factory function to create new Axios instances with custom config",
              "signature": "function createInstance(defaultConfig)"
            }
          ]
        },
        {
          "title": "core",
          "node_id": "0004",
          "type": "directory",
          "path": "lib/core/",
          "summary": "Core functionality: Axios class, interceptors, request dispatching",
          "nodes": [
            {
              "title": "Axios.js",
              "node_id": "0005",
              "type": "file",
              "path": "lib/core/Axios.js",
              "summary": "Main Axios class with request method and interceptor management"
            },
            {
              "title": "InterceptorManager.js",
              "node_id": "0006",
              "type": "file",
              "path": "lib/core/InterceptorManager.js",
              "summary": "Manages request/response interceptors with use(), eject() methods"
            }
          ]
        }
      ]
    },
    {
      "title": "types",
      "node_id": "0010",
      "type": "directory",
      "path": "types/",
      "summary": "TypeScript type definitions for Axios API",
      "nodes": [...]
    }
  ]
}
```

### Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     REPOMIX OUTPUT PROCESSING                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. PARSE REPOMIX OUTPUT                                                │
│     ├── Extract directory_structure section                             │
│     ├── Extract files section with paths and content                    │
│     └── Extract metadata (token counts, file summary)                   │
│                                                                         │
│  2. BUILD DIRECTORY TREE                                                │
│     ├── Parse directory structure into nested tree                      │
│     ├── Map files to their directory nodes                              │
│     └── Calculate aggregated metrics (tokens, files per dir)            │
│                                                                         │
│  3. CODE STRUCTURE EXTRACTION (per file)                                │
│     ├── Detect language from extension                                  │
│     ├── Extract code elements:                                          │
│     │   ├── Classes / Interfaces / Types                                │
│     │   ├── Functions / Methods                                         │
│     │   ├── Exports / Imports                                           │
│     │   └── Constants / Variables                                       │
│     └── Build sub-tree for significant files                            │
│                                                                         │
│  4. GENERATE SUMMARIES (LLM-powered)                                    │
│     ├── Directory-level: What this module/package does                  │
│     ├── File-level: Purpose and key exports                             │
│     └── Function-level: What it does, parameters, usage                 │
│                                                                         │
│  5. TREE OPTIMIZATION                                                   │
│     ├── Collapse small directories (< threshold tokens)                 │
│     ├── Expand important files (entry points, types)                    │
│     └── Add cross-references (imports, exports)                         │
│                                                                         │
│  6. OUTPUT FINAL INDEX                                                  │
│     └── Save as pageindex.json with full tree structure                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Plan

### Phase 1: Repomix Parser Module

Create `pageindex/page_index_repomix.py`:

```python
# Core functions needed:

def parse_repomix_xml(xml_path: str) -> dict:
    """Parse Repomix XML output into structured data."""

def parse_repomix_markdown(md_path: str) -> dict:
    """Parse Repomix Markdown output into structured data."""

def parse_repomix_json(json_path: str) -> dict:
    """Parse Repomix JSON output into structured data."""

def extract_directory_tree(repomix_data: dict) -> list:
    """Convert directory_structure into tree nodes."""

def extract_file_content(repomix_data: dict) -> dict:
    """Map file paths to their content."""
```

### Phase 2: Code Structure Analyzer

```python
# Language-aware code parsing:

def analyze_code_structure(content: str, language: str) -> list:
    """Extract classes, functions, exports from code."""

def detect_language(file_path: str) -> str:
    """Detect programming language from file extension."""

# Use Tree-sitter or regex patterns for:
SUPPORTED_LANGUAGES = {
    'python': PythonAnalyzer,
    'javascript': JavaScriptAnalyzer,
    'typescript': TypeScriptAnalyzer,
    'go': GoAnalyzer,
    'rust': RustAnalyzer,
    # etc.
}
```

### Phase 3: Tree Builder

```python
def build_repomix_tree(
    directory_tree: list,
    file_contents: dict,
    options: dict
) -> dict:
    """Build PageIndex tree from Repomix data."""

def merge_code_structure_into_tree(
    tree: dict,
    code_analysis: dict
) -> dict:
    """Add code-level nodes to file nodes."""

def optimize_tree_depth(tree: dict, config: dict) -> dict:
    """Balance tree depth vs breadth for navigation."""
```

### Phase 4: Summary Generation

```python
async def generate_summaries_for_repomix(
    structure: dict,
    model: str
) -> dict:
    """Generate LLM summaries at each tree level."""

# Specialized prompts for code:
DIRECTORY_SUMMARY_PROMPT = """
Analyze this directory's contents and provide a 1-2 sentence summary
of what this module/package does and its key functionality.
"""

FILE_SUMMARY_PROMPT = """
Summarize this source file's purpose, key exports, and how it fits
into the larger codebase.
"""

FUNCTION_SUMMARY_PROMPT = """
Describe what this function does, its parameters, return value,
and common usage patterns.
"""
```

### Phase 5: CLI Integration

Update `run_pageindex.py`:

```python
parser.add_argument('--repomix_path', type=str,
    help='Path to Repomix output file (xml, md, or json)')
parser.add_argument('--repomix_style', type=str, default='xml',
    choices=['xml', 'markdown', 'json'],
    help='Repomix output format')
parser.add_argument('--code-depth', type=str, default='file',
    choices=['directory', 'file', 'function'],
    help='Depth of code structure analysis')
```

### Phase 6: Multi-Dependency Management

```python
# New module: pageindex/dependency_manager.py

class DependencyIndex:
    """Manages multiple indexed dependencies."""

    def __init__(self, dependencies_dir: str):
        self.deps_dir = dependencies_dir
        self.indexes = {}

    def add_dependency(self, name: str, repomix_path: str):
        """Index a new dependency from Repomix output."""

    def search(self, query: str, dependencies: list = None) -> list:
        """Search across one or more dependency indexes."""

    def get_context(self, node_id: str, dependency: str) -> str:
        """Get full context for a specific node."""
```

---

## Usage Workflow

### 1. Index a Dependency (Code + Docs)

```bash
# Create dependency directory structure
mkdir -p .dependencies/axios/{code,docs}

# === CODE INDEXING ===
# Generate Repomix output
npx repomix --remote axios/axios --output .dependencies/axios/code/repomix-output.xml

# Index with PageIndex
python run_pageindex.py \
  --repomix_path .dependencies/axios/code/repomix-output.xml \
  --code-depth function \
  --if-add-node-summary yes \
  --output .dependencies/axios/code/pageindex.json

# === DOCS INDEXING ===
# Gather documentation (manual or scripted)
# Option 1: Clone docs from repo
git clone --depth 1 --filter=blob:none --sparse https://github.com/axios/axios
cd axios && git sparse-checkout set docs && mv docs/* ../.dependencies/axios/docs/

# Option 2: Download from documentation site
# (Custom script to scrape/download markdown docs)

# Option 3: Add curated guides/tutorials
# Copy relevant blog posts, tutorials, Stack Overflow answers as markdown

# Index docs with qmd
qmd collection add .dependencies/axios/docs --name axios-docs

# Add context for better search
qmd context add qmd://axios-docs "Axios HTTP client documentation, guides, and tutorials"

# Generate embeddings
qmd embed
```

### 2. Batch Index All Project Dependencies

```bash
#!/bin/bash
# scripts/index_all_deps.sh

DEPS_DIR=".dependencies"

# Read dependencies from package.json
DEPS=$(jq -r '.dependencies | keys[]' package.json)

for dep in $DEPS; do
  echo "Indexing $dep..."

  mkdir -p "$DEPS_DIR/$dep"/{code,docs}

  # Code indexing
  npx repomix --remote "npm:$dep" --output "$DEPS_DIR/$dep/code/repomix-output.xml"
  python run_pageindex.py \
    --repomix_path "$DEPS_DIR/$dep/code/repomix-output.xml" \
    --output "$DEPS_DIR/$dep/code/pageindex.json"

  # Doc indexing (if docs directory exists in package)
  if [ -d "node_modules/$dep/docs" ]; then
    cp -r "node_modules/$dep/docs"/* "$DEPS_DIR/$dep/docs/"
    qmd collection add "$DEPS_DIR/$dep/docs" --name "$dep-docs"
  fi
done

# Generate all embeddings at once
qmd embed
```

### 3. Agent Query Flow - Unified Search

```python
# depindex/unified_search.py
import subprocess
import json
from pageindex import DependencyIndex

class UnifiedDependencySearch:
    """Combined PageIndex + qmd search for dependencies."""

    def __init__(self, deps_dir: str):
        self.deps_dir = deps_dir
        self.code_index = DependencyIndex(deps_dir)

    def search(
        self,
        query: str,
        dependency: str,
        search_code: bool = True,
        search_docs: bool = True
    ) -> dict:
        """Search both code and documentation for a dependency."""

        results = {
            "query": query,
            "dependency": dependency,
            "code_results": None,
            "doc_results": None
        }

        # Search code via PageIndex
        if search_code:
            results["code_results"] = self.code_index.search(
                query=query,
                dependencies=[dependency]
            )

        # Search docs via qmd
        if search_docs:
            results["doc_results"] = self._qmd_search(
                query=query,
                collection=f"{dependency}-docs"
            )

        return results

    def _qmd_search(self, query: str, collection: str) -> list:
        """Execute qmd hybrid search."""
        result = subprocess.run(
            ["qmd", "query", query, "-c", collection, "--json", "-n", "5"],
            capture_output=True,
            text=True
        )
        if result.returncode == 0:
            return json.loads(result.stdout)
        return []

    def get_doc_content(self, doc_id: str) -> str:
        """Retrieve full document content from qmd."""
        result = subprocess.run(
            ["qmd", "get", doc_id, "--full"],
            capture_output=True,
            text=True
        )
        return result.stdout if result.returncode == 0 else ""


# Usage in coding agent:
search = UnifiedDependencySearch(".dependencies")

# Agent query: "How do I implement retry logic with axios?"
results = search.search(
    query="implement retry logic",
    dependency="axios"
)

# Results structure:
# {
#   "query": "implement retry logic",
#   "dependency": "axios",
#   "code_results": {
#     "relevant_nodes": [
#       {
#         "title": "lib/helpers/retryAfter.js",
#         "summary": "Helper for parsing retry-after headers",
#         "path": "lib/helpers/retryAfter.js",
#         "line_start": 1
#       }
#     ]
#   },
#   "doc_results": [
#     {
#       "docid": "#a1b2c3",
#       "score": 0.89,
#       "title": "Implementing Retry Logic",
#       "path": "docs/guides/retry-logic.md",
#       "snippet": "To implement retry logic with axios..."
#     },
#     {
#       "docid": "#d4e5f6",
#       "score": 0.76,
#       "title": "Error Handling Guide",
#       "path": "docs/guides/error-handling.md",
#       "snippet": "When requests fail, you can catch..."
#     }
#   ]
# }
```

### 4. MCP Server Integration

For AI agents that support MCP (Model Context Protocol), both tools can be exposed:

```json
// claude_desktop_config.json or agent MCP config
{
  "mcpServers": {
    "qmd": {
      "command": "qmd",
      "args": ["mcp"]
    },
    "depindex": {
      "command": "python",
      "args": ["-m", "depindex.mcp_server"]
    }
  }
}
```

The agent then has access to:
- `qmd_query` - Search documentation semantically
- `qmd_get` - Retrieve document content
- `depindex_search` - Navigate code trees
- `depindex_get_code` - Get code sections with context

---

## Configuration Options

```yaml
# depindex.config.yaml - Unified configuration

# General settings
dependencies:
  storage_dir: .dependencies
  auto_update: false
  include_dev_deps: false

# === CODE INDEXING (PageIndex + Repomix) ===
code:
  repomix:
    default_style: xml
    compress: false          # Use Tree-sitter compression
    remove_comments: false

  pageindex:
    code_analysis:
      enabled: true
      depth: function        # directory | file | function
      languages:
        - python
        - javascript
        - typescript
        - go
        - rust

    tree_optimization:
      collapse_threshold: 1000   # tokens - collapse small dirs
      expand_entry_points: true  # Always expand index.js, main.py, etc.
      max_depth: 6               # Maximum tree depth

    summaries:
      directory_level: true
      file_level: true
      function_level: false      # Can be expensive for large codebases

# === DOCUMENTATION INDEXING (qmd) ===
docs:
  sources:
    - type: repo_docs         # Clone docs/ from repo
      enabled: true
    - type: readme            # Include README files
      enabled: true
    - type: custom            # Custom curated docs
      paths:
        - ./custom-guides/

  qmd:
    # Search settings
    default_search_mode: query  # search | vsearch | query
    min_score: 0.3              # Minimum relevance score
    max_results: 10

    # Embedding settings
    chunk_size: 800             # Tokens per chunk
    chunk_overlap: 0.15         # 15% overlap

    # Context descriptions (auto-generated or custom)
    auto_context: true          # Generate context from README

# === UNIFIED SEARCH ===
search:
  # Default behavior when agent searches
  default_targets:
    code: true
    docs: true

  # Result merging
  merge_strategy: interleave   # interleave | code_first | docs_first

  # Response format
  include_snippets: true
  max_snippet_length: 500
  include_file_paths: true
```

### qmd Collection Setup

Each dependency's docs are registered as a qmd collection:

```bash
# View all indexed collections
qmd collection list

# Output:
# axios-docs       .dependencies/axios/docs        245 files   125,432 tokens
# react-query-docs .dependencies/react-query/docs  189 files    98,234 tokens
# zod-docs         .dependencies/zod/docs           67 files    45,123 tokens

# Check index health
qmd status

# Update after adding new docs
qmd update

# Re-embed if needed
qmd embed -f
```

---

## Bidirectional Link Index

Instead of relying solely on semantic search to connect documentation to code, we precompute explicit mappings during indexing. This creates a **link index** that enables instant, deterministic navigation between layers.

### The Problem with Pure Search

```
Agent: "How do I use axios interceptors?"

Without links:
  1. Search docs for "interceptors" → finds guide
  2. Search code for "interceptors" → finds files
  3. Hope they're related (fuzzy matching)

With links:
  1. Search docs for "interceptors" → finds guide
  2. Guide has explicit links to: InterceptorManager.js:15, Axios.js:42
  3. Jump directly to implementation (deterministic)
```

### Link Types

| Link Type | Direction | Example |
|-----------|-----------|---------|
| **Symbol Reference** | doc → code | Doc mentions `axios.get()` → links to `lib/core/Axios.js:get()` |
| **File Reference** | doc → code | Doc says "see `lib/helpers/`" → links to directory node |
| **Code Example** | doc → code | Code block uses `interceptors.use()` → links to implementation |
| **API Header** | doc → code | `### axios.create(config)` → links to function definition |
| **Docstring Ref** | code → doc | `// See: docs/guides/interceptors.md` → links to doc |
| **JSDoc @see** | code → doc | `@see {@link docs/api.md#create}` → links to doc section |
| **README Link** | code → doc | Inline `[guide](./docs/guide.md)` → links to doc |

### Link Target Strategy: What Links Point To

Links use a **multi-layer reference** system with PageIndex node_id as the primary anchor:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         LINK REFERENCE LAYERS                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ LAYER 1: PageIndex Node ID (Primary - for retrieval & navigation)      │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  "code_node": "0023"                                                    │    │
│  │                                                                         │    │
│  │  Why primary:                                                           │    │
│  │  ✓ Has pre-computed summary (understand without reading full code)     │    │
│  │  ✓ Has hierarchical context (parent dir, sibling files)                │    │
│  │  ✓ Knows content boundaries (start_index, end_index)                   │    │
│  │  ✓ Enables tree navigation ("show me the parent module")               │    │
│  │  ✓ Works even if original source not available                         │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ LAYER 2: Original Path + Line (Secondary - for display & debugging)    │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  "path": "lib/core/InterceptorManager.js"                               │    │
│  │  "line": 15                                                             │    │
│  │  "symbol": "use"                                                        │    │
│  │                                                                         │    │
│  │  Why include:                                                           │    │
│  │  ✓ Human-readable ("this is at lib/core/InterceptorManager.js:15")     │    │
│  │  ✓ Matches what developers see in IDE                                   │    │
│  │  ✓ Useful for logging, debugging, citations                             │    │
│  │  ✓ Can open directly if original source available                       │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ LAYER 3: Repomix Content Locator (Tertiary - for actual retrieval)     │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  "repomix_file_index": 23        # Which <file> element in XML          │    │
│  │  "content_start_line": 1842      # Line in repomix output               │    │
│  │  "content_end_line": 1920                                               │    │
│  │                                                                         │    │
│  │  Why include:                                                           │    │
│  │  ✓ Fast content retrieval from single consolidated file                 │    │
│  │  ✓ No need to have original source available                            │    │
│  │  ✓ Precise byte/line offsets for extraction                             │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Why PageIndex Node as Primary Anchor:**

| Capability | Direct File Reference | PageIndex Node Reference |
|------------|----------------------|-------------------------|
| Get code content | ✓ | ✓ (via node → repomix locator) |
| Get summary without reading | ✗ | ✓ (node has summary) |
| Navigate to parent module | ✗ | ✓ (tree structure) |
| See related files | ✗ | ✓ (sibling nodes) |
| Works without original source | ✗ | ✓ |
| Understand code in context | Partial | ✓ (hierarchical context) |

**Example: Full Link Reference**

```json
{
  "doc_id": "#a1b2c3",
  "doc_path": "docs/guides/interceptors.md",
  "code_refs": [
    {
      // Layer 1: PageIndex node (PRIMARY)
      "node_id": "0023",
      "node_summary": "Manages request/response interceptor chains with use() and eject() methods",

      // Layer 2: Original source location (for display)
      "path": "lib/core/InterceptorManager.js",
      "symbol": "InterceptorManager",
      "line_start": 1,
      "line_end": 85,

      // Layer 3: Repomix locator (for retrieval)
      "repomix_locator": {
        "file_index": 23,
        "xml_path": "/repomix-output/files/file[@path='lib/core/InterceptorManager.js']",
        "content_lines": [1842, 1920]
      },

      // Link metadata
      "extraction_method": "code_example",
      "confidence": 0.95
    }
  ]
}
```

**Retrieval Flow:**

```
Agent: "Show me the InterceptorManager implementation"

1. Lookup node_id "0023" in PageIndex
   → Get: summary, parent (lib/core/), siblings, token count

2. Agent decides: "Yes, this is what I need, show me the code"

3. Use repomix_locator to extract content:
   → Parse repomix-output.xml
   → Find file at index 23 (or XPath)
   → Extract lines 1842-1920
   → Return actual source code

4. Agent also gets context:
   → Parent: "lib/core/ - Core functionality: Axios class, interceptors..."
   → Siblings: Axios.js, dispatchRequest.js
   → Can navigate: "Show me how Axios.js uses InterceptorManager"
```

**Why Not Link Directly to Repomix Offsets?**

Linking only to repomix file locations would lose important context:

```
❌ Repomix-only link:
{
  "repomix_line": 1842,
  "repomix_end": 1920
}
// Agent gets raw code, but:
// - What does this code do? (no summary)
// - What module is this part of? (no hierarchy)
// - What other files are related? (no siblings)
// - What's the "real" path? (line 1842 is meaningless to a developer)

✓ PageIndex node link:
{
  "node_id": "0023",
  "summary": "Manages request/response interceptor chains...",
  "parent": "0004",  // lib/core/
  "path": "lib/core/InterceptorManager.js"
}
// Agent can:
// - Understand code without reading it (summary)
// - Navigate up/down/sideways (tree)
// - Show human-readable path (lib/core/...)
// - Still get actual content when needed (via locator)
```

### Link Index Structure

```json
{
  "dependency": "axios",
  "version": "1.6.0",
  "generated_at": "2024-01-15T10:30:00Z",

  "symbols": {
    "axios": {
      "node_id": "0001",
      "type": "module",
      "path": "lib/axios.js",
      "repomix_file_index": 0
    },
    "axios.get": {
      "node_id": "0015",
      "type": "method",
      "path": "lib/core/Axios.js",
      "line": 52,
      "repomix_file_index": 5
    },
    "InterceptorManager": {
      "node_id": "0023",
      "type": "class",
      "path": "lib/core/InterceptorManager.js",
      "line": 1,
      "repomix_file_index": 7
    },
    "interceptors.use": {
      "node_id": "0025",
      "type": "method",
      "path": "lib/core/InterceptorManager.js",
      "line": 15,
      "repomix_file_index": 7
    }
  },

  "links": {
    "doc_to_code": [
      {
        "doc_id": "#a1b2c3",
        "doc_path": "docs/guides/interceptors.md",
        "doc_section": "Adding Interceptors",
        "code_refs": [
          {
            // Layer 1: PageIndex node (primary anchor)
            "node_id": "0023",

            // Layer 2: Original source (for display)
            "path": "lib/core/InterceptorManager.js",
            "symbol": "InterceptorManager",
            "line_start": 1,
            "line_end": 85,

            // Layer 3: Repomix locator (for retrieval)
            "repomix_locator": {
              "file_index": 7,
              "content_lines": [1842, 1920]
            },

            // Link metadata
            "extraction_method": "code_example",
            "confidence": 0.95
          },
          {
            "node_id": "0025",
            "path": "lib/core/InterceptorManager.js",
            "symbol": "use",
            "line_start": 15,
            "line_end": 28,
            "repomix_locator": { "file_index": 7, "content_lines": [1856, 1869] },
            "extraction_method": "code_example",
            "confidence": 0.95
          }
        ]
      }
    ],

    "code_to_doc": [
      {
        "node_id": "0023",
        "path": "lib/core/InterceptorManager.js",
        "doc_refs": [
          {
            "doc_id": "#a1b2c3",
            "doc_path": "docs/guides/interceptors.md",
            "section": "Adding Interceptors",
            "qmd_locator": {
              "collection": "axios-docs",
              "line_start": 45,
              "line_end": 120
            }
          },
          {
            "doc_id": "#x7y8z9",
            "doc_path": "docs/api-reference.md",
            "section": "Request Interceptors",
            "qmd_locator": {
              "collection": "axios-docs",
              "line_start": 234,
              "line_end": 289
            }
          }
        ],
        "extraction_method": "reverse_lookup"
      }
    ]
  },

  // Indexes for fast lookup
  "indexes": {
    // node_id → all docs that reference it
    "node_to_docs": {
      "0023": ["#a1b2c3", "#x7y8z9"],
      "0025": ["#a1b2c3"]
    },
    // doc_id → all nodes it references
    "doc_to_nodes": {
      "#a1b2c3": ["0023", "0025", "0026"],
      "#x7y8z9": ["0023"]
    },
    // symbol name → node_id (for quick symbol lookup)
    "symbol_to_node": {
      "InterceptorManager": "0023",
      "interceptors.use": "0025",
      "use": "0025"  // Short form alias
    }
  },

  "coverage": {
    "symbols_total": 57,
    "symbols_with_docs": 45,
    "symbols_without_docs": 12,
    "docs_total": 28,
    "docs_with_code_refs": 23,
    "docs_without_code_refs": 5,
    "total_links": 156,
    "avg_confidence": 0.89
  }
}
```

### Link Extraction Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           LINK EXTRACTION PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ PHASE 1: Build Symbol Table from Code (PageIndex)                       │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  Input: pageindex.json (code tree)                                      │    │
│  │                                                                         │    │
│  │  Extract:                                                               │    │
│  │  ├── Exported functions    → axios.get, axios.post, axios.create       │    │
│  │  ├── Classes               → Axios, InterceptorManager, CancelToken    │    │
│  │  ├── Methods               → interceptors.use, interceptors.eject      │    │
│  │  ├── Types/Interfaces      → AxiosRequestConfig, AxiosResponse         │    │
│  │  ├── Constants             → HttpStatusCode, METHOD_*                  │    │
│  │  └── Module paths          → lib/core, lib/helpers, lib/adapters       │    │
│  │                                                                         │    │
│  │  Output: symbol_table.json                                              │    │
│  │  {                                                                      │    │
│  │    "axios.create": { node: "0012", path: "lib/axios.js", line: 15 },   │    │
│  │    "InterceptorManager": { node: "0023", path: "lib/core/..." },       │    │
│  │    ...                                                                  │    │
│  │  }                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ PHASE 2: Extract References from Documentation                          │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  For each doc in qmd collection:                                        │    │
│  │                                                                         │    │
│  │  A. API Headers (Markdown)                                              │    │
│  │     Pattern: ### `axios.create(config)` or ## axios.get()               │    │
│  │     → Match against symbol_table → Link to code node                    │    │
│  │                                                                         │    │
│  │  B. Inline Code References                                              │    │
│  │     Pattern: `interceptors.use()` or `AxiosInstance`                    │    │
│  │     → Match against symbol_table → Link to code node                    │    │
│  │                                                                         │    │
│  │  C. Code Block Analysis                                                 │    │
│  │     ```javascript                                                       │    │
│  │     axios.interceptors.request.use(config => {                          │    │
│  │       // ...                                                            │    │
│  │     });                                                                  │    │
│  │     ```                                                                  │    │
│  │     → Parse AST → Extract function calls → Match symbols                │    │
│  │     → Links: interceptors, request, use                                 │    │
│  │                                                                         │    │
│  │  D. File Path References                                                │    │
│  │     Pattern: "see `lib/core/Axios.js`" or "in the helpers directory"   │    │
│  │     → Match against code tree paths → Link to directory/file node       │    │
│  │                                                                         │    │
│  │  E. Explicit Doc Links                                                  │    │
│  │     Pattern: [Interceptors Guide](./interceptors.md)                    │    │
│  │     → Cross-reference within docs                                       │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ PHASE 3: Extract References from Code                                   │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  For each file in repomix output:                                       │    │
│  │                                                                         │    │
│  │  A. JSDoc @see / @link tags                                             │    │
│  │     /**                                                                 │    │
│  │      * @see {@link docs/guides/interceptors.md}                         │    │
│  │      */                                                                 │    │
│  │     → Extract doc path → Link to qmd doc                                │    │
│  │                                                                         │    │
│  │  B. Comment References                                                  │    │
│  │     // See: docs/api.md for usage                                       │    │
│  │     # Reference: docs/configuration.md                                  │    │
│  │     → Pattern match → Link to qmd doc                                   │    │
│  │                                                                         │    │
│  │  C. README/Doc Links in Code                                            │    │
│  │     Look for markdown links in comments, docstrings                     │    │
│  │     → Resolve relative paths → Link to qmd doc                          │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ PHASE 4: Build Bidirectional Index                                      │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  1. Merge all extracted links                                           │    │
│  │  2. Deduplicate (same doc→code pair from multiple extractions)          │    │
│  │  3. Compute reverse mappings (code→doc from doc→code)                   │    │
│  │  4. Calculate confidence scores based on extraction method              │    │
│  │  5. Generate coverage statistics                                        │    │
│  │                                                                         │    │
│  │  Output: link_index.json                                                │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Link Extraction Patterns

```python
# depindex/link_extractor.py

import re
from typing import List, Dict
import ast

class LinkExtractor:
    """Extract doc↔code links from documentation and source files."""

    def __init__(self, symbol_table: Dict, doc_index: Dict):
        self.symbols = symbol_table
        self.docs = doc_index

    # ═══════════════════════════════════════════════════════════════════════
    # DOC → CODE EXTRACTION
    # ═══════════════════════════════════════════════════════════════════════

    def extract_api_headers(self, markdown: str) -> List[Dict]:
        """Extract API headers like ### `axios.create(config)`"""
        links = []
        # Match: ## axios.get() or ### `axios.create(config)`
        pattern = r'^#{2,4}\s+`?(\w+(?:\.\w+)*)\s*\([^)]*\)`?'
        for match in re.finditer(pattern, markdown, re.MULTILINE):
            symbol = match.group(1)
            if symbol in self.symbols:
                links.append({
                    "symbol": symbol,
                    "code_node": self.symbols[symbol]["node_id"],
                    "method": "api_header",
                    "confidence": 0.95
                })
        return links

    def extract_inline_code_refs(self, markdown: str) -> List[Dict]:
        """Extract inline code like `interceptors.use()`"""
        links = []
        # Match backtick-wrapped identifiers
        pattern = r'`(\w+(?:\.\w+)*)\(?[^`]*\)?`'
        for match in re.finditer(pattern, markdown):
            symbol = match.group(1).rstrip('()')
            if symbol in self.symbols:
                links.append({
                    "symbol": symbol,
                    "code_node": self.symbols[symbol]["node_id"],
                    "method": "inline_code",
                    "confidence": 0.85
                })
        return links

    def extract_code_block_refs(self, markdown: str) -> List[Dict]:
        """Parse code examples and extract function calls."""
        links = []
        # Extract code blocks
        code_pattern = r'```(?:javascript|typescript|js|ts)?\n(.*?)```'
        for match in re.finditer(code_pattern, markdown, re.DOTALL):
            code = match.group(1)
            # Extract function calls (simplified - use real parser for production)
            call_pattern = r'(\w+(?:\.\w+)*)\s*\('
            for call_match in re.finditer(call_pattern, code):
                symbol = call_match.group(1)
                # Normalize: axios.interceptors.request.use → interceptors.use
                normalized = self._normalize_symbol(symbol)
                if normalized in self.symbols:
                    links.append({
                        "symbol": normalized,
                        "code_node": self.symbols[normalized]["node_id"],
                        "method": "code_example",
                        "confidence": 0.90,
                        "context": code[:100]  # First 100 chars for context
                    })
        return links

    def extract_file_refs(self, markdown: str) -> List[Dict]:
        """Extract file/directory references like 'see lib/core/'"""
        links = []
        pattern = r'`((?:lib|src|packages)/[\w/.-]+)`'
        for match in re.finditer(pattern, markdown):
            path = match.group(1)
            # Match against code tree paths
            node = self._find_node_by_path(path)
            if node:
                links.append({
                    "path": path,
                    "code_node": node["node_id"],
                    "method": "file_reference",
                    "confidence": 0.95
                })
        return links

    # ═══════════════════════════════════════════════════════════════════════
    # CODE → DOC EXTRACTION
    # ═══════════════════════════════════════════════════════════════════════

    def extract_jsdoc_refs(self, code: str) -> List[Dict]:
        """Extract @see and @link references from JSDoc comments."""
        links = []
        # Match @see {@link path} or @see path
        pattern = r'@see\s+(?:\{@link\s+)?([^\s}]+)'
        for match in re.finditer(pattern, code):
            doc_path = match.group(1)
            doc = self._find_doc_by_path(doc_path)
            if doc:
                links.append({
                    "doc_path": doc_path,
                    "doc_id": doc["doc_id"],
                    "method": "jsdoc_see",
                    "confidence": 0.98
                })
        return links

    def extract_comment_refs(self, code: str) -> List[Dict]:
        """Extract doc references from comments."""
        links = []
        patterns = [
            r'//\s*[Ss]ee:?\s*(docs?/[\w/.-]+\.md)',
            r'//\s*[Rr]ef(?:erence)?:?\s*(docs?/[\w/.-]+\.md)',
            r'#\s*[Ss]ee:?\s*(docs?/[\w/.-]+\.md)',  # Python comments
        ]
        for pattern in patterns:
            for match in re.finditer(pattern, code):
                doc_path = match.group(1)
                doc = self._find_doc_by_path(doc_path)
                if doc:
                    links.append({
                        "doc_path": doc_path,
                        "doc_id": doc["doc_id"],
                        "method": "comment_reference",
                        "confidence": 0.90
                    })
        return links


# Usage:
extractor = LinkExtractor(symbol_table, doc_index)

# Process all docs
for doc in docs:
    links = []
    links.extend(extractor.extract_api_headers(doc.content))
    links.extend(extractor.extract_inline_code_refs(doc.content))
    links.extend(extractor.extract_code_block_refs(doc.content))
    links.extend(extractor.extract_file_refs(doc.content))
    doc_to_code_links[doc.id] = links

# Process all code files
for file in code_files:
    links = []
    links.extend(extractor.extract_jsdoc_refs(file.content))
    links.extend(extractor.extract_comment_refs(file.content))
    code_to_doc_links[file.node_id] = links
```

### Using Links at Query Time

```python
class LinkedDependencySearch:
    """Search with explicit link traversal."""

    def __init__(self, deps_dir: str):
        self.code_index = PageIndex(deps_dir)
        self.link_index = LinkIndex(deps_dir)

    def search_with_links(self, query: str, dependency: str) -> dict:
        """Search docs, then follow links to code."""

        # Step 1: Search documentation
        doc_results = qmd_search(query, f"{dependency}-docs")

        # Step 2: For each doc result, get linked code
        enriched_results = []
        for doc in doc_results:
            linked_code = self.link_index.get_code_for_doc(doc["doc_id"])

            enriched_results.append({
                "doc": doc,
                "linked_code": [
                    {
                        "node": self.code_index.get_node(link["code_node"]),
                        "symbol": link["symbol"],
                        "confidence": link["confidence"],
                        "link_type": link["method"]
                    }
                    for link in linked_code
                ]
            })

        return {
            "query": query,
            "results": enriched_results,
            # Also include coverage info
            "unlinked_code_mentions": self._find_unlinked_symbols(doc_results)
        }

    def get_docs_for_symbol(self, symbol: str, dependency: str) -> list:
        """Reverse lookup: find all docs that mention a symbol."""
        code_node = self.code_index.find_by_symbol(symbol)
        if not code_node:
            return []

        return self.link_index.get_docs_for_code(code_node["node_id"])


# Agent interaction with links:

search = LinkedDependencySearch(".dependencies")

# Query: "How do I add request interceptors?"
results = search.search_with_links("add request interceptors", "axios")

# Results now include explicit links:
# {
#   "results": [
#     {
#       "doc": {
#         "doc_id": "#a1b2",
#         "title": "Interceptors Guide",
#         "score": 0.92
#       },
#       "linked_code": [
#         {
#           "node": { "title": "InterceptorManager", "path": "lib/core/..." },
#           "symbol": "interceptors.use",
#           "confidence": 0.95,
#           "link_type": "code_example"
#         },
#         {
#           "node": { "title": "Axios.js", "path": "lib/core/Axios.js" },
#           "symbol": "Axios.interceptors",
#           "confidence": 0.90,
#           "link_type": "inline_code"
#         }
#       ]
#     }
#   ]
# }

# Agent can now:
# 1. Read the doc (from qmd)
# 2. DIRECTLY jump to linked code (no fuzzy search needed)
# 3. Get implementation details with exact line numbers
```

### Link Quality Indicators

```yaml
# Link confidence scoring:

confidence_weights:
  # Explicit references (high confidence)
  jsdoc_see: 0.98          # @see tag is intentional
  api_header: 0.95         # Markdown API header matches symbol
  file_reference: 0.95     # Explicit file path mention

  # Derived references (medium confidence)
  code_example: 0.90       # Function call in code block
  comment_reference: 0.90  # Comment mentions doc path
  inline_code: 0.85        # `symbol` in text

  # Inferred references (lower confidence)
  symbol_mention: 0.70     # Plain text mentions symbol name
  fuzzy_match: 0.50        # Similar names, not exact

# Coverage thresholds:
coverage_targets:
  symbols_documented: 0.80  # 80% of exports should have doc links
  docs_linked: 0.90         # 90% of docs should link to code
  warn_orphan_docs: true    # Warn about docs with no code refs
  warn_undocumented: true   # Warn about exports with no doc refs
```

### Directory Structure with Links

```
.dependencies/axios/
├── code/
│   ├── repomix-output.xml
│   └── pageindex.json
├── docs/
│   ├── guides/
│   └── api-reference.md
├── link_index.json          # ← NEW: Bidirectional links
└── symbol_table.json        # ← NEW: Exported symbols registry
```

### For Coding Agents

1. **Complete Understanding**: Both code structure AND usage guidance
2. **Structured Navigation**: PageIndex tree for "where is it?"
3. **Semantic Search**: qmd for "how do I use it?"
4. **Context-Aware Retrieval**: Summaries help agents decide where to look
5. **Efficient Token Usage**: Progressive disclosure, only load what's needed
6. **Explainable Results**: Trace exactly where information came from
7. **Offline Capable**: All processing is local, no API calls during search
8. **Version Consistent**: Indexed docs match the dependency version you're using

### Compared to Alternatives

| Approach | Code Understanding | Usage Guidance | Speed | Reliability |
|----------|-------------------|----------------|-------|-------------|
| **Raw File Reading** | Complete | None | Slow | High |
| **Vector RAG** | Partial (chunks) | Partial | Fast | Medium |
| **Web Search** | Links only | Variable | Slow | Low (outdated) |
| **LLM Knowledge** | Outdated | Generic | Fast | Low (hallucinations) |
| **PageIndex + qmd** | Hierarchical | Semantic search | Fast | High |

### Why Two Search Layers?

| Question Type | Best Tool | Why |
|--------------|-----------|-----|
| "Where is X implemented?" | PageIndex | Hierarchical code navigation |
| "How do I use X?" | qmd | Semantic doc search |
| "What does function Y do?" | PageIndex | Code + summaries |
| "Best practices for X?" | qmd | Guides and tutorials |
| "What parameters does Z take?" | PageIndex | Function signatures |
| "Common patterns with X?" | qmd | Examples in docs |

### Example Agent Interaction

```
Agent Task: Implement request retry logic with exponential backoff using axios.

═══════════════════════════════════════════════════════════════════════════════
STEP 1: Search Documentation (qmd)
═══════════════════════════════════════════════════════════════════════════════

$ qmd query "retry logic exponential backoff" -c axios-docs --json

Results:
┌─────────┬───────┬─────────────────────────────────────────────────────────┐
│ Score   │ DocID │ Title & Path                                            │
├─────────┼───────┼─────────────────────────────────────────────────────────┤
│ 0.92    │ #a1b2 │ "Implementing Retry Logic"                              │
│         │       │ docs/guides/retry-logic.md                              │
│         │       │ Snippet: "Use axios-retry for automatic retries with    │
│         │       │ exponential backoff. Configure with retries: 3..."      │
├─────────┼───────┼─────────────────────────────────────────────────────────┤
│ 0.85    │ #c3d4 │ "Error Handling Best Practices"                         │
│         │       │ docs/guides/error-handling.md                           │
│         │       │ Snippet: "For transient failures, implement retry..."   │
├─────────┼───────┼─────────────────────────────────────────────────────────┤
│ 0.71    │ #e5f6 │ "Interceptors Guide"                                    │
│         │       │ docs/guides/interceptors.md                             │
│         │       │ Snippet: "Response interceptors can catch errors..."    │
└─────────┴───────┴─────────────────────────────────────────────────────────┘

Agent reads top doc:
$ qmd get #a1b2 --full

→ Learns: Use axios-retry plugin or implement via interceptors
→ Learns: exponentialDelay helper exists
→ Learns: Common pattern with config options

═══════════════════════════════════════════════════════════════════════════════
STEP 2: Navigate Code (PageIndex)
═══════════════════════════════════════════════════════════════════════════════

Agent navigates code tree for implementation details:

axios (pageindex.json)
└── lib/
    ├── core/
    │   └── Axios.js
    │       └── request() ← "Main request method, calls dispatchRequest"
    ├── helpers/
    │   └── retryAfter.js ← "Parses Retry-After header from responses"
    └── defaults/
        └── index.js ← "Default config including timeout settings"

Agent retrieves code:
→ Gets: retryAfter.js implementation (parsing logic)
→ Gets: How interceptors are called in request flow
→ Gets: Default timeout/retry configuration structure

═══════════════════════════════════════════════════════════════════════════════
STEP 3: Synthesize and Implement
═══════════════════════════════════════════════════════════════════════════════

Agent now has:
✓ HOW: Tutorial showing axios-retry pattern (from docs)
✓ WHERE: Interceptor hooks in lib/core/Axios.js (from code)
✓ WHAT: retryAfter helper for header parsing (from code)
✓ WHY: Best practices for transient errors (from docs)

Agent implements:
┌────────────────────────────────────────────────────────────────────────────┐
│ import axios from 'axios';                                                  │
│ import axiosRetry from 'axios-retry';                                       │
│                                                                             │
│ // Based on docs/guides/retry-logic.md pattern                              │
│ axiosRetry(axios, {                                                         │
│   retries: 3,                                                               │
│   retryDelay: axiosRetry.exponentialDelay,                                  │
│   retryCondition: (error) => {                                              │
│     // From docs: retry on network errors and 5xx                           │
│     return axiosRetry.isNetworkOrIdempotentRequestError(error)              │
│       || error.response?.status >= 500;                                     │
│   }                                                                         │
│ });                                                                         │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Alternative Architecture: Cross-Docpack Meta-Tree

Instead of PageIndex per-dependency, we can use **qmd as the unified search layer** across ALL documentation, with **PageIndex as a meta-navigation tree** that helps agents decide which docpacks to search.

### The Insight

| Current Approach | Meta-Tree Approach |
|-----------------|-------------------|
| PageIndex per dependency (code) | qmd indexes ALL docs across ALL dependencies |
| qmd per dependency (docs) | PageIndex creates ONE tree across all docpacks |
| Agent searches one dependency at a time | Agent navigates to find the RIGHT dependency |

**Key Question the Meta-Tree Answers**: "Which dependency should I look at for this task?"

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      CROSS-DOCPACK META-TREE ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                        DOCUMENTATION LAYER (qmd)                         │    │
│  │                     Single index across ALL docpacks                     │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  qmd collection: "project-deps"                                         │    │
│  │                                                                         │    │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │    │
│  │  │   axios     │ │ react-query │ │     zod     │ │   lodash    │       │    │
│  │  │   docs/     │ │   docs/     │ │   docs/     │ │   docs/     │       │    │
│  │  │             │ │             │ │             │ │             │       │    │
│  │  │ • guides    │ │ • queries   │ │ • schemas   │ │ • arrays    │       │    │
│  │  │ • api-ref   │ │ • mutations │ │ • inference │ │ • objects   │       │    │
│  │  │ • examples  │ │ • caching   │ │ • errors    │ │ • functions │       │    │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘       │    │
│  │         │               │               │               │              │    │
│  │         └───────────────┴───────────────┴───────────────┘              │    │
│  │                                   │                                     │    │
│  │                         Unified semantic search                         │    │
│  │                    "How do I validate API responses?"                   │    │
│  │                                   │                                     │    │
│  │                    Returns results from: zod, axios, react-query        │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                       ▲                                          │
│                                       │                                          │
│                              Bidirectional links                                 │
│                                       │                                          │
│                                       ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                      META-NAVIGATION LAYER (PageIndex)                   │    │
│  │                  Hierarchical tree across ALL docpacks                   │    │
│  ├─────────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  docpack_tree.json                                                      │    │
│  │  │                                                                      │    │
│  │  ├── HTTP & Networking                                                  │    │
│  │  │   ├── axios [summary: "Promise-based HTTP client..."]               │    │
│  │  │   │   ├── Request Configuration                                      │    │
│  │  │   │   ├── Response Handling                                          │    │
│  │  │   │   ├── Interceptors                                               │    │
│  │  │   │   └── Error Handling                                             │    │
│  │  │   └── ky [summary: "Tiny HTTP client based on fetch..."]            │    │
│  │  │                                                                      │    │
│  │  ├── Data Fetching & Caching                                            │    │
│  │  │   ├── react-query [summary: "Async state management..."]            │    │
│  │  │   │   ├── Queries                                                    │    │
│  │  │   │   ├── Mutations                                                  │    │
│  │  │   │   └── Cache Invalidation                                         │    │
│  │  │   └── swr [summary: "React hooks for data fetching..."]             │    │
│  │  │                                                                      │    │
│  │  ├── Validation & Schemas                                               │    │
│  │  │   ├── zod [summary: "TypeScript-first schema validation..."]        │    │
│  │  │   │   ├── Basic Types                                                │    │
│  │  │   │   ├── Object Schemas                                             │    │
│  │  │   │   ├── Transformations                                            │    │
│  │  │   │   └── Error Handling                                             │    │
│  │  │   └── yup [summary: "Schema builder for validation..."]             │    │
│  │  │                                                                      │    │
│  │  └── Utilities                                                          │    │
│  │      ├── lodash [summary: "Utility library for arrays..."]             │    │
│  │      └── date-fns [summary: "Modern date utility library..."]          │    │
│  │                                                                         │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Meta-Tree Structure

```json
{
  "tree_name": "project-dependencies",
  "generated_at": "2024-01-15T10:30:00Z",
  "total_docpacks": 12,
  "total_docs": 847,

  "structure": [
    {
      "title": "HTTP & Networking",
      "node_id": "cat-001",
      "type": "category",
      "summary": "Libraries for making HTTP requests, handling responses, and managing network communication",
      "nodes": [
        {
          "title": "axios",
          "node_id": "dep-axios",
          "type": "docpack",
          "summary": "Promise-based HTTP client for browser and node.js with interceptors, transforms, and cancellation",
          "metadata": {
            "version": "1.6.0",
            "doc_count": 23,
            "qmd_collection": "project-deps",
            "qmd_path_prefix": "axios/",
            "primary_use_cases": [
              "REST API calls",
              "Request/response interceptors",
              "File uploads",
              "Request cancellation"
            ],
            "related_deps": ["react-query", "ky"]
          },
          "nodes": [
            {
              "title": "Request Configuration",
              "node_id": "axios-config",
              "type": "topic",
              "summary": "How to configure requests: headers, params, timeout, auth",
              "qmd_docs": ["axios/guides/config.md", "axios/api/request-config.md"],
              "keywords": ["headers", "params", "timeout", "baseURL", "auth"]
            },
            {
              "title": "Interceptors",
              "node_id": "axios-interceptors",
              "type": "topic",
              "summary": "Request and response interceptors for logging, auth, error handling",
              "qmd_docs": ["axios/guides/interceptors.md"],
              "keywords": ["interceptors", "middleware", "request transform", "response transform"],
              "code_links": [
                {"node_id": "0023", "symbol": "InterceptorManager"}
              ]
            },
            {
              "title": "Error Handling",
              "node_id": "axios-errors",
              "type": "topic",
              "summary": "Handling network errors, status codes, and response validation",
              "qmd_docs": ["axios/guides/error-handling.md"],
              "keywords": ["errors", "catch", "status codes", "network errors"]
            }
          ]
        },
        {
          "title": "ky",
          "node_id": "dep-ky",
          "type": "docpack",
          "summary": "Tiny and elegant HTTP client based on browser Fetch API with retries and hooks",
          "metadata": {
            "version": "1.2.0",
            "doc_count": 8,
            "qmd_collection": "project-deps",
            "qmd_path_prefix": "ky/"
          },
          "nodes": [...]
        }
      ]
    },
    {
      "title": "Data Fetching & State",
      "node_id": "cat-002",
      "type": "category",
      "summary": "Libraries for fetching data and managing async state in applications",
      "nodes": [
        {
          "title": "react-query",
          "node_id": "dep-react-query",
          "type": "docpack",
          "summary": "Powerful async state management for React with caching, background updates, and optimistic updates",
          "metadata": {
            "version": "5.0.0",
            "doc_count": 45,
            "primary_use_cases": [
              "Server state management",
              "Data caching",
              "Background refetching",
              "Optimistic updates"
            ],
            "related_deps": ["axios", "zod"]
          },
          "nodes": [...]
        }
      ]
    },
    {
      "title": "Validation & Schemas",
      "node_id": "cat-003",
      "type": "category",
      "summary": "Libraries for runtime validation, type checking, and schema definition",
      "nodes": [
        {
          "title": "zod",
          "node_id": "dep-zod",
          "type": "docpack",
          "summary": "TypeScript-first schema validation with static type inference",
          "metadata": {
            "version": "3.22.0",
            "doc_count": 32,
            "primary_use_cases": [
              "API response validation",
              "Form validation",
              "Environment variable validation",
              "Type inference"
            ],
            "related_deps": ["react-hook-form", "axios"]
          },
          "nodes": [...]
        }
      ]
    }
  ],

  // Cross-cutting concerns that span multiple docpacks
  "cross_references": {
    "api_validation": {
      "description": "Validating API responses",
      "involves": ["axios", "zod", "react-query"],
      "example_flow": "axios fetches → zod validates → react-query caches"
    },
    "error_handling": {
      "description": "Handling errors across the stack",
      "involves": ["axios", "react-query", "zod"],
      "docs": ["axios/guides/error-handling.md", "react-query/guides/errors.md"]
    }
  }
}
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            AGENT QUERY FLOW                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Agent: "I need to validate API responses and cache the results"                │
│                                                                                  │
│  ═══════════════════════════════════════════════════════════════════════════    │
│  STEP 1: Navigate Meta-Tree (PageIndex)                                         │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                  │
│  Agent reads docpack_tree.json summaries:                                       │
│                                                                                  │
│  → "Validation & Schemas" category                                              │
│    → zod: "TypeScript-first schema validation..."                               │
│      → primary_use_cases: ["API response validation", ...]  ✓ MATCH             │
│                                                                                  │
│  → "Data Fetching & State" category                                             │
│    → react-query: "...caching, background updates..."                           │
│      → primary_use_cases: ["Data caching", ...]  ✓ MATCH                        │
│                                                                                  │
│  → cross_references.api_validation:                                             │
│    → "axios fetches → zod validates → react-query caches"  ✓ EXACT MATCH        │
│                                                                                  │
│  Agent decides: Search zod + react-query docs                                   │
│                                                                                  │
│  ═══════════════════════════════════════════════════════════════════════════    │
│  STEP 2: Semantic Search (qmd)                                                  │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                  │
│  $ qmd query "validate API response and cache" -c project-deps                  │
│        --filter-path "zod/*,react-query/*"                                      │
│                                                                                  │
│  Results:                                                                        │
│  ┌─────────┬──────────────────────────────────────────────────────────────┐     │
│  │ 0.94    │ zod/guides/api-validation.md                                 │     │
│  │         │ "Validating API responses with Zod schemas..."              │     │
│  ├─────────┼──────────────────────────────────────────────────────────────┤     │
│  │ 0.91    │ react-query/guides/query-functions.md                       │     │
│  │         │ "Fetching and caching data with useQuery..."                │     │
│  ├─────────┼──────────────────────────────────────────────────────────────┤     │
│  │ 0.87    │ react-query/guides/typescript.md                            │     │
│  │         │ "Type-safe queries with Zod integration..."                 │     │
│  └─────────┴──────────────────────────────────────────────────────────────┘     │
│                                                                                  │
│  ═══════════════════════════════════════════════════════════════════════════    │
│  STEP 3: Follow Links to Code (if needed)                                       │
│  ═══════════════════════════════════════════════════════════════════════════    │
│                                                                                  │
│  From link_index.json:                                                          │
│  → zod/guides/api-validation.md links to: z.object(), z.parse()                │
│  → Get code implementations from repomix outputs                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Directory Structure

```
project/
├── src/
├── .dependencies/
│   │
│   ├── docpack_tree.json          # ← PageIndex meta-tree (ONE file)
│   ├── link_index.json            # ← Cross-docpack links
│   │
│   ├── axios/
│   │   ├── code/
│   │   │   ├── repomix-output.xml
│   │   │   └── pageindex.json     # Code-level tree (optional)
│   │   └── docs/                  # ← Indexed by qmd
│   │       ├── guides/
│   │       └── api-reference.md
│   │
│   ├── react-query/
│   │   ├── code/
│   │   └── docs/                  # ← Indexed by qmd
│   │
│   └── zod/
│       ├── code/
│       └── docs/                  # ← Indexed by qmd
│
└── .qmd/
    └── index.sqlite               # ← Single qmd index for ALL docs
```

### Benefits of Meta-Tree Approach

| Benefit | Description |
|---------|-------------|
| **Cross-Dependency Discovery** | "What libraries help with X?" - navigate categories |
| **Unified Search** | One qmd search across ALL documentation |
| **Relationship Awareness** | Cross-references show how deps work together |
| **Efficient Navigation** | Read summaries before diving into specific docs |
| **Scalable** | Add new docpacks without restructuring |

### When to Use Each Approach

| Scenario | Recommended Approach |
|----------|---------------------|
| Single dependency deep-dive | Per-dependency PageIndex |
| "Which library for X?" | Meta-tree navigation |
| Cross-library patterns | Meta-tree cross_references |
| Specific API lookup | Direct qmd search |
| Implementation details | Per-dependency code PageIndex |

### Meta-Tree Generation

```python
# depindex/meta_tree.py

class MetaTreeBuilder:
    """Build PageIndex tree across all docpacks."""

    def __init__(self, deps_dir: str):
        self.deps_dir = deps_dir
        self.docpacks = []

    def scan_docpacks(self) -> list:
        """Find all docpack directories."""
        # Scan .dependencies/ for subdirs with docs/
        pass

    def categorize_docpacks(self, docpacks: list) -> dict:
        """Use LLM to categorize docpacks by purpose."""
        prompt = """
        Categorize these libraries into logical groups:
        {docpacks}

        Categories might include:
        - HTTP & Networking
        - Data Fetching & State
        - Validation & Schemas
        - UI Components
        - Utilities
        - Testing
        ...

        Return JSON with categories and which libraries belong to each.
        """
        pass

    def generate_docpack_summary(self, docpack: str) -> dict:
        """Generate summary and use cases for a docpack."""
        # Read README, key docs
        # Use LLM to summarize purpose and use cases
        pass

    def extract_topics(self, docpack: str) -> list:
        """Extract main topics/sections from docpack."""
        # Parse doc structure
        # Group by theme
        # Generate topic summaries
        pass

    def find_cross_references(self, docpacks: list) -> dict:
        """Find relationships between docpacks."""
        # Look for:
        # - Mentions of other deps in docs
        # - Common patterns (fetch → validate → cache)
        # - Integration guides
        pass

    def build_tree(self) -> dict:
        """Build the complete meta-tree."""
        docpacks = self.scan_docpacks()
        categories = self.categorize_docpacks(docpacks)

        tree = {"structure": [], "cross_references": {}}

        for category, deps in categories.items():
            category_node = {
                "title": category,
                "type": "category",
                "summary": self.generate_category_summary(category, deps),
                "nodes": []
            }

            for dep in deps:
                docpack_node = {
                    "title": dep,
                    "type": "docpack",
                    "summary": self.generate_docpack_summary(dep),
                    "metadata": self.get_docpack_metadata(dep),
                    "nodes": self.extract_topics(dep)
                }
                category_node["nodes"].append(docpack_node)

            tree["structure"].append(category_node)

        tree["cross_references"] = self.find_cross_references(docpacks)

        return tree
```

### Unified Search with Meta-Tree

```python
class MetaTreeSearch:
    """Search with meta-tree navigation."""

    def __init__(self, deps_dir: str):
        self.meta_tree = load_json(f"{deps_dir}/docpack_tree.json")
        self.qmd_collection = "project-deps"

    def find_relevant_docpacks(self, query: str) -> list:
        """Use meta-tree to identify relevant docpacks."""

        # Strategy 1: Check cross_references for patterns
        for pattern, info in self.meta_tree["cross_references"].items():
            if self._matches_pattern(query, info["description"]):
                return info["involves"]

        # Strategy 2: Search category/docpack summaries
        relevant = []
        for category in self.meta_tree["structure"]:
            for docpack in category["nodes"]:
                if self._matches_summary(query, docpack):
                    relevant.append(docpack["title"])

        return relevant

    def search(self, query: str) -> dict:
        """Full search flow."""

        # Step 1: Find relevant docpacks via meta-tree
        relevant_docpacks = self.find_relevant_docpacks(query)

        # Step 2: Build path filter for qmd
        if relevant_docpacks:
            path_filter = ",".join(f"{dp}/*" for dp in relevant_docpacks)
        else:
            path_filter = None  # Search all

        # Step 3: Run qmd search
        results = qmd_query(
            query,
            collection=self.qmd_collection,
            filter_path=path_filter
        )

        # Step 4: Enrich with meta-tree context
        for result in results:
            docpack = self._get_docpack_from_path(result["path"])
            result["docpack_summary"] = self.meta_tree.get_docpack(docpack)["summary"]
            result["related_docpacks"] = self.meta_tree.get_related(docpack)

        return {
            "query": query,
            "identified_docpacks": relevant_docpacks,
            "results": results
        }
```

---

## Future Enhancements

### Code Layer (PageIndex + Repomix)
1. **Cross-Reference Graph**: Build import/export dependency graph between files
2. **Type-Aware Search**: Use TypeScript types for better function matching
3. **Call Graph Navigation**: "Show me everything that calls this function"
4. **Multi-Language Support**: Extend code analysis to more languages

### Documentation Layer (qmd)
5. **Auto-Doc Collection**: Scrape official docs sites automatically
6. **Version-Pinned Docs**: Match docs version to package.json version
7. **Community Content**: Index relevant Stack Overflow, blog posts
8. **Example Extraction**: Parse code examples from docs into runnable snippets

### Unified System
9. **Version Diffing**: Compare indexes across dependency versions
10. **Incremental Updates**: Update index when dependencies change
11. **Cross-Layer Linking**: Link doc mentions to code implementations
12. **Agent Memory**: Cache agent's frequently-used dependency patterns
13. **Project Context**: Index YOUR project's usage of dependencies for patterns

### Distribution & Sharing
14. **Pre-built Indexes**: Publish indexes for popular packages (npm-style)
15. **Index CDN**: `depindex install axios` downloads pre-built index
16. **Team Sharing**: Share curated doc collections across team

---

## Implementation Roadmap

### Phase 1: Core Integration (MVP)
- [ ] Create `page_index_repomix.py` with XML parsing
- [ ] Add basic code structure extraction (Python/JS/TS)
- [ ] Build symbol table from PageIndex output (exports, classes, functions)
- [ ] Integrate into `run_pageindex.py` CLI
- [ ] Document qmd setup workflow for dependencies
- [ ] Build example index of axios (code + docs)

### Phase 2: Link Extraction
- [ ] Implement `link_extractor.py` module
- [ ] Doc→Code: API header pattern matching (`### axios.create()`)
- [ ] Doc→Code: Inline code reference extraction (`` `symbol` ``)
- [ ] Doc→Code: Code block AST parsing for function calls
- [ ] Doc→Code: File path reference matching
- [ ] Code→Doc: JSDoc @see/@link extraction
- [ ] Code→Doc: Comment reference pattern matching
- [ ] Build bidirectional link index (link_index.json)
- [ ] Add link confidence scoring

### Phase 3: Unified Search with Links
- [ ] Create `depindex` unified search module
- [ ] Implement link-aware search (doc → linked code)
- [ ] Implement reverse lookup (code symbol → related docs)
- [ ] Add result enrichment with linked context
- [ ] Create MCP server for agent integration
- [ ] Add coverage reporting (unlinked symbols, orphan docs)

### Phase 4: Automation
- [ ] Script to auto-index from package.json/requirements.txt
- [ ] Auto-download docs from common sources (GitHub, npm, PyPI)
- [ ] Incremental update detection (re-link on changes)
- [ ] Link validation (detect broken references)

### Phase 5: Polish & Distribution
- [ ] Pre-built index format specification (including links)
- [ ] Index publishing/sharing mechanism
- [ ] VS Code extension for browsing indexes + links
- [ ] Demo notebook with full agent workflow
- [ ] Link coverage dashboard / quality metrics

---

## Quick Start Guide

```bash
# 1. Install tools
pip install pageindex
npm install -g repomix
npm install -g qmd  # or: cargo install qmd

# 2. Create dependency knowledge base
mkdir -p .dependencies/axios/{code,docs}

# 3. Index code (generates pageindex.json + symbol_table.json)
npx repomix --remote axios/axios -o .dependencies/axios/code/repomix-output.xml
python -m pageindex --repomix .dependencies/axios/code/repomix-output.xml \
  --output .dependencies/axios/code/pageindex.json \
  --symbols .dependencies/axios/code/symbol_table.json

# 4. Add documentation
git clone --depth 1 https://github.com/axios/axios /tmp/axios
cp -r /tmp/axios/docs/* .dependencies/axios/docs/
qmd collection add .dependencies/axios/docs --name axios-docs
qmd embed

# 5. Generate bidirectional links
python -m depindex link \
  --code-index .dependencies/axios/code/pageindex.json \
  --symbols .dependencies/axios/code/symbol_table.json \
  --docs-dir .dependencies/axios/docs \
  --output .dependencies/axios/link_index.json

# Link extraction output:
# ✓ Found 45 exported symbols
# ✓ Extracted 78 doc→code links from 12 documents
# ✓ Extracted 23 code→doc links from source files
# ✓ Coverage: 89% of symbols have documentation
# ✓ Saved to .dependencies/axios/link_index.json

# 6. Search with links!
python -m depindex search "interceptors" --dep axios

# Returns:
# DOC: "Interceptors Guide" (score: 0.92)
#   └─→ LINKED CODE:
#       ├─ InterceptorManager.js:15 (use method) [confidence: 0.95]
#       ├─ InterceptorManager.js:32 (eject method) [confidence: 0.95]
#       └─ Axios.js:42 (interceptors property) [confidence: 0.90]
```

---

## Appendix: Repomix Output Examples

### XML Format Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<repomix-output>
  <file_summary>
    This file is a merged representation of the entire codebase...
    File Count: 45
    Total Tokens: 125,432
  </file_summary>

  <directory_structure>
lib/
  adapters/
    http.js
    xhr.js
  core/
    Axios.js
    InterceptorManager.js
    dispatchRequest.js
  helpers/
    ...
  </directory_structure>

  <files>
    <file path="lib/axios.js">
'use strict';

import utils from './utils.js';
import bind from './helpers/bind.js';
import Axios from './core/Axios.js';
// ... rest of file
    </file>
    <!-- more files -->
  </files>
</repomix-output>
```

### Markdown Format Structure
```markdown
# Repository: axios

## File Summary
- Files: 45
- Tokens: 125,432
- Primary Language: JavaScript

## Directory Structure
\`\`\`
lib/
  adapters/
    http.js
    xhr.js
  core/
    Axios.js
\`\`\`

## Files

### lib/axios.js
\`\`\`javascript
'use strict';

import utils from './utils.js';
// ...
\`\`\`
```
