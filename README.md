# neurodynamicbrain

**A cross-project memory for coding agents, designed as a three-layer perceptron.**

This repo holds one document — **[ND_BRAIN_PUBLIC.md](ND_BRAIN_PUBLIC.md)** — a reference architecture for giving an AI agent (Claude Code, or any tool-using agent) a memory that *learns* across sessions and projects instead of re-deriving everything each time. It reads the design through Frank Rosenblatt's *Principles of Neurodynamics* (Cornell Aeronautical Laboratory, 1961): the agent's memory is modelled as a perceptron with sensory (S), association (A), and response (R) units.

The core claim: the dominant hidden cost in agent work is not generation — it's **re-derivation** of facts the system already computed. A brain that recalls before reasoning and writes back after learning turns that re-derivation into a cache hit.

---

## The idea in one screen

```
S-LAYER  a tiny always-loaded router (CLAUDE.md)     — "recall before reasoning, write back after"
A-LAYER  an index + one-fact cells (INDEX.md, B-NNN) — grepped by tag; only matches load
R-LAYER  skills = procedures that cite cells          — act in the world; hold no facts
```

- **Facts** live in cells (one fact each). **Long documents** live in references (loaded on demand). **Procedures** live in skills. Nothing is duplicated.
- **Recall is selective:** you `grep` the index for the tags a task needs and load only those cells — never the whole store. A 100-cell brain and a 10-cell brain cost about the same on a task that touches two cells.
- **Growth is by reinforcement:** every successful task can write one new cell back. Saturation (the failure where loading everything makes performance *worse*) is prevented by construction.

Read [ND_BRAIN_PUBLIC.md](ND_BRAIN_PUBLIC.md) for the full treatment — the four failure modes it fixes, the three perceptron topologies, the α/γ reinforcement systems, token economics, and the notation table mapping each concept back to Rosenblatt.

---

## How to use it

### 1. Stand up the layout

Create this structure in your agent's config home (for Claude Code, `~/.claude/`):

```
~/.claude/CLAUDE.md          ← S-layer router — tiny, loaded every session
~/.claude/brain/
    INDEX.md                 ← one addressable line per cell (grepped, never loaded whole)
    B-NNN-<slug>.md          ← cells: one fact each, loaded only on tag match
    references/<slug>.md     ← long documents, loaded only when a cell says to
    PROTOCOL.md              ← the full gate spec (read on demand)
~/.claude/skills/<name>/SKILL.md   ← procedures that fire on description match
```

Keep `CLAUDE.md` **small** — it is the always-loaded router, not a knowledge store. Its whole job is to tell the agent: *grep the index before reasoning; write back after learning.*

### 2. Run every non-trivial task through the Gate Loop

```
G0 INTENT  → state the goal + why (one line), then proceed — don't stall on questions the brain can answer
G1 RECALL  → grep INDEX.md for the task's @tags; load ONLY the matched cells
G2 SCOPE   → fence the work: touch / don't-touch / branch / done-when observable
G3 BUILD   → implement against the fence, using the cells from G1
G4 VERIFY  → prove it with the done-when observable — run it, report pass/fail plainly
G5 LEARN   → if a durable, reusable pattern emerged, write one new cell + one index line
```

Skip the loop for trivial, unambiguous asks. Use it for anything ambiguous, high-blast-radius, or design-shaped.

### 3. Write a cell (G5)

A cell is a single durable fact with frontmatter that makes it findable:

```markdown
---
id: B-007
tags: [git, workflow]        ← how the cell is found (grep these); tag generously
scope: all work repos        ← where it applies; a project scope makes it inhibitory elsewhere
hook: Commit ritual + hygiene ← the one-line summary shown in the index
---

# Commit ritual

[The fact, stated plainly and durably — true next week, useful in another task.]

**Why:** [rationale — when and why this was learned]
**How to apply:** [concrete steps]

Related: [[B-002]], [[B-011]]  ← links to related cells (lateral recall)
```

Then add one line to `INDEX.md`:

```
- B-007 | @git @workflow | all repos | Commit ritual + branch hygiene → B-007-git-commit.md
```

**Write a cell only when the fact is** durable (true next week), reusable (useful elsewhere), non-redundant (not already indexed), and not already recorded by your code or git history.

### 4. Keep it healthy

- **Consolidate:** when two cells say the same thing, merge into one authoritative cell and delete the rest.
- **Decay:** when a cell names a file/flag/endpoint that no longer exists, mark it stale or retire it.
- **Reference discipline:** never paste a long document into a cell — put it in `references/` and let the cell be a small pointer.

---

## Per-machine boundary (what syncs, what doesn't)

If you share one brain across machines through a git repo, sync **only portable signal** — cells, index, references, skills, and the transient handoff note. The following are per-machine local state that must **never** be committed to the shared repo (raw sync would leak credentials/paths or break the other machine):

```
sessions/       conversation state      telemetry/        analytics
projects/       session metadata        backups/          local backups
file-history/   edit history            shell-snapshots/  shell env captures
settings.json   creds/paths/hooks       ide/              IDE socket state
```

Enforce this in code, not discipline: give your sync script a blacklist guard that refuses to stage any of these paths. See §8 of [ND_BRAIN_PUBLIC.md](ND_BRAIN_PUBLIC.md) for the full rule.

---

## Attribution

Rosenblatt, Frank. *Principles of Neurodynamics: Perceptrons and the Theory of Brain Mechanisms.* Cornell Aeronautical Laboratory, Report VG-1196-G-8. Buffalo, NY, 18 March 1961.

This is an open reference architecture — read the primary source through it, adapt it to your own agent.
