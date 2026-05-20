# Spatio-Temporal Graph Wiki

A pattern for building version-aware knowledge bases using content-addressed metadata and Git as a temporal index.

## The core idea

Most version-controlled documentation systems treat Git as backup. The history exists, but queries always target the current state. To see "what did the documentation say in version 2024.1?", you manually checkout that commit and search.

This system builds a *persistent knowledge graph* indexing both semantic structure (products, domains, concepts) and temporal evolution (which files existed when). The graph is a dual-axis index: spatial ("what exists about X?") and temporal ("what existed at version Y?"). Query by metadata and version to get precise, consistent results without searching text.

## The problem it solves

Traditional approaches to version-aware documentation have fundamental limitations:

- **Directory-based versioning** - Duplicates content and prevents cross-version queries. 
- **Vector search / RAG** - Version-blind. Embeddings don't distinguish versions, potentially mixing v2024.1 and v2024.2 content. 
- **Manual navigation** - Requires detective work to find relevant files in large repositories.

This system uses metadata filtering for precision, Git for temporal boundaries, and graph queries for intelligent entry points.

## Architecture

There are three layers:

1. **Common Source Knowledge Base (CSKB)** - Markdown files with YAML front matter in a Git repository. Each Knowledge File is a conceptual "bucket" for content that shares identical metadata. Source content is chunked, classified, and recombined. Chunks with matching metadata merge into the same Knowledge File. Files are named using deterministic metadata hashes, so identical metadata always produces the same filename. Front matter contains categories (product, domain, feature, version), relationships (prerequisites, related topics), and citation metadata (references to original sources). Content includes numbered citations that map back to source documents, providing an audit trail for verification.

2. **The knowledge graph** - A persistent index built incrementally during ingestion and commits, with a spatial and temporal axis: 
    - **Spatial** (files, products, domains, concepts).
    - **Temporal** (commits, versions).
    - **Edges** represent semantic relationships (file X requires Y) and temporal ones (file created in commit Z).
Stored as YAML or in a graph database. Can be reconstructed from files and Git history for validation, but intended workflow is incremental building.

3. **The query interface** - Queries the graph for relevant files by metadata and version, then reconstructs the entire CSKB in the specified state from Git. The graph then filters thousands of files to a handful of relevant ones, providing intelligent entry points which the agent can then navigate, following links and relationships while maintaining version boundaries.

## Key concepts

- **Content-Addressed Metadata** - Filenames are deterministic hashes of metadata. Example: `{product: X, domain: Y, type: task}` → `a7f3c9d1-0001.md`. Same metadata = same filename; different metadata = different filename. If metadata changes, the filename changes, forcing a new file. This creates stable cross-version references: file `f6e5c8d4-0007.md` in both v2024.1 and v2024.2 represents the same conceptual bucket.

- **Version as Metadata** - Version is metadata on files, not Git tags. A version spans many commits. Files declare `version: ["v2024.1", "v2024.2"]` in front matter. Queries filter by version metadata across commits. This treats versions as logical groupings of content, allowing continuous updates to v2024.1 even after v2024.2 ships.

- **Schema-Driven Classification** - Define metadata categories upfront based on search needs and desired granularity: `product`, `domain`, `feature`, `info_type`, `version`. Categories are structural. Changing them requires potentially updating all existing Knowledge Files and regenerating hashes. An LLM Classifier suggests *values* within categories (new products, features, etc.), which humans approve. Values can be added freely without breaking changes. Categories stay stable; values expand organically.

- **Incremental Graph Construction** - The graph builds incrementally: ingestion adds spatial nodes and edges, commits add temporal nodes and edges. The graph emerges from metadata and Git history. Reconstruction from files is possible for validation but is not the intended workflow.

- **Dual-Axis Graph** - Enables queries like: "What files about domain X existed in v2024.1?" (spatial + temporal), "What changed between versions?" (temporal diff), "Show prerequisite chain at v2023.2" (spatial traversal at temporal point).
    - **Spatial edges**: File → Product/Domain, File → File (prerequisites, relationships)
    - **Temporal edges**: File → Commit (created/modified/deleted), File → Version, Commit → Commit

## How it works

### Building the graph

1. Ingest source → chunk → classify by metadata
2. Create/update Knowledge Files (chunks with matching metadata merge into same file)
3. Update graph (add nodes, semantic edges)
4. Commit to Git
5. Index commit (add temporal edges)

The graph builds incrementally with each ingestion cycle. Reconstruction from Git history is possible for validation but not the primary workflow.

### Querying the system

1. **User query**: "Show me configuration tasks for product Alpha in version 2024.1"

2. **Graph query**: Filter files where `product=Alpha`, `version includes v2024.1`, `info_type=task`, `domain contains configuration` → Returns ~500 files and 3 entrypoint suggestions.

3. **Git Retrieval**: Reconstruct the CSKB in the specified state by loading the ~500 files from the applicable commits.

4. **Agent navigation**: The agent starts with the 3 entrypoint files but can explore further, following prerequisites in front matter, markdown links to related topics, or querying the graph again for additional context. All navigation stays within the same version boundary (files tagged with v2024.1) to maintain version consistency because the reconstructed CSKB only contains files that meet the specified criteria.

### Multi-version snapshots

Support customers running mixed versions. Query files by `product=Alpha, version=v2024.1` and `product=Alpha, version=v2024.2` separately. Files reference each other safely via metadata-addressed filenames. Same hash means same conceptual entity across versions.

## Why this works

- **Precision**: Metadata filtering returns exact matches, not similarity scores
- **Version safety**: No accidental mixing of version content (unless you deliberately query multiple versions)
- **Scalability**: Graph queries over 50K files take 20-100ms
- **Temporal queries**: Compare versions structurally via graph traversal
- **Token efficiency**: Filter 10K files to 3-5 before loading—65-70% fewer tokens than traditional RAG
- **Verifiable**: Graph rebuilds from files; no hidden database state

## Use cases

The pattern applies anywhere you need version-aware, structured knowledge with both semantic structure (what relates to what) and temporal awareness (when things existed or changed).

- **Documentation generation**: Filter and assemble specific content for PDFs, HTML, help systems.
- **LLM context**: Provide precise, version-correct context for chatbots, support agents, code assistants.
- **API documentation**: Generate version-specific API references.
- **Training materials**: Assemble courseware from curated knowledge.
- **Compliance reporting**: Extract audit-ready documentation with citation trails.
- **Wiki navigation**: Browse interactively (though scale may require graph-powered filtering).

The goal is a curated Common Source Knowledge Base (CSKB) that provides context for downstream tools and pipelines. A CSKB of tens of thousands of files can provide thousands of relevant, pre-approved files to multiple consumers.


## Implementation notes

- **Metadata schema**: Define categories upfront in a YAML or JSON file based on search needs and knowledge file granularity. Choose categories carefully as they determine filename hashes and changing them later will likely necessitate updating all existing files. Typical categories: `product`, `domain`, `feature`, `info_type` (concept/task/reference), `version`.

- **Vocabulary management**: You can add values that you know such as your list of products, or start with an empty vocabulary and let the LLM suggest values as you ingest content. The LLM proposes new products, features, domains, etc., which humans review and approve. Adding a new value doesn't change pre-existing filenames. Over time you will build a controlled vocabulary that prevents drift while adapting to your content.

- **Filename encoding**: Hash metadata (blake2b) + optional sequence id: `{hash}-{sequence}.md`. 

- **Image and other filetypes**: A sequence number is useful for image files where multiple files may have the same metadata. The metadata hash means the image can be matched directly to related Knowledge Files with the same metadata hash.

- **Graph storage**: Start with YAML for simplicity (works fine up to ~10K files). Move to Neo4j or similar for larger scale (50K+ files) or complex graph queries. The graph structure is simple: nodes are entities (files, commits, concepts), edges are typed relationships.

- **Ingestion workflow**: Source → chunks → classify → files → commit → graph update (seconds to minutes). The graph updates incrementally with each cycle, so there's no separate "indexing" step; it happens as part of ingestion.

- **Version metadata**: Include version as a metadata field on your files (e.g., `version: ["v2024.1", "v2024.2"]`). Files can belong to multiple versions. The graph indexes version metadata, enabling queries like "show me all v2024.1 content" to filter files by their version field rather than by Git tags.

- **Citations and verification**: Knowledge files contain numbered citations (e.g., [1], [2]) within their content. Citation metadata is stored in front matter, mapping citation numbers to original source documents. This provides an audit trail for every piece of information, enabling verification that content accurately reflects sources.

- **Optional linting**: Citations also enable optional linting similar to LLM-Wiki patterns, checking for contradictions, outdated claims, or descrepancies between Knowledge Files and their sources. In a well-organized CSKB with clear category boundaries and proper classification, the content should be internally consistent making regular linting unnecessary. However, citation-based validation acts as a safeguard against contamination from bad source data, extraction errors, or accidentally ingested content, and maintains content integrity.

## Note

This document is intentionally abstract. It describes the pattern, not a specific implementation. The exact graph schema, the metadata categories, the filename format, the query interface - all of that depends on your domain and requirements.

Use this is as a starting point for your own implementation. The core concepts: content-addressed metadata, Git as temporal index, and the spatio-temporal graph structure, are the essential pieces. Everything else is negotiable.

Your implementation might use different tools, different metadata schemas, different hash functions, different query interfaces. That's expected and encouraged.
