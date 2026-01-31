# PageIndex + Repomix + qmd: TypeScript Unified Project Plan

## Executive Summary

This document outlines a unified TypeScript implementation that combines three tools into a single monorepo:

1. **PageIndex** - Hierarchical tree structures for reasoning-based code/document navigation
2. **Repomix** - Repository consolidation into AI-optimized formats
3. **qmd** - Hybrid search engine (BM25 + vector + LLM re-ranking)

The goal: Enable AI coding agents to have complete understanding of any dependency through both structured code navigation AND semantic documentation search.

---

## Architecture Decision: Monorepo vs Dependencies

### Recommendation: **Unified Monorepo**

After analyzing both codebases, incorporating them as direct code (not standalone dependencies) is the better approach:

| Factor | Monorepo | Separate Dependencies |
|--------|----------|----------------------|
| **Integration Depth** | Deep - shared types, direct function calls | Shallow - CLI/API boundaries |
| **Type Safety** | Full TypeScript types across all modules | Type definitions at boundaries only |
| **Build Optimization** | Single build, tree-shaking, shared deps | Multiple builds, duplicate dependencies |
| **Customization** | Easy to modify any component | Fork/patch required for changes |
| **Version Coordination** | Automatic - single version | Manual - dependency version matrix |
| **Bundle Size** | Optimized - only used code | Larger - full packages included |
| **Development Speed** | Fast iteration, no publish cycles | Slower - publish/install cycles |

**Why Monorepo Works Here:**

1. **qmd is already TypeScript** - No conversion needed, direct integration
2. **PageIndex needs porting anyway** - Converting from Python, might as well integrate
3. **Shared concerns** - All three deal with document processing, LLM calls, tree structures
4. **Single MCP server** - One unified server exposing all capabilities
5. **Common dependencies** - SQLite, embeddings, LLM clients are shared

---

## Project Structure

```
depindex/
├── package.json                    # Monorepo root (workspaces)
├── tsconfig.json                   # Shared TS config
├── turbo.json                      # Turborepo for builds
│
├── packages/
│   ├── core/                       # Shared utilities
│   │   ├── src/
│   │   │   ├── llm/               # LLM client abstraction
│   │   │   │   ├── index.ts
│   │   │   │   ├── openai.ts
│   │   │   │   └── local.ts       # node-llama-cpp
│   │   │   ├── store/             # SQLite + FTS5 + sqlite-vec
│   │   │   │   ├── index.ts
│   │   │   │   ├── schema.ts
│   │   │   │   └── migrations.ts
│   │   │   ├── types/             # Shared type definitions
│   │   │   │   ├── index.ts
│   │   │   │   ├── tree.ts
│   │   │   │   ├── search.ts
│   │   │   │   └── document.ts
│   │   │   └── utils/
│   │   │       ├── tokens.ts      # Token counting
│   │   │       ├── hash.ts        # Content hashing
│   │   │       └── chunk.ts       # Text chunking
│   │   └── package.json
│   │
│   ├── pageindex/                  # PageIndex (ported from Python)
│   │   ├── src/
│   │   │   ├── index.ts           # Main exports
│   │   │   ├── pdf/               # PDF processing
│   │   │   │   ├── parser.ts      # pdfjs-dist integration
│   │   │   │   ├── toc.ts         # TOC detection/extraction
│   │   │   │   └── mapper.ts      # Page-to-section mapping
│   │   │   ├── markdown/          # Markdown processing
│   │   │   │   ├── parser.ts      # Header extraction
│   │   │   │   └── thinning.ts    # Tree optimization
│   │   │   ├── repomix/           # Repomix adapter (NEW)
│   │   │   │   ├── parser.ts      # XML/MD/JSON parsing
│   │   │   │   ├── analyzer.ts    # Code structure extraction
│   │   │   │   └── tree.ts        # Build code tree
│   │   │   ├── tree/
│   │   │   │   ├── builder.ts     # Hierarchy construction
│   │   │   │   ├── summary.ts     # LLM summary generation
│   │   │   │   └── traversal.ts   # Tree utilities
│   │   │   └── cli.ts             # CLI entry point
│   │   └── package.json
│   │
│   ├── qmd/                        # qmd (from tobi/qmd)
│   │   ├── src/
│   │   │   ├── index.ts           # Main exports
│   │   │   ├── store.ts           # Database operations
│   │   │   ├── search/
│   │   │   │   ├── fts.ts         # BM25 full-text
│   │   │   │   ├── vector.ts      # Semantic search
│   │   │   │   ├── hybrid.ts      # RRF + reranking
│   │   │   │   └── rrf.ts         # Reciprocal rank fusion
│   │   │   ├── collections.ts     # Collection management
│   │   │   ├── formatter.ts       # Output formatting
│   │   │   ├── llm.ts             # Embeddings, reranking
│   │   │   └── cli.ts             # CLI entry point
│   │   └── package.json
│   │
│   ├── link-index/                 # Bidirectional linking (NEW)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── extractor.ts       # Doc↔Code link extraction
│   │   │   ├── symbols.ts         # Symbol table builder
│   │   │   └── resolver.ts        # Link resolution
│   │   └── package.json
│   │
│   └── mcp-server/                 # Unified MCP server
│       ├── src/
│       │   ├── index.ts
│       │   ├── tools/
│       │   │   ├── search.ts      # qmd search tools
│       │   │   ├── navigate.ts    # PageIndex navigation
│       │   │   └── index.ts       # Indexing operations
│       │   └── resources.ts       # Document resources
│       └── package.json
│
├── apps/
│   └── cli/                        # Unified CLI
│       ├── src/
│       │   ├── index.ts           # Main entry
│       │   ├── commands/
│       │   │   ├── index.ts       # depindex index <source>
│       │   │   ├── search.ts      # depindex search <query>
│       │   │   ├── navigate.ts    # depindex nav <path>
│       │   │   ├── link.ts        # depindex link <dep>
│       │   │   └── serve.ts       # depindex mcp
│       │   └── config.ts
│       └── package.json
│
└── examples/
    ├── axios/                      # Example: axios dependency
    └── react-query/                # Example: react-query
```

---

## Core Type Definitions

### Tree Structures

```typescript
// packages/core/src/types/tree.ts

/**
 * Base node in any hierarchical tree
 */
export interface TreeNode {
  /** Unique identifier (4-digit: 0001, 0002, etc.) */
  nodeId: string;

  /** Display title */
  title: string;

  /** AI-generated summary */
  summary?: string;

  /** Child nodes */
  nodes?: TreeNode[];
}

/**
 * PDF document tree node
 */
export interface PdfTreeNode extends TreeNode {
  /** Starting page (1-indexed) */
  startIndex: number;

  /** Ending page (1-indexed) */
  endIndex: number;

  /** Full text content (optional) */
  text?: string;

  /** Summary when node has children */
  prefixSummary?: string;
}

/**
 * Markdown document tree node
 */
export interface MarkdownTreeNode extends TreeNode {
  /** Line number where section starts */
  lineNum: number;

  /** Header level (1-6) */
  level: number;

  /** Section text content */
  text?: string;
}

/**
 * Code repository tree node (from Repomix)
 */
export interface CodeTreeNode extends TreeNode {
  /** Node type */
  type: 'directory' | 'file' | 'class' | 'function' | 'interface' | 'type' | 'export';

  /** File path relative to repo root */
  path: string;

  /** Starting line in file */
  lineStart?: number;

  /** Ending line in file */
  lineEnd?: number;

  /** Function/method signature */
  signature?: string;

  /** Programming language */
  language?: string;

  /** Repomix content locator */
  repomixLocator?: {
    fileIndex: number;
    contentLines: [number, number];
  };
}

/**
 * Document index output structure
 */
export interface DocumentIndex<T extends TreeNode = TreeNode> {
  /** Document/repository name */
  docName: string;

  /** Document type */
  docType: 'pdf' | 'markdown' | 'repomix';

  /** AI-generated description */
  docDescription?: string;

  /** Metadata */
  metadata?: {
    version?: string;
    totalFiles?: number;
    totalTokens?: number;
    language?: string;
    processedAt: string;
  };

  /** Hierarchical structure */
  structure: T[];
}
```

### Search Structures

```typescript
// packages/core/src/types/search.ts

/**
 * Document in the search index
 */
export interface IndexedDocument {
  /** Short document ID (first 6 chars of hash) */
  docId: string;

  /** Full content hash (SHA256) */
  hash: string;

  /** Collection name */
  collection: string;

  /** Display path within collection */
  path: string;

  /** Document title */
  title: string;

  /** Folder/path context description */
  context?: string;

  /** Last modified timestamp */
  modifiedAt: string;

  /** Content size in bytes */
  bodyLength: number;
}

/**
 * Search result
 */
export interface SearchResult extends IndexedDocument {
  /** Relevance score (0.0-1.0) */
  score: number;

  /** Search source */
  source: 'fts' | 'vec' | 'hybrid';

  /** Result snippet */
  snippet?: string;

  /** Character position (for vector chunk results) */
  chunkPos?: number;
}

/**
 * Search options
 */
export interface SearchOptions {
  /** Search mode */
  mode: 'search' | 'vsearch' | 'query';

  /** Collection to search (optional, searches all if not specified) */
  collection?: string;

  /** Maximum results */
  limit?: number;

  /** Minimum score threshold */
  minScore?: number;
}
```

### Link Index Structures

```typescript
// packages/core/src/types/links.ts

/**
 * Code symbol in the symbol table
 */
export interface CodeSymbol {
  /** PageIndex node ID */
  nodeId: string;

  /** Symbol type */
  type: 'module' | 'class' | 'function' | 'method' | 'interface' | 'type' | 'constant';

  /** File path */
  path: string;

  /** Line number */
  line?: number;

  /** Repomix file index */
  repomixFileIndex?: number;
}

/**
 * Bidirectional link between documentation and code
 */
export interface DocCodeLink {
  /** How link was extracted */
  extractionMethod:
    | 'api_header'      // ### `axios.create()`
    | 'inline_code'     // `interceptors.use()`
    | 'code_example'    // Function call in code block
    | 'file_reference'  // `lib/core/Axios.js`
    | 'jsdoc_see'       // @see tag
    | 'comment_ref';    // // See: docs/...

  /** Confidence score (0.0-1.0) */
  confidence: number;
}

/**
 * Doc → Code link reference
 */
export interface DocToCodeRef extends DocCodeLink {
  /** PageIndex node ID (primary anchor) */
  nodeId: string;

  /** Node summary (from PageIndex) */
  nodeSummary?: string;

  /** Original source path */
  path: string;

  /** Symbol name */
  symbol?: string;

  /** Line range */
  lineStart?: number;
  lineEnd?: number;

  /** Repomix locator for content retrieval */
  repomixLocator?: {
    fileIndex: number;
    contentLines: [number, number];
  };
}

/**
 * Code → Doc link reference
 */
export interface CodeToDocRef extends DocCodeLink {
  /** qmd document ID */
  docId: string;

  /** Document path */
  docPath: string;

  /** Section within document */
  section?: string;

  /** qmd locator */
  qmdLocator?: {
    collection: string;
    lineStart: number;
    lineEnd: number;
  };
}

/**
 * Complete link index for a dependency
 */
export interface LinkIndex {
  /** Dependency name */
  dependency: string;

  /** Dependency version */
  version?: string;

  /** Generation timestamp */
  generatedAt: string;

  /** Symbol table (symbol name → code location) */
  symbols: Record<string, CodeSymbol>;

  /** Doc → Code links */
  docToCode: Array<{
    docId: string;
    docPath: string;
    docSection?: string;
    codeRefs: DocToCodeRef[];
  }>;

  /** Code → Doc links (reverse index) */
  codeToDoc: Array<{
    nodeId: string;
    path: string;
    docRefs: CodeToDocRef[];
  }>;

  /** Fast lookup indexes */
  indexes: {
    /** node_id → doc_ids that reference it */
    nodeToDocs: Record<string, string[]>;

    /** doc_id → node_ids it references */
    docToNodes: Record<string, string[]>;

    /** symbol name → node_id */
    symbolToNode: Record<string, string>;
  };

  /** Coverage statistics */
  coverage: {
    symbolsTotal: number;
    symbolsWithDocs: number;
    docsTotal: number;
    docsWithCodeRefs: number;
    totalLinks: number;
    avgConfidence: number;
  };
}
```

---

## PageIndex Adapter for qmd Integration

The key integration point: allowing qmd to index PageIndex-generated trees so they become searchable.

### Adapter Design

```typescript
// packages/pageindex/src/adapters/qmd-adapter.ts

import { createStore, insertContent, insertDocument } from '@depindex/qmd';
import { hashContent } from '@depindex/core';
import type { DocumentIndex, CodeTreeNode, TreeNode } from '@depindex/core';

export interface QmdIndexOptions {
  /** Collection name for this index */
  collectionName: string;

  /** Include full text content (increases size) */
  includeText?: boolean;

  /** Include code signatures */
  includeSignatures?: boolean;

  /** Custom context for the collection */
  context?: string;
}

/**
 * Convert PageIndex tree to qmd-indexable markdown documents
 */
export async function indexPageIndexTree(
  tree: DocumentIndex<CodeTreeNode>,
  options: QmdIndexOptions
): Promise<{ indexed: number; skipped: number }> {
  const store = createStore();
  const now = new Date().toISOString();
  let indexed = 0;
  let skipped = 0;

  async function processNode(
    node: CodeTreeNode,
    parentPath: string = '',
    breadcrumb: string[] = []
  ): Promise<void> {
    const nodePath = parentPath ? `${parentPath}/${node.title}` : node.title;
    const fullBreadcrumb = [...breadcrumb, node.title];

    // Convert node to markdown for indexing
    const markdown = nodeToMarkdown(node, fullBreadcrumb, options);

    if (!markdown.trim()) {
      skipped++;
      return;
    }

    const hash = await hashContent(markdown);

    // Insert into qmd store
    insertContent(store.db, hash, markdown, now);
    insertDocument(
      store.db,
      options.collectionName,
      normalizePathForQmd(nodePath, node.type),
      node.title,
      hash,
      now,
      now
    );

    indexed++;

    // Recursively process children
    for (const child of node.nodes || []) {
      await processNode(child, nodePath, fullBreadcrumb);
    }
  }

  // Process all root nodes
  for (const root of tree.structure) {
    await processNode(root);
  }

  store.close();

  return { indexed, skipped };
}

/**
 * Convert a single tree node to markdown document
 */
function nodeToMarkdown(
  node: CodeTreeNode,
  breadcrumb: string[],
  options: QmdIndexOptions
): string {
  const lines: string[] = [];

  // Title
  lines.push(`# ${node.title}`);
  lines.push('');

  // Breadcrumb navigation
  if (breadcrumb.length > 1) {
    lines.push(`> Path: ${breadcrumb.join(' > ')}`);
    lines.push('');
  }

  // Type badge
  lines.push(`**Type:** ${node.type}`);
  if (node.path) {
    lines.push(`**Path:** \`${node.path}\``);
  }
  if (node.lineStart && node.lineEnd) {
    lines.push(`**Lines:** ${node.lineStart}-${node.lineEnd}`);
  }
  if (node.language) {
    lines.push(`**Language:** ${node.language}`);
  }
  lines.push('');

  // Signature (for functions/methods)
  if (options.includeSignatures && node.signature) {
    lines.push('## Signature');
    lines.push('```' + (node.language || ''));
    lines.push(node.signature);
    lines.push('```');
    lines.push('');
  }

  // Summary
  if (node.summary) {
    lines.push('## Summary');
    lines.push(node.summary);
    lines.push('');
  }

  // Children summary
  if (node.nodes && node.nodes.length > 0) {
    lines.push('## Contains');
    for (const child of node.nodes) {
      const typeIcon = getTypeIcon(child.type);
      lines.push(`- ${typeIcon} **${child.title}**${child.summary ? `: ${child.summary.slice(0, 100)}...` : ''}`);
    }
    lines.push('');
  }

  return lines.join('\n');
}

function normalizePathForQmd(path: string, type: string): string {
  // Convert tree path to file-like path for qmd
  const normalized = path
    .toLowerCase()
    .replace(/[^a-z0-9/.-]/g, '-')
    .replace(/-+/g, '-');

  const extension = type === 'directory' ? '/index.md' : '.md';
  return normalized + extension;
}

function getTypeIcon(type: string): string {
  const icons: Record<string, string> = {
    directory: '[dir]',
    file: '[file]',
    class: '[class]',
    function: '[fn]',
    method: '[method]',
    interface: '[iface]',
    type: '[type]',
    export: '[export]',
  };
  return icons[type] || '[?]';
}
```

### Search Integration

```typescript
// packages/pageindex/src/adapters/unified-search.ts

import { createStore, searchFTS, searchVec, query as hybridQuery } from '@depindex/qmd';
import type { SearchResult, SearchOptions } from '@depindex/core';

export interface UnifiedSearchOptions extends SearchOptions {
  /** Search code index */
  searchCode?: boolean;

  /** Search documentation */
  searchDocs?: boolean;

  /** Dependency name */
  dependency: string;
}

export interface UnifiedSearchResult {
  query: string;
  dependency: string;
  codeResults?: SearchResult[];
  docResults?: SearchResult[];
  linkedContext?: LinkedContext[];
}

export interface LinkedContext {
  /** Source (doc or code) */
  source: 'doc' | 'code';

  /** Primary result */
  result: SearchResult;

  /** Linked items from the other layer */
  linked: Array<{
    type: 'doc' | 'code';
    nodeId?: string;
    docId?: string;
    title: string;
    summary?: string;
    confidence: number;
  }>;
}

/**
 * Unified search across code and documentation
 */
export async function unifiedSearch(
  query: string,
  options: UnifiedSearchOptions
): Promise<UnifiedSearchResult> {
  const store = createStore();
  const result: UnifiedSearchResult = {
    query,
    dependency: options.dependency,
  };

  const searchFn = options.mode === 'search'
    ? (q: string, c: string) => searchFTS(store.db, q, options.limit || 10, c)
    : options.mode === 'vsearch'
    ? (q: string, c: string) => searchVec(store.db, q, 'embeddinggemma', options.limit || 10, c)
    : (q: string, c: string) => hybridQuery(store.db, q, 'embeddinggemma', options.limit || 10, c);

  // Search code collection
  if (options.searchCode !== false) {
    const codeCollection = `${options.dependency}-code`;
    result.codeResults = await searchFn(query, codeCollection);
  }

  // Search docs collection
  if (options.searchDocs !== false) {
    const docsCollection = `${options.dependency}-docs`;
    result.docResults = await searchFn(query, docsCollection);
  }

  store.close();

  return result;
}

/**
 * Search with automatic link enrichment
 */
export async function searchWithLinks(
  query: string,
  options: UnifiedSearchOptions,
  linkIndex: LinkIndex
): Promise<UnifiedSearchResult> {
  const baseResults = await unifiedSearch(query, options);

  // Enrich results with linked context
  baseResults.linkedContext = [];

  // For each doc result, find linked code
  for (const docResult of baseResults.docResults || []) {
    const nodeIds = linkIndex.indexes.docToNodes[docResult.docId] || [];

    if (nodeIds.length > 0) {
      baseResults.linkedContext.push({
        source: 'doc',
        result: docResult,
        linked: nodeIds.map(nodeId => {
          const symbol = Object.entries(linkIndex.symbols)
            .find(([_, s]) => s.nodeId === nodeId);

          return {
            type: 'code' as const,
            nodeId,
            title: symbol?.[0] || nodeId,
            confidence: 0.9,
          };
        }),
      });
    }
  }

  // For each code result, find linked docs
  for (const codeResult of baseResults.codeResults || []) {
    const docIds = linkIndex.indexes.nodeToDocs[codeResult.docId] || [];

    if (docIds.length > 0) {
      baseResults.linkedContext.push({
        source: 'code',
        result: codeResult,
        linked: docIds.map(docId => ({
          type: 'doc' as const,
          docId,
          title: docId, // Would need doc lookup for actual title
          confidence: 0.9,
        })),
      });
    }
  }

  return baseResults;
}
```

---

## Repomix Parser Implementation

```typescript
// packages/pageindex/src/repomix/parser.ts

import { DOMParser } from '@xmldom/xmldom';
import type { CodeTreeNode, DocumentIndex } from '@depindex/core';

export interface RepomixOutput {
  format: 'xml' | 'markdown' | 'json';
  fileSummary: {
    fileCount: number;
    totalTokens: number;
    primaryLanguage?: string;
  };
  directoryStructure: string;
  files: Array<{
    path: string;
    content: string;
    index: number;
  }>;
  instruction?: string;
}

/**
 * Parse Repomix XML output
 */
export function parseRepomixXml(xmlContent: string): RepomixOutput {
  const parser = new DOMParser();
  const doc = parser.parseFromString(xmlContent, 'text/xml');

  // Extract file_summary
  const summaryEl = doc.getElementsByTagName('file_summary')[0];
  const summaryText = summaryEl?.textContent || '';
  const fileCountMatch = summaryText.match(/File Count:\s*(\d+)/i);
  const tokenMatch = summaryText.match(/Total Tokens:\s*([\d,]+)/i);

  // Extract directory_structure
  const dirEl = doc.getElementsByTagName('directory_structure')[0];
  const directoryStructure = dirEl?.textContent?.trim() || '';

  // Extract files
  const fileEls = doc.getElementsByTagName('file');
  const files: RepomixOutput['files'] = [];

  for (let i = 0; i < fileEls.length; i++) {
    const fileEl = fileEls[i];
    const path = fileEl.getAttribute('path') || '';
    const content = fileEl.textContent || '';
    files.push({ path, content, index: i });
  }

  // Extract instruction
  const instructionEl = doc.getElementsByTagName('instruction')[0];

  return {
    format: 'xml',
    fileSummary: {
      fileCount: fileCountMatch ? parseInt(fileCountMatch[1], 10) : files.length,
      totalTokens: tokenMatch ? parseInt(tokenMatch[1].replace(/,/g, ''), 10) : 0,
    },
    directoryStructure,
    files,
    instruction: instructionEl?.textContent?.trim(),
  };
}

/**
 * Parse Repomix Markdown output
 */
export function parseRepomixMarkdown(mdContent: string): RepomixOutput {
  const files: RepomixOutput['files'] = [];
  let directoryStructure = '';
  let fileCount = 0;
  let totalTokens = 0;

  // Extract directory structure
  const dirMatch = mdContent.match(/## Directory Structure\s*```\s*([\s\S]*?)```/i);
  if (dirMatch) {
    directoryStructure = dirMatch[1].trim();
  }

  // Extract files (### path followed by code block)
  const fileRegex = /### ([^\n]+)\s*```\w*\s*([\s\S]*?)```/g;
  let match;
  let index = 0;

  while ((match = fileRegex.exec(mdContent)) !== null) {
    files.push({
      path: match[1].trim(),
      content: match[2],
      index: index++,
    });
  }

  // Extract summary info
  const filesMatch = mdContent.match(/Files:\s*(\d+)/i);
  const tokensMatch = mdContent.match(/Tokens:\s*([\d,]+)/i);

  return {
    format: 'markdown',
    fileSummary: {
      fileCount: filesMatch ? parseInt(filesMatch[1], 10) : files.length,
      totalTokens: tokensMatch ? parseInt(tokensMatch[1].replace(/,/g, ''), 10) : 0,
    },
    directoryStructure,
    files,
  };
}

/**
 * Build directory tree from Repomix directory structure string
 */
export function buildDirectoryTree(dirStructure: string): CodeTreeNode[] {
  const lines = dirStructure.split('\n').filter(line => line.trim());
  const root: CodeTreeNode[] = [];
  const stack: { node: CodeTreeNode; indent: number }[] = [];

  for (const line of lines) {
    const indent = line.search(/\S/);
    const name = line.trim();
    const isDirectory = name.endsWith('/');

    const node: CodeTreeNode = {
      nodeId: '', // Assigned later
      title: isDirectory ? name.slice(0, -1) : name,
      type: isDirectory ? 'directory' : 'file',
      path: name,
    };

    // Find parent based on indentation
    while (stack.length > 0 && stack[stack.length - 1].indent >= indent) {
      stack.pop();
    }

    if (stack.length === 0) {
      root.push(node);
    } else {
      const parent = stack[stack.length - 1].node;
      parent.nodes = parent.nodes || [];
      parent.nodes.push(node);

      // Build full path
      node.path = `${parent.path}${isDirectory ? name : name}`;
    }

    if (isDirectory) {
      stack.push({ node, indent });
    }
  }

  return root;
}

/**
 * Parse Repomix output and build PageIndex tree
 */
export async function parseRepomix(
  content: string,
  format: 'xml' | 'markdown' | 'json' = 'xml'
): Promise<DocumentIndex<CodeTreeNode>> {
  const parsed = format === 'xml'
    ? parseRepomixXml(content)
    : format === 'markdown'
    ? parseRepomixMarkdown(content)
    : JSON.parse(content);

  // Build directory tree
  const structure = buildDirectoryTree(parsed.directoryStructure);

  // Map file contents to tree nodes
  const fileMap = new Map(parsed.files.map(f => [f.path, f]));

  function enrichNode(node: CodeTreeNode, basePath: string = ''): void {
    const fullPath = basePath ? `${basePath}/${node.title}` : node.title;
    node.path = fullPath;

    if (node.type === 'file') {
      const file = fileMap.get(fullPath) || fileMap.get(node.title);
      if (file) {
        node.repomixLocator = {
          fileIndex: file.index,
          contentLines: [0, file.content.split('\n').length],
        };

        // Detect language
        node.language = detectLanguage(node.title);
      }
    }

    for (const child of node.nodes || []) {
      enrichNode(child, node.type === 'directory' ? fullPath : basePath);
    }
  }

  for (const node of structure) {
    enrichNode(node);
  }

  return {
    docName: 'repomix-output',
    docType: 'repomix',
    metadata: {
      totalFiles: parsed.fileSummary.fileCount,
      totalTokens: parsed.fileSummary.totalTokens,
      processedAt: new Date().toISOString(),
    },
    structure,
  };
}

function detectLanguage(filename: string): string | undefined {
  const ext = filename.split('.').pop()?.toLowerCase();
  const langMap: Record<string, string> = {
    ts: 'typescript',
    tsx: 'typescript',
    js: 'javascript',
    jsx: 'javascript',
    py: 'python',
    go: 'go',
    rs: 'rust',
    java: 'java',
    rb: 'ruby',
    php: 'php',
    cs: 'csharp',
    cpp: 'cpp',
    c: 'c',
    h: 'c',
    hpp: 'cpp',
    md: 'markdown',
    json: 'json',
    yaml: 'yaml',
    yml: 'yaml',
  };
  return ext ? langMap[ext] : undefined;
}
```

---

## CLI Design

```typescript
// apps/cli/src/index.ts

import { Command } from 'commander';

const program = new Command();

program
  .name('depindex')
  .description('Unified dependency knowledge system')
  .version('0.1.0');

// Index a dependency
program
  .command('index <source>')
  .description('Index a dependency (repo URL, local path, or npm package)')
  .option('-n, --name <name>', 'Dependency name')
  .option('-t, --type <type>', 'Source type: repo, npm, local', 'auto')
  .option('--code-depth <depth>', 'Code analysis depth: directory, file, function', 'file')
  .option('--include-docs', 'Also index documentation')
  .option('--docs-path <path>', 'Path to documentation within repo')
  .action(async (source, options) => {
    // 1. Fetch/clone source
    // 2. Run Repomix to generate consolidated output
    // 3. Parse Repomix output with PageIndex
    // 4. Index code tree in qmd
    // 5. If --include-docs: index docs in qmd
    // 6. Generate link index
  });

// Search
program
  .command('search <query>')
  .description('Search indexed dependencies')
  .option('-d, --dep <name>', 'Dependency to search')
  .option('-m, --mode <mode>', 'Search mode: search, vsearch, query', 'query')
  .option('--code-only', 'Search code only')
  .option('--docs-only', 'Search documentation only')
  .option('-n, --limit <n>', 'Max results', '10')
  .option('--json', 'Output as JSON')
  .action(async (query, options) => {
    // Run unified search with options
  });

// Navigate code tree
program
  .command('nav [path]')
  .description('Navigate the code tree')
  .option('-d, --dep <name>', 'Dependency')
  .option('--children', 'Show children')
  .option('--parent', 'Show parent')
  .option('--siblings', 'Show siblings')
  .action(async (path, options) => {
    // Tree navigation
  });

// Get content
program
  .command('get <ref>')
  .description('Get content by node ID, doc ID, or path')
  .option('-d, --dep <name>', 'Dependency')
  .option('--full', 'Get full content')
  .option('-l, --lines <range>', 'Line range (e.g., 10-50)')
  .action(async (ref, options) => {
    // Content retrieval
  });

// Link operations
program
  .command('link <dependency>')
  .description('Generate or update link index for a dependency')
  .option('--rebuild', 'Force rebuild links')
  .option('--report', 'Show coverage report')
  .action(async (dependency, options) => {
    // Link generation
  });

// MCP server
program
  .command('mcp')
  .description('Start MCP server')
  .action(async () => {
    // Start unified MCP server
  });

// Collection management
program
  .command('list')
  .description('List indexed dependencies')
  .action(async () => {
    // List all indexed deps
  });

program.parse();
```

---

## MCP Server Tools

```typescript
// packages/mcp-server/src/tools/index.ts

export const tools = [
  // === SEARCH TOOLS ===
  {
    name: 'depindex_search',
    description: 'Search dependency code and documentation using hybrid search (BM25 + vector + reranking)',
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Search query' },
        dependency: { type: 'string', description: 'Dependency name (optional, searches all if omitted)' },
        mode: { type: 'string', enum: ['search', 'vsearch', 'query'], default: 'query' },
        target: { type: 'string', enum: ['all', 'code', 'docs'], default: 'all' },
        limit: { type: 'number', default: 10 },
      },
      required: ['query'],
    },
  },

  // === NAVIGATION TOOLS ===
  {
    name: 'depindex_nav',
    description: 'Navigate the code tree structure',
    inputSchema: {
      type: 'object',
      properties: {
        dependency: { type: 'string', description: 'Dependency name' },
        nodeId: { type: 'string', description: 'Node ID to navigate to' },
        action: { type: 'string', enum: ['children', 'parent', 'siblings', 'path'], default: 'children' },
      },
      required: ['dependency'],
    },
  },

  // === CONTENT RETRIEVAL ===
  {
    name: 'depindex_get',
    description: 'Get full content for a node or document',
    inputSchema: {
      type: 'object',
      properties: {
        ref: { type: 'string', description: 'Node ID (#0001), doc ID (#abc123), or path' },
        dependency: { type: 'string' },
        lines: { type: 'string', description: 'Line range (e.g., "10-50")' },
      },
      required: ['ref'],
    },
  },

  // === LINK TOOLS ===
  {
    name: 'depindex_links',
    description: 'Get linked code for a doc or linked docs for code',
    inputSchema: {
      type: 'object',
      properties: {
        ref: { type: 'string', description: 'Node ID or doc ID' },
        dependency: { type: 'string' },
        direction: { type: 'string', enum: ['doc_to_code', 'code_to_doc', 'both'], default: 'both' },
      },
      required: ['ref', 'dependency'],
    },
  },

  // === SYMBOL LOOKUP ===
  {
    name: 'depindex_symbol',
    description: 'Look up a symbol (function, class, etc.) by name',
    inputSchema: {
      type: 'object',
      properties: {
        symbol: { type: 'string', description: 'Symbol name (e.g., "axios.create", "InterceptorManager")' },
        dependency: { type: 'string' },
      },
      required: ['symbol'],
    },
  },
];
```

---

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1-2)
- [ ] Set up monorepo structure with Turborepo
- [ ] Port qmd core modules (store, search, collections)
- [ ] Implement shared types and utilities
- [ ] Basic CLI scaffolding

### Phase 2: PageIndex Port (Week 2-3)
- [ ] Port PDF processing pipeline
- [ ] Port Markdown processing pipeline
- [ ] Implement tree building and traversal
- [ ] Summary generation with LLM

### Phase 3: Repomix Integration (Week 3-4)
- [ ] Repomix XML/Markdown/JSON parsers
- [ ] Code structure analyzer (functions, classes, exports)
- [ ] Build code tree from Repomix output
- [ ] Code-level summary generation

### Phase 4: qmd Adapter (Week 4)
- [ ] PageIndex → qmd indexing adapter
- [ ] Unified search across code and docs
- [ ] Search result enrichment

### Phase 5: Link Index (Week 5)
- [ ] Symbol table extraction
- [ ] Doc → Code link extraction
- [ ] Code → Doc link extraction
- [ ] Bidirectional index generation

### Phase 6: MCP Server & Polish (Week 6)
- [ ] Unified MCP server
- [ ] CLI completion
- [ ] Documentation
- [ ] Example indexes (axios, react-query)

---

## Summary: Can qmd Index PageIndex Output?

**YES - and here's why it works well:**

1. **Format Compatibility**: PageIndex outputs hierarchical JSON with summaries. Converting each node to a markdown document is straightforward.

2. **Natural Hierarchy**: qmd's collection + path system maps perfectly to PageIndex's tree structure. Each node becomes a document with its path reflecting its position in the tree.

3. **Search Synergy**:
   - BM25 (keyword) finds exact function names, file paths
   - Vector (semantic) finds conceptual matches ("authentication" → login code)
   - Hybrid combines both for best results

4. **Summary Advantage**: PageIndex pre-generates summaries at each node. These become excellent search targets for qmd's semantic search.

5. **MCP Ready**: qmd's existing MCP server can be extended to expose PageIndex navigation alongside search.

**The Adapter Strategy:**
```
PageIndex JSON Tree
    ↓
Convert each node to markdown document:
  - Title = node.title
  - Metadata = type, path, lines
  - Content = summary + children list
    ↓
Index in qmd as collection
    ↓
Search returns relevant nodes with:
  - score, snippet, docId
  - Can map docId back to nodeId
  - Navigate tree from result
```

This unified approach gives AI agents complete dependency understanding through structured code navigation AND semantic search in a single, cohesive system.
