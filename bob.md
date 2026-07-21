# The Brain of Brains ( BOB )

*A portable architecture for giving your AI coding agent a unified, addressable, token-efficient memory — plus the procedures to act on it — across every project you work on.*

This guide is generic. Copy the templates, swap in your own facts, and you have the system running in ~15 minutes. It assumes an agent that (a) loads a global instruction file on every session and (b) can read/grep/write local files. Examples use [Claude Code](https://claude.com/claude-code) paths (`~/.claude/`), but the pattern ports to any agent with a global config file and filesystem access.

---

## 1. The problem it solves

Most AI-agent memory is **fragmented and lossy**:

- **Per-project silos.** What the agent learned about your conventions in repo A is invisible in repo B. You re-teach the same rules ("don't add that commit trailer", "size dev cheaply") in every repo.
- **Token waste through re-derivation.** Each session re-reasons facts it already figured out last week — re-reading code, re-deriving architecture, re-asking questions. That's tokens (and latency) spent rebuilding state that should be cached.
- **Load-everything memory.** Dumping all memory into the context window every session is the opposite failure: it's expensive and drowns the signal.
- **Drift and duplication.** The same rule, written slightly differently in five places, eventually contradicts itself.

The Brain of Brains fixes all four with one idea: **one unified memory, addressable like a filesystem, recalled selectively like a search index, and consulted before the agent does expensive reasoning.**

---

## 2. The mental model

Think of it as a human brain with explicit, addressable memory locations:

| Brain analogy | The system | What it does |
|---|---|---|
| **Reflexes / operating rules** | A tiny global instruction file | Loaded every session. Says *how to use the brain* — never holds the knowledge itself. |
| **Hippocampus (the index)** | `INDEX.md` | A cheap-to-scan map: one tagged line per memory. You grep it to find *where* a memory is. |
| **Memory cells** | `B-NNN-*.md` files | One fact/pattern each. Loaded **only** when its tag matches the task. |
| **Motor skills** | Skills / procedures | Invokable workflows that *act on* the facts (commit rituals, review checklists, API calls). |

The win is **selective recall**: scan a small index → load only the one or two cells the task needs → act. You never pay to load the whole brain.

---

## 3. The three layers

```
~/.claude/CLAUDE.md        Layer 1 · PROTOCOL   tiny, loads every session — how to use the brain
~/.claude/brain/
    INDEX.md               Layer 2 · INDEX      addressable tagged map (grepped on demand); grouped into regions
    B-NNN-<slug>.md        Layer 2 · CELLS      one fact each (loaded only on tag match)
    references/<slug>.md   Layer 2 · STORE      deferred long-docs; reached via a small @reference handle cell, loaded on demand
    PROTOCOL.md            Layer 2 · SPEC       full gate + reference/decay/consolidation detail (read on demand)
    SKILLS.md              Layer 3 · CATALOG    R-unit registry: each skill + the cells it cites
~/.claude/skills/
    <skill>/SKILL.md       Layer 3 · SKILLS     procedures that act on the facts (auto-activate)
    <skill>/*.sh|py|mjs    Layer 3 · EFFECTORS  retained reusable code a skill runs (its arms/legs)
    _lib/                  Layer 3 · EFFECTORS  effector code shared across skills
```

**Rule of separation:** the **brain stores facts**; **skills store procedures** (and their retained effector code); **references store long documents**. A fact ("commits use identity X") is a cell. A procedure ("run the commit ritual…") is a skill that *cites* that cell and may call a retained script. A long spec is a `references/` doc behind a one-line handle cell. One source of truth — skills stay thin, cells stay authoritative, long text stays out of context until needed. (This three-layer split is the S/A/R perceptron model — see brain cell B-059.)

---

## 4. The Gate Loop

Every non-trivial task runs through six gates. This is what makes the memory *get used* instead of ignored.

| Gate | Name | What happens |
|---|---|---|
| **G0** | INTENT | Establish goal **+ why**. If the user gave intent, echo it in one line and proceed. If not, infer it, state your one-line read, and proceed — don't stall on questions the brain or code can answer. |
| **G1** | RECALL | Grep the index for matching tags; load only the matched cells; **reuse cached knowledge instead of re-deriving it.** This is the token-saving gate. |
| **G2** | SCOPE | Fence the work: what to touch / not touch / which branch / the done-when observable. |
| **G3** | BUILD | Implement against the fence. |
| **G4** | VERIFY | Prove it with the done-when observable (run it, check the row, hit the endpoint). Report plainly. |
| **G5** | LEARN | If a durable, reusable pattern emerged, write it back as a new cell (+ index line, tagged). |

Skip gates for trivial, unambiguous asks. Use the full loop for ambiguous, high-blast-radius, or design work.

**Two token leaks the loop plugs:** G1 (re-derivation → cache hit) and G0 (clarification round-trips → explicit intent up front).

---

## 5. Setup in 6 steps

1. **Create the brain dir:** `mkdir -p ~/.claude/brain ~/.claude/skills`
2. **Write the global protocol** (`~/.claude/CLAUDE.md`) — template in §6. Keep it small; it loads every session.
3. **Seed the index** (`~/.claude/brain/INDEX.md`) — template in §6.
4. **Write your first cells** — start with a `user` cell (who you are, how you work) and your top 3–5 conventions. Template in §6.
5. **Drop in `PROTOCOL.md`** — the full gate spec — so sessions can read the detail on demand without it costing tokens every time.
6. **Retrofit procedures into skills** as they emerge (§7). Don't pre-build skills you don't need — let G5 surface them.

That's it. From the next session, the agent reads the protocol, and on each task greps the index and pulls only what's relevant.

---

## 6. Templates (copy these)

### 6a. `~/.claude/CLAUDE.md` — the global protocol (keep it tiny)

```markdown
# Global Operating Protocol — read once per session

This file loads into every session across every project. Keep it SMALL.
The knowledge lives in the brain (~/.claude/brain/), not here. This only says how to USE it.

## The Brain (unified cross-project memory)
- Index: ~/.claude/brain/INDEX.md — one addressable line per cell, tagged @region.
- Cells: ~/.claude/brain/B-NNN-*.md — one fact each. Load a cell ONLY when its tag matches.
- RECALL (before generating code, planning, or researching):
  grep INDEX.md for the @tags relevant to the task → read only the matched cells.
  If the brain already answers it, REUSE it. Do not re-derive what's cached.
- WRITE-BACK (after solving something non-obvious):
  append a new B-NNN cell + one INDEX line. Tag it. Link related cells with [[B-NNN]].
  Promote anything that applies to >1 project.
- Full detail: ~/.claude/brain/PROTOCOL.md (read on demand).

## The Gate Loop — run every non-trivial task through these
- G0 INTENT  — goal + why; echo a one-line read and proceed.
- G1 RECALL  — consult the brain; reuse, don't re-derive.
- G2 SCOPE   — touch / don't-touch / branch / done-when.
- G3 BUILD   — implement.
- G4 VERIFY  — prove it with the done-when observable; report plainly.
- G5 LEARN   — write durable patterns back to the brain (and a skill if it's a procedure).
Skip gates for trivial/terse asks. Use the full loop for ambiguous or high-risk work.

## Intent shorthand (user types this to save round-trips)
>> <goal> | why: <intent> | touch: <files/layer> | done: <observable>
Any field omittable. When present, treat it as the G0/G2 contract and skip clarification.

## Skills (procedures over the brain)
List your ~/.claude/skills/ here, each with the cells it cites. They auto-activate by description.
```

### 6b. `~/.claude/brain/INDEX.md` — the hippocampus

```markdown
# Brain Index — addressable memory map

Grep for @tags matching the task, then read ONLY the matched B-NNN cells.
Format: `B-NNN | @tags | scope | one-line hook → file`

Tag regions: @user @prompting @git @workflow @code @infra @reference @gotcha @meta @arch:<x> @project:<x>

---
- B-001 | @user @meta | global | Who I am + how I work → B-001-user-profile.md
- B-002 | @git | all work repos | Commit/PR conventions → B-002-git-conventions.md
- B-003 | @workflow | global | The Gate loop + write-back discipline → PROTOCOL.md
```

### 6c. A memory cell — `~/.claude/brain/B-NNN-<slug>.md`

```markdown
---
id: B-007
tags: [git]
scope: all work repos
hook: One-line summary used during recall
---

# Title

The fact, stated plainly and durably (true next week, useful in another task).

**Why:** the rationale / when it was established. (For preference/feedback cells, this matters most.)

**How to apply:** the concrete steps or checks to honor it.

Related: [[B-003]], [[B-012]].   ← link other cells liberally; a link to a not-yet-written cell is fine.
```

### 6d. `PROTOCOL.md` — the full spec (the part you read on demand)

Hold the detail that's too big for the always-loaded `CLAUDE.md`: the gate table, the **tag vocabulary**, and the **write-back discipline** (below). See §8.

### 6e. A skill — `~/.claude/skills/<name>/SKILL.md`

```markdown
---
name: safe-commit
description: >
  What it does + WHEN to use it (be specific so it auto-activates correctly and doesn't over-fire).
  Name the triggers and the skip conditions.
---

# Skill title

Facts + rationale live in the brain (cite the cells: B-002, B-007). This skill is the procedure.

## Steps
1. ...
2. ...
```

---

## 7. Retrofitting patterns into skills

Not every fact is a skill. Use this test:

- **Passive convention** ("never commit secrets", "use identity X") → **cell only.** It's an always-on rule the agent applies; the protocol + cell are enough.
- **Task-shaped procedure** ("run the deploy", "review a Terraform diff", "do the commit ritual") → **skill** that cites the relevant cells.

Good skills are **procedural, reusable, and bundle several cells into one motion.** Examples (generic):

| Skill | When it fires | Bundles |
|---|---|---|
| `safe-commit` | before any commit/PR | identity + branch hygiene + secret scan + no-trailer |
| `iac-guard` | editing IaC or before apply | sizing rules + no-shortcuts + no-secrets |
| `status-log` | after a deploy/infra event | where + how to log status |
| `task-gates` | ambiguous / high-risk task | runs the Gate loop |

Keep skills **thin**: the steps + a pointer to the cells. When a cell changes, the skill still works because it doesn't duplicate the fact.

---

## 8. Operating discipline

### Tag vocabulary (regions of the brain)
Pick a small, stable set and reuse it. A starter kit:
`@user @prompting @git @workflow @code @infra @reference @gotcha @meta`, plus namespaced
`@arch:<system>` and `@project:<name>` for system- or project-specific knowledge. Tag generously —
a cell with three tags is found three ways.

### Write-back discipline (G5)
Write a cell when something is **durable and reusable** (true next week, useful elsewhere). Do **not**
write conversation trivia or facts the codebase/git already records.

1. Pick the next free `B-NNN`.
2. Create the cell with `id / tags / scope / hook` frontmatter + the fact.
3. Add one line to `INDEX.md`.
4. Link related cells with `[[B-NNN]]`.
5. **Promote** anything spanning >1 project to the global brain.

### Consolidation (dedup)
When you find the same rule written in N places, **merge into one cell** and delete the duplicates.
Five slightly-different copies of a rule will eventually contradict each other; one authoritative cell
won't. The index line is the only place the cell's existence is advertised.

### Conflict reconciliation
When two cells disagree, the **stronger, repeated, more recent** signal wins. Record the resolution in
the surviving cell ("supersedes the old X note") so the contradiction doesn't resurface. Silence in one
context does not outvote explicit statements in several others.

### Hygiene
- **Verify before acting on stale-able facts.** If a cell names a file, flag, or endpoint, confirm it
  still exists before relying on it. Cells reflect what was true when written.
- **Delete cells that turn out wrong.** A wrong memory is worse than no memory.
- **Mark volatile cells** with the decay marker ("⚠ possibly stale — verify") so future sessions treat
  them with suspicion.
- **Keep the always-loaded `CLAUDE.md` small.** Everything heavy is on-demand. The whole point is that
  recall is cheap and selective.
- **Let `brain.sh` do the chores.** Beyond `doctor`/`next-id`/`tags`/`links`/`stats`, the read-only
  reports `stale` (decay scan), `dups` (consolidation candidates), `saturation` (cell count vs ceiling +
  aging) and `tags --undeclared` (vocabulary governance) surface what to clean up — they never auto-mutate.

### Long documents → references/
Don't paste a whole spec/transcript into a cell (that's a saturation bomb). Put it in
`brain/references/<slug>.md` and write a small `@reference` handle cell pointing to it — the
**reference-ingest** skill (and its `ingest.sh` effector) does the copy + provenance + handle + `doctor`.
Load the full doc only when a task needs the detail.

---

## 9. Why this saves tokens (the economics)

- **The protocol is loaded once per session and cached** — a fixed, small cost.
- **The index is grepped, not loaded** — you read matched lines, not the whole map.
- **Only matched cells enter context** — a git task loads git cells, nothing else.
- **G1 turns re-derivation into a cache hit** — the single biggest recurring saving. Re-reading code and
  re-reasoning architecture every session is the dominant hidden cost; the brain pays it once.
- **G0 + the intent shorthand remove clarification round-trips** — fewer turns, fewer tokens.

The brain grows, but per-task cost stays roughly flat because recall is selective. A 100-cell brain and
a 10-cell brain cost about the same on a task that touches two cells.

---

## 10. Adoption tips & anti-patterns

**Do**
- Start tiny: a `user` cell + your top 3 conventions. Let the brain grow via G5.
- Write cells in plain, durable language — a teammate (or future you) should understand them cold.
- Re-run consolidation occasionally: scan the index for near-duplicate hooks and merge.

**Don't**
- Don't put knowledge in `CLAUDE.md`. It's the *router*, not the *store*.
- Don't auto-load the whole brain "to be safe" — that defeats the design.
- Don't duplicate a fact into a skill; cite the cell.
- Don't let cells rot — verify file/flag/endpoint references, delete wrong ones.
- Don't pre-build skills speculatively; retrofit them when a procedure actually repeats.

---

## 11. One-screen summary

> **Protocol** (tiny, always loaded) tells the agent to **RECALL before reasoning** and **WRITE-BACK after learning**.
> The **Index** is a cheap tagged map; **Cells** are one-fact memories loaded only on a tag match.
> **Skills** are thin procedures that act on the cells. Every task runs the **Gate loop**
> (Intent → Recall → Scope → Build → Verify → Learn). Facts live in the brain; procedures live in skills;
> nothing is duplicated; recall is selective, so it stays cheap as it grows.

*Stand it up once, globally, and every project inherits the same brain.*
