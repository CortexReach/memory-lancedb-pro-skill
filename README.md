# memory-lancedb-pro — OpenClaw Memory Skill

> **Claude Code Skill** for [memory-lancedb-pro](https://github.com/CortexReach/memory-lancedb-pro) — production-grade long-term memory plugin for OpenClaw AI agents.

This skill gives Claude Code deep, accurate knowledge of every feature in `memory-lancedb-pro` (v1.1.0-beta.8): installation, optimal configuration, Smart Extraction, hybrid retrieval, Weibull decay lifecycle, multi-scope isolation, self-improvement governance, and all MCP tools.

---

## What this skill does

When installed, Claude Code can:

- **Guide you through a 7-step optimal configuration workflow** — just say _"help me enable the best config"_
- **Present 4 deployment plans** (Full Power / Budget / Simple / Fully Local) with provider links and tradeoffs
- **Install, configure, and verify** the plugin using `openclaw plugins install` or git clone
- **Set up Ollama** for fully local, zero-API-cost deployment
- **Configure every feature**: Smart Extraction, hybrid retrieval, reranking, multi-scope, Weibull decay, session memory, self-improvement governance
- **Use all 9 MCP tools** correctly: `memory_recall`, `memory_store`, `memory_forget`, `memory_update`, `memory_stats`, `memory_list`, `self_improvement_log`, `self_improvement_extract_skill`, `self_improvement_review`
- **Avoid common pitfalls** — workspace plugin enablement, `autoRecall` default-false, jiti cache, env vars, scope isolation, etc.

---

## Installation

### Prerequisites

**For Claude Code users:**
- [Claude Code](https://claude.ai/code) CLI installed
- [memory-lancedb-pro](https://github.com/CortexReach/memory-lancedb-pro) plugin configured as an MCP server

**For OpenClaw users:**
- [OpenClaw](https://openclaw.ai) gateway running
- `memory-lancedb-pro` plugin installed via `openclaw plugins install memory-lancedb-pro@beta`

### Install the skill

> **Note:** This is a *skill* (knowledge file for AI agents), not a plugin. Skills are installed by placing them in your skills directory — there is no `openclaw skills install` command.

**Option A — clone this repo (recommended):**

For **Claude Code** users:
```bash
git clone https://github.com/CortexReach/memory-lancedb-pro-skill.git ~/.claude/skills/memory-lancedb-pro
```

For **OpenClaw** users:
```bash
git clone https://github.com/CortexReach/memory-lancedb-pro-skill.git ~/.openclaw/workspace/skills/memory-lancedb-pro-skill
```

**Option B — download ZIP from GitHub:**

1. Click **Code → Download ZIP** on the [repository page](https://github.com/CortexReach/memory-lancedb-pro-skill)
2. Extract and place in your skills directory:

```bash
# Claude Code
unzip memory-lancedb-pro-skill-main.zip
mv memory-lancedb-pro-skill-main ~/.claude/skills/memory-lancedb-pro

# OpenClaw
unzip memory-lancedb-pro-skill-main.zip
mv memory-lancedb-pro-skill-main ~/.openclaw/workspace/skills/memory-lancedb-pro-skill
```

**Verify the skill is loaded:**

```bash
# Claude Code: the skill loads automatically based on trigger conditions
# To test: ask Claude Code "help me configure memory-lancedb-pro"

# OpenClaw: check skill discovery
openclaw skills list
```

---

## Skill Structure

```
memory-lancedb-pro/
├── SKILL.md                      # Main skill file (loaded into context automatically)
└── references/
    └── full-reference.md         # Deep technical reference (loaded on demand)
```

### Progressive disclosure

| Level | What loads | When |
|-------|-----------|------|
| Metadata (`name` + `description`) | Always | ~100 words, negligible |
| `SKILL.md` body | When skill triggers | Operational workflows, all config options |
| `references/full-reference.md` | On demand | DB schema, Weibull formulas, source file map, scoring internals |

---

## Trigger phrases

Claude Code loads this skill automatically when you mention:

- `memory-lancedb-pro`, `memory pro`, `lancedb pro`
- `help me enable the best config` / `apply optimal configuration`
- `memory_recall`, `memory_store`, `memory_forget`, `memory_update`
- `Smart Extraction`, `autoCapture`, `autoRecall`
- `hybrid retrieval`, `reranker`, `BM25`, `Weibull decay`
- `self_improvement_log`, `LEARNINGS.md`, `ERRORS.md`

---

## Covered Features

### Installation & Setup
- 3 installation methods: `openclaw plugins install`, git clone with manual path, existing deployment migration
- Plugin enablement rules: `plugins.allow`, `plugins.entries.<id>.enabled`, `plugins.slots.memory`
- Workspace plugin gotchas (disabled by default, requires explicit `allow`)
- Custom path env vars: `OPENCLAW_HOME`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`
- Post-installation smoke test checklist

### 7-Step Optimal Config Workflow
When you say **"help me enable the best config"**, Claude will:
1. Present 4 deployment plans with provider links
2. Ask about your existing API keys and config location
3. Find and read your current `openclaw.json`
4. Build a merged config block for your chosen plan
5. Apply it with the correct template (Method 1 vs Method 2)
6. Validate and restart the gateway
7. Run a full smoke test

### 4 Deployment Plans

| Plan | Embedding | Reranker | LLM | API Keys |
|------|-----------|----------|-----|----------|
| **A — Full Power** | Jina `jina-embeddings-v5-text-small` | Jina `jina-reranker-v3` | OpenAI `gpt-4o-mini` | Jina + OpenAI |
| **B — Budget** | Jina embeddings | SiliconFlow BGE (free tier) | OpenAI `gpt-4o-mini` | Jina + SiliconFlow + OpenAI |
| **C — Simple** | OpenAI `text-embedding-3-small` | None | OpenAI `gpt-4o-mini` | OpenAI only |
| **D — Local** | Ollama `nomic-embed-text` (768-dim) | None | Ollama `qwen2.5:7b` | None (free) |

Each plan includes: API key acquisition links, cost notes, RAM requirements (Plan D), and tradeoff explanations.

### Smart Extraction
- 6-category LLM-powered classification: Profile → `fact`, Preferences → `preference`, Entities → `entity`, Events → `decision`, Cases → `fact`, Patterns → `other`
- L0/L1/L2 layered storage (Abstract / Overview / Full Content)
- Two-stage deduplication: vector pre-filter (≥ 0.7) + LLM decision (`CREATE | MERGE | SKIP | SUPPORT | CONTEXTUALIZE | CONTRADICT`)
- Config: `smartExtraction`, `extractMinMessages`, `extractMaxChars`, `llm.*`

### Hybrid Retrieval
- Fusion: `(vectorScore × 0.7) + (bm25Score × 0.3)` via RRF
- Pipeline: RRF → Cross-Encoder Rerank → Lifecycle Decay Boost → Length Norm → Hard Min Score → MMR Diversity
- BM25 keyword preservation (score ≥ 0.75 bypasses semantic filter — protects API keys, ticket numbers)
- 4 reranker providers: Jina, SiliconFlow, Voyage AI, Pinecone

### Memory Lifecycle (Weibull Decay)
- 3 tiers: Core (β=0.8, floor=0.9) / Working (β=1.0, floor=0.7) / Peripheral (β=1.3, floor=0.5)
- Promotion/demotion rules based on access count, composite score, importance, age
- Composite score: Recency 40% + Frequency 30% + Intrinsic 30%
- Access reinforcement: frequently recalled memories decay more slowly

### Multi-Scope Isolation
- Scope formats: `global`, `agent:<id>`, `custom:<name>`, `project:<id>`, `user:<id>`
- `scopes.agentAccess` mapping for multi-scope agents
- Disable memory entirely: `{ "plugins": { "slots": { "memory": "none" } } }`

### All 9 MCP Tools
Core (auto-registered): `memory_recall`, `memory_store`, `memory_forget`, `memory_update`
Management (opt-in): `memory_stats`, `memory_list`
Self-improvement (opt-in): `self_improvement_log`, `self_improvement_extract_skill`, `self_improvement_review`

### Self-Improvement Governance
- `LEARNINGS.md` (IDs: `LRN-YYYYMMDD-XXX`) and `ERRORS.md` (IDs: `ERR-YYYYMMDD-XXX`)
- Entry lifecycle: `pending → resolved → promoted_to_skill`
- Skill scaffold generation from learning entries

### CLI Reference
Full coverage of all `openclaw memory-pro` commands: `list`, `search`, `stats`, `delete`, `delete-bulk`, `export`, `import`, `reembed`, `upgrade`, `migrate`

### Ollama Local Deployment (Plan D)
- Step-by-step model pull commands
- Ollama health check and embedding endpoint verification
- JSON output reliability notes per model
- Remote Ollama host configuration
- Fallback when Smart Extraction fails with local LLM

### Iron Rules & Slash Commands
- 5 Iron Rules for AI agents (dual-layer storage, LanceDB hygiene, recall-before-retry, etc.)
- `/lesson` and `/remember` custom slash command templates for `AGENTS.md`

---

## What's in `references/full-reference.md`

Deep technical content loaded only when needed:

- **Database schema**: LanceDB `memories` table fields and metadata keys
- **Source file map**: All 31 source files with sizes and responsibilities
- **Retrieval pipeline**: Full scoring formula chain with all 9 parameters and defaults
- **Weibull decay formulas**: `recency = exp(-lambda × daysSince^beta)` with tier-specific parameters
- **Embedding config interface**: All `EmbeddingConfig` options
- **Document chunking**: 5-level splitting hierarchy, smart chunking math
- **Smart metadata system**: Three-tier content fields, bounded array limits, normalization functions
- **LLM client internals**: Temperature, response parsing, error recovery strategy
- **Noise filter details**: 5 built-in noise categories, auto-learning prototypes (bank cap: 200)
- **Adaptive retrieval full logic**: Skip/force conditions with CJK equivalents
- **Access tracking & reinforcement**: Debounce timer, logarithmic reinforcement curve
- **Reflection storage subsystem**: 4 storage types, importance weights, dedup threshold

---

## About memory-lancedb-pro

The underlying plugin is maintained at [CortexReach/memory-lancedb-pro](https://github.com/CortexReach/memory-lancedb-pro).

Key specs:
- **Version**: 1.1.0-beta.8
- **Storage**: LanceDB (embedded, no separate server)
- **Retrieval**: Hybrid vector + BM25 with RRF fusion
- **Node.js**: 22.16+ required, 24 recommended
- **License**: MIT

---

## License

MIT

---

## Buy Me a Coffee

[!["Buy Me A Coffee"](https://storage.ko-fi.com/cdn/kofi2.png?v=3)](https://ko-fi.com/aila)

## Contact

<img src="assets/wechat-qrcode.jpeg" width="200" alt="WeChat QR Code" />
