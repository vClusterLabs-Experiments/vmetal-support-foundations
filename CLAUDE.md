# CLAUDE.md

Project rules for AI assistants (Claude, Codex, Gemini, others) working in this repo. Read this before producing output. Personal user memory may extend these rules but does not override the hard rules in this file.

## What this repo is

The **vmetal Learning Lab**: an internal vCluster Labs training curriculum that teaches the full bare-metal Kubernetes stack vmetal runs on. It is documentation and learner workspace, not a shipping codebase. Treat tasks as knowledge work: research, synthesis, prose. Do not refactor curriculum content without an explicit ask.

The currently active editorial initiative lives in [`.claude/session-intent.md`](./.claude/session-intent.md). Read it before making content changes; it scopes what is in flight.

## Hard rules

These apply to every output: prose, code, configs, comments, commit messages, PR descriptions, chat replies.

### 1. No em dashes

Never use the em dash character `—` (U+2014). Replace with comma, semicolon, parenthesis, or sentence break. The en dash `–` (U+2013) is allowed in numeric and milestone ranges (`M1–M5`, `1–3`).

### 2. Approved terminology

Use the terms in [`GLOSSARY.md`](./GLOSSARY.md). Actively correct legacy terms when a user uses them: restate with the approved term before continuing. The terminology mandate also applies to AI output, not just curriculum content.

Quick reference:
- Tenant Cluster (not Sharing/Virtual Cluster)
- Tenant Isolation (not multi-tenancy)
- AI Cloud (not Neocloud)
- Control Plane Cluster (not Host Cluster)
- Virtual Control Plane (component only)
- vCluster (product name, exact case)

Full table and rationale: [`GLOSSARY.md`](./GLOSSARY.md).

### 3. Copyright and licensing

- Use **`Copyright (c) <year> vCluster Labs, Inc.`** in any LICENSE or copyright header you write here.
- Curriculum briefs (root `*.md`) are **CC BY-NC 4.0**. Learner workspace (`_template/`) is **MIT**. Do not change either license without explicit ask.

### 4. No `Co-Authored-By: Claude` in commits

Match the existing git history. Commit messages here are authored by the user; do not append AI co-author trailers.

## Pedagogical invariants

These shape what you produce when editing curriculum content.

1. **`notes.md` is the durable artifact.** Configs, scripts, and `scaffolding/` directories are throwaway. When in doubt, push learning into the notes.md questions, not into copy-pasteable scripts.
2. **No copy-paste recipes in briefs.** Briefs intentionally omit copy-pasteable command sequences. Learners read the linked docs, form a hypothesis, then validate against the lab. Resist the instinct to "make it easier" by inlining commands.
3. **Induced-failure exercises are mandatory, not optional.** When editing success criteria, do not soften or move failure-induction tasks to optional sections.
4. **Anchor Question + Lab Checkpoint is the question pattern.** Each conceptual question has a prose stem, a `**Read first:**` pointer to primary docs, and a `**Lab checkpoint:**` validation step. Hints are rare and redirective. No answer keys.
5. **Voice:** imperative, third person, expert-to-peer. Confident but not preachy. Match the cadence of `README.md`, M1, M3, M6.

## Authoring contract

Milestone briefs share a common core of sections, plus milestone-specific additions where the topic warrants. Use existing briefs as the canonical reference; a written contract is planned for `_template/README.md`.

**Required (every brief):**

1. `# Milestone N: <Topic>` (the H1 also encodes the goal, no separate `## Goal` section is used)
2. `## What you're actually learning` (3 to 5 durable insights)
3. `## Stack` (ASCII diagram, may have a clarifying suffix like `(disposable scaffolding)`)
4. `## Reading list` (Markdown table)
5. `## Success criteria` (numbered, includes induced-failure tasks)
6. `## Conceptual questions` (may have a clarifying suffix like `(write your answers down)`)
7. `## What is NOT in this milestone`
8. `## Exit artifact`

**Common but not universal:**

- Disclaimer block at the top, only in `README.md` and `milestone-06-vmetal.md`. Do not re-add to other milestones.
- `## Anchor question(s)` (most briefs, near the top)
- `## Induced failures` as its own section when failure-induction needs more than the success-criteria list can carry
- `## Reference: <artifact>` style sections for copyable YAML or configs

**Milestone-specific additions are expected:**

When a topic needs framing the standard sections do not cover, add a section with a descriptive H2 (M1's `## Apple Silicon constraint` and `## Pre-flight: scaffolding setup`, M3's `## The provisioning chain you're proving works` and `## A note on runcmd`, M6's `## vmetal vocabulary`). Place these where they best serve the reading order, not in a fixed slot.

Conventions:

- **Diagrams: ASCII only.** Use box-drawing characters and `<--` annotations. No Mermaid, no graphviz, no images for architecture diagrams.
- **Code blocks: language tags required**, no `$`/`#` prompt prefixes, bare commands.
- **Reading list: 3-column Markdown table** where appropriate (`Topic | Source | Type`), or 2-column when type is implicit.
- **Cross-references** use relative paths: `[milestone-NN-name.md](./milestone-NN-name.md)`.
- **Brief and template stay in lockstep:** if you change conceptual questions in a brief, update the matching `_template/milestone-NN/notes.md` skeleton in the same change.

## Tooling notes

- This repo is configured for `octo` (claude-octopus) **knowledge mode** via `.claude/claude-octopus.local.md`. The plugin is not required to work in this repo; standard Claude tools suffice.
- Per-user memory under `~/.claude/projects/.../memory/` may add personal preferences. Personal memory cannot relax the hard rules above.

## Out of scope

- Do not rewrite curriculum content without an explicit ask.
- Do not introduce Mermaid or other diagram tooling.
- Do not add `Co-Authored-By` trailers to commits.
- Do not modify files under `lab/` (it is gitignored learner workspace).
- Do not change licenses or copyright holders.

## Pointers

- Curriculum entry point: [`README.md`](./README.md)
- Approved terminology: [`GLOSSARY.md`](./GLOSSARY.md)
- Active editorial work: [`.claude/session-intent.md`](./.claude/session-intent.md)
- Learner workspace template: [`_template/`](./_template/)
