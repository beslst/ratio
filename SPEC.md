# Ratio — MVP Specification

> **Ratio** is a versioning system for agentic content generation.  
> It versions *reasoning*, not just artifacts.

---

## 0. Guiding Principles

1. **Reasoning is the artifact.** Files are a side effect. The decision tree that produced them is the real history.
2. **Nothing is lost.** Every intermediate agent state is captured. Humans decide what to promote, not what to save.
3. **Git interop is a feature, not an afterthought.** Ratio wraps existing file trees; it does not replace the filesystem.
4. **MVP is a CLI + local storage.** No server, no collaboration, no UI — prove the core model first.

---

## 1. Scope

### In MVP
- `ratio init` — initialize a repo
- `ratio run` — invoke an agent with full capture
- `ratio log` — browse the decision tree
- `ratio show <id>` — inspect a node (prompt, reasoning, diff, context)
- `ratio checkout <id>` — restore filesystem to any node's state
- `ratio milestone <id>` — promote a node to a named milestone (analogous to a commit)
- `ratio diff <id1> <id2>` — compare two nodes semantically
- Local storage only (SQLite + content-addressed blob store)
- Git bridge: auto-commit to a shadow Git repo at every milestone

### Out of Scope for MVP
- Multi-agent graphs
- Remote/collaborative repos
- UI / web interface
- Probabilistic branching (auto-fork on uncertainty)
- LLM-judge evaluation pipeline
- Semantic AST diffing (MVP uses structural text diff + metadata)
- Authentication / signing

---

## 2. Core Data Model

### 2.1 DecisionNode

The fundamental unit of history.

```
DecisionNode {
  id:           string          // uuid v4, content-addressed on creation
  parent_id:    string | null   // parent node id; null for root
  run_id:       string          // groups nodes from one agent invocation
  type:         enum            // ROOT | STEP | MILESTONE | CHECKPOINT
  
  // Input
  prompt:       string          // the instruction given to the agent
  context:      ContextSnapshot // see 2.2

  // Process
  reasoning:    string          // chain-of-thought / scratchpad (if available)
  actions:      Action[]        // see 2.3

  // Output
  artifact_ids: string[]        // content-addressed blob ids of output files
  summary:      string          // agent-generated one-liner of what changed

  // Meta
  model:        string          // e.g. "claude-sonnet-4-20250514"
  timestamp:    datetime
  duration_ms:  int
  confidence:   float | null    // 0-1, agent self-report if available
  tags:         string[]
  milestone_name: string | null // set when type = MILESTONE
}
```

### 2.2 ContextSnapshot

What the agent knew when it ran.

```
ContextSnapshot {
  id:             string       // content-addressed
  files_in:       FileRef[]    // files passed to the agent as context
  tools:          string[]     // tool names available
  system_prompt:  string
  model_params:   {
    temperature:  float
    max_tokens:   int
    // ...
  }
  env_hash:       string       // hash of relevant env vars / config
}
```

### 2.3 Action

A single thing the agent did.

```
Action {
  seq:      int           // order within the node
  type:     enum          // TOOL_CALL | FILE_WRITE | FILE_DELETE | FILE_RENAME | MESSAGE
  tool:     string | null // if TOOL_CALL
  input:    json          // arguments
  output:   json          // result (truncated if large, full stored as blob)
  error:    string | null
  duration_ms: int
}
```

### 2.4 FileRef

```
FileRef {
  path:     string   // relative to repo root
  blob_id:  string   // content-addressed SHA-256
  mode:     string   // file permissions
}
```

---

## 3. Storage Layout

```
.ratio/
  config.json          # repo config (model defaults, ignore patterns, etc.)
  HEAD                 # current node id
  
  db/
    ratio.db           # SQLite: nodes, runs, milestones, tags
  
  objects/
    blobs/             # content-addressed file snapshots
      ab/
        cd1234...      # blob keyed by SHA-256
    contexts/          # serialized ContextSnapshots
    reasoning/         # raw reasoning text, stored separately (can be large)
  
  git/                 # shadow Git repo (bare)
    ...                # auto-committed at every milestone
```

### SQLite Schema (simplified)

```sql
CREATE TABLE nodes (
  id            TEXT PRIMARY KEY,
  parent_id     TEXT REFERENCES nodes(id),
  run_id        TEXT NOT NULL,
  type          TEXT NOT NULL,
  prompt        TEXT,
  reasoning_id  TEXT,     -- ref to objects/reasoning/
  context_id    TEXT,     -- ref to objects/contexts/
  summary       TEXT,
  model         TEXT,
  timestamp     TEXT,
  duration_ms   INTEGER,
  confidence    REAL,
  milestone_name TEXT
);

CREATE TABLE node_artifacts (
  node_id  TEXT REFERENCES nodes(id),
  path     TEXT,
  blob_id  TEXT,
  mode     TEXT,
  PRIMARY KEY (node_id, path)
);

CREATE TABLE actions (
  id        TEXT PRIMARY KEY,
  node_id   TEXT REFERENCES nodes(id),
  seq       INTEGER,
  type      TEXT,
  tool      TEXT,
  input     TEXT,   -- JSON
  output    TEXT,   -- JSON (possibly truncated)
  error     TEXT,
  duration_ms INTEGER
);

CREATE TABLE tags (
  node_id TEXT REFERENCES nodes(id),
  tag     TEXT,
  PRIMARY KEY (node_id, tag)
);
```

---

## 4. CLI Reference

### `ratio init`

Initialize a Ratio repo in the current directory.

```
ratio init [--model <model>] [--git-bridge]
```

- Creates `.ratio/` directory and structure
- Writes `config.json` with defaults
- Optionally initializes shadow Git repo if `--git-bridge`
- Does **not** require an existing Git repo

---

### `ratio run`

Invoke an agent, capturing the full decision node.

```
ratio run "<prompt>" [options]

Options:
  --context <glob>      Files to include in context (default: all tracked files)
  --model <model>       Override default model
  --parent <id>         Explicitly set parent node (default: HEAD)
  --tag <tag>           Tag this node on creation
  --dry-run             Show what would be sent; don't execute
```

**Behavior:**
1. Resolves context files, builds `ContextSnapshot`
2. Creates a `STEP` node (pending)
3. Invokes the agent, streaming actions into the node as they occur
4. Captures all file writes as blobs
5. Finalizes the node; updates `HEAD`
6. Prints the node id and summary

**Example:**
```
$ ratio run "Refactor the auth module to use JWT instead of sessions"

> run:a3f9c1 started
  model: claude-sonnet-4-20250514
  context: 12 files (34.2 KB)

  -> TOOL_CALL read_file auth/session.py
  -> TOOL_CALL read_file auth/middleware.py
  -> FILE_WRITE auth/jwt.py            (+187 lines)
  -> FILE_WRITE auth/middleware.py     (modified, +23 -41 lines)
  -> FILE_DELETE auth/session.py

✓ node:7b2d84 committed
  "Replaced session-based auth with JWT; added jwt.py, updated middleware"
  3 files changed · 2.1s
```

---

### `ratio log`

Browse the decision tree.

```
ratio log [options]

Options:
  --from <id>      Start from a specific node (default: HEAD)
  --depth <n>      Max depth to traverse (default: 20)
  --all            Show all branches, not just current lineage
  --milestones     Show only milestone nodes
  --run <id>       Show only nodes from a specific run
  --format <fmt>   tree (default) | list | json
```

**Example output:**
```
$ ratio log

* node:7b2d84  [HEAD]
|  "Replaced session-based auth with JWT"
|  2 min ago · claude-sonnet-4-20250514 · 2.1s

* node:3c8f12  [milestone: initial-auth]
|  "Scaffolded auth module with session support"
|  1 hour ago · claude-sonnet-4-20250514 · 4.7s

o node:1a4e90  (branch)
|  "Attempted OAuth flow — abandoned"
|  1 hour ago

* node:0d2b11  [root]
   "Init"
```

---

### `ratio show <id>`

Inspect a node in full.

```
ratio show <id> [--reasoning] [--actions] [--context] [--diff]
```

**Default output:** summary, prompt, files changed, confidence, timestamp

**With flags:**
- `--reasoning` — print the full chain-of-thought
- `--actions` — print all tool calls and file ops in sequence
- `--context` — list files and params in the context snapshot
- `--diff` — show file diffs vs parent node

**Example:**
```
$ ratio show node:7b2d84 --reasoning --diff

NODE  node:7b2d84
Type  STEP
Time  2026-03-09 14:32:01
Model claude-sonnet-4-20250514 (t=0.7)
Conf  0.87

PROMPT
  Refactor the auth module to use JWT instead of sessions

REASONING
  The current implementation uses server-side sessions stored in Redis.
  To migrate to JWT I need to:
  1. Create a new jwt.py module with encode/decode helpers
  2. Update middleware to validate Bearer tokens instead of session cookies
  3. Remove session.py — nothing else imports it directly
  ...

DIFF  auth/middleware.py
  - from auth.session import validate_session
  + from auth.jwt import validate_token
  ...

DIFF  auth/jwt.py  (new file, 187 lines)
  + import jwt
  + ...
```

---

### `ratio checkout <id>`

Restore the working directory to the state of any node.

```
ratio checkout <id> [--dry-run]
```

- Writes all tracked files from the node's artifact snapshot to disk
- Does not change HEAD (detached mode, like `git checkout <sha>`)
- `--dry-run` prints what would change without writing

---

### `ratio milestone <id> <name>`

Promote a node to a named milestone.

```
ratio milestone <id> <name> [--note <text>]
```

- Sets `milestone_name` on the node
- If `--git-bridge` is active, creates a Git commit in the shadow repo at this point
- Milestones are the human-established "save points" — the equivalent of commits

---

### `ratio diff <id1> <id2>`

Compare two nodes.

```
ratio diff <id1> <id2> [--files-only] [--summary-only]
```

**MVP output:**
- List of files added / modified / deleted between the two snapshots
- Per-file unified text diff
- Metadata delta (model, prompt, confidence)

**Example:**
```
$ ratio diff node:3c8f12 node:7b2d84

  M  auth/middleware.py    (+23 -41)
  A  auth/jwt.py           (+187)
  D  auth/session.py

  Prompt changed:
    before: "Scaffold auth module"
    after:  "Refactor auth to use JWT"
  
  Confidence: 0.72 -> 0.87
```

---

## 5. Config File

`.ratio/config.json`

```json
{
  "version": 1,
  "default_model": "claude-sonnet-4-20250514",
  "default_context": ["**/*"],
  "ignore": [".ratio", "node_modules", "__pycache__", ".git"],
  "git_bridge": true,
  "capture": {
    "reasoning": true,
    "actions": true,
    "context_snapshot": true,
    "max_blob_size_mb": 10
  },
  "display": {
    "default_log_depth": 20,
    "truncate_reasoning_chars": 2000
  }
}
```

---

## 6. Agent Integration Protocol

Ratio wraps agents via a **capture adapter**. For MVP, the adapter intercepts:

1. **Stdin/stdout** of subprocess agents (e.g. Claude Code)
2. **File system writes** via an FS watcher during the run
3. **Tool call logs** streamed as newline-delimited JSON to a capture socket

The adapter does not need to modify the agent. It observes from the outside.

### Capture Socket Protocol

Agents that natively support Ratio emit events to a Unix socket at `$RATIO_SOCKET`:

```jsonl
{"type": "reasoning", "text": "I need to first read the file..."}
{"type": "tool_call", "tool": "read_file", "input": {"path": "auth/session.py"}}
{"type": "tool_result", "output": "...file contents..."}
{"type": "file_write", "path": "auth/jwt.py", "content_hash": "ab12cd..."}
{"type": "done", "summary": "Replaced session auth with JWT", "confidence": 0.87}
```

For agents without native support, Ratio falls back to FS-watch + stdout parsing.

---

## 7. Git Bridge

When `git_bridge: true`, Ratio maintains a shadow bare Git repo at `.ratio/git/`.

- On every `ratio milestone`, Ratio commits the full file snapshot to the shadow repo, using the node id and milestone name as the commit message.
- The shadow repo can be pushed to any Git remote: `ratio git remote add origin <url>`
- `ratio git push` pushes the shadow repo, enabling GitHub/GitLab compatibility for teams not yet on Ratio natively.

This allows Ratio repos to be reviewed in standard Git tooling while preserving the full reasoning history in `.ratio/`.

---

## 8. MVP Implementation Plan

### Phase 1 — Storage Layer
- [ ] Content-addressed blob store
- [ ] SQLite schema and Node CRUD
- [ ] ContextSnapshot serialization
- [ ] `ratio init`

### Phase 2 — Capture
- [ ] FS watcher for file-write capture
- [ ] Unix socket capture server
- [ ] Claude Code adapter (subprocess wrapper)
- [ ] Node finalization + HEAD update

### Phase 3 — CLI
- [ ] `ratio run`
- [ ] `ratio log` (tree + list formats)
- [ ] `ratio show`
- [ ] `ratio checkout`
- [ ] `ratio milestone`
- [ ] `ratio diff`

### Phase 4 — Git Bridge
- [ ] Shadow repo init
- [ ] Milestone → Git commit
- [ ] `ratio git` passthrough

### Phase 5 — Polish
- [ ] `ratio config`
- [ ] Ignore patterns
- [ ] Large blob handling (truncation + warnings)
- [ ] Error recovery (interrupted runs leave pending nodes)

---

## 9. Success Criteria for MVP

| Criterion | Target |
|---|---|
| `ratio run` captures a full Claude Code session end-to-end | ✓ |
| Full reasoning trace retrievable via `ratio show` | ✓ including chain-of-thought |
| Can checkout any historical node state | ✓ exact file restoration |
| `ratio diff` shows meaningful file-level delta | ✓ unified diff + metadata |
| Git bridge works with GitHub | ✓ `git push` compatible |
| Storage overhead per node | < 2x raw file size |
| `ratio log` renders a tree of 50+ nodes | < 200ms |

---

## 10. Future Directions (Post-MVP)

- **Semantic diffing** — AST-aware diffs for code; intent diffs for prose
- **Probabilistic branching** — auto-fork when agent expresses uncertainty
- **LLM-judge evaluation** — attach quality scores to nodes
- **Multi-agent graphs** — hypergraph history for collaborative agent sessions
- **`ratio replay`** — re-run any node with a different model or prompt
- **`ratio bisect`** — binary search the tree for a regression
- **Remote repos + collaboration**
- **Web UI** — visual reasoning tree explorer

---

*Ratio MVP — v0.1 draft*
