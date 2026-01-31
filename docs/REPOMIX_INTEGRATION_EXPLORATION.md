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

## Benefits of This Approach

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
- [ ] Integrate into `run_pageindex.py` CLI
- [ ] Document qmd setup workflow for dependencies
- [ ] Build example index of axios (code + docs)

### Phase 2: Unified Search
- [ ] Create `depindex` unified search module
- [ ] Implement combined search API (code + docs)
- [ ] Add result merging and ranking
- [ ] Create MCP server for agent integration

### Phase 3: Automation
- [ ] Script to auto-index from package.json/requirements.txt
- [ ] Auto-download docs from common sources (GitHub, npm, PyPI)
- [ ] Incremental update detection

### Phase 4: Polish & Distribution
- [ ] Pre-built index format specification
- [ ] Index publishing/sharing mechanism
- [ ] VS Code extension for browsing indexes
- [ ] Demo notebook with full agent workflow

---

## Quick Start Guide

```bash
# 1. Install tools
pip install pageindex
npm install -g repomix
npm install -g qmd  # or: cargo install qmd

# 2. Create dependency knowledge base
mkdir -p .dependencies/axios/{code,docs}

# 3. Index code
npx repomix --remote axios/axios -o .dependencies/axios/code/repomix-output.xml
python -m pageindex --repomix .dependencies/axios/code/repomix-output.xml

# 4. Add documentation
git clone --depth 1 https://github.com/axios/axios /tmp/axios
cp -r /tmp/axios/docs/* .dependencies/axios/docs/
qmd collection add .dependencies/axios/docs --name axios-docs
qmd embed

# 5. Search!
# Code: python -m pageindex search "interceptors" --dep axios
# Docs: qmd query "how to use interceptors" -c axios-docs
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
