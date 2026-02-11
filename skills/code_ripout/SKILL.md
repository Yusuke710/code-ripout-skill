---
name: code_ripout
description: |
  Rip out functionality from a complex library into a self-contained file using DeepWiki MCP + GitHub CLI.
  Inspired by Karpathy's workflow for increasing software malleability — the idea that LLMs can
  understand source code well enough to extract the essential core from over-engineered libraries.

  Triggers:
  - "rip out", "extract from library", "self-contained version"
  - User is frustrated with a library's complexity and suspects the core is simpler
  - User wants to remove a dependency by inlining its essential logic
  - "make it self-contained", "distill this library", "rewrite without dependency"
  - User mentions a GitHub repo and wants to understand then extract functionality

  Do NOT use when:
  - User just wants to understand a library (use DeepWiki directly)
  - User wants to use the library as-is (just help them with the API)
  - The library functionality is genuinely complex and can't be meaningfully simplified
---

## Workflow

### Step 1: Parse the request

Extract from user's request or $ARGUMENTS:
- **repo**: GitHub repository in `owner/repo` format
- **functionality**: What specific functionality to extract
- **output_file**: Target filename (default: infer from functionality)

### Step 2: Understand via DeepWiki MCP

Build a high-level understanding BEFORE reading source code.

```
mcp__deepwiki__ask_question(
  repoName: "<owner/repo>",
  question: "How does <repo> implement <functionality>? What are the core components,
  key abstractions, and the minimal set of operations needed? What is essential vs
  what is orchestration/generalization overhead?"
)
```

Also get the wiki structure to identify relevant sections:
```
mcp__deepwiki__read_wiki_structure(repoName: "<owner/repo>")
```

From the DeepWiki response, identify:
- The core algorithm/logic (KEEP)
- What low-level primitives or stdlib calls it ultimately delegates to

### Step 3: Read source via GitHub CLI

Read the actual source files identified by DeepWiki. Start with the transitive
core path (entry point -> main logic -> primitives). Expand to additional files
only when unresolved dependencies appear.

List directory:
```bash
gh api repos/<owner>/<repo>/contents/<path> --jq '.[].name'
```

Read file contents via blob:
```bash
gh api repos/<owner>/<repo>/contents/<filepath> --jq '.sha'
gh api repos/<owner>/<repo>/git/blobs/<sha> --jq '.content' | base64 -d
```

### Step 4: Analyze and plan the extraction

Before writing code, explicitly classify what you found in EACH source file:

**KEEP** — Core logic:
- The actual algorithm or computation
- Correctness-critical details (precision, layouts, protocols, invariants)
- Error handling that protects against real failure modes

**STRIP** — Orchestration overhead (common categories):
- Plugin/registry systems and dynamic dispatch tables
- Config/settings hierarchies and feature flags
- Integration adapters (framework hooks, middleware, serialization)
- Distributed/parallel orchestration (unless requested)
- Compatibility shims and deprecation wrappers
- Telemetry, logging infrastructure, and observability hooks
- CLI scaffolding and argument parsing
- Abstract base classes used for a single concrete implementation

**SIMPLIFY** — Over-generalized code:
- Complex abstractions with only one real code path -> inline the concrete path
- Indirect calls through registries -> direct function calls
- Configurable behavior with one common setting -> hardcode the default

Key principles:
- **Call primitives directly** instead of going through dispatch/registry layers
- **Hardcode the common recipe** instead of supporting every configuration
- **Inline small utilities** instead of importing from the library
- **Keep the same public API** so it's a drop-in replacement
- **Preserve correctness invariants** — these are the things easiest to
  get wrong when simplifying (memory layouts, protocol ordering, precision,
  encoding, authentication handshakes, etc.)

### Step 5: Implement the self-contained file

Write a single file (or minimal set of files) that:

1. **Has a clear provenance header** with:
   - Source repo URL and commit SHA
   - Which files were distilled
   - License of the original (preserve it)
   - What was kept vs stripped
   - The core algorithm in plain English

2. **Only imports stdlib + the base framework** (no library dependency)

3. **Preserves the essential public API** so users can swap it in

4. **Comments tricky correctness details** — especially constraints that exist
   for non-obvious reasons (hardware requirements, protocol ordering, precision
   choices, encoding rules)

5. **Includes a basic test/demo** in `if __name__ == "__main__"` or equivalent

### Step 6: Verify parity

Before declaring success, verify the ripout matches the original:
- Run the demo/test to confirm basic functionality works
- If possible, compare outputs against the original library on representative inputs
- Check edge cases the original handles (empty inputs, error paths, boundary values)
- Verify the public API signature matches (same function names, same arguments)

## Anti-patterns to avoid

- **Don't over-abstract**: Write operations inline rather than creating generic helpers
  that obscure the actual logic. Three specific calls is better than one leaky wrapper.
- **Don't cargo-cult from the original**: Understand WHY each detail exists before keeping
  it. Some things exist for edge cases you don't need. Some things exist for correctness
  you absolutely do need. Know the difference.
- **Don't skip reading the source**: DeepWiki gives you the map, but the source code
  is the territory. Always read the actual implementation before writing yours.
- **Don't strip invariants without testing**: If the original has a check, assertion, or
  specific ordering, understand why before removing it. These often guard against subtle
  bugs.
- **Don't forget provenance**: Always record which repo, commit, and files you distilled
  from. The upstream will evolve and someone will need to know where this code came from.
- **Don't aim for full API parity**: Rip out the specific functionality needed, not the
  entire library surface. Scope creep defeats the purpose.
