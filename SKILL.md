---
name: memory-lancedb-pro
description: This skill should be used when working with memory-lancedb-pro, a production-grade long-term memory MCP plugin for OpenClaw AI agents. Use when installing, configuring, or using any feature of memory-lancedb-pro including Smart Extraction, hybrid retrieval, memory lifecycle management, multi-scope isolation, self-improvement governance, or any MCP memory tools (memory_recall, memory_store, memory_forget, memory_update, memory_stats, memory_list, self_improvement_log, self_improvement_extract_skill, self_improvement_review).
---

# memory-lancedb-pro

Production-grade long-term memory system (v1.1.0-beta.8) for OpenClaw AI agents. Provides persistent, intelligent memory storage using LanceDB with hybrid vector + BM25 retrieval, LLM-powered Smart Extraction, Weibull decay lifecycle, and multi-scope isolation.

For full technical details (thresholds, formulas, database schema, source file map), see `references/full-reference.md`.

---

## Applying the Optimal Config (Step-by-Step Workflow)

When the user says "help me enable the best config", "apply optimal configuration", or similar, follow this exact procedure:

### Step 1 — Present configuration plans and let user choose

Present these three plans in a clear comparison, then ask the user to pick one:

---

**Plan A — 🏆 Full Power (Best Quality)**
- Embedding: Jina `jina-embeddings-v5-text-small` (task-aware, 1024-dim)
- Reranker: Jina `jina-reranker-v3` (cross-encoder, same key)
- LLM: OpenAI `gpt-4o-mini` (Smart Extraction)
- Keys needed: `JINA_API_KEY` + `OPENAI_API_KEY`
- Get keys: Jina → https://jina.ai/api-key · OpenAI → https://platform.openai.com/api-keys
- Cost: Both paid (Jina has free tier with limited quota)
- Best for: Production deployments, highest retrieval quality

**Plan B — 💰 Budget (Free Reranker)**
- Embedding: Jina `jina-embeddings-v5-text-small`
- Reranker: SiliconFlow `BAAI/bge-reranker-v2-m3` (free tier available)
- LLM: OpenAI `gpt-4o-mini`
- Keys needed: `JINA_API_KEY` + `SILICONFLOW_API_KEY` + `OPENAI_API_KEY`
- Get keys: Jina → https://jina.ai/api-key · SiliconFlow → https://cloud.siliconflow.cn/account/ak · OpenAI → https://platform.openai.com/api-keys
- Cost: Jina embedding paid, SiliconFlow reranker free tier, OpenAI paid
- Best for: Cost-sensitive deployments that still want reranking

**Plan C — 🟢 Simple (OpenAI Only)**
- Embedding: OpenAI `text-embedding-3-small`
- Reranker: None (vector+BM25 fusion only, no cross-encoder)
- LLM: OpenAI `gpt-4o-mini`
- Keys needed: `OPENAI_API_KEY` only
- Get key: https://platform.openai.com/api-keys
- Cost: OpenAI paid only
- Best for: Users who already have OpenAI and want minimal setup

**Plan D — 🖥️ Fully Local (Ollama, No API Keys)**
- Embedding: Ollama `nomic-embed-text` (768-dim, local at `http://localhost:11434/v1`)
- Reranker: **None** — Ollama has no cross-encoder reranker; retrieval uses vector+BM25 fusion only
- LLM: Ollama via OpenAI-compatible endpoint — recommended models with reliable JSON output: `qwen2.5:7b`, `llama3.2`, `mistral`
- Keys needed: **None** — fully local, no external API calls
- Prerequisites:
  - Ollama installed: https://ollama.com/download
  - Models pulled (see Step 4 below)
  - Ollama running: macOS = launch the app from Applications; Linux = `systemctl start ollama` or `ollama serve`
- Cost: Free (hardware only)
- RAM requirements: nomic-embed-text ~270MB; qwen2.5:7b ~4.7GB; llama3.2 ~2GB
- Trade-offs: No cross-encoder reranking = lower retrieval precision than Plans A/B; Smart Extraction quality depends on local LLM — if extraction produces garbage, set `"smartExtraction": false`
- Best for: Privacy-sensitive deployments, air-gapped environments, zero API cost

---

After user selects a plan, ask in one message:
1. Do you already have the required API key(s)?
2. Are the env vars already set in your OpenClaw Gateway process? (If unsure, answer No)
3. Where is your `openclaw.json`? (Skip if you want me to find it automatically)

If the user already stated their provider/keys in context, skip asking and proceed.

### Step 2 — Find openclaw.json

Check these locations in order:
```bash
# Most common locations
ls ~/.openclaw/openclaw.json
ls ~/openclaw.json
# Ask the gateway where it's reading config from
openclaw config get --show-path 2>/dev/null || echo "not found"
```

If not found, ask the user for the path.

### Step 3 — Read current config

```bash
# Read and display current plugins config before changing anything
openclaw config get plugins.entries.memory-lancedb-pro 2>/dev/null
openclaw config get plugins.slots.memory 2>/dev/null
```

**Check what already exists** — never blindly overwrite existing settings.

### Step 4 — Build the merged config based on chosen plan

Use the config block for the chosen plan. Substitute actual API keys inline if the user provided them directly; keep `${ENV_VAR}` syntax if they confirmed env vars are set in the gateway process.

**Plan A config (`plugins.entries.memory-lancedb-pro.config`):**
```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  },
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "rerank": "cross-encoder",
    "rerankProvider": "jina",
    "rerankModel": "jina-reranker-v3",
    "rerankEndpoint": "https://api.jina.ai/v1/rerank",
    "rerankApiKey": "${JINA_API_KEY}",
    "candidatePoolSize": 12,
    "minScore": 0.6,
    "hardMinScore": 0.62,
    "filterNoise": true
  },
  "sessionMemory": { "enabled": false }
}
```

**Plan B config:**
```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  },
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "rerank": "cross-encoder",
    "rerankProvider": "siliconflow",
    "rerankModel": "BAAI/bge-reranker-v2-m3",
    "rerankEndpoint": "https://api.siliconflow.com/v1/rerank",
    "rerankApiKey": "${SILICONFLOW_API_KEY}",
    "candidatePoolSize": 12,
    "minScore": 0.5,
    "hardMinScore": 0.55,
    "filterNoise": true
  },
  "sessionMemory": { "enabled": false }
}
```

**Plan C config:**
```json
{
  "embedding": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "text-embedding-3-small",
    "baseURL": "https://api.openai.com/v1"
  },
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "filterNoise": true,
    "minScore": 0.3,
    "hardMinScore": 0.35
  },
  "sessionMemory": { "enabled": false }
}
```

**Plan D config (replace `qwen2.5:7b` with your preferred local LLM):**
```json
{
  "embedding": {
    "apiKey": "ollama",
    "model": "nomic-embed-text",
    "baseURL": "http://localhost:11434/v1",
    "dimensions": 768
  },
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 4000,
  "llm": {
    "apiKey": "ollama",
    "model": "qwen2.5:7b",
    "baseURL": "http://localhost:11434/v1"
  },
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "filterNoise": true,
    "minScore": 0.25,
    "hardMinScore": 0.28
  },
  "sessionMemory": { "enabled": false }
}
```

**Plan D prerequisites — run BEFORE applying config:**
```bash
# 1. Verify Ollama is running (should return JSON with model list)
curl http://localhost:11434/api/tags

# 2. Pull embedding model
ollama pull nomic-embed-text

# 3. Pull LLM for Smart Extraction (choose one):
ollama pull qwen2.5:7b    # recommended: good JSON output, ~4.7GB
ollama pull llama3.2      # alternative: ~2GB, lighter
ollama pull mistral       # alternative: ~4.1GB

# 4. Verify both models are installed
ollama list

# 5. Quick sanity check — embedding endpoint works:
curl http://localhost:11434/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"nomic-embed-text","input":"test"}'
# Should return a JSON with a 768-element vector
```

**If Smart Extraction produces garbled/invalid output:** The local LLM may not support structured JSON reliably. Disable it and use simpler storage:
```json
{ "smartExtraction": false }
```

**If Ollama is on a different host or Docker:** Replace `http://localhost:11434/v1` with the actual host, e.g. `http://192.168.1.100:11434/v1`. Also set `OLLAMA_HOST=0.0.0.0` in the Ollama process to allow remote connections.

For the **`plugins.entries.memory-lancedb-pro.config`** block, merge into the existing `openclaw.json` rather than replacing the whole file. Use a targeted edit of only the memory plugin config section.

### Step 5 — Apply the config

Read the current `openclaw.json` first, then apply a surgical edit to the `plugins.entries.memory-lancedb-pro` section. Use the template that matches your installation method:

**Method 1 — `openclaw plugins install` (plugin was installed via the plugin manager):**
No `load.paths` or `allow` needed — the plugin manager already registered the plugin.
```json
{
  "plugins": {
    "slots": { "memory": "memory-lancedb-pro" },
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": {
          "<<OPTIMAL CONFIG HERE>>"
        }
      }
    }
  }
}
```

**Method 2 — git clone with manual path (workspace plugin):**
Both `load.paths` AND `allow` are required — workspace plugins are disabled by default.
```json
{
  "plugins": {
    "load": { "paths": ["plugins/memory-lancedb-pro"] },
    "allow": ["memory-lancedb-pro"],
    "slots": { "memory": "memory-lancedb-pro" },
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": {
          "<<OPTIMAL CONFIG HERE>>"
        }
      }
    }
  }
}
```

### Step 6 — Validate and restart

```bash
openclaw config validate
openclaw gateway restart
openclaw logs --follow --plain | rg "memory-lancedb-pro"
```

Expected output confirms:
- `memory-lancedb-pro: smart extraction enabled`
- `memory-lancedb-pro@...: plugin registered`

### Step 7 — Verify

```bash
openclaw plugins info memory-lancedb-pro
openclaw hooks list --json | grep -E "before_agent_start|agent_end|command:new"
openclaw memory-pro stats
```

Then do a quick smoke test:
1. Store: call `memory_store` with `text: "test memory for verification"`
2. Recall: call `memory_recall` with `query: "test memory"`
3. Confirm the memory is returned

---

## Installation

### Requirements
- Node.js 24 recommended (Node 22 LTS minimum, `22.16+`)
- LanceDB ≥ 0.26.2
- OpenAI SDK ≥ 6.21.0
- TypeBox 0.34.48

### Install Method 1 — via OpenClaw plugin manager (recommended)

```bash
# Install directly from npm — auto-copies to ~/.openclaw/extensions/ and enables
openclaw plugins install memory-lancedb-pro@beta

# Or install from a local clone (copies into extensions dir and enables)
git clone https://github.com/CortexReach/memory-lancedb-pro.git /tmp/memory-lancedb-pro
openclaw plugins install /tmp/memory-lancedb-pro
```

Then bind the memory slot and add your config (see Configuration section below):
```json
{
  "plugins": {
    "slots": { "memory": "memory-lancedb-pro" },
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": { "<<your config here>>" }
      }
    }
  }
}
```

Restart and verify:
```bash
openclaw gateway restart
openclaw plugins info memory-lancedb-pro
```

### Install Method 2 — git clone with manual path (Path A for development)

> ⚠️ **Critical**: Workspace plugins (git-cloned paths) are **disabled by default** in OpenClaw. You MUST explicitly enable them.

```bash
# 1. Clone into workspace
cd /path/to/your/openclaw/workspace
git clone https://github.com/CortexReach/memory-lancedb-pro.git plugins/memory-lancedb-pro
cd plugins/memory-lancedb-pro && npm install
```

Add to `openclaw.json` — the `enabled: true` and the `allow` entry are both required:
```json
{
  "plugins": {
    "load": { "paths": ["plugins/memory-lancedb-pro"] },
    "allow": ["memory-lancedb-pro"],
    "slots": { "memory": "memory-lancedb-pro" },
    "entries": {
      "memory-lancedb-pro": {
        "enabled": true,
        "config": {
          "embedding": {
            "apiKey": "${JINA_API_KEY}",
            "model": "jina-embeddings-v5-text-small",
            "baseURL": "https://api.jina.ai/v1",
            "dimensions": 1024,
            "taskQuery": "retrieval.query",
            "taskPassage": "retrieval.passage",
            "normalized": true
          }
        }
      }
    }
  }
}
```

Validate and restart:
```bash
openclaw config validate
openclaw gateway restart
openclaw logs --follow --plain | rg "memory-lancedb-pro"
```

Expected log output:
- `memory-lancedb-pro: smart extraction enabled`
- `memory-lancedb-pro@...: plugin registered`

### Install Method 3 — Existing deployments (Path B)

Use **absolute paths** in `plugins.load.paths`. Add to `plugins.allow`. Bind memory slot: `plugins.slots.memory = "memory-lancedb-pro"`. Set `plugins.entries.memory-lancedb-pro.enabled: true`.

### Upgrading plugin code vs. data

**Command distinction (important):**

| Command | Purpose |
|---------|---------|
| `openclaw plugins update memory-lancedb-pro` | Update the **plugin code** (npm-installed only) |
| `openclaw plugins update --all` | Update all npm-installed plugins |
| `openclaw memory-pro upgrade` | Migrate **memory data** from older format to v1.1.0 schema |
| `openclaw memory-pro migrate` | Only for built-in `memory-lancedb` → Pro data migration |
| `openclaw memory-pro reembed` | Rebuild embeddings after changing embedding model |

Safe data upgrade sequence:
```bash
openclaw memory-pro export --scope global --output memories-backup.json
openclaw memory-pro upgrade --dry-run
openclaw memory-pro upgrade
openclaw memory-pro stats
openclaw memory-pro search "your known keyword" --scope global --limit 5
```

### Plugin management commands

```bash
openclaw plugins list                           # show all discovered plugins
openclaw plugins info memory-lancedb-pro        # show plugin status and config
openclaw plugins enable memory-lancedb-pro      # enable a disabled plugin
openclaw plugins disable memory-lancedb-pro     # disable without removing
openclaw plugins update memory-lancedb-pro      # update npm-installed plugin
openclaw plugins update --all                   # update all npm plugins
openclaw plugins doctor                         # health check for all plugins
openclaw plugins install ./path/to/plugin       # install local plugin (copies + enables)
openclaw plugins install @scope/plugin@beta     # install from npm registry
openclaw plugins install -l ./path/to/plugin    # symlink for dev (no copy)
```

### Easy-to-Miss Setup Steps

1. **Workspace plugins are DISABLED by default**: After git clone, you MUST add `plugins.allow: ["memory-lancedb-pro"]` AND `plugins.entries.memory-lancedb-pro.enabled: true` — without these the plugin silently does not load.
2. **Env vars in gateway process**: `${OPENAI_API_KEY}` requires env vars set in the *OpenClaw Gateway service* process—not just your shell.
3. **Absolute vs. relative paths**: For existing deployments, always use absolute paths in `plugins.load.paths`.
4. **jiti cache invalidation**: After modifying `.ts` files under plugins, run `rm -rf /tmp/jiti/` BEFORE `openclaw gateway restart`.
5. **Unknown plugin id = error**: OpenClaw treats unknown ids in `entries`, `allow`, `deny`, or `slots` as validation errors. The plugin id must be discoverable before referencing it.
6. **Separate LLM config**: If embedding and LLM use different providers, configure the `llm` section separately — it falls back to embedding key/URL otherwise.
7. **Scope isolation**: Multi-scope requires explicit `scopes.agentAccess` mapping — without it, agents only see `global` scope.
8. **Session memory hook**: Fires on `/new` command — test with an actual `/new` invocation.
9. **Reranker credentials**: When switching providers, update both `rerankApiKey` AND `rerankEndpoint`.
10. **Config check before assuming defaults**: Run `openclaw config get plugins.entries.memory-lancedb-pro` to verify what's actually loaded.
11. **Custom config/state paths via env vars**: OpenClaw respects the following environment variables for custom paths:
    - `OPENCLAW_HOME` — sets the root config/data directory (default: `~/.openclaw/`)
    - `OPENCLAW_CONFIG_PATH` — absolute path to `openclaw.json` override
    - `OPENCLAW_STATE_DIR` — override for runtime state/data directory
    Set these in the OpenClaw Gateway process's environment if the default `~/.openclaw/` path is not appropriate.

### Post-Installation Verification
```bash
openclaw doctor                                 # full health check (recommended)
openclaw config validate                        # config schema check only
openclaw plugins info memory-lancedb-pro        # plugin status
openclaw plugins doctor                         # plugin-specific health
openclaw hooks list --json | grep memory        # confirm hooks registered
openclaw memory-pro stats
openclaw memory-pro list --scope global --limit 5
```

Full smoke test checklist:
- ✅ Plugin info shows `enabled: true` and config loaded
- ✅ Hooks include `before_agent_start`, `agent_end`, `command:new`
- ✅ One `memory_store` → `memory_recall` round trip via tools
- ✅ One exact-ID search hit
- ✅ One natural-language search hit
- ✅ If session memory enabled: one real `/new` test

---

## Configuration

### Minimal Quick-Start
```json
{
  "embedding": {
    "provider": "openai-compatible",
    "apiKey": "${OPENAI_API_KEY}",
    "model": "text-embedding-3-small"
  },
  "autoCapture": true,
  "autoRecall": true,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "sessionMemory": { "enabled": false }
}
```

**Note:** `autoRecall` is **disabled by default** in the plugin schema — explicitly set it to `true` for new deployments.

### Optimal Production Config (recommended)
Uses Jina for both embedding and reranking — best retrieval quality:

```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  },
  "dbPath": "~/.openclaw/memory/lancedb-pro",
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "enableManagementTools": false,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "rerank": "cross-encoder",
    "rerankProvider": "jina",
    "rerankModel": "jina-reranker-v3",
    "rerankEndpoint": "https://api.jina.ai/v1/rerank",
    "rerankApiKey": "${JINA_API_KEY}",
    "candidatePoolSize": 12,
    "minScore": 0.6,
    "hardMinScore": 0.62,
    "filterNoise": true,
    "lengthNormAnchor": 500,
    "timeDecayHalfLifeDays": 60,
    "reinforcementFactor": 0.5,
    "maxHalfLifeMultiplier": 3
  },
  "sessionMemory": { "enabled": false, "messageCount": 15 }
}
```

**Why these settings excel:**
- **Jina embeddings**: Task-aware vectors (`taskQuery`/`taskPassage`) optimized for retrieval
- **Hybrid mode 0.7/0.3**: Balances semantic understanding with exact keyword matching
- **Jina reranker v3**: Cross-encoder reranking significantly improves relevance
- **`candidatePoolSize: 12` + `minScore: 0.6`**: Aggressive filtering reduces noise
- **`captureAssistant: false`**: Prevents storing agent-generated boilerplate
- **`sessionMemory: false`**: Avoids polluting retrieval with session summaries

### Full Config (all options)
```json
{
  "embedding": {
    "apiKey": "${JINA_API_KEY}",
    "model": "jina-embeddings-v5-text-small",
    "baseURL": "https://api.jina.ai/v1",
    "dimensions": 1024,
    "taskQuery": "retrieval.query",
    "taskPassage": "retrieval.passage",
    "normalized": true
  },
  "dbPath": "~/.openclaw/memory/lancedb-pro",
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini",
    "baseURL": "https://api.openai.com/v1"
  },
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "enableManagementTools": false,
  "retrieval": {
    "mode": "hybrid",
    "vectorWeight": 0.7,
    "bm25Weight": 0.3,
    "minScore": 0.3,
    "hardMinScore": 0.35,
    "rerank": "cross-encoder",
    "rerankProvider": "jina",
    "rerankModel": "jina-reranker-v3",
    "rerankEndpoint": "https://api.jina.ai/v1/rerank",
    "rerankApiKey": "${JINA_API_KEY}",
    "candidatePoolSize": 20,
    "recencyHalfLifeDays": 14,
    "recencyWeight": 0.1,
    "filterNoise": true,
    "lengthNormAnchor": 500,
    "timeDecayHalfLifeDays": 60,
    "reinforcementFactor": 0.5,
    "maxHalfLifeMultiplier": 3
  },
  "scopes": {
    "default": "global",
    "definitions": {
      "global": { "description": "Shared knowledge" },
      "agent:discord-bot": { "description": "Discord bot private" }
    },
    "agentAccess": {
      "discord-bot": ["global", "agent:discord-bot"]
    }
  },
  "sessionStrategy": "none",
  "memoryReflection": {
    "storeToLanceDB": true,
    "injectMode": "inheritance+derived",
    "agentId": "memory-distiller",
    "messageCount": 120,
    "maxInputChars": 24000,
    "thinkLevel": "medium"
  },
  "selfImprovement": {
    "enabled": true,
    "beforeResetNote": true,
    "ensureLearningFiles": true
  },
  "mdMirror": { "enabled": false },
  "decay": {
    "recencyHalfLifeDays": 30,
    "recencyWeight": 0.4,
    "frequencyWeight": 0.3,
    "intrinsicWeight": 0.3,
    "betaCore": 0.8,
    "betaWorking": 1.0,
    "betaPeripheral": 1.3
  },
  "tier": {
    "coreAccessThreshold": 10,
    "coreCompositeThreshold": 0.7,
    "coreImportanceThreshold": 0.8,
    "workingAccessThreshold": 3,
    "workingCompositeThreshold": 0.4,
    "peripheralCompositeThreshold": 0.15,
    "peripheralAgeDays": 60
  }
}
```

---

## Configuration Field Reference

### Embedding
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `apiKey` | string | — | API key (supports `${ENV_VAR}`); array for multi-key failover |
| `model` | string | — | Model identifier |
| `baseURL` | string | provider default | API endpoint |
| `dimensions` | number | provider default | Vector dimensionality |
| `taskQuery` | string | — | Task hint for query embeddings (`retrieval.query`) |
| `taskPassage` | string | — | Task hint for passage embeddings (`retrieval.passage`) |
| `normalized` | boolean | false | Request L2-normalized embeddings |
| `provider` | string | `openai-compatible` | Provider type selector |
| `chunking` | boolean | true | Auto-chunk documents exceeding embedding context limits |

### Top-Level
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dbPath` | string | `~/.openclaw/memory/lancedb-pro` | LanceDB data directory |
| `autoCapture` | boolean | true | Auto-extract memories after agent replies (via `agent_end` hook) |
| `autoRecall` | boolean | **false** (schema default) | Inject memories before agent processing — **set to true explicitly** |
| `captureAssistant` | boolean | false | Include assistant messages in extraction |
| `smartExtraction` | boolean | true | LLM-powered 6-category extraction |
| `extractMinMessages` | number | 2 | Min conversation turns before extraction triggers |
| `extractMaxChars` | number | 8000 | Max context chars sent to extraction LLM |
| `enableManagementTools` | boolean | false | Register CLI management tools as agent tools |
| `autoRecallMinLength` | number | 15 | Min prompt chars to trigger auto-recall (6 for CJK) |
| `autoRecallMinRepeated` | number | 0 | Min turns before same memory can re-inject in same session |
| `sessionStrategy` | string | `none` | Session pipeline: `memoryReflection` / `systemSessionMemory` / `none` |

### LLM (for Smart Extraction)
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `llm.apiKey` | string | falls back to `embedding.apiKey` | LLM API key |
| `llm.model` | string | `openai/gpt-oss-120b` | LLM model for extraction |
| `llm.baseURL` | string | falls back to `embedding.baseURL` | LLM endpoint |

### Retrieval
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | string | `hybrid` | `hybrid` / `vector` (`bm25`-only mode does not exist in schema) |
| `vectorWeight` | number | 0.7 | Weight for vector search |
| `bm25Weight` | number | 0.3 | Weight for BM25 full-text search |
| `minScore` | number | 0.3 | Minimum relevance threshold |
| `hardMinScore` | number | 0.35 | Hard cutoff post-reranking |
| `rerank` | string | `cross-encoder` | Reranking strategy |
| `rerankProvider` | string | `jina` | `jina` / `siliconflow` / `voyage` / `pinecone` / `dashscope` |
| `rerankModel` | string | `jina-reranker-v3` | Reranker model name |
| `rerankEndpoint` | string | provider default | Reranker API URL |
| `rerankApiKey` | string | — | Reranker API key |
| `candidatePoolSize` | number | 20 | Candidates to rerank before final filter |
| `recencyHalfLifeDays` | number | 14 | Freshness decay half-life |
| `recencyWeight` | number | 0.1 | Weight of recency in scoring |
| `timeDecayHalfLifeDays` | number | 60 | Memory age decay factor |
| `reinforcementFactor` | number | 0.5 | Access-based half-life multiplier (0–2, set 0 to disable) |
| `maxHalfLifeMultiplier` | number | 3 | Hard cap on reinforcement boost |
| `filterNoise` | boolean | true | Filter refusals, greetings, etc. |
| `lengthNormAnchor` | number | 500 | Reference length for normalization (chars) |

**Access reinforcement note:** Reinforcement is whitelisted to `source: "manual"` only — auto-recall does NOT strengthen memories, preventing noise amplification.

### Session Strategy (v1.1.0+)

Use `sessionStrategy` (top-level field) to configure the session pipeline:

| Value | Behavior |
|-------|----------|
| `"none"` (default) | Session summaries disabled |
| `"memoryReflection"` | Advanced LLM-powered reflection with inheritance/derived injection |
| `"systemSessionMemory"` | Built-in session memory (simpler, legacy-equivalent) |

**`memoryReflection` config** (used when `sessionStrategy: "memoryReflection"`):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `storeToLanceDB` | boolean | true | Persist reflections to LanceDB |
| `writeLegacyCombined` | boolean | true | Also write legacy combined row |
| `injectMode` | string | `inheritance+derived` | `inheritance-only` / `inheritance+derived` |
| `agentId` | string | — | Dedicated reflection agent (e.g. `"memory-distiller"`) |
| `messageCount` | number | 120 | Messages to include in reflection |
| `maxInputChars` | number | 24000 | Max chars sent to reflection LLM |
| `timeoutMs` | number | 20000 | Reflection LLM timeout (ms) |
| `thinkLevel` | string | `medium` | Reasoning depth: `off` / `minimal` / `low` / `medium` / `high` |
| `errorReminderMaxEntries` | number | 3 | Max error entries injected into reflection |
| `dedupeErrorSignals` | boolean | true | Deduplicate error signals before injection |

### Session Memory (deprecated — legacy compat only)

> ⚠️ **`sessionMemory` is a legacy compatibility shim since v1.1.0.** Prefer `sessionStrategy` instead.
> - `sessionMemory.enabled: true` → maps to `sessionStrategy: "systemSessionMemory"`
> - `sessionMemory.enabled: false` → maps to `sessionStrategy: "none"`

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `sessionMemory.enabled` | boolean | false | Legacy: enable session summaries on `/new` |
| `sessionMemory.messageCount` | number | 15 | Legacy: maps to `memoryReflection.messageCount` |

### Self-Improvement Governance

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `selfImprovement.enabled` | boolean | **true** | Enable self-improvement tools (`self_improvement_log` etc.) — **on by default** |
| `selfImprovement.beforeResetNote` | boolean | true | Inject learning reminder before session reset |
| `selfImprovement.skipSubagentBootstrap` | boolean | true | Skip bootstrap for sub-agents |
| `selfImprovement.ensureLearningFiles` | boolean | true | Auto-create `LEARNINGS.md` / `ERRORS.md` if missing |

**Tool activation rules:**
- `self_improvement_log`: requires `selfImprovement.enabled: true` (default — active unless explicitly disabled)
- `self_improvement_extract_skill` + `self_improvement_review`: additionally require `enableManagementTools: true`

### Markdown Mirror

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mdMirror.enabled` | boolean | false | Mirror memory entries as `.md` files |
| `mdMirror.dir` | string | — | Directory for markdown mirror files |

### Decay
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `decay.recencyHalfLifeDays` | number | 30 | Base Weibull decay half-life |
| `decay.recencyWeight` | number | 0.4 | Weight of recency in lifecycle score (distinct from `retrieval.recencyWeight`) |
| `decay.frequencyWeight` | number | 0.3 | Weight of access frequency |
| `decay.intrinsicWeight` | number | 0.3 | Weight of importance × confidence |
| `decay.betaCore` | number | 0.8 | Weibull shape for core memories |
| `decay.betaWorking` | number | 1.0 | Weibull shape for working memories |
| `decay.betaPeripheral` | number | 1.3 | Weibull shape for peripheral memories |
| `decay.coreDecayFloor` | number | 0.9 | Minimum lifecycle score for core tier |
| `decay.workingDecayFloor` | number | 0.7 | Minimum lifecycle score for working tier |
| `decay.peripheralDecayFloor` | number | 0.5 | Minimum lifecycle score for peripheral tier |
| `decay.staleThreshold` | number | 0.3 | Score below which a memory is considered stale |
| `decay.searchBoostMin` | number | 0.3 | Minimum search boost applied to lifecycle score |
| `decay.importanceModulation` | number | 1.5 | Multiplier for importance in lifecycle score |

### Tier Management
| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `tier.coreAccessThreshold` | number | 10 | Access count for core promotion |
| `tier.coreCompositeThreshold` | number | 0.7 | Lifecycle score for core promotion |
| `tier.coreImportanceThreshold` | number | 0.8 | Minimum importance for core promotion |
| `tier.workingAccessThreshold` | number | 3 | Access count for working promotion |
| `tier.workingCompositeThreshold` | number | 0.4 | Lifecycle score for working promotion |
| `tier.peripheralCompositeThreshold` | number | 0.15 | Score below which demotion occurs |
| `tier.peripheralAgeDays` | number | 60 | Age threshold for stale memory demotion |

---

## MCP Tools

### Core Tools (auto-registered)

**`memory_recall`** — Search long-term memory via hybrid retrieval
| Parameter | Type | Required | Default | Notes |
|-----------|------|----------|---------|-------|
| `query` | string | yes | — | Search query |
| `limit` | number | no | 5 | Max 20 |
| `scope` | string | no | — | Specific scope to search |
| `category` | enum | no | — | `preference\|fact\|decision\|entity\|reflection\|other` |

**`memory_store`** — Save information to long-term memory
| Parameter | Type | Required | Default | Notes |
|-----------|------|----------|---------|-------|
| `text` | string | yes | — | Information to remember |
| `importance` | number | no | 0.7 | Range 0–1 |
| `category` | enum | no | — | Memory classification |
| `scope` | string | no | `agent:<id>` | Target scope |

**`memory_forget`** — Delete memories by search or direct ID
| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `query` | string | one of | Search query to locate memory |
| `memoryId` | string | one of | Full UUID or 8+ char prefix |
| `scope` | string | no | Scope for search/deletion |

**`memory_update`** — Update memory (preserves original timestamp; `preference`/`entity` text updates create a new versioned row preserving history)
| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `memoryId` | string | yes | Full UUID or 8+ char prefix |
| `text` | string | no | New content (triggers re-embedding; `preference`/`entity` creates supersede version) |
| `importance` | number | no | New score 0–1 |
| `category` | enum | no | New classification |

### Management Tools (enable with `enableManagementTools: true`)

**`memory_stats`** — Usage statistics
- `scope` (string, optional): Filter by scope

**`memory_list`** — List recent memories with filtering
- `limit` (number, optional, default 10, max 50), `scope`, `category`, `offset` (pagination)

### Self-Improvement Tools

> `self_improvement_log` is **enabled by default** (`selfImprovement.enabled: true`). `self_improvement_extract_skill` and `self_improvement_review` additionally require `enableManagementTools: true`.

**`self_improvement_log`** — Log learning/error entries into LEARNINGS.md / ERRORS.md
| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `type` | enum | yes | `"learning"` or `"error"` |
| `summary` | string | yes | One-line summary |
| `details` | string | no | Detailed context |
| `suggestedAction` | string | no | Action to prevent recurrence |
| `category` | string | no | Learning: `correction\|best_practice\|knowledge_gap`; Error: `correction\|bug_fix\|integration_issue` |
| `area` | string | no | `frontend\|backend\|infra\|tests\|docs\|config` |
| `priority` | string | no | `low\|medium\|high\|critical` |

**`self_improvement_extract_skill`** — Create skill scaffold from a learning entry
| Parameter | Type | Required | Default | Notes |
|-----------|------|----------|---------|-------|
| `learningId` | string | yes | — | Format `LRN-YYYYMMDD-001` or `ERR-*` |
| `skillName` | string | yes | — | Lowercase with hyphens |
| `sourceFile` | enum | no | `LEARNINGS.md` | `LEARNINGS.md\|ERRORS.md` |
| `outputDir` | string | no | `"skills"` | Relative output directory |

**`self_improvement_review`** — Summarize governance backlog (no parameters)

---

## Smart Extraction

LLM-powered automatic memory classification and storage triggered after conversations.

### Enable
```json
{
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 8000,
  "llm": {
    "apiKey": "${OPENAI_API_KEY}",
    "model": "gpt-4o-mini"
  }
}
```

Minimal (reuses embedding API key — no separate `llm` block needed):
```json
{
  "embedding": { "apiKey": "${OPENAI_API_KEY}", "model": "text-embedding-3-small" },
  "smartExtraction": true
}
```

Disable: `{ "smartExtraction": false }`

### 6-Category Classification

| Input Category | Stored As | Dedup Behavior |
|---------------|-----------|----------------|
| Profile | `fact` | Always merge (auto-consolidates) |
| Preferences | `preference` | Conditional merge |
| Entities | `entity` | Conditional merge |
| Events | `decision` | Append-only (no merge) |
| Cases | `fact` | Append-only (no merge) |
| Patterns | `other` | Conditional merge |

### L0/L1/L2 Layered Content per Memory
- **L0 (Abstract)**: Single-sentence index (min 5 chars)
- **L1 (Overview)**: Structured markdown summary
- **L2 (Content)**: Full narrative detail

### Two-Stage Deduplication
1. **Vector pre-filter**: Similarity ≥ 0.7 finds candidates
2. **LLM decision**: `CREATE | MERGE | SKIP | SUPPORT | CONTEXTUALIZE | CONTRADICT`

---

## Embedding Providers

| Provider | Model | Base URL | Dimensions | Notes |
|----------|-------|----------|-----------|-------|
| Jina (recommended) | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 | Task-aware, normalized |
| OpenAI | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 | Widely compatible |
| Google Gemini | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 | High-dim |
| Ollama (local) | `nomic-embed-text` | `http://localhost:11434/v1` | varies | Offline capable |

**Multi-key failover:** Set `apiKey` as an array for round-robin rotation on 429/503 errors.

---

## Reranker Providers

| Provider | `rerankProvider` | Endpoint | Model | Notes |
|----------|-----------------|----------|-------|-------|
| Jina (default) | `jina` | `https://api.jina.ai/v1/rerank` | `jina-reranker-v3` | Production-grade |
| SiliconFlow | `siliconflow` | `https://api.siliconflow.com/v1/rerank` | `BAAI/bge-reranker-v2-m3` | Free tier available |
| Voyage AI | `voyage` | `https://api.voyageai.com/v1/rerank` | `rerank-2.5` | Sends `{model, query, documents}`, no `top_n` |
| Pinecone | `pinecone` | `https://api.pinecone.io/rerank` | `bge-reranker-v2-m3` | Pinecone customers only |
| DashScope (Aliyun) | `dashscope` | `https://dashscope.aliyuncs.com/api/v1/services/rerank/text-rerank/text-rerank` | `gte-rerank-v2` | Alibaba Cloud |

Jina key can be reused for both embedding and reranking.

---

## Multi-Scope Isolation

| Scope Format | Description |
|-------------|-------------|
| `global` | Shared across all agents |
| `agent:<id>` | Agent-specific memories |
| `custom:<name>` | Custom-named scopes |
| `project:<id>` | Project-specific memories |
| `user:<id>` | User-specific memories |

Default access: `global` + `agent:<id>`. Multi-scope requires explicit `scopes.agentAccess` — see Full Config above.

**To disable memory entirely** (unbind the slot without removing the plugin):
```json
{ "plugins": { "slots": { "memory": "none" } } }
```

---

## Memory Lifecycle (Weibull Decay)

### Three Tiers

| Tier | Decay Floor | Beta | Behavior |
|------|-------------|------|----------|
| Core | 0.9 | 0.8 | Gentle sub-exponential decline |
| Working | 0.7 | 1.0 | Standard exponential (default) |
| Peripheral | 0.5 | 1.3 | Rapid super-exponential fade |

### Promotion/Demotion Rules
- **Peripheral → Working:** access ≥ 3 AND score ≥ 0.4
- **Working → Core:** access ≥ 10 AND score ≥ 0.7 AND importance ≥ 0.8
- **Working → Peripheral:** score < 0.15 OR (age > 60 days AND access < 3)
- **Core → Working:** score < 0.15 AND access < 3

---

## Hybrid Retrieval

**Fusion:** `weightedFusion = (vectorScore × 0.7) + (bm25Score × 0.3)`

**Pipeline:** RRF Fusion → Cross-Encoder Rerank → Lifecycle Decay Boost → Length Norm → Hard Min Score → MMR Diversity (cosine > 0.85 demoted)

**Reranking:** 60% cross-encoder score + 40% original fused score. Falls back to cosine similarity on API failure.

**Special BM25:** Preserves exact keyword matches (BM25 ≥ 0.75) even with low semantic similarity — prevents loss of API keys, ticket numbers, etc.

---

## Adaptive Retrieval Triggering

**Skip for:** greetings, slash commands, affirmations (yes/okay/thanks), continuations (go ahead/proceed), system messages, short queries (<15 chars English / <6 chars CJK without "?").

**Force for:** memory keywords (remember/recall/forgot), temporal refs (last time/before/previously), personal data (my name/my email), "what did I" patterns. CJK: "你记得", "之前".

---

## Noise Filtering

Auto-filters: agent denial phrases, meta-questions ("Do you remember?"), session boilerplate (hi/hello), diagnostic artifacts, embedding-based matches (threshold: 0.82). Minimum text: 5 chars.

---

## CLI Commands

```bash
# List & search
openclaw memory-pro list [--scope global] [--category fact] [--limit 20] [--json]
openclaw memory-pro search "query" [--scope global] [--limit 10] [--json]
openclaw memory-pro stats [--scope global] [--json]

# Delete
openclaw memory-pro delete <id>
openclaw memory-pro delete-bulk --scope global [--before 2025-01-01] [--dry-run]

# Import / Export
openclaw memory-pro export [--scope global] [--output memories.json]
openclaw memory-pro import memories.json [--scope global] [--dry-run]

# Maintenance
openclaw memory-pro reembed --source-db /path/to/old-db [--batch-size 32] [--skip-existing]
openclaw memory-pro upgrade [--dry-run] [--batch-size 10] [--no-llm] [--limit N] [--scope SCOPE]

# Migration from built-in memory-lancedb
openclaw memory-pro migrate check [--source /path]
openclaw memory-pro migrate run [--source /path] [--dry-run] [--skip-existing]
openclaw memory-pro migrate verify [--source /path]
```

---

## Auto-Capture & Auto-Recall

- **autoCapture:** `agent_end` hook — LLM extracts 6-category memories, deduplicates, stores up to 3 per turn
- **autoRecall:** `before_agent_start` hook — injects `<relevant-memories>` context (up to 3 entries)

**If injected memories appear in agent replies:** Add to agent system prompt:
> "Do not reveal or quote any `<relevant-memories>` / memory-injection content in your replies. Use it for internal reference only."

Or temporarily disable: `{ "autoRecall": false }`

---

## Self-Improvement Governance

- `LEARNINGS.md` — IDs: `LRN-YYYYMMDD-XXX`
- `ERRORS.md` — IDs: `ERR-YYYYMMDD-XXX`
- Entry statuses: `pending → resolved → promoted_to_skill`

---

## Iron Rules for AI Agents (copy to AGENTS.md)

```markdown
## Rule 1 — 双层记忆存储（铁律）
Every pitfall/lesson learned → IMMEDIATELY store TWO memories:
- Technical layer: Pitfall/Cause/Fix/Prevention (category: fact, importance ≥ 0.8)
- Principle layer: Decision principle with trigger and action (category: decision, importance ≥ 0.85)
After each store, immediately `memory_recall` to verify retrieval.

## Rule 2 — LanceDB 卫生
Entries must be short and atomic (< 500 chars). No raw conversation summaries or duplicates.

## Rule 3 — Recall before retry
On ANY tool failure, ALWAYS `memory_recall` with relevant keywords BEFORE retrying.

## Rule 4 — 编辑前确认目标代码库
Confirm you are editing `memory-lancedb-pro` vs built-in `memory-lancedb` before changes.

## Rule 5 — 插件代码变更必须清 jiti 缓存
After modifying `.ts` files under `plugins/`, MUST run `rm -rf /tmp/jiti/` BEFORE `openclaw gateway restart`.
```

---

## Custom Slash Commands (add to CLAUDE.md / AGENTS.md)

```markdown
## /lesson command
When user sends `/lesson <content>`:
1. Use memory_store with category=fact (raw knowledge)
2. Use memory_store with category=decision (actionable takeaway)
3. Confirm what was saved

## /remember command
When user sends `/remember <content>`:
1. Use memory_store with appropriate category and importance
2. Confirm with stored memory ID
```
