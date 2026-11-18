# Claude Code Workflow Skeleton — idea → deployment

A reusable project skeleton for driving any feature from a vague idea to a merged, deployed PR using [Matt Pocock's skills](https://github.com/mattpocock/skills) plus [graphify](https://pypi.org/project/graphifyy/) for repo understanding. Copy this skeleton into any project and the whole pipeline is live — no line-by-line repo reading, no ad-hoc prompting.

## What's in the skeleton

| Path | What it is | Copy to new project? |
|---|---|---|
| `.claude/skills/<name>/` | 15 skills (13 pipeline skills + `graphify` engine + your `repo-map` alias; whole folders — `SKILL.md` + support files), auto-discovered by Claude Code from the project | ✅ yes |
| `.claude/settings.json` | graphify's hooks: `PreToolUse` guard on Bash/Grep/Read/Glob (forces graph-first), `SessionStart` rule re-injection after compaction | ✅ yes |
| `.claude/settings.local.json` | per-machine overrides (proxy env, theme…) — **contains secrets**, git-ignored | ❌ never committed; copy the example instead |
| `.claude/settings.local.example.json` | template for the above (token redacted) — safe to commit | ✅ yes; per machine: copy to `settings.local.json`, fill in your token |
| `CLAUDE.md` (project root) | the permanent rules graphify installs: "map before grep", keep graph fresh | ✅ yes — the most-missed file |
| `docs/adr/` | architecture decision records (written by `domain-modeling` during grilling) | ✅ yes (empty) |
| `GLOSSARY.md` (root, appears later) | the domain language, written during grilling | ✅ yes once it exists |
| `graphify-out/` | this repo's knowledge graph — **answers belong to this repo only** | ❌ never; rebuild per project with `/repo-map .` |

## Prerequisites (once per machine)

| Tool | Install (Windows) | Verify | Why |
|---|---|---|---|
| Claude Code CLI | per Anthropic docs | `claude --version` | runs the pipeline |
| git | `winget install --id Git.Git` | `git --version` | commits, branches, PRs |
| **Rust** | `winget install --id Rustlang.Rustup`, then `rustup default stable` | `cargo --version` | required: internal implementation/testing happens in Rust (see `CLAUDE.md` → "Build discipline") |
| uv | `winget install --id astral-sh.uv` | `uv --version` | installs the graphify CLI (Option B step 2) |

### Keeping project files inside the project folder

By default Claude Code's **auto memory** writes notes about each project into your home folder (`~/.claude/projects/<project>/memory/`). If you want all project-related files to stay inside the project directory instead of spread across the system, turn it off with `"autoMemoryEnabled": false` — either **globally** in `~/.claude/settings.json` (recommended: it's a machine-wide switch, so it's the one setting that legitimately lives in the user folder) or **per project** in `.claude/settings.local.json` (the included example already sets it, so copying the example covers each new project). With auto memory off, this skeleton keeps its persistent notes in `NOTES.md` at the repo root (git-ignored) — see the "Project-local memory" rule in `CLAUDE.md`. Claude Code still keeps its own **session transcripts** under `~/.claude/projects/`; that's its history record, not project data, and is not relocatable.

## Setup in a new project

### Option A — copy the skeleton

```bash
cp -r <skeleton>/.claude/skills <project>/.claude/
cp <skeleton>/.claude/settings.json <project>/.claude/
cp <skeleton>/CLAUDE.md <project>/
mkdir -p <project>/docs/adr
```

Prerequisite: the `graphify` CLI must be on PATH (see Option B step 2). Then in the new project: `/repo-map .` to build its own map.

### Option B — build from scratch

```bash
# 1. Skills (copy whole folders — several ship support files)
#    Any local location works — set SKILLS to wherever you cloned it.
git clone https://github.com/mattpocock/skills "$HOME/pocock-skills"   # once per machine
SKILLS="$HOME/pocock-skills"
mkdir -p .claude/skills
for s in engineering/domain-modeling engineering/to-spec engineering/to-tickets \
         engineering/implement engineering/tdd engineering/code-review engineering/pr engineering/wizard \
         engineering/retro engineering/prototype productivity/grill-me productivity/grilling productivity/handoff; do
  cp -r "$SKILLS/skills/$s" .claude/skills/
done
rm -rf .claude/skills/*/agents   # Codex-only files, not needed for Claude Code

# 1b. Personal entry points (NOT stock upstream — recreate exactly):
#  - grill-me's upstream body calls only `grilling`; ours also calls `domain-modeling`
#    (the interview writes the GLOSSARY/ADRs as decisions harden):
cat > .claude/skills/grill-me/SKILL.md <<'EOF'
---
name: grill-me
description: A relentless interview to sharpen a plan or design, which also creates docs (ADR's and glossary) as we go.
disable-model-invocation: true
---

Call the Skill tool twice, for "grilling" and "domain-modeling".
EOF
#  - repo-map is our alias for graphify's vendor-named skill:
mkdir -p .claude/skills/repo-map && cat > .claude/skills/repo-map/SKILL.md <<'EOF'
---
name: repo-map
description: Alias for graphify — build/refresh the repo's knowledge-graph map and answer questions from it instead of reading files line by line.
disable-model-invocation: true
---

Call the Skill tool with "graphify", forwarding any arguments you were given.
EOF

# 2. graphify (once per machine; project registration per project)
# NOTE: the PyPI package is "graphifyy" (two y's; CLI is `graphify`). Pin the version —
# source repo: https://github.com/Graphify-Labs/graphify (Apache-2.0). Verify before installing.
uv tool install graphifyy==0.9.112
graphify install --project        # skill files + CLAUDE.md into this project only
graphify claude install --project # /graphify skill + PreToolUse hooks into .claude/
graphify hook install             # optional: git hooks keep the graph fresh on commit
```

Verify: type `/grill` — autocomplete offers `/grill-me`; type `/repo-map` — the alias exists. (`/graphify` also stays registered — graphify's updater owns that skill file; `/repo-map .` and `/graphify .` are the same command.) First `/repo-map .` builds the map; `graphify-out/graph.json` appearing means it worked.

## The pipeline, phase by phase

**Golden rule of context:** grilling → `/to-spec` → `/to-tickets` run in ONE unbroken session (the "smart zone"). Each `/implement` starts a FRESH session. Crossing either boundary without thinking is the #1 way this pipeline degrades — use `/handoff` or `/clear` deliberately.

### Phase 0 — Understand the repo (continuous)
| Skill | Usage | What it does |
|---|---|---|
| `repo-map` → `graphify` | `/repo-map .` (identical to `/graphify .`) then it's automatic: `graphify query "<question>"`, `path "A" "B"`, `explain "<concept>"`, `section "heading"`; `graphify update .` after code changes | Turns the repo into a persistent knowledge graph; hooks force graph-first lookups so agents never grep blindly |

### Phase 1 — Sharpen the idea
| Skill | Usage | What it does |
|---|---|---|
| `grill-me` (ours: wraps `grilling` + `domain-modeling`) | `/grill-me <one-sentence idea>` | The interview: rounds of numbered questions, each with a recommended answer; stops only when nothing is left assumed. Writes as you decide |
| `grilling` | invoked by the wrapper (or say "grill me") | The interview engine: design tree, frontier questions, decisions stay yours |
| `domain-modeling` | runs inside grilling | Coins `GLOSSARY.md` terms and ADRs (`docs/adr/NNNN-slug.md`) the moment a decision hardens — only for hard-to-reverse + surprising + real-trade-off choices |
| `prototype` | `/prototype <design question>` | Throwaway code to answer one question (state model? UI shape?). Always bridged back by `/handoff` |
| `handoff` | `/handoff <next session's focus>` | Compacts this conversation into a temp-file brief for a fresh agent — carries decisions across session boundaries |

### Phase 2 — The spec (PRD)
| Skill | Usage | What it does |
|---|---|---|
| `to-spec` | `/to-spec` (same session as grilling!) | Synthesizes the grilling + docs into a PRD: Problem, Solution, User Stories, Implementation & Testing Decisions (incl. agreed test seams), Out of Scope. It interviews nobody — that's the point |

### Phase 3 — Break down
| Skill | Usage | What it does |
|---|---|---|
| `to-tickets` | `/to-tickets` | Slices the spec into tracer-bullet vertical tickets with blocking edges (ticket graph); each ticket ends at your one agreed seam |

### Phase 4 — Build (fresh session per ticket)
| Skill | Usage | What it does |
|---|---|---|
| `implement` | `/implement <ticket ref>` | Runs one ticket end-to-end: fresh context, builds on the committed result of blocking tickets, finishes with review + commit |
| `tdd` | invoked inside implement | Red-green at the pre-agreed seams only: failing test first, minimal code to pass. Support files: `tests.md`, `mocking.md` |
| `code-review` | after a ticket is built (say "review since <commit>") | Two axes in parallel sub-agents: **Standards** (repo conventions) vs **Spec** (does it match the originating ticket/spec). Note: name collides with Claude Code's built-in — the project skill should win; check which one fires. |

### Phase 5 — Ship
| Skill | Usage | What it does |
|---|---|---|
| `pr` | `/pr` (drafts body from the diff) | Smallest possible visual summary of what changed, before/after evidence, and a "danger on merge" call so a human can decide fast |
| `wizard` | `/wizard <what only humans can do>` | Generates an interactive bash wizard for steps no agent can do: cloud consoles, credential provisioning, CI secrets, production cutovers |

### Phase 6 — Deploy monitoring & improvement
| Skill | Usage | What it does |
|---|---|---|
| `retro` | `/retro` after shipping a milestone | Blameless retro on the *environment, not the code*: which skill/harness/docs gap caused friction → concrete improvements to this skeleton |
| *(diagnosing-bugs)* | not installed — pull from upstream if needed | Debug loop for hard bugs/regressions reported post-deploy: `cp -r "$SKILLS/skills/engineering/diagnosing-bugs" .claude/skills/` |

## House rules

- **Facts are the agent's job, decisions are yours.** Grilling dispatches sub-agents for environment lookups; every design choice goes through you.
- **One seam, highest possible point** for tests unless grilled otherwise — it is chosen during grilling and recorded in the spec's Testing Decisions.
- A whole-file diff in `graphify-out/graph.json` on any PR is **expected** — never treat it as churn (CLAUDE.md says so).
- Skills are project-scoped: they travel with the repo, so teammates and future agents get the same pipeline.
- Keep `.claude/settings.local.json` (secrets/machine config) out of git; commit `.claude/settings.json`, skills, `CLAUDE.md`, and `docs/`.
