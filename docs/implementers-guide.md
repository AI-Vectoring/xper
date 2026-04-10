# xper — Implementer's Guide

This document is written for the people building xper: the decisions made, the reasoning behind them, and the open problems still to solve. It is a working document, not a polished spec.

---

## Origin

xper started as a test: could a plain-text topic tree, navigated by an LLM following `[Link]` references, serve as a more precise and controllable alternative to vector RAG?

The prototype used a Mautic 7.0 installation guide, authored by an LLM from a source RST file at a Mautic GitHub URL. The LLM that authored the tree later navigated it successfully in a separate session — including finding a hidden test message embedded in a leaf node to verify correct traversal.

The system worked. This document captures what we learned.

---

## Core Concept

The knowledge base is a directed graph of nodes. Each node:

- Covers exactly one topic
- Is small enough to fit comfortably in a prompt
- Declares named links to other nodes
- Optionally carries a one-line description of each link target

An LLM navigates the graph by reading a node, reasoning about which link leads toward the answer, and following it. This is **navigational RAG** — retrieval by reasoning, not by similarity.

---

## Node Anatomy

Current format (plain text):

```
NODE TITLE
==========

[Parent Node] > [Current Node]   ← breadcrumb

Body text covering exactly one topic.

SUBTOPICS
---------

[Link Label]
  One-line description of what that node covers.

[Another Link]
  One-line description.
```

Rules:
- One topic per node. Split aggressively.
- Every link must have a description. This is what allows the LLM to reason about whether to follow it without reading the target.
- Breadcrumbs are useful for orientation but should not be relied on for navigation — they go stale if nodes are moved.

---

## Authoring Pipeline

The tree does not need to be hand-authored. The pipeline is:

```
Source (URL / document / corpus)
        ↓
  LLM reads source content
        ↓
  LLM generates node structure:
    - identifies topics and subtopics
    - writes focused node content
    - names links semantically
        ↓
  Nodes stored in database
        ↓
  LLM navigates in separate session
```

Key property: the LLM that authors and the LLM that navigates share the same reasoning patterns. This means nodes tend to be placed where a navigating model would look for them. Ambiguous placement is partially self-correcting.

For large sources spanning multiple pages or cross-referencing heavily, the authoring session may need to be broken into chunks. Structural decisions made in one chunk should be recorded (e.g., a manifest) so later chunks place nodes consistently.

---

## The Navigation Tool

The current prototype uses direct file reads + directory listings to resolve links. This is a proof of concept only. The production tool must abstract this entirely.

### Interface

```
follow_link(label: str) -> NodeContent
```

Returns:
- The full text of the target node
- The list of available links in that node (labels + descriptions)
- The node's position in the breadcrumb trail

The LLM never sees file paths, database IDs, or directory structure. It only sees node content and link labels.

### Tool responsibilities
- Resolve `[Label]` to the correct node (case-insensitive, tolerant of minor variations)
- Return a structured error if the link does not exist
- Track traversal path for the current session (enables backtracking)
- Log navigation paths for analysis and improvement

---

## Database Design

Files work for prototyping. The production system uses a database.

### Minimum schema

```
nodes
-----
id          UUID, primary key
slug        text, unique — machine-readable identifier
title       text — human-readable node title
body        text — full node content
parent_id   UUID, nullable, foreign key → nodes.id
created_at  timestamp
updated_at  timestamp

links
-----
id          UUID, primary key
source_id   UUID, foreign key → nodes.id
target_id   UUID, foreign key → nodes.id
label       text — the [Link Label] as it appears in content
description text — one-line description shown to navigating LLM
```

### Vector extension

Add an `embedding` column (vector type, e.g. pgvector) to the `nodes` table. Populated at write time. Used only as a fallback when navigation fails — see Hybrid Mode below.

---

## Hybrid Mode: Navigate + Search

Navigation is the primary mode. It is precise, deterministic, and auditable. It fails in three cases:

1. **The question spans multiple branches** — no single path leads to the full answer
2. **The tree is too wide or deep** — the right path is not inferable from node descriptions alone
3. **Ambiguous placement** — the author put the node where the navigating LLM would not look

In all three cases, vector similarity search on the `nodes` table returns the relevant nodes directly. The LLM then synthesizes from those nodes without navigating.

Because the nodes are already in the database for navigation, the vector index is free — no separate ingestion pipeline.

The LLM decides which mode to use. A simple heuristic: if after two or three hops the model is not confident it is converging on an answer, switch to search.

---

## Testing

Embed verification messages in leaf nodes during authoring:

```
You reached this node correctly. Session ID: [whatever].
```

In a separate navigation session, ask a question whose answer is in that node. If the session ID appears in the response, the full traversal chain worked. This is a black-box integration test for the entire pipeline: authoring, storage, tool resolution, and navigation.

---

## Known Open Problems

- **Link label disambiguation**: Two nodes in different branches with the same link label. The tool needs to use traversal context to resolve correctly.
- **Stale breadcrumbs**: Manual breadcrumbs go stale when nodes are renamed or moved. Consider generating them dynamically from the `links` table instead.
- **Authoring consistency at scale**: Large corpora authored in multiple sessions may produce structural inconsistencies. A manifest or schema document authored first and referenced throughout mitigates this.
- **Backtracking**: If the LLM follows the wrong path, it should be able to backtrack cheaply. The tool should support `go_back()` returning to the previous node.
- **Node granularity**: Too coarse and nodes become hard to navigate; too fine and the tree becomes too deep. No hard rule yet — calibrate by testing with real questions.
