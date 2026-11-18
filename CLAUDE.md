## Build discipline

- Internal implementation and testing — scratch code, prototypes, verification harnesses, agent-driven checks — is done in **Rust**, regardless of the project's final language/framework. Reason: a compile gate (types, borrow-checking, exhaustive matching) forces structural correctness before anything runs, which makes the agent's internal loop fast and self-checking. Logic bugs still require tests; the compiler is a floor, not a ceiling.
- The **delivered artifact's** language/framework is a per-project decision, made during grilling and recorded as an ADR in `docs/adr/`.
- Exception: inside a repo with established conventions (e.g. an existing Python or TypeScript codebase), the repo's toolchain wins — don't introduce Rust into someone else's house.

## Project-local memory

- Keep ALL persistent project notes (decisions, state, working agreements) in `NOTES.md` at the repo root — it is git-ignored and lives only on this machine. Never write project-specific facts into user-level folders or global memory; everything related to a project stays inside the project directory.
- If `NOTES.md` exists, read it before starting project work.
- Settings hygiene: personal/machine overrides belong in `.claude/settings.local.json` (git-ignored). Never create or edit the shared `.claude/settings.json` to store personal config — it is team-committed and holds only shared tool config (e.g. graphify hooks). Keep every project-related file inside this project directory; nothing project-specific goes in user-home folders.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- When graphify-out/graph.json exists, run `graphify query "<question>"` before your first grep, search or source-file read for a codebase question, and put this rule in every subagent prompt that explores code. Read raw files only for lines the graph or an earlier read already located. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. When the answer is under a Markdown heading, `graphify section "<heading>"` prints that section's text, so the .md need not be opened. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
- A committed graphify-out/graph.json is rewritten whole on every build, so a large diff to it in a pull request is expected. Do not flag it as churn or suggest skipping the graph rebuild or its hooks.
