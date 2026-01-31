# PageIndex + Repomix Integration Exploration

## Executive Summary

This document explores the integration of PageIndex with Repomix to enable intelligent, hierarchical search over code dependencies for AI coding agents. The core idea is to use Repomix to generate structured representations of code dependencies, then adapt PageIndex to create navigable tree structures that coding agents can efficiently search.

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

---

## Integration Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           INDEXING PHASE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────────────┐  │
│  │ Dependency   │     │   Repomix    │     │ PageIndex for Repomix  │  │
│  │ Repository   │────▶│  (Generate)  │────▶│     (Index + Tree)     │  │
│  │ (e.g. axios) │     │              │     │                        │  │
│  └──────────────┘     └──────────────┘     └────────────────────────┘  │
│                              │                        │                 │
│                              ▼                        ▼                 │
│                     repomix-output.xml      dependency_index.json      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          RETRIEVAL PHASE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────────────────┐  │
│  │ Coding Agent │     │ PageIndex    │     │ Repomix Output +       │  │
│  │   Query:     │────▶│  Navigator   │────▶│ Index                  │  │
│  │ "How to use  │     │ (LLM-based)  │     │                        │  │
│  │  interceptors│     │              │◀────│ Returns relevant       │  │
│  │  in axios?"  │     └──────────────┘     │ code sections          │  │
│  └──────────────┘                          └────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Directory Structure for Dependencies

```
project/
├── src/                          # Your project code
├── .dependencies/                # PageIndex dependency storage
│   ├── axios/
│   │   ├── repomix-output.xml    # Raw Repomix output
│   │   └── pageindex.json        # Indexed tree structure
│   ├── react-query/
│   │   ├── repomix-output.xml
│   │   └── pageindex.json
│   └── zod/
│       ├── repomix-output.xml
│       └── pageindex.json
└── pageindex.config.yaml         # Configuration
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

### 1. Index Dependencies

```bash
# Generate Repomix output for a dependency
cd ~/.dependencies/axios
npx repomix --remote yamadashy/axios --output repomix-output.xml

# Index with PageIndex
python run_pageindex.py \
  --repomix_path ~/.dependencies/axios/repomix-output.xml \
  --code-depth function \
  --if-add-node-summary yes
```

### 2. Batch Index Project Dependencies

```bash
# Script to index all dependencies from package.json
python scripts/index_dependencies.py \
  --package-json ./package.json \
  --output-dir ./.dependencies
```

### 3. Agent Query Flow

```python
# In coding agent:
from pageindex import DependencyIndex

deps = DependencyIndex("./.dependencies")

# Agent needs to understand axios interceptors
context = deps.search(
    query="How to add request interceptors",
    dependencies=["axios"]
)

# Returns structured results:
# {
#   "relevant_nodes": [
#     {
#       "title": "InterceptorManager.js",
#       "path": "lib/core/InterceptorManager.js",
#       "summary": "Manages request/response interceptors...",
#       "content_snippet": "..."
#     }
#   ],
#   "navigation_path": ["lib", "core", "InterceptorManager.js"],
#   "related_nodes": [...]
# }
```

---

## Configuration Options

```yaml
# pageindex.config.yaml

# Repomix-specific settings
repomix:
  default_style: xml
  code_analysis:
    enabled: true
    depth: function        # directory | file | function
    languages:
      - python
      - javascript
      - typescript
      - go

  tree_optimization:
    collapse_threshold: 1000   # tokens - collapse small dirs
    expand_entry_points: true  # Always expand index.js, main.py, etc.
    max_depth: 6               # Maximum tree depth

  summaries:
    directory_level: true
    file_level: true
    function_level: false      # Can be expensive for large codebases

# Dependency management
dependencies:
  storage_dir: .dependencies
  auto_update: false
  include_dev_deps: false
```

---

## Benefits of This Approach

### For Coding Agents

1. **Structured Navigation**: Instead of searching through raw files, agents navigate a logical tree
2. **Context-Aware Retrieval**: Summaries at each level help agents decide where to look
3. **Efficient Token Usage**: Tree structure allows progressive disclosure
4. **Explainable Results**: Can trace exactly where information came from

### Compared to Alternatives

| Approach | Pros | Cons |
|----------|------|------|
| **Raw File Reading** | Complete info | Overwhelming, no structure |
| **Vector RAG** | Fast, scalable | Loses context, opaque results |
| **Web Search** | Up-to-date | Slow, may find wrong version |
| **PageIndex + Repomix** | Structured, explainable, hierarchical | Requires pre-indexing |

### Example Agent Interaction

```
Agent: I need to implement request retry logic with axios.

[PageIndex Navigation]
1. Search "retry" across axios index
2. Find: lib/helpers/retryAfter.js (summary: "Helper for retry-after header parsing")
3. Find: lib/defaults/index.js (summary: "Default config including retry settings")
4. Navigate to lib/core/Axios.js → request method
5. Retrieve relevant code sections with context

Agent receives:
- Exact file paths and line numbers
- Function signatures
- Usage context from surrounding code
- Related files (imports, exports)
```

---

## Future Enhancements

1. **Cross-Reference Graph**: Build import/export dependency graph between files
2. **Type-Aware Search**: Use TypeScript types for better function matching
3. **Version Diffing**: Compare indexes across dependency versions
4. **Incremental Updates**: Update index when dependency updates
5. **Usage Examples Extraction**: Find and index code examples from tests/docs
6. **Multi-Language Support**: Extend code analysis to more languages

---

## Next Steps

1. [ ] Create `page_index_repomix.py` with XML parsing
2. [ ] Add basic code structure extraction (Python/JS/TS)
3. [ ] Integrate into `run_pageindex.py` CLI
4. [ ] Build example index of a popular library (e.g., axios, requests)
5. [ ] Create demo notebook showing agent usage
6. [ ] Add Markdown and JSON format support
7. [ ] Build dependency manager for multi-library support

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
