<div align="center">

```
██████╗  █████╗ ████████╗██╗ ██████╗
██╔══██╗██╔══██╗╚══██╔══╝██║██╔═══██╗
██████╔╝███████║   ██║   ██║██║   ██║
██╔══██╗██╔══██║   ██║   ██║██║   ██║
██║  ██║██║  ██║   ██║   ██║╚██████╔╝
╚═╝  ╚═╝╚═╝  ╚═╝   ╚═╝   ╚═╝ ╚═════╝
```

**Version control for agentic content generation.**  
*Git versions files. Ratio versions reasoning.*

[![License: MIT](https://img.shields.io/badge/License-MIT-black.svg)](LICENSE)
[![Status: MVP](https://img.shields.io/badge/Status-MVP-orange.svg)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)]()

</div>

---

Git was designed for humans making deliberate, line-level changes. It has no concept of *why* a change was made, what the agent considered and rejected, or what the prompt was that produced the output.

When an agent rewrites 400 files in a single shot, `git diff` tells you *what* changed. It tells you nothing about the decision that caused it — and that decision is exactly what you need when something goes wrong, when you want to reproduce a result, or when you need to audit what happened.

Ratio captures the full picture: **prompt → reasoning → actions → output**, structured as a queryable decision tree you can traverse, replay, and audit.

---

## How It Works

Every `ratio run` creates a **decision node** — not just a snapshot of your files, but the complete record of an agent session:

```
node:7b2d84
├── prompt:    "Refactor auth module to use JWT"
├── context:   12 files · claude-sonnet-4 · t=0.7
├── reasoning: "The current implementation uses Redis-backed sessions.
│               To migrate I need to: (1) create jwt.py with encode/decode
│               helpers, (2) update middleware to validate Bearer tokens..."
├── actions:   read_file × 3 · write_file × 2 · delete_file × 1
├── output:    auth/jwt.py (+187) · auth/middleware.py (+23 -41)
└── confidence: 0.87
```

Nodes form a tree. Branches emerge naturally when you explore alternatives, retry a failed run, or fork from a previous state. Every intermediate state is preserved. Nothing is lost.

---

## Quick Start

```bash
# Install
pip install ratio-vcs

# Initialize a repo
cd my-project
ratio init --git-bridge

# Run an agent session (captured automatically)
ratio run "Add input validation to all API endpoints"

# Browse the decision tree
ratio log

# Inspect what the agent actually did and why
ratio show node:7b2d84 --reasoning --diff

# Restore any previous state
ratio checkout node:3c8f12

# Promote a node to a named milestone (like a commit)
ratio milestone node:7b2d84 "auth-jwt-migration"
```

---

## CLI Overview

| Command | Description |
|---|---|
| `ratio init` | Initialize a repo in the current directory |
| `ratio run "<prompt>"` | Run an agent session with full capture |
| `ratio log` | Browse the decision tree |
| `ratio show <id>` | Inspect a node: prompt, reasoning, actions, diff |
| `ratio checkout <id>` | Restore filesystem to any node's state |
| `ratio milestone <id> <name>` | Promote a node to a named milestone |
| `ratio diff <id1> <id2>` | Compare two nodes |

---

## `ratio log`

```
$ ratio log

* node:7b2d84  [HEAD]
|  "Replaced session-based auth with JWT"
|  2 min ago · claude-sonnet-4 · conf: 0.87

* node:3c8f12  [milestone: initial-auth]
|  "Scaffolded auth module with session support"
|  1 hour ago · claude-sonnet-4 · conf: 0.72

o node:1a4e90  (branch — abandoned)
|  "Attempted OAuth flow"
|  1 hour ago

* node:0d2b11  [root]
   "Init"
```

---

## `ratio show`

```
$ ratio show node:7b2d84 --reasoning --diff

NODE  node:7b2d84
Type  STEP
Time  2026-03-09 14:32:01
Model claude-sonnet-4 (t=0.7)
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

DIFF  auth/jwt.py  (new file)
  + import jwt
  + SECRET_KEY = os.environ["JWT_SECRET"]
  + ...
```

---

## `ratio diff`

```
$ ratio diff node:3c8f12 node:7b2d84

  M  auth/middleware.py    (+23 -41)
  A  auth/jwt.py           (+187)
  D  auth/session.py

  Prompt changed:
    before: "Scaffold auth module"
    after:  "Refactor auth to use JWT"

  Confidence: 0.72 → 0.87
```

---

## Why Not Git?

Git assumes humans are the authors. It stores what changed, not why. It has no concept of:

- The **prompt** that triggered a change
- The **reasoning** the agent used to arrive at a solution
- The **alternatives** the agent considered and discarded
- The **intermediate states** produced during generation
- The **model, temperature, and context** that produced the output

Ratio is built for the agentic loop — where the real unit of work is a decision, not a line edit.

| | Git | Ratio |
|---|---|---|
| Unit of change | line diff | decision node |
| History | linear + branches | reasoning DAG |
| `blame` → | who wrote this line | which prompt caused this |
| `log` → | commit messages | prompt + reasoning traces |
| Branching | manual | automatic on retry/fork |
| Intermediate states | lost unless committed | always captured |
| Context preserved | ✗ | ✓ model, params, files-in |

---

## Git Bridge

Ratio plays well with existing tooling. With `--git-bridge`, every milestone auto-commits to a shadow Git repo that you can push to GitHub, review in PRs, and treat like a normal repository.

```bash
ratio init --git-bridge
ratio git remote add origin https://github.com/you/your-repo
ratio git push
```

Your team gets a normal Git history. You get the full reasoning tree. Both coexist.

---

## Storage

Everything lives in `.ratio/` alongside your project:

```
.ratio/
  config.json       # model defaults, ignore patterns
  HEAD              # current node id
  db/ratio.db       # SQLite: nodes, actions, tags
  objects/
    blobs/          # content-addressed file snapshots (SHA-256)
    contexts/       # context window snapshots
    reasoning/      # chain-of-thought logs
  git/              # shadow Git repo (if --git-bridge)
```

No server. No cloud dependency. Local-first.

---

## Roadmap

- [x] Core data model (DecisionNode, ContextSnapshot, Action)
- [x] SQLite + content-addressed blob store
- [x] `ratio init / run / log / show / checkout / milestone / diff`
- [x] Claude Code capture adapter
- [x] Git bridge
- [ ] Semantic AST-aware diffing for code
- [ ] `ratio replay` — re-run any node with a different model or prompt
- [ ] `ratio bisect` — binary search the tree for a regression
- [ ] LLM-judge evaluation scores on nodes
- [ ] Probabilistic branching — auto-fork on agent uncertainty
- [ ] Multi-agent session graphs
- [ ] Remote repos + collaboration
- [ ] Web UI — visual reasoning tree explorer

---

## Contributing

Ratio is early and the design space is wide open. If you're building with agents and hitting the limits of Git, we'd love your input.

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/your-idea`
3. Open a PR with context on what problem you're solving

See [SPEC.md](SPEC.md) for the full MVP specification and data model.

---

## License

MIT © Ratio Contributors

---

<div align="center">
<sub>Built for the agentic era. Git for the rest.</sub>
</div>
