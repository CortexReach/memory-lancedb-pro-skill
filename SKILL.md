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
- Embedding: Ollama `mxbai-embed-large` (1024-dim, recommended) or `nomic-embed-text:v1.5` (768-dim, lighter)
- Reranker: **None** — Ollama has no cross-encoder reranker; retrieval uses vector+BM25 fusion only
- LLM: Ollama via OpenAI-compatible endpoint — recommended models with reliable JSON/structured output:
  - `qwen3:8b` (**recommended** — best JSON output, native structured output, ~5.2GB)
  - `qwen3:14b` (better quality, ~9GB, needs 16GB VRAM)
  - `llama4:scout` (multimodal MoE, 10M ctx, ~12GB)
  - `mistral-small3.2` (24B, 128K ctx, excellent instruction following, ~15GB)
  - `mistral-nemo` (12B, 128K ctx, efficient, ~7GB)
- Keys needed: **None** — fully local, no external API calls
- Prerequisites:
  - Ollama installed: https://ollama.com/download
  - Models pulled (see Step 5 below)
  - Ollama running: macOS = launch the app from Applications; Linux = `systemctl start ollama` or `ollama serve`
- Cost: Free (hardware only)
- RAM requirements: mxbai-embed-large ~670MB; qwen3:8b ~5.2GB; qwen3:14b ~9GB; llama4:scout ~12GB; mistral-small3.2 ~15GB
- Trade-offs: No cross-encoder reranking = lower retrieval precision than Plans A/B; Smart Extraction quality depends on local LLM — if extraction produces garbage, set `"smartExtraction": false`
- Best for: Privacy-sensitive deployments, air-gapped environments, zero API cost

---

After user selects a plan, ask in one message:
1. Please provide the required API key(s) for your chosen plan (paste directly, or say "already set as env vars")
2. Are the env vars already set in your OpenClaw Gateway process? (If unsure, answer No)
3. Where is your `openclaw.json`? (Skip if you want me to find it automatically)

If the user already stated their provider/keys in context, skip asking and proceed.

**Do NOT proceed to Step 2 until API keys have been collected and verified (Step 2 below).**

### Step 2 — Verify API Keys (MANDATORY — do not skip)

**Run ALL key checks for the chosen plan before touching any config.** If any check fails, STOP and tell the user which key failed and why. Do not proceed to Step 3.

**Plan A / Plan B — Jina embedding check:**
```bash
curl -s -o /dev/null -w "%{http_code}" \
  https://api.jina.ai/v1/embeddings \
  -H "Authorization: Bearer <JINA_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"jina-embeddings-v5-text-small","input":["test"]}'
```

**Plan A / B / C — OpenAI check:**
```bash
curl -s -o /dev/null -w "%{http_code}" \
  https://api.openai.com/v1/models \
  -H "Authorization: Bearer <OPENAI_API_KEY>"
```

**Plan B — SiliconFlow reranker check:**
```bash
curl -s -o /dev/null -w "%{http_code}" \
  https://api.siliconflow.com/v1/rerank \
  -H "Authorization: Bearer <SILICONFLOW_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-reranker-v2-m3","query":"test","documents":["test doc"]}'
```

**Plan D — Ollama check:**
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:11434/api/tags
```

**Interpret results:**

| HTTP code | Meaning | Action |
|-----------|---------|--------|
| `200` / `201` | Key valid, quota available | ✅ Continue |
| `401` / `403` | Invalid or expired key | ❌ STOP — ask user to check key |
| `402` | Payment required / no credits | ❌ STOP — ask user to top up account |
| `429` | Rate limited or quota exceeded | ❌ STOP — ask user to check billing/quota |
| `000` / connection refused | Service unreachable | ❌ STOP — ask user to check network / Ollama running |

**If any check fails:** Tell the user exactly which provider failed, the HTTP code received, and what to fix. **Do not proceed with installation until all required keys pass their checks.**

If the user says keys are set as env vars in the gateway process, run checks using `${VAR_NAME}` substituted inline or ask them to paste the key temporarily for verification.

### Step 3 — Find openclaw.json

Check these locations in order:
```bash
# Most common locations
ls ~/.openclaw/openclaw.json
ls ~/openclaw.json
# Ask the gateway where it's reading config from
openclaw config get --show-path 2>/dev/null || echo "not found"
```

If not found, ask the user for the path.

### Step 4 — Read current config

```bash
# Read and display current plugins config before changing anything
openclaw config get plugins.entries.memory-lancedb-pro 2>/dev/null
openclaw config get plugins.slots.memory 2>/dev/null
```

**Check what already exists** — never blindly overwrite existing settings.

### Step 5 — Build the merged config based on chosen plan

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

**Plan D config (replace models as needed — `qwen3:8b` recommended for LLM, `mxbai-embed-large` for embedding):**
```json
{
  "embedding": {
    "apiKey": "ollama",
    "model": "mxbai-embed-large",
    "baseURL": "http://localhost:11434/v1",
    "dimensions": 1024
  },
  "autoCapture": true,
  "autoRecall": true,
  "captureAssistant": false,
  "smartExtraction": true,
  "extractMinMessages": 2,
  "extractMaxChars": 4000,
  "llm": {
    "apiKey": "ollama",
    "model": "qwen3:8b",
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
  "sessionStrategy": "none"
}
```

**Plan D prerequisites — run BEFORE applying config:**
```bash
# 1. Verify Ollama is running (should return JSON with model list)
curl http://localhost:11434/api/tags

# 2. Pull embedding model (choose one):
ollama pull mxbai-embed-large          # recommended: 1024-dim, beats text-embedding-3-large, ~670MB
ollama pull snowflake-arctic-embed2    # best multilingual local option, ~670MB
ollama pull nomic-embed-text:v1.5      # classic stable, 768-dim, ~270MB

# 3. Pull LLM for Smart Extraction (choose one based on RAM):
ollama pull qwen3:8b           # recommended: best JSON/structured output, ~5.2GB
ollama pull qwen3:14b          # better quality, ~9GB, needs 16GB VRAM
ollama pull llama4:scout       # multimodal MoE, 10M ctx, ~12GB
ollama pull mistral-small3.2   # 24B, 128K ctx, excellent, ~15GB
ollama pull mistral-nemo       # 12B, 128K ctx, efficient, ~7GB

# 4. Verify models are installed
ollama list

# 5. Quick sanity check — embedding endpoint works:
curl http://localhost:11434/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"test"}'
# Should return a JSON with a 1024-element vector
```

**If Smart Extraction produces garbled/invalid output:** The local LLM may not support structured JSON reliably. Try `qwen3:8b` first — it has native structured output support. If still failing, disable:
```json
{ "smartExtraction": false }
```

**If Ollama is on a different host or Docker:** Replace `http://localhost:11434/v1` with the actual host, e.g. `http://192.168.1.100:11434/v1`. Also set `OLLAMA_HOST=0.0.0.0` in the Ollama process to allow remote connections.

For the **`plugins.entries.memory-lancedb-pro.config`** block, merge into the existing `openclaw.json` rather than replacing the whole file. Use a targeted edit of only the memory plugin config section.

### Step 6 — Apply the config

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

### Step 7 — Validate and restart

```bash
openclaw config validate
openclaw gateway restart
openclaw logs --follow --plain | rg "memory-lancedb-pro"
```

Expected output confirms:
- `memory-lancedb-pro: smart extraction enabled`
- `memory-lancedb-pro@...: plugin registered`

### Step 8 — Verify

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

### Quick Install (Beginner-Friendly)

For new users, the community one-click installer handles everything automatically — path detection, schema validation, auto-update, provider selection, and rollback:

```bash
curl -fsSL https://raw.githubusercontent.com/CortexReach/toolbox/main/memory-lancedb-pro-setup/setup-memory.sh -o setup-memory.sh
bash setup-memory.sh
```

Options: `--dry-run` (preview only), `--beta` (include pre-release), `--ref v1.2.0` (pin version), `--selfcheck-only`, `--uninstall`.

Source: https://github.com/CortexReach/toolbox/tree/main/memory-lancedb-pro-setup

---

### Requirements
- Node.js 24 recommended (Node 22 LTS minimum, `22.16+`)
- LanceDB ≥ 0.26.2
- OpenAI SDK ≥ 6.21.0
- TypeBox 0.34.48

### Install Method 1 — via OpenClaw plugin manager (recommended)

```bash
# Install from npm registry (@beta tag = latest pre-release, e.g. 1.1.0-beta.8)
openclaw plugins install memory-lancedb-pro@beta

# Install stable release from npm (@latest tag, e.g. 1.0.32)
openclaw plugins install memory-lancedb-pro

# Or install from a local git clone — use master branch (matches npm @beta)
git clone -b master https://github.com/CortexReach/memory-lancedb-pro.git /tmp/memory-lancedb-pro
openclaw plugins install /tmp/memory-lancedb-pro
```

> **npm vs GitHub branches:** `@beta` installs from the npm registry (not directly from GitHub). The repo has two long-lived branches: **`master`** is the release branch (matches npm `@beta`), **`main`** is older/behind. Always clone `master` if you want code that matches the published beta.

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
git clone -b master https://github.com/CortexReach/memory-lancedb-pro.git plugins/memory-lancedb-pro
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

Then restart and verify:
```bash
openclaw config validate
openclaw gateway restart
openclaw logs --follow --plain | rg "memory-lancedb-pro"
```

### New User First-Install Checklist

After the plugin starts successfully, determine which scenario applies and run the corresponding steps:

---

**Scenario A — Coming from built-in `memory-lancedb` plugin (most common upgrade path)**

The old plugin stores data in LanceDB at `~/.openclaw/memory/lancedb`. Use the migrate command:

```bash
# 1. Check if old data exists and is readable
openclaw memory-pro migrate check

# 2. Preview what would be migrated (dry run)
openclaw memory-pro migrate run --dry-run

# 3. Run the actual migration
openclaw memory-pro migrate run

# 4. Verify migrated data
openclaw memory-pro migrate verify
openclaw memory-pro stats
```

If the old database is at a non-default path:
```bash
openclaw memory-pro migrate check --source /path/to/old/lancedb
openclaw memory-pro migrate run --source /path/to/old/lancedb
```

---

**Scenario B — Existing memories exported as JSON**

If you have memories in the standard JSON export format:

```bash
# Preview import (dry run)
openclaw memory-pro import memories.json --scope global --dry-run

# Import
openclaw memory-pro import memories.json --scope global
```

Expected JSON schema:
```json
{
  "version": "1.0",
  "memories": [
    {
      "text": "Memory content (required)",
      "category": "preference|fact|decision|entity|other",
      "importance": 0.7,
      "timestamp": 1234567890000
    }
  ]
}
```

---

**Scenario C — Memories stored in Markdown files (AGENTS.md, MEMORY.md, etc.)**

There is **no direct markdown import** — the import command only accepts JSON. You need to convert first.

Manual conversion approach:
1. Open the markdown file(s) containing memories
2. For each memory entry, create a JSON object with `text`, `category`, `importance`
3. Save as a JSON file following the schema above
4. Run `openclaw memory-pro import`

Or use `memory_store` tool directly in the agent to store individual entries one at a time:
```
memory_store(text="<extracted memory>", category="fact", importance=0.8)
```

> **Note:** Markdown-based memory files (MEMORY.md, AGENTS.md) are workspace context files, not the same as the LanceDB memory store. You only need to migrate them if you want that content searchable via `memory_recall`.

---

**Scenario D — Fresh install, no prior memories**

No migration needed. Verify the plugin is working with a quick smoke test:
```bash
openclaw memory-pro stats     # should show 0 memories
```
Then trigger a conversation — `autoCapture` will start storing memories automatically.

---

### LanceDB Version Compatibility

> **No manual action required for LanceDB version changes.**

The plugin requires `@lancedb/lancedb ^0.26.2` as an npm dependency — this is installed automatically when you install or update the plugin. You do not need to manually install or upgrade LanceDB.

LanceDB 0.26+ changed how numeric columns are returned (Arrow `BigInt` type for `timestamp`, `importance`, `_distance`, `_score`). The plugin handles this transparently at runtime via internal `Number(...)` coercion — no migration commands are needed when moving between LanceDB versions.

**TL;DR:** LanceDB version compatibility is fully automatic. See the table below for when each maintenance command actually applies.

### Upgrading plugin code vs. data

**Command distinction (important):**

| Command | When to use |
|---------|-------------|
| `openclaw plugins update memory-lancedb-pro` | Update **plugin code** after a new release (npm-installed only) |
| `openclaw plugins update --all` | Update all npm-installed plugins at once |
| `openclaw memory-pro upgrade` | Enrich **old memory-lancedb-pro entries** that predate the smart-memory schema (missing L0/L1/L2 metadata + 6-category system) — NOT related to LanceDB version |
| `openclaw memory-pro migrate` | One-time migration from the separate `memory-lancedb` built-in plugin → Pro |
| `openclaw memory-pro reembed` | Rebuild all embeddings after switching embedding model or provider |

**When do you need `memory-pro upgrade`?**

Run it if you installed memory-lancedb-pro before the smart-memory format was introduced (i.e., entries are missing `memory_category` in their metadata). Signs you need it:
- `memory_recall` returns results but without meaningful categories
- `memory-pro list --json` shows entries with no `l0_abstract` / `l1_overview` fields

Safe upgrade sequence:
```bash
# 1. Backup first
openclaw memory-pro export --scope global --output memories-backup.json

# 2. Preview what would change
openclaw memory-pro upgrade --dry-run

# 3. Run upgrade (uses LLM by default for L0/L1/L2 generation)
openclaw memory-pro upgrade

# 4. Verify results
openclaw memory-pro stats
openclaw memory-pro search "your known keyword" --scope global --limit 5
```

Upgrade options:
```bash
openclaw memory-pro upgrade --no-llm          # skip LLM, use simple text truncation
openclaw memory-pro upgrade --batch-size 5    # slower but safer for large collections
openclaw memory-pro upgrade --limit 50        # process only first N entries
openclaw memory-pro upgrade --scope global    # limit to one scope
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

> **Gateway restart required** after: `plugins install`, `plugins enable`, `plugins disable`, `plugins update`, or any change to `openclaw.json`. Changes do not take effect until the gateway is restarted.
>
> ```bash
> openclaw gateway restart
> ```

### Easy-to-Miss Setup Steps

1. **Gateway restart required after any change**: After installing, enabling, disabling, updating, or changing config in `openclaw.json`, you MUST run `openclaw gateway restart` — changes are NOT hot-reloaded.
2. **Workspace plugins are DISABLED by default**: After git clone, you MUST add `plugins.allow: ["memory-lancedb-pro"]` AND `plugins.entries.memory-lancedb-pro.enabled: true` — without these the plugin silently does not load.
3. **Env vars in gateway process**: `${OPENAI_API_KEY}` requires env vars set in the *OpenClaw Gateway service* process—not just your shell.
4. **Absolute vs. relative paths**: For existing deployments, always use absolute paths in `plugins.load.paths`.
5. **`baseURL` not `baseUrl`**: The embedding (and llm) config field is `baseURL` (capital URL), NOT `baseUrl`. Using the wrong casing causes a schema validation error: "must NOT have additional properties". Also note the required `/v1` suffix: `http://localhost:11434/v1`, not `http://localhost:11434`. Do not confuse with `agents.defaults.memorySearch.remote.baseUrl` which uses a different casing.
6. **jiti cache invalidation**: After modifying `.ts` files under plugins, run `rm -rf /tmp/jiti/` BEFORE `openclaw gateway restart`.
7. **Unknown plugin id = error**: OpenClaw treats unknown ids in `entries`, `allow`, `deny`, or `slots` as validation errors. The plugin id must be discoverable before referencing it.
8. **Separate LLM config**: If embedding and LLM use different providers, configure the `llm` section separately — it falls back to embedding key/URL otherwise.
9. **Scope isolation**: Multi-scope requires explicit `scopes.agentAccess` mapping — without it, agents only see `global` scope.
10. **Session memory hook**: Fires on `/new` command — test with an actual `/new` invocation.
11. **Reranker credentials**: When switching providers, update both `rerankApiKey` AND `rerankEndpoint`.
12. **Config check before assuming defaults**: Run `openclaw config get plugins.entries.memory-lancedb-pro` to verify what's actually loaded.
13. **Custom config/state paths via env vars**: OpenClaw respects the following environment variables for custom paths:
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

## Troubleshooting — Error Message Quick Reference

**Config validation tool** (from [CortexReach/toolbox](https://github.com/CortexReach/toolbox)):
```bash
# Download once
curl -fsSL https://raw.githubusercontent.com/CortexReach/toolbox/main/memory-lancedb-pro-setup/scripts/config-validate.mjs -o config-validate.mjs
# Run against your openclaw.json
node config-validate.mjs
# Or validate a specific config snippet
node config-validate.mjs --json '{"embedding":{"baseURL":"http://localhost:11434/v1","model":"bge-m3","apiKey":"ollama"}}'
```
Exit code 0 = pass/warn, 1 = errors found.

| Error message | Root cause | Fix |
|---------------|-----------|-----|
| `must NOT have additional properties` + `config.embedding` | Field name typo in embedding config (e.g. `baseUrl` instead of `baseURL`) | Check all field names against the schema table below — field names are case-sensitive |
| `must NOT have additional properties` (top-level config) | Unknown top-level field in plugin config | Remove or correct the field |
| `memory-lancedb-pro: plugin not found` / plugin silently not loading | `plugins.allow` missing (git-clone install) or `enabled: false` | Add `plugins.allow: ["memory-lancedb-pro"]` and set `enabled: true`, then restart |
| `Unknown plugin id` validation error | Plugin referenced in `entries`/`slots` before it's discoverable | Install/register the plugin first, then add config references |
| `${OPENAI_API_KEY}` not expanding / auth errors despite env var set | Env var not set in the **gateway process** environment | Set the env var in the service that runs OpenClaw gateway, not just your shell |
| Hooks (`before_agent_start`, `agent_end`) not firing | Gateway not restarted after install/config change | Run `openclaw gateway restart` |
| Embedding errors with Ollama | Wrong `baseURL` format | Must be `http://localhost:11434/v1` (with `/v1`), field must be `baseURL` not `baseUrl` |
| `memory-pro stats` shows 0 entries after conversation | `autoCapture` false or `extractMinMessages` not reached | Set `autoCapture: true`; need at least `extractMinMessages` (default 2) turns |
| Memories not injected before agent replies | `autoRecall` is false (schema default) | Explicitly set `"autoRecall": true` |
| `jiti` cache error after editing plugin `.ts` files | Stale compiled cache | Run `rm -rf /tmp/jiti/` then `openclaw gateway restart` |

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
| `sessionStrategy` | string | `systemSessionMemory` | Session pipeline: `memoryReflection` / `systemSessionMemory` / `none` |
| `autoRecallTopK` | number | 3 | Max memories injected per auto-recall (max 20) |
| `autoRecallSelectionMode` | string | `mmr` | Selection algorithm: `mmr` / `legacy` / `setwise-v2` |
| `autoRecallCategories` | array | `["preference","fact","decision","entity","other"]` | Categories eligible for auto-recall injection |
| `autoRecallExcludeReflection` | boolean | true | Exclude reflection-type memories from auto-recall |
| `autoRecallMaxAgeDays` | number | 30 | Max age (days) of memories considered for auto-recall |
| `autoRecallMaxEntriesPerKey` | number | 10 | Max entries per scope key in auto-recall results |

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
| `rerank` | string | `cross-encoder` | Reranking strategy: `cross-encoder` / `lightweight` / `none` |
| `rerankProvider` | string | `jina` | `jina` / `siliconflow` / `voyage` / `pinecone` / `vllm` (Docker Model Runner) |
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
| `"systemSessionMemory"` **(default)** | Built-in session memory (simpler) |
| `"memoryReflection"` | Advanced LLM-powered reflection with inheritance/derived injection |
| `"none"` | Session summaries disabled |

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

**`memoryReflection.recall` sub-object** (controls which past reflections are retrieved for injection):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | string | `fixed` | Recall mode: `fixed` / `dynamic` |
| `topK` | number | 6 | Max reflection entries retrieved (max 20) |
| `includeKinds` | array | `["invariant"]` | Which kinds to include: `invariant` / `derived` |
| `maxAgeDays` | number | 45 | Max age of reflections to retrieve |
| `maxEntriesPerKey` | number | 10 | Max entries per scope key |
| `minRepeated` | number | 2 | Min times an entry must appear to be included |
| `minScore` | number | 0.18 | Minimum relevance score (range 0–5) |
| `minPromptLength` | number | 8 | Min prompt length to trigger recall |

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
| Jina (recommended) | `jina-embeddings-v5-text-small` | `https://api.jina.ai/v1` | 1024 | Latest (Feb 2026), task-aware LoRA, 32K ctx |
| Jina (multimodal) | `jina-embeddings-v4` | `https://api.jina.ai/v1` | 1024 | Text + image, Qwen2.5-VL backbone |
| OpenAI | `text-embedding-3-large` | `https://api.openai.com/v1` | 3072 | Best OpenAI quality (MTEB 64.6%) |
| OpenAI | `text-embedding-3-small` | `https://api.openai.com/v1` | 1536 | Cost-efficient |
| DashScope (Alibaba) | `text-embedding-v4` | `https://dashscope.aliyuncs.com/compatible-mode/v1` | 1024 | Recommended for Chinese users; also supports rerank (see note below) |
| Google Gemini | `gemini-embedding-2-preview` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 | Latest (Mar 2026), multimodal, 100+ languages |
| Google Gemini | `gemini-embedding-001` | `https://generativelanguage.googleapis.com/v1beta/openai/` | 3072 | Stable text-only |
| Ollama (local) | `mxbai-embed-large` | `http://localhost:11434/v1` | 1024 | **Recommended local** — beats text-embedding-3-large |
| Ollama (local) | `snowflake-arctic-embed2` | `http://localhost:11434/v1` | 1024 | Best multilingual local option |
| Ollama (local) | `nomic-embed-text:v1.5` | `http://localhost:11434/v1` | 768 | Lightweight classic, 270MB |

**DashScope rerank note:** DashScope is not a `rerankProvider` enum value, but its rerank API response is Jina-compatible. Use `rerankProvider: "jina"` with DashScope's endpoint:
```json
"retrieval": {
  "rerank": "cross-encoder",
  "rerankProvider": "jina",
  "rerankModel": "qwen3-rerank",
  "rerankEndpoint": "https://dashscope.aliyuncs.com/compatible-api/v1/reranks",
  "rerankApiKey": "${DASHSCOPE_API_KEY}"
}
```

**Multi-key failover:** Set `apiKey` as an array for round-robin rotation on 429/503 errors.

---

## Reranker Providers

| Provider | `rerankProvider` | Endpoint | Model | Notes |
|----------|-----------------|----------|-------|-------|
| Jina (default) | `jina` | `https://api.jina.ai/v1/rerank` | `jina-reranker-v3` | **Latest text reranker** (2025, Qwen3 backbone, 131K ctx) |
| Jina (multimodal) | `jina` | `https://api.jina.ai/v1/rerank` | `jina-reranker-m0` | Multimodal (text+images), use when docs contain images |
| SiliconFlow | `siliconflow` | `https://api.siliconflow.com/v1/rerank` | `BAAI/bge-reranker-v2-m3` | Free tier available |
| Voyage AI | `voyage` | `https://api.voyageai.com/v1/rerank` | `rerank-2.5` | Sends `{model, query, documents}`, no `top_n` |
| Pinecone | `pinecone` | `https://api.pinecone.io/rerank` | `bge-reranker-v2-m3` | Pinecone customers only |
| vLLM / Docker Model Runner | `vllm` | Custom endpoint | any compatible model | Self-hosted via Docker Model Runner |

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
