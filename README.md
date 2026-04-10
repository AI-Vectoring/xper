# xper

A navigational context management system for LLMs.

xper structures knowledge as a navigable topic tree — a hierarchy of focused nodes linked by named references. Instead of dumping an entire knowledge base into a prompt or relying on similarity search alone, an LLM follows links through the tree, loading only the context it needs to answer a question.

## How It Works

Knowledge is authored as nodes: small, focused text files covering a single topic. Each node declares named links to related nodes. An LLM navigates the tree by reasoning about which path leads to the answer. When navigation alone is insufficient, a vector search fallback locates the right node directly.

The same LLM that navigates the tree can also author it — given a source URL or document, it generates the node structure automatically.

## Documents

- [Implementer's Guide](docs/implementers-guide.md) — Architecture, tooling, authoring pipeline, development notes
- [User's Guide](docs/users-guide.md) — How to query xper and what to expect
- [Technical Reference](docs/technical.md) — Deep-dive: architecture, tradeoffs, failure modes, hybrid design
- [Marketing](docs/marketing.md) — Product positioning and use cases
