# code_ripout

Extract and distill functionality from a complex GitHub library into a minimal,
self-contained implementation. Uses DeepWiki MCP for understanding + GitHub CLI
for reading source code.

Inspired by [Karpathy's workflow](https://x.com/karpathy/status/2021633574089416993) for increasing software
malleability — the idea that LLMs can understand source code well enough to extract
the essential core from over-engineered libraries.

## Context

Libraries often wrap simple core ideas in layers of generalization: plugin registries,
config systems, compatibility shims, integration adapters, telemetry, CLI scaffolding,
etc. Sometimes you just need the core 20% that does 80% of the work.

This skill automates the workflow:
1. Understand what the library actually does (via DeepWiki)
2. Read the actual source code (via gh CLI)
3. Identify what's essential vs. orchestration overhead
4. Produce a minimal self-contained file with identical core API
5. Verify parity with the original

## Installation

```bash
/plugin marketplace add Yusuke710/code-ripout-skill
/plugin install code_ripout@code-ripout-skill
```

## Requirements

### DeepWiki MCP

Provides `mcp__deepwiki__ask_question` and `mcp__deepwiki__read_wiki_structure` for querying any GitHub repo.

```bash
claude mcp add -s user -t http deepwiki https://mcp.deepwiki.com/mcp
```

> Source: https://docs.devin.ai/work-with-devin/deepwiki-mcp

### GitHub CLI

Used to read source files directly from repositories.

```bash
brew install gh
gh auth login
```
