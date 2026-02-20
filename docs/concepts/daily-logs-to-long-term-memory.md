---
title: "How Daily Logs Turn Into Long-Term Memory"
summary: "Complete pipeline from daily logs to long-term memory in OpenClaw"
---

# How Daily Logs Turn Into Long-Term Memory

OpenClaw uses a **two-layer memory architecture** that mirrors human cognition: daily logs serve as short-term working memory, while `MEMORY.md` acts as curated long-term storage.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    TWO-LAYER MEMORY SYSTEM                       │
└─────────────────────────────────────────────────────────────────┘

Layer 1: DAILY LOGS (Short-term Context)
├─ Location: ~/.openclaw/workspace/memory/YYYY-MM-DD.md
├─ Load Policy: Today + Yesterday at session start
├─ Content: Day-to-day running notes, session context
└─ Retention: Indexed indefinitely, searchable via vector search

Layer 2: LONG-TERM MEMORY (Curated Knowledge)
├─ Location: ~/.openclaw/workspace/MEMORY.md
├─ Load Policy: Main/private sessions only (never groups)
├─ Content: Durable facts, preferences, decisions, lessons
└─ Retention: Permanent reference material
```

## The Complete Pipeline

### 1. Session Start: Loading Memory

```
[Session Start]
    ↓
[loadWorkspaceBootstrapFiles()]
    ├─ Read MEMORY.md (if main/private session)
    ├─ Read memory/ directory
    ├─ Filter: today + yesterday daily logs
    └─ Inject into system prompt as bootstrap context
    ↓
[Embedding Index Sync]
    ├─ Location: ~/.openclaw/memory/{agentId}.sqlite
    ├─ List all memory files, compute SHA256 hashes
    ├─ For changed files:
    │  ├─ Chunk markdown (~400 tokens, 80-token overlap)
    │  ├─ Generate embeddings (OpenAI/Gemini/local)
    │  └─ Update SQLite vector index
    └─ Ready for memory_search tool
    ↓
[Agent Ready]
    ├─ Has today + yesterday logs in context
    ├─ Can search all indexed memories semantically
    └─ Can write to memory/YYYY-MM-DD.md
```

**Source:** `src/agents/workspace.ts:412-466`, `src/agents/bootstrap-files.ts`

### 2. During Session: Writing Daily Logs

Agents write observations, decisions, and context to daily logs throughout the session:

```typescript
// Agent writes to daily log (via file write tool)
// File: ~/.openclaw/workspace/memory/2026-02-20.md

## Session with Peter - Gateway Setup
- Configured Discord channel routing
- Fixed memory flush threshold (was triggering too early)
- Decision: Use hybrid search (70% vector, 30% BM25)
- TODO: Test temporal decay with 30-day half-life
```

**Key Characteristics:**
- **Append-only**: Never overwrite existing content (system prompts warn against this)
- **Raw notes**: Unfiltered observations, no need for perfect formatting
- **Today's file**: Creates `memory/YYYY-MM-DD.md` if it doesn't exist

**Source:** System prompts in bootstrap files, memory flush prompts

### 3. Pre-Compaction: Automatic Memory Flush

**Trigger Condition** (`src/auto-reply/reply/memory-flush.ts:113-144`):

```javascript
if (
  totalTokens > (contextWindow - reserveTokensFloor - softThresholdTokens)
  && compactionCount !== memoryFlushCompactionCount  // Once per compaction cycle
  && workspace_is_writable
) {
  // Run memory flush turn
}
```

**Default Thresholds:**
- `contextWindow`: Model-specific (e.g., 200K for Sonnet)
- `reserveTokensFloor`: 20,000 tokens
- `softThresholdTokens`: 4,000 tokens
- **Trigger point**: ~176K tokens (for 200K context window)

**Memory Flush Process:**

```
[Token Count Approaches Limit]
    ↓
[shouldRunMemoryFlush() → true]
    ↓
[Silent Agentic Turn]
    ├─ System Prompt: "Pre-compaction memory flush turn."
    ├─ User Prompt: "Store durable memories now (use memory/YYYY-MM-DD.md)"
    └─ Reminder: "APPEND only, do not overwrite existing entries"
    ↓
[Model Writes to Daily Log]
    ├─ Appends key insights to memory/2026-02-20.md
    ├─ May also update MEMORY.md (curated facts)
    └─ Replies with NO_REPLY (silent, user never sees this)
    ↓
[Update Session Metadata]
    ├─ Set memoryFlushAt timestamp
    ├─ Set memoryFlushCompactionCount = compactionCount
    └─ Prevent duplicate flush in same compaction cycle
    ↓
[Context Compaction Proceeds]
    ├─ Truncate old messages (keep recent context)
    ├─ Preserve bootstrap injections
    └─ Session continues with freed context space
```

**Source:** `src/auto-reply/reply/memory-flush.ts`, `src/agents/pi-embedded-helpers/bootstrap.ts`

**Example Memory Flush Output:**

```markdown
## Pre-compaction flush - 14:23
Key insights from session:
- User prefers TypeScript strict mode, no `any` types
- Gateway runs on Mac Studio (192.168.10.5)
- Memory search uses hybrid mode (vector + BM25)
- Temporal decay enabled: 30-day half-life
- Next: Test MMR re-ranking with daily notes
```

### 4. Manual Curation: Daily Logs → MEMORY.md

Users (or agents on request) manually **curate important facts** from daily logs into `MEMORY.md`:

**Daily Log Entry** (`memory/2026-02-15.md`):
```markdown
## Gateway configuration
Figured out the memory flush was triggering too early.
Increased softThresholdTokens from 2000 to 4000.
Works much better now - only flushes when really needed.
```

**Curated to MEMORY.md** (manually or via explicit agent request):
```markdown
## Configuration Preferences

### Memory System
- Memory flush threshold: 4000 soft tokens (not 2000)
- Reason: Previous setting triggered too frequently
- Hybrid search: 70% vector, 30% BM25
- Temporal decay: 30-day half-life for daily notes
```

**Curation Triggers:**
1. **Manual editing** by user
2. **Agent request**: "Remember this" → agent writes to MEMORY.md
3. **Periodic review**: User reviews daily logs and promotes key facts

**Design Philosophy:**
- Daily logs = **everything** (append-only, verbose, dated)
- MEMORY.md = **curated knowledge** (evergreen, no dates, reference material)

### 5. Long-Term Indexing & Search

**Vector Index** (`~/.openclaw/memory/{agentId}.sqlite`):

```
[Memory File Watcher]
    ├─ Watches: MEMORY.md + memory/*.md
    ├─ Debounce: 1.5 seconds
    └─ Triggers: Index sync on changes
    ↓
[Sync Process]
    ├─ Hash check (SHA256)
    ├─ Chunk changed files (~400 tokens, 80-token overlap)
    ├─ Generate embeddings (cached if possible)
    ├─ Update SQLite tables:
    │  ├─ files: path, hash, mtime, size
    │  ├─ chunks: text, embeddings, line ranges
    │  ├─ chunks_fts: BM25 full-text index
    │  └─ chunks_vec: Vector search (sqlite-vec)
    └─ Ready for semantic search
```

**Search Pipeline** (`memory_search` tool):

```
[User Query: "What's the gateway IP?"]
    ↓
[Hybrid Search] — enabled by default
    ├─ Vector Search (semantic similarity)
    │  └─ Embedding model encodes query
    │     → Cosine similarity vs indexed chunks
    │        → Top candidates
    ├─ BM25 Full-Text Search (keyword matching)
    │  └─ SQLite FTS5 ranks chunks
    │     → Top candidates
    └─ Merge Results
       └─ finalScore = 0.7 × vectorScore + 0.3 × bm25Score
    ↓
[Optional: Temporal Decay] — off by default
    ├─ Detect dated files (YYYY-MM-DD pattern)
    ├─ Apply exponential decay:
    │  └─ decayedScore = score × e^(-λ × ageInDays)
    │     where λ = ln(2) / halfLifeDays
    ├─ Evergreen files (MEMORY.md, network.md) never decay
    └─ Recent notes rank higher than old ones
    ↓
[Optional: MMR Re-ranking] — off by default
    ├─ Remove redundant/duplicate results
    ├─ Balance relevance vs diversity
    └─ lambda = 0.7 (70% relevance, 30% diversity)
    ↓
[Return Top-K Results] — default: 6 snippets
    ├─ Snippet text (~700 chars max)
    ├─ File path + line range
    ├─ Score (0-1)
    └─ Source metadata
```

**Search Result Example:**

```
memory_search("gateway IP address")

Results:
1. MEMORY.md:15-17 (score: 0.92)
   "Gateway runs on Mac Studio at 192.168.10.5.
    Tailscale hostname: gateway-host."

2. memory/2026-02-15.md:34-36 (score: 0.78)
   "Configured static IP for gateway: 192.168.10.5
    (was DHCP, kept getting different IPs)"

3. memory/network.md:8-10 (score: 0.71)
   "Network topology:
    - Gateway: 192.168.10.5 (Mac Studio)
    - AdGuard DNS: 192.168.10.2"
```

**Source:** `src/memory/manager-search.ts`, `src/memory/hybrid.ts`, `src/agents/tools/memory-tool.ts`

## Key Data Structures

### SQLite Schema (`~/.openclaw/memory/{agentId}.sqlite`)

```sql
-- Metadata
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT
);

-- File tracking
CREATE TABLE files (
  id INTEGER PRIMARY KEY,
  path TEXT UNIQUE NOT NULL,
  hash TEXT NOT NULL,
  mtimeMs INTEGER,
  size INTEGER,
  source TEXT DEFAULT 'memory'
);

-- Indexed chunks
CREATE TABLE chunks (
  id INTEGER PRIMARY KEY,
  fileId INTEGER NOT NULL,
  text TEXT NOT NULL,
  fromLine INTEGER,
  toLine INTEGER,
  embedding BLOB,
  FOREIGN KEY (fileId) REFERENCES files(id)
);

-- Full-text search index (BM25)
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  chunkId UNINDEXED,
  text,
  content=chunks,
  content_rowid=id
);

-- Vector search table (sqlite-vec extension)
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  chunkId INTEGER PRIMARY KEY,
  embedding FLOAT[1536]  -- Dimension varies by model
);

-- Embedding cache (avoid re-computing unchanged text)
CREATE TABLE embedding_cache (
  textHash TEXT PRIMARY KEY,
  embedding BLOB NOT NULL,
  provider TEXT,
  model TEXT,
  createdAt INTEGER
);
```

**Source:** `src/memory/memory-schema.ts`

### Memory File Entry

```typescript
type MemoryFileEntry = {
  path: string;        // Relative to workspace
  absPath: string;     // Absolute filesystem path
  mtimeMs: number;     // Last modification timestamp
  size: number;        // File size in bytes
  hash: string;        // SHA256 of content
};
```

**Source:** `src/memory/types.ts`

## Configuration Examples

### Basic Setup (Remote Embeddings)

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
      memorySearch: {
        enabled: true,
        provider: "openai",  // or "gemini", "voyage", "local"
        model: "text-embedding-3-small",
        query: {
          maxResults: 6,
          minScore: 0.35,
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3
          }
        },
        sync: {
          onSessionStart: true,
          onSearch: true,
          watch: true,
          watchDebounceMs: 1500,
          intervalMinutes: 5
        }
      },
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          prompt: "Store durable memories to memory/YYYY-MM-DD.md",
          systemPrompt: "Pre-compaction memory flush turn"
        }
      }
    }
  }
}
```

### Advanced: Temporal Decay + MMR

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            // Reduce redundant daily log entries
            mmr: {
              enabled: true,
              lambda: 0.7  // 70% relevance, 30% diversity
            },
            // Boost recent notes over old ones
            temporalDecay: {
              enabled: true,
              halfLifeDays: 30  // Score halves every 30 days
            }
          }
        }
      }
    }
  }
}
```

### Local Embeddings (No API Keys)

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "local",
        fallback: "none",  // Don't fall back to remote
        local: {
          modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf"
        }
      }
    }
  }
}
```

## Timeline: Daily Log Lifecycle

```
Day 0 (2026-02-20):
├─ memory/2026-02-20.md created
├─ Loaded at session start (today)
├─ Agent writes observations throughout session
├─ Pre-compaction flush appends key insights
└─ Indexed and searchable

Day 1 (2026-02-21):
├─ memory/2026-02-21.md created (new today)
├─ memory/2026-02-20.md loaded (yesterday)
└─ Both indexed and searchable

Day 30 (2026-03-22):
├─ memory/2026-02-20.md is 30 days old
├─ Still indexed and searchable
├─ If temporal decay enabled:
│  └─ Score × 0.50 (50% of original)
└─ Older memories rank lower but still found

Day 180 (2026-08-19):
├─ memory/2026-02-20.md is 180 days old
├─ If temporal decay enabled (30-day half-life):
│  └─ Score × 0.016 (~1.6% of original)
├─ Very old notes fade unless explicitly curated to MEMORY.md
└─ Important facts should be promoted to MEMORY.md (evergreen)
```

## Best Practices

### For Users

1. **Let daily logs be messy**: Write everything to `memory/YYYY-MM-DD.md`, don't worry about formatting
2. **Curate to MEMORY.md**: Periodically review daily logs and promote key facts/decisions
3. **Use search, not recall**: Don't try to remember everything; use `memory_search` to find it
4. **Enable temporal decay**: If you have months of daily notes, enable decay to boost recent context

### For Agents

1. **Write liberally to daily logs**: Append observations throughout the session
2. **Trigger memory flush**: System handles this automatically before compaction
3. **Search semantically**: Use `memory_search` for queries, not just file reads
4. **Respect MEMORY.md scope**: Only load in main/private sessions (security)

## Key Insights

1. **Two-layer architecture mirrors human memory**:
   - Daily logs = working memory (short-term, verbose, dated)
   - MEMORY.md = long-term storage (curated, evergreen, reference)

2. **Memory flush prevents context loss**:
   - Automatic before compaction
   - Silent operation (user never sees it)
   - One flush per compaction cycle (tracked via metadata)

3. **Always append, never overwrite**:
   - Daily logs are append-only
   - Prevents losing prior session data
   - Prompts explicitly warn against overwrites

4. **Embeddings enable semantic search**:
   - Find "the machine running the gateway" when searching "Mac Studio"
   - Hybrid search combines semantic + keyword matching
   - Temporal decay boosts recent notes over old ones

5. **Security-first design**:
   - MEMORY.md only loads in main/private sessions
   - Never in groups/channels (prevents leaking private facts)
   - Only indexed files in `memory/` or `MEMORY.md`

## References

### Documentation
- `/docs/concepts/memory.md` - User-facing memory documentation
- `/docs/reference/session-management-compaction.md` - Compaction lifecycle

### Implementation
- `src/agents/workspace.ts:412-466` - Bootstrap file loading
- `src/auto-reply/reply/memory-flush.ts` - Automatic memory flush
- `src/memory/manager.ts` - Memory index manager
- `src/memory/manager-search.ts` - Search implementation
- `src/memory/hybrid.ts` - Hybrid BM25+vector search
- `src/memory/temporal-decay.ts` - Recency boosting
- `src/memory/mmr.ts` - Diversity re-ranking
- `src/agents/tools/memory-tool.ts` - memory_search and memory_get tools
- `src/memory/sync-memory-files.ts` - File watcher and sync

### Tests
- `src/auto-reply/reply/memory-flush.test.ts` - Memory flush logic tests
- `src/agents/tools/memory-tool.e2e.test.ts` - Memory tool integration tests

## Summary Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DAILY LOGS → LONG-TERM MEMORY                      │
└──────────────────────────────────────────────────────────────────────┘

Session Start
    ↓
Load Bootstrap (today + yesterday daily logs + MEMORY.md)
    ↓
Sync Embeddings (hash check → chunk → embed → index)
    ↓
Agent Reasoning
    ├─ Writes to memory/YYYY-MM-DD.md (observations, decisions)
    ├─ Uses memory_search (semantic queries)
    └─ Uses memory_get (direct file reads)
    ↓
Token Count Increases
    ↓
[Threshold Crossed]
    ↓
Memory Flush Turn (silent)
    ├─ System: "Pre-compaction memory flush"
    ├─ Agent appends key insights to memory/YYYY-MM-DD.md
    └─ Reply: NO_REPLY (user never sees this)
    ↓
Context Compaction
    ├─ Truncate old messages
    ├─ Preserve bootstrap files
    └─ Session continues
    ↓
Session End
    ├─ All written memories persist to disk
    └─ Index updated (file watcher)
    ↓
User Reviews Daily Logs (periodic)
    ├─ Promote important facts to MEMORY.md
    └─ Curated knowledge becomes evergreen reference
    ↓
Future Sessions
    ├─ Bootstrap loads: today + yesterday + MEMORY.md
    ├─ memory_search finds old insights via embeddings
    └─ Temporal decay ranks recent > old (if enabled)
```

---

**Key Takeaway**: Daily logs capture **everything** in real-time; memory flush preserves **insights** before compaction; manual curation promotes **durable knowledge** to MEMORY.md; embeddings make **all of it searchable** semantically across sessions.
