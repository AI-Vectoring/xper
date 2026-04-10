# xper — User's Guide

---

## What xper Does

xper is a knowledge system you interact with by asking questions in plain language. It finds answers by navigating a structured topic tree — following a chain of focused nodes until it reaches the information you need.

You do not search. You do not specify where to look. You ask, and xper reasons its way to the answer.

---

## Asking Questions

Ask as you normally would:

> "What are the PHP requirements for installing Mautic?"

> "How do I install Mautic using Composer?"

> "What is the minimum execution time setting for the web installer?"

xper works best with **specific, factual questions** about topics covered in the knowledge base. You do not need to know the structure of the tree or how the information is organized.

---

## What Happens When You Ask

1. xper starts at the root of the topic tree
2. It reads the available topics and reasons about which branch leads toward your answer
3. It follows that branch, reading each node, until it reaches the answer
4. If navigation alone is not sufficient, it searches the full knowledge base directly and synthesizes from multiple nodes

You may see xper report its path — which nodes it visited on the way to the answer. This is intentional. It lets you verify that the answer came from the right place.

---

## What xper Is Good At

- **Precise, factual lookups** in well-structured knowledge bases (documentation, guides, policies, specifications)
- **Multi-step questions** that require combining information from a few related topics
- **Questions where the answer location is not obvious** — xper reasons about the structure, it does not require you to know where things are
- **Consistent answers** — the same question navigates the same path, producing the same answer

---

## What xper Is Not Good At

- **Unstructured or unindexed content** — xper only knows what is in its knowledge base. It cannot answer questions about content that has not been authored into the tree.
- **Opinion, analysis, or synthesis across large amounts of content** — xper finds specific answers, it does not summarize entire domains
- **Real-time information** — the knowledge base reflects a point-in-time snapshot of the source material

---

## When You Get an Incomplete Answer

If xper reaches a node that references external documentation without repeating the details (e.g., "see the official requirements page"), that is a gap in the knowledge base, not a system failure. Report it so the missing node can be authored.

---

## Tips

- Be specific. "What are the requirements?" is harder to navigate than "What are the PHP requirements for a production install?"
- If the answer seems wrong, ask xper to show its navigation path. An incorrect answer usually means it took the wrong branch — which is diagnosable and fixable in the tree.
- xper does not hallucinate from outside its knowledge base. If it cannot find an answer, it will say so.
