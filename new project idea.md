# Konomis — Portable AI Memory & Code Context Platform

> **The One-Liner:**  
> Git syncs your code across machines, but **Konomis syncs your AI's working memory and codebase map**—so when you switch from your laptop to your PC (or switch AI tools), your agent picks up right where you left off without re-scraping files or burning tokens.

*(Name origin: **Oikonomia** — Greek for "stewardship / careful resource management" + **Index/System**. Konomis acts as a strict steward of your AI's context window.)*

---

## 1. The Problem We Are Solving

When coding with AI across multiple machines (e.g. **Laptop** $\leftrightarrow$ **Desktop PC**) or across tools (**Cursor**, **Claude Code**, **Antigravity**, **Codex**):

1. **Broken Context on Device Switch:**  
   When you `git pull` on another machine, you get the code, **not the conversation**. Your chat history is stuck in the other machine's local cache. Your AI has total amnesia.
2. **Repeated Mistakes:**  
   The new session doesn't know what approaches you already tried and rejected 30 minutes ago, so it suggests the exact same broken ideas again.
3. **Token Waste (Brute-Force Scraping):**  
   To understand the repo, the AI runs `grep`, `find`, and reads huge raw files line by line—burning 30,000–80,000 tokens before making a single edit.

### How Konomis is Different (vs. Repowise & Similar Tools)
* **Repowise** focuses on **repo governance & code quality** (PR review bots, 1–10 health scores, dead code, test coverage, security).
* **Konomis** focuses on **portable working memory + token-efficient context**: syncing *your active session state, architectural decisions, and a lightweight symbol map* across devices and AI editors.

---

## 2. How It Works for a User (The Everyday Flow)

Konomis works like **Git + GitHub**, but for AI context:

### Step 1: One-Time Setup (`konomis init`)
In any project folder, you run:
```bash
konomis init
```
This automatically:
1. Links the repo to your Konomis server (local or cloud).
2. Registers the **Konomis MCP server** inside your installed tools (`.cursor/mcp.json`, Claude config, etc.).
3. Injects clear instructions into `.cursorrules` / `CLAUDE.md` telling the AI: *"Always check Konomis first before running `grep` or reading whole files."*

### Step 2: Working & Checkpointing (On Your Laptop)
As you code, Konomis captures two things:
* **Code Structure:** Only the files changed since the last commit are parsed for functions, classes, types, and who calls whom.
* **Session State:** Either automatically (by reading your IDE's session transcript) or manually via CLI:
  ```bash
  konomis checkpoint -m "Fixing BullMQ retry race condition; Redlock was too slow"
  ```
Instead of saving 150k tokens of raw chat messages, Konomis compresses it into a crisp **Handoff Card**:
* **Current Goal** (what you are trying to finish on this branch)
* **Touched Symbols** (exact functions/files involved)
* **What Failed** (dead ends so the AI never repeats them)
* **Decisions Made** (and why)
* **Next Step** (where you stopped)

### Step 3: Switching Devices (On Your Desktop PC)
You sit at your PC, pull your code, and sync context:
```bash
git pull
konomis resume
```
Then in Cursor / Claude Code / Antigravity, you simply say: *"Continue where I left off."*  
Your AI calls Konomis via MCP, loads the **Handoff Card + relevant symbol map (~800 tokens)** in **one call**, and continues mid-thought.

### Step 4: The Web Dashboard ("GitHub for AI Context")
In the browser, you see your projects organized like GitHub:
* **Branches $\to$ Session Timeline:** Every checkpoint made across Laptop/PC and which AI tool created it.
* **Decision Log:** All "Why we built it this way" notes linked to files and commits.
* **Symbol Explorer & Savings Meter:** Visual graph of indexed functions and a live counter showing **Tokens Saved vs. Raw File Reads**.

---

## 3. System Architecture (Simple View)

```
┌──────────────────────────────────┐       ┌──────────────────────────────────┐
│        LAPTOP (Cursor/CLI)       │       │      DESKTOP PC (Claude/IDE)     │
│  • Local Code + Git Branch       │       │  • Local Code + Git Branch       │
│  • konomis CLI (init/push/resume)│       │  • konomis MCP Client            │
└────────────────┬─────────────────┘       └────────────────▲─────────────────┘
                 │ Push diff + checkpoint                   │ Pull handoff + symbols
                 ▼                                          │
┌─────────────────────────────────────────────────────────────────────────────┐
│                         KONOMIS SERVER (Cloud or Self-Hosted)               │
│                                                                             │
│  ┌─────────────────────────┐          ┌──────────────────────────────────┐  │
│  │   API & MCP Gateway     │          │      Background Workers          │  │
│  │  • REST API (CLI/UI)    │─────────▶│  • BullMQ Queue (Redis)          │  │
│  │  • MCP Endpoint (Tools) │          │  • Tree-sitter AST Diff Worker   │  │
│  └────────────┬────────────┘          │  • Session Summarizer Worker     │  │
│               │                       └────────────────┬─────────────────┘  │
│               ▼                                        ▼                    │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    PostgreSQL + pgvector                              │  │
│  │  • Repos & Branches        • AST Symbols & Call Graph                 │  │
│  │  • Session Checkpoints     • Decisions & Embeddings                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. The Two Core Engines

### Engine A: Cross-Device Session Handoff (Your Unique Edge)
* **Universal Session Adapters:** Small adapters (inspired by the `HarnessAdapter` pattern) that read local session logs from Claude Code, Cursor, Codex, etc., and turn them into one shared format.
* **Smart Compression:** Summarizes noisy tool logs and long chats into a tiny, structured checkpoint so restoring context never blows up the new agent's token limit.
* **Branch-Aware Memory:** Context belongs to a Git branch (`feat/auth`), so switching branches automatically switches the AI's mental context too.

### Engine B: Smart Code Map (Zero-Scraping Index)
* **Incremental AST Parsing (Tree-sitter):** When you commit, Konomis checks `git diff-tree` and **only re-parses modified files** (fast, `< 200ms`), extracting function signatures, types, imports, and call edges.
* **Token-Budget Graph Pruner:** When an AI asks about a function, Konomis runs a **Breadth-First Search (BFS)** across callers/callees and stops once a strict token cap (e.g., `2,000` or `4,000` tokens) is reached.
  * *Priority order:* Function Signature > Direct Callers > Type/Interface Contracts > Implementation details.
* **Hybrid Search (pgvector):** Lets agents search past decisions and summaries semantically (*"Why did we stop using distributed locks?"*).

---

## 5. How We Guarantee AI Agents Actually Use Konomis

Building an MCP server isn't enough—by default, LLMs try to run `grep` and `read_file`. Konomis enforces usage with 3 layers:

1. **Auto-Injected Workspace Rules:** `konomis init` writes a high-priority instruction block into `.cursorrules`, `CLAUDE.md`, and `AGENTS.md` forcing the agent to call `konomis_get_session_handoff` and `konomis_get_symbol_graph` *before* touching raw files.
2. **Action-Oriented Tool Descriptions:** MCP tool descriptions explicitly instruct the model: *"AUTHORITATIVE LOOKUP: Use this INSTEAD of grep or reading whole files to inspect architecture and save tokens."*
3. **One-Line Resume Prompt:** `konomis resume` copies a ready-to-paste starter prompt (or prints it) so the agent immediately pulls the latest checkpoint on boot.

---

## 6. The 4 MCP Tools Exposed to AI Agents

| Tool Name | What It Does | Why It Saves Tokens |
| :--- | :--- | :--- |
| `get_session_handoff(branch)` | Returns the latest checkpoint (goal, touched symbols, failed attempts, next steps). | Instant continuity across Laptop $\to$ PC without re-explaining anything. |
| `get_symbol_graph(symbol, budget)` | Returns signatures, types, callers, and callees within a strict token limit. | Turns 5,000 lines of raw file reads into ~600 tokens of exact structure. |
| `record_decision(target, choice, why)` | Saves an architectural choice or dead end tied to a file/symbol/branch. | Future sessions (or teammates) hear the "why" in one line. |
| `search_memory(query)` | Searches past checkpoints, PR notes, and architectural decisions via pgvector. | Answers *"why is this built like this?"* in one lookup. |

---

## 7. Database Schema (PostgreSQL + pgvector)

```sql
-- 1. Projects / Repos
CREATE TABLE repositories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    remote_url TEXT,
    default_branch VARCHAR(64) DEFAULT 'main',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Git Commits / Revisions
CREATE TABLE revisions (
    id SERIAL PRIMARY KEY,
    repository_id INT REFERENCES repositories(id) ON DELETE CASCADE,
    branch VARCHAR(128) NOT NULL,
    commit_sha VARCHAR(40) NOT NULL UNIQUE,
    message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Code Symbols (Functions, Classes, Types)
CREATE TABLE code_symbols (
    id SERIAL PRIMARY KEY,
    repository_id INT REFERENCES repositories(id) ON DELETE CASCADE,
    revision_id INT REFERENCES revisions(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    kind VARCHAR(64) NOT NULL,        -- 'function' | 'class' | 'interface' | 'type'
    file_path TEXT NOT NULL,
    start_line INT NOT NULL,
    end_line INT NOT NULL,
    signature TEXT,
    embedding vector(1536)
);

-- 4. Symbol Call / Import Edges (Graph)
CREATE TABLE symbol_dependencies (
    source_symbol_id INT REFERENCES code_symbols(id) ON DELETE CASCADE,
    target_symbol_id INT REFERENCES code_symbols(id) ON DELETE CASCADE,
    relation_type VARCHAR(32) NOT NULL, -- 'calls' | 'implements' | 'imports'
    PRIMARY KEY (source_symbol_id, target_symbol_id, relation_type)
);

-- 5. Cross-Device Session Checkpoints (The Laptop <-> PC Bridge)
CREATE TABLE session_checkpoints (
    id SERIAL PRIMARY KEY,
    repository_id INT REFERENCES repositories(id) ON DELETE CASCADE,
    branch VARCHAR(128) NOT NULL,
    device_name VARCHAR(128),         -- e.g. 'laptop-win', 'desktop-pc'
    agent_harness VARCHAR(64),        -- e.g. 'cursor', 'claude-code', 'antigravity'
    working_goal TEXT NOT NULL,
    failed_attempts JSONB DEFAULT '[]',
    next_steps JSONB DEFAULT '[]',
    touched_symbols TEXT[] DEFAULT '{}',
    raw_summary TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 6. Architectural Decisions ("Why" Memory)
CREATE TABLE context_decisions (
    id SERIAL PRIMARY KEY,
    repository_id INT REFERENCES repositories(id) ON DELETE CASCADE,
    branch VARCHAR(128),
    file_path TEXT,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    source_commit_sha VARCHAR(40),
    embedding vector(1536),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 8. Recommended Tech Stack

Matches speed, reliability, and backend scalability:
* **Language:** TypeScript (Node.js) — shared types across CLI, API, Workers, and MCP server.
* **AST Parsing:** `tree-sitter` (`web-tree-sitter` or native bindings for TS/JS/Python first).
* **MCP SDK:** `@modelcontextprotocol/sdk` (supports both local `stdio` and remote `SSE / Streamable HTTP`).
* **Database & Search:** PostgreSQL + `pgvector`.
* **Async Queue:** Redis + BullMQ (for background indexing & transcript summarization).
* **Web Dashboard:** Next.js + Tailwind CSS (GitHub-style repo/branch/session explorer).

---

## 9. How to Proceed: Step-by-Step Build Plan

> **Rule of thumb:** Build the **smallest working loop first** so you can actually use it between your laptop and PC by Week 2—then add fancy features.

### Phase 1: Project Skeleton & Database (Days 1–3)
- [ ] Set up a monorepo (`apps/api`, `apps/cli`, `apps/mcp`, `packages/shared`, or clean modular folders).
- [ ] Spin up PostgreSQL (`pgvector`) + Redis via `docker-compose.yml`.
- [ ] Create the database tables above (using Prisma or Drizzle).

### Phase 2: The Core Code Indexer (Week 1)
- [ ] Build a Tree-sitter parser for **TypeScript / JavaScript** (add Python/Go later).
- [ ] Extract functions, classes, signatures, imports, and call edges into `code_symbols` & `symbol_dependencies`.
- [ ] Add **Incremental Git Diffing**: compare `HEAD~1` vs `HEAD` with `git diff-tree` so only changed files are re-parsed.
- [ ] Write the **BFS Token-Budget Pruner** (`get_symbol_graph`) that stops once the token budget is full.

### Phase 3: The Cross-Device Checkpoint & MCP Server (Week 2)
- [ ] Build the API endpoints: `POST /checkpoints`, `GET /checkpoints/latest?repo=...&branch=...`, `POST /decisions`.
- [ ] Build the MCP server using `@modelcontextprotocol/sdk` exposing the 4 tools (`get_session_handoff`, `get_symbol_graph`, `record_decision`, `search_memory`).
- [ ] Build the CLI (`konomis init`, `konomis checkpoint`, `konomis sync`, `konomis resume`).
- [ ] **Real-World Test #1:** Use it yourself! Start a coding task on your laptop, run `konomis checkpoint`, pull on your PC, and verify the AI resumes cleanly.

### Phase 4: Transcript Adapters & Background Workers (Week 3)
- [ ] Hook up **BullMQ** workers so `konomis push` / Git webhooks index commits asynchronously without blocking the CLI.
- [ ] Build the first **Session Transcript Adapter** (e.g., Claude Code `.jsonl` or Codex logs) + an LLM summarizer job so users don't even have to type manual checkpoint messages—Konomis extracts the goal, failed attempts, and next steps automatically.
- [ ] Add `pgvector` embeddings to `context_decisions` and `code_symbols`.

### Phase 5: Web Dashboard & Benchmarks (Week 4)
- [ ] Build the **Next.js Dashboard**: Repos $\to$ Branches $\to$ Session Timeline + Decisions + Symbol Map.
- [ ] Add **Token Savings Telemetry**: log `tokens_returned` vs `raw_file_tokens_avoided` on every MCP call.
- [ ] Run a real benchmark on 2–3 open-source repos comparing *Raw AI Exploration* vs *Konomis MCP* and put the exact numbers in your README.

---

## 10. Scope Guardrails (What NOT to Build in v1)

To make sure this ships as a real product and doesn't stall:
* **Skip** 28 programming languages on day 1 — start with **TypeScript/JS + Python**.
* **Skip** complex GitHub PR review bots, code-health scores, dead-code linters, and CI coverage mapping (that's Repowise's lane, not yours).
* **Skip** syncing giant raw chat logs verbatim — always compress into **Goals + Decisions + Failed Attempts + Touched Symbols**.
