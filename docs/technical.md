# xper — Technical Reference

---

## Overview

xper is a **navigational context management system** for large language models. It addresses a core problem in LLM deployment: how to give a model access to a large knowledge base without overwhelming its context window or degrading answer quality through irrelevant retrieval.

The approach is navigational rather than retrieval-based. Knowledge is organized as a directed graph of focused nodes. The LLM traverses the graph by reasoning — following named links from node to node until it reaches the information needed to answer a query.

---

## Why Not Just Vector RAG?

Vector RAG (Retrieval-Augmented Generation) retrieves document chunks by embedding similarity — chunks whose vector representation is closest to the query vector get injected into the prompt.

This works well for unstructured, flat corpora. It has known failure modes:

- **False positives**: Chunks that are semantically similar to the query but contextually wrong
- **False negatives**: Correct chunks that happen to use different vocabulary than the query
- **No structure exploitation**: The retrieval mechanism is blind to the document's logical organization
- **Non-deterministic**: The same query can retrieve different chunks across runs

xper's navigational approach exploits explicit structure. The model reasons about the organization of knowledge, not just surface similarity. In testing, this produces more precise results on structured knowledge bases — the model navigates to the correct node through reasoning, even when no single term in the query matches any term in the answer.

---

## Architecture

### The Topic Tree

Knowledge is stored as a directed acyclic graph (in practice, a tree with occasional cross-links). Each node is a self-contained text covering exactly one topic. Nodes declare named links to child nodes, with a one-line description of each link's target.

```
Root Node
├── [Topic A]       "Overview of topic A and its subtopics"
│   ├── [Subtopic A1]   "Specific aspect of A"
│   └── [Subtopic A2]   "Another aspect of A"
└── [Topic B]       "Overview of topic B"
    └── [Subtopic B1]   "Specific aspect of B"
```

The descriptions on links are critical. They allow the LLM to reason about whether to follow a link without reading the target node — keeping each navigation step cheap.

### The Navigation Tool

The LLM interacts with the knowledge base through a single tool:

```
follow_link(label: str) -> NodeContent
```

This tool accepts a link label and returns the target node's content plus its available outbound links. The LLM never interacts with file paths, database IDs, or any storage implementation detail. All resolution is handled by the tool layer.

Supporting operations:
- `go_back()` — return to the previous node
- `search(query: str)` — vector search fallback (see Hybrid Mode)
- `get_path()` — return the current traversal path (for auditability)

### The Database

Production storage uses a relational database with an optional vector extension (e.g., PostgreSQL + pgvector).

Core tables:

**nodes** — one row per node
```
id          UUID
slug        text, unique
title       text
body        text
parent_id   UUID (nullable)
embedding   vector(1536)   -- optional, for search fallback
created_at  timestamp
updated_at  timestamp
```

**links** — one row per directed edge
```
id          UUID
source_id   UUID → nodes.id
target_id   UUID → nodes.id
label       text
description text
```

Link integrity is enforced at the database level. A link cannot point to a non-existent node. This eliminates the broken-link failure mode present in file-based implementations.

---

## Hybrid Mode

Navigation is precise but has three failure modes:

| Failure Mode | Description |
|---|---|
| Multi-branch synthesis | The answer spans nodes in different branches with no common ancestor near the root |
| Inference failure | The tree is wide or deep enough that the correct path is not inferable from node descriptions alone |
| Authoring ambiguity | The author placed a node in a branch where the navigating model would not look |

In all three cases, the fallback is vector similarity search on the `nodes` table. The model queries the embedding index directly, retrieves the most relevant nodes, and synthesizes an answer without navigation.

The two modes use the same underlying data. No separate ingestion pipeline is needed for search — nodes are embedded at write time.

**Mode selection heuristic**: If the model has traversed two or three hops without converging on an answer, switch to search. This can be implemented as a tool-level timeout or as explicit model reasoning ("I am not finding this through navigation; switching to search").

---

## The Authoring Pipeline

Nodes do not need to be hand-authored. An LLM can generate the tree from source material:

```
Source material (URL, document, corpus)
        ↓
  LLM reads and understands source structure
        ↓
  LLM generates nodes:
    - one node per coherent topic unit
    - link labels named to match navigating model's reasoning
    - link descriptions that enable follow/skip decisions
        ↓
  Nodes written to database
  Embeddings generated at write time
```

A key property: the authoring LLM and the navigating LLM share reasoning patterns. This means the tree's structure tends to align with how a model would search it — ambiguous placement is partially self-correcting.

For large sources, authoring should be structured:
1. Author a manifest node first, declaring the top-level structure
2. Author subtrees one branch at a time, referencing the manifest for consistency
3. Validate with integration tests (see Testing) before deploying

---

## Comparison with Vector RAG

| Property | xper (Navigational) | Vector RAG |
|---|---|---|
| Retrieval mechanism | Reasoning over explicit structure | Embedding similarity |
| Determinism | High — same query, same path | Low — varies across runs |
| Handles structured knowledge | Natively | Poorly |
| Handles unstructured corpora | Requires authoring/transformation | Natively |
| Handles fuzzy/semantic queries | Well — model reasons about structure | Well — similarity handles vocabulary variation |
| Multi-branch synthesis | Poor (without search fallback) | Moderate |
| Auditability | High — traversal path is explicit | Low — retrieval is opaque |
| False positive rate | Low | Moderate |
| Infrastructure required | Database (+ optional vector index) | Vector database + embedding pipeline |
| Authoring cost | Automatable via LLM pipeline | None (ingest raw content) |

---

## Testing

### Unit tests
- Each node resolves correctly from its link label
- Broken links fail at write time, not at read time
- Link descriptions are present and non-empty

### Integration tests

Embed a verification string in a target leaf node during authoring:

```
Verification: [unique token]
```

In a separate session, submit a question whose answer is in that node. Confirm the verification token appears in the response. A passing test proves the full chain: authoring, storage, tool resolution, and navigation all work correctly end-to-end.

### Regression tests

After any structural change to the tree (new nodes, renamed links, moved subtrees), re-run the integration test suite. Navigation paths are deterministic — a change that breaks a path will break the corresponding test.

---

## Failure Modes and Mitigations

| Failure Mode | Mitigation |
|---|---|
| Wrong branch taken | Search fallback; backtrack tool; improve link descriptions |
| Missing node | Gap reporting from users; coverage metrics |
| Stale content | `updated_at` tracking; re-authoring pipeline from source |
| Authoring inconsistency across sessions | Author manifest first; reference it throughout |
| Link label ambiguity | Namespaced labels; tool uses traversal context to disambiguate |
| Navigation loop | Tool tracks visited nodes; refuse to revisit |

---

## Scalability

The navigational approach scales with tree depth and width, not corpus size. A well-organized tree of 10,000 nodes is navigable in the same number of hops as a tree of 100 nodes, provided the branching factor and description quality are maintained.

The binding constraint is authoring quality, not retrieval infrastructure. A poorly organized tree — wrong branching, weak descriptions, ambiguous placement — degrades navigational performance regardless of scale. The search fallback absorbs this degradation for the subset of queries that navigation fails on.
