# PRINCIPLES OF AGENT NEURODYNAMICS
## Brain of Brains — read through Rosenblatt's *Principles of Neurodynamics* (CAL VG-1196-G-8, 1961)

> *"A perceptron is first and foremost a brain model, not an invention for pattern recognition.
> As a brain model, its utility is in enabling us to determine the physical structures
> and neurodynamic principles which underlie natural intelligence."*
>
> — Frank Rosenblatt, Cornell Aeronautical Laboratory, 18 March 1961

---

## §0 · WHAT THIS IS, AND HOW TO LOAD IT

A cross-project agent memory, built as a three-layer perceptron. Facts live in cells, long documents in references, procedures in skills. Nothing is duplicated. Recall is selective — you grep for what a task needs, never load the whole store. The brain grows through reinforcement and stays cheap because saturation is architecturally prevented.

```
~/.claude/CLAUDE.md          ← S-layer router — tiny, loads every session
~/.claude/brain/
    INDEX.md                 ← A-unit registry (grepped, never loaded whole)
    B-NNN-<slug>.md          ← A-unit cells (one fact each — loaded on tag match)
    references/<slug>.md      ← long-term store (loaded only when a cell says to)
    PROTOCOL.md              ← full gate spec (read on demand)
~/.claude/skills/<name>/SKILL.md  ← R-unit motor programs (fire on description match)
```

**Session boot (once, automatically):**
1. Read `CLAUDE.md` — initializes the S-layer.
2. Per task, run the Gate Loop (§4).
3. G1 RECALL fires first — grep `INDEX.md` before reasoning.
4. Only matched cells enter context. Never load the whole brain.
5. After success, G5 LEARN writes new weights back.

---

## §1 · WHY AGENT MEMORY BREAKS

Rosenblatt opened his 1961 report by diagnosing why brain models fail. Each failure he named maps onto a failure of agent memory. The substrate changed; the diagnosis did not.

### The McCulloch-Pitts trap

McCulloch and Pitts (1943) proved a network of two-state logical devices could represent any psychological function definable in finite words. Their five postulates: all-or-nothing firing; fixed threshold; synaptic delay only; absolute inhibition; and — the lethal one — **fixed structure that does not change with time**.

Postulate 5 makes learning impossible. A fixed-structure network has no non-volatile memory, no state that survives a period of inactivity. Agent memory built as a static context dump *is* a McCulloch-Pitts network: perfectly logical, incapable of learning, doomed to re-derive everything each session. Rosenblatt's move was to relax Postulate 5 — make the weights variable. That single change turns a calculator into a brain, and it is what this system is built on.

### Four failure modes

- **FAILURE-01 · Localized memory.** Lashley's *equipotentiality*: memory traces are distributed, not pinned to the site of experience. Per-project silos do the opposite — knowledge from repo A is invisible in repo B, and one missing file makes the system inoperable. Rosenblatt on monotypic models: *"a single misconnection would be sufficient to make the system inoperable."*
- **FAILURE-02 · Zero-weight A-layer.** An agent that re-reads code and re-reasons architecture each session is a perceptron whose association matrix is reset to zero every trial. The dominant token cost in agent work is not generation — it is re-derivation of facts already computed.
- **FAILURE-03 · Saturation.** Rosenblatt proved bounded perceptrons reach *"a saturation condition in which their performance is actually poorer than during the transient learning phase."* Loading the entire brain each session *is* that regime: past the capacity ceiling, signal drowns in noise.
- **FAILURE-04 · Unreconciled conflict.** Five near-copies of a rule in five places have no reconciliation mechanism; they contradict. Without one authoritative A-unit, the output is undefined. Rule: *"the stronger, repeated, more recent signal wins; silence does not outvote explicit statements."*

---

## §2 · THE THREE SIGNAL UNITS

Rosenblatt's perceptron is a signal-transmission network of three unit types. Here they are not metaphors — they are files and filesystem operations.

**Signal** — *"any measurable variable… characterized by its amplitude, time, and location."* In agent terms: any information token moving through the system — a word in a prompt, a tag in a grep, a byte written to a cell.

### S-unit — Sensory (the task prompt + G0 INTENT)

> *"A sensory unit is any transducer responding to physical energy by emitting a signal which is some function of the input… output λᵢ = 1 if the input exceeds threshold θᵢ, and 0 otherwise."*

The S-layer transduces human language into a structured signal: a one-line goal, implied `@tags`, a done-when observable. It fires (λ=1) when the prompt clears the intent threshold — enough is present to proceed. It stays silent (λ=0) only when the prompt is ambiguous beyond what the brain can resolve.

| Input | Behavior |
|---|---|
| Clear goal + implicit tags | Fires; G0 echoes and proceeds |
| Goal, no context | Infers context from brain; fires; states the inference |
| Irresolvably ambiguous | Does not fire; **one** focused question |

The intent shorthand pre-saturates the S-unit — fires it at maximum amplitude with no clarification round-trip:
```
>> <goal> | why: <intent> | touch: <files/layer> | done: <observable>
```
Any field is omittable.

### A-unit — Association (the INDEX line + B-cell + tag-match at G1)

> *"An association unit generates an output signal if the algebraic sum of its input signals ∑ cᵢⱼ·λᵢ meets a threshold θ, and no signal otherwise."*

The A-unit is the core of the brain. **Each B-cell is one A-unit.** Its `@tags` are its coupling coefficients — the weights deciding whether it fires for a task. Its threshold is the match criterion: how many tags must overlap before the cell loads.

```markdown
---
id: B-007                        ← unit identifier
tags: [git, workflow]            ← coupling coefficients (excitatory weights)
scope: all work repos            ← stimulus domain — which inputs may activate it
hook: Commit ritual + hygiene    ← output signal when fired
---

# [Title of the fact]

[The fact, stated plainly and durably — true next week, useful in another task.]

**Why:** [rationale + when this weight was set]
**How to apply:** [concrete steps]

Related: [[B-002]], [[B-012]]    ← lateral A-A connections
```

The INDEX is the interaction-matrix register — grepped, not loaded:
```
- B-001 | @user @meta    | global    | Who I am and how I work → B-001-user-profile.md
- B-002 | @git           | all repos | Commit and PR conventions → B-002-git-conventions.md
- B-007 | @git @workflow | all repos | Commit ritual + branch hygiene → B-007-git-commit.md
```

**Activation (G1 RECALL):** parse the S-signal for implied tags → grep INDEX for them (the weighted-sum) → load each matched cell → follow relevant `[[Related]]` links → load a `references/` path only if the task needs document detail.

**Excitatory vs. inhibitory:** `@tags` are excitatory — a match pushes the cell toward firing. The `scope` field is inhibitory — a cell tagged `@project:repoA` is suppressed when the task is in repoB. Scope is what prevents irrelevant cells from firing across contexts.

### R-unit — Response (the external output + skills, at G3/G4)

> *"A response unit… emits a signal transmitted outside the network. r = +1 if the sum of its inputs is strictly positive, −1 if strictly negative."*

The R-unit fires *only after* the A-layer resolves which cells are active — never from raw S-input (that is the zero-A-layer failure). Its output is the compiled result of everything the A-layer knows about the task: code written, files changed, answers given.

**Skills are R-unit motor programs.** They sequence A-unit activations into external action. A skill holds no facts — it *cites* the cells it acts on, keeping Rosenblatt's separation of signal propagation from memory storage: the procedure never duplicates the weights.

```markdown
---
name: safe-commit
description: >
  Run before any commit/PR. Fires on: "commit", "push", "open PR", "merge".
  Skip for: documentation-only diffs with no runnable artifacts.
---

# Safe Commit — R-Unit Motor Program
Facts live in the brain (cited below). This skill is the procedure.

## Cells (load at G1)
- [[B-002]] commit/PR conventions · [[B-007]] commit ritual · [[B-011]] secret scan

## Steps
1. G1 RECALL: confirm B-002, B-007, B-011 loaded
2. Set identity per [[B-002]]  ·  3. Verify branch per [[B-007]]  ·  4. Secret scan per [[B-011]]
5. Commit; report plainly  ·  6. G5: write back any new pattern
```

---

## §3 · SIGNAL TOPOLOGY — THREE ARCHITECTURES

Rosenblatt classified perceptrons by connection topology. All three are present here.

**Series-coupled** — *"connections from units at logical distance d terminate on units at distance d+1."* The brain's three-layer path:

```
S-LAYER  ~/.claude/CLAUDE.md              distance 0 — loaded every session, constant cost
   ↓
A-LAYER  INDEX.md → B-NNN cells → references/   distance 1 — grepped, selectively loaded
   ↓
R-LAYER  ~/.claude/skills/<name>/SKILL.md  distance 2 — motor programs, emit external action
```
Facts travel S→A→R. The protocol lives at S, weights at A, procedures at R. Nothing propagates backward except reinforcement at G5.

**Cross-coupled** — *"connections join units of the same type, allowing lateral propagation."* The `Related: [[B-NNN]]` links are A-A cross-couplings: a loaded cell activates an adjacent cell it references without a new grep. This is how skill chains work — loading B-002 (git conventions) reaches B-007 (commit ritual) laterally, modeling the hippocampal spread of activation.

**Back-coupled (feedback)** — *"connections originate from R-units and terminate on A- or S-units, creating feedback loops."* This is **G5 LEARN**. When the R-unit fires successfully (G4 passes), a signal propagates back to the A-layer: a new cell is written, indexed, tagged — positive reinforcement of the pathway that worked. When G4 fails, no cell is written; if a wrong cell caused the error, it is deleted — error-correction that weakens the bad pathway. Back-coupling is also what gives the system **selective attention** (§10): firing a skill suppresses cells irrelevant to the task, which is why loading only matched cells beats loading everything.

---

## §4 · THE INTERACTION MATRIX

> *"The interaction matrix is the matrix of coupling coefficients between all pairs of units. The memory state is the configuration of all variable-valued connections at a specified time."*

**The interaction matrix is the complete set of B-cells + their tags.** Each cell is one row: id = row index, tags = column entries (coupling to task signals), hook + body = the value emitted when fired.

The matrix is sparse by design — most cells carry 2–4 tags, most tasks activate 2–5 cells. Per-task cost is proportional to *active* A-units, not total matrix size: **a 100-cell brain and a 10-cell brain cost about the same on a task that touches two cells.**

**Phase space** — *"the space of all possible memory states."* Here, the space of all possible INDEX configurations. Each G5 write-back steps the brain one move through it; the convergence theorems (§6) say which regions reinforcement drives it toward.

---

## §5 · THE GATE LOOP — THE REINFORCEMENT SYSTEM

> *"A reinforcement system is any set of rules by which the memory state may be altered through time. Positive reinforcement increases the value of a connection terminating on a correct response; negative reinforcement decreases one terminating on an incorrect response."*

The Gate Loop is a reinforcement system in Rosenblatt's exact sense — the rule set by which task experience alters the matrix.

```
G0  INTENT  ── S-unit transduction ──────────────── fire or wait
G1  RECALL  ── A-unit activation ────────────── grep → load cells
G2  SCOPE   ── stimulus boundary ──────────── fence the work domain
G3  BUILD   ── R-unit emission ──────────────── implement the fence
G4  VERIFY  ── error-correction feedback ── observable met? reinforce/prune
G5  LEARN   ── write-back reinforcement ── new cell + index line + tags
```

**G0 · INTENT.** Establish goal + why; echo a one-line read and proceed. Explicit intent is the G0/G2 contract — skip clarification. Otherwise infer from the brain (G1 fires first), state the inference, proceed. *Do not stall* — infer, proceed, correct at G4 if wrong. Each clarification round-trip is an S→A→R cycle with no external output; the shorthand eliminates it.

**G1 · RECALL — the token-saving gate.** Grep INDEX for tags matching the task; load only matched cells; reuse cached knowledge. **This is the single most important gate:** it converts re-derivation into a cache hit. Each tag is a coupling coefficient, the grep is the summation, a cell fires when its tag overlap meets threshold.

*Reference discipline (deferred potentiation):* long documents live in `references/`. A cell pointing to one is a small recall handle — it says the knowledge exists and how to reach it, but does not load the document until a task needs detail. The pathway exists; it fires only under the right conditions.
```markdown
---
id: B-014
tags: [reference, arch:platform]
hook: Platform API design spec — full reference
---
Brain reference: ~/.claude/brain/references/platform-api-spec.md
Provenance: ~/Downloads/Platform-API-Design-Spec.docx
Load the full reference only when reviewing API surfaces or contract detail.
Do not paste this document into context on recall. Related: [[B-003]], [[B-008]]
```

*Tag vocabulary:* `@user @prompting @git @workflow @code @infra @reference @gotcha @meta @arch:<x> @project:<x>` (project tags are inhibitory cross-project). **Tag generously** — a three-tag cell is found three ways; a one-tag cell is missed whenever the task uses a synonym.

**G2 · SCOPE — stimulus boundary.** Fence the work: touch / don't-touch / branch / done-when. The perceptron must know which stimulus class it is being trained on before the R-unit fires. **The done-when observable is the error-correction criterion** — without it, G4 produces no reinforcement signal and the brain cannot learn from the task.
```
Touch: [files/layers/services in scope]   Do not touch: [explicit exclusions]
Branch: [target]                          Done-when: [run X, see Y, check Z]
```

**G3 · BUILD — R-unit emission.** Implement against the fence using the active A-set. The R-unit acts on what the A-layer provided; it does not re-derive. **Never fire from raw S-input** — skipping G1 is the zero-A-layer failure: inconsistent output, every cached pattern missed.

**G4 · VERIFY — error-correction feedback.** Prove the output with the done-when observable — run it, hit the endpoint, read the log. Report plainly: pass or fail, not "it should work."
- Pass → positive signal → G5, consider writing a cell.
- Fail → negative signal → write nothing; identify and consider pruning the cell that misfired.
- Ambiguous → write nothing; one focused diagnostic step; re-verify.

*Convergence (Theorem 3):* if the correct answer is stored anywhere in the matrix, repeated G3→G4 cycles with honest error signals reach it in finite iterations. If it is not stored, the cycle surfaces the gap and G5 fills it.

**G5 · LEARN — write-back reinforcement.** If a durable, reusable pattern emerged, write a new cell, add one INDEX line, tag it, link related cells, promote cross-project facts globally. Creating the cell + assigning tags + registering it *is* incrementing coupling coefficients.

*Write when the fact is:* durable (true next week — the *pattern*, not the pass/fail detail) · reusable · non-redundant (checked the index) · not already in git.
*Do not write:* conversation trivia · what git already records · lucky G4 passes (verify twice for anything surprising) · speculation.
```
1. Next free B-NNN (highest in INDEX)     5. Related: links
2. Create B-NNN-<slug>.md w/ frontmatter  6. Append one INDEX line
3. State the fact plainly + durably        7. Cross-project → promote (drop @project)
4. Add Why + How to apply                  8. Long source doc → use reference-ingest
```

**Skip the loop for trivial, unambiguous asks** (rename a variable, define a term). The loop is for ambiguous, high-blast-radius, and design work. When in doubt, run it — running it on a trivial task wastes a few tokens; skipping it on a consequential one can cause an unrecoverable error.

---

## §6 · ALPHA AND GAMMA — REINFORCEMENT VARIANTS

Rosenblatt defined two reinforcement systems with different convergence properties. Both appear in write-back.

**Alpha (α) — unbounded.** *"All active connections on a correct R-unit increase by δ; all on an incorrect one decrease by δ."* Standard write-back: every success adds a cell (α-increment), every confirmed error removes or corrects one (α-decrement). Alpha is **universal** — it converges for any linearly-separable classification from any start — at the cost of unbounded growth. Consolidation, hygiene, and the durability test (§8, §5·G5) are what keep that growth from degrading the brain.

**Gamma (γ) — conservative.** *"Connections not terminating on an active unit are adjusted oppositely to keep the sum of all values constant."* Consolidation and dedup: when two cells say the same thing, the survivor absorbs the deleted one's value — knowledge is conserved, only redistributed to one authoritative location. This is why **one authoritative cell beats five copies**: alpha guarantees convergence, gamma guarantees conservation, dedup keeps the brain convergent.

---

## §7 · SIGNAL FORMS — CONCRETE ANATOMY

Every signal has amplitude, time, location, and coupling. Each below is a concrete file, string, or operation. (The S/A/R units are defined in §2; here are the two reinforcement signals and the anatomy fields that matter operationally.)

**S-signal (task):** `{goal, why, touch[], done_when, implied_tags[], amplitude}` where amplitude = HIGH (all fields → fire, proceed, no echo) / MEDIUM (goal + some → fire, echo the inference) / LOW (ambiguous → don't fire, one question). The intent shorthand is a pre-formed S-signal that fires at HIGH on receipt.

**A-signal (cell activation):** `{id, tags[], scope, hook, body, lateral[], ref_path?}`. Coupling: tag match = excitatory (loads) · scope mismatch = inhibitory (suppressed) · `Related:` = lateral spread · `ref_path` = deferred, activated only when G3 needs detail. **The hook is the A-unit's emission to the R-layer** — the one-line summary that lets the R-unit know knowledge exists without loading the full cell.

**R-signal (external emission):** `{type: code|command|file_edit|answer|plan, content, amplitude: +1|−1, feedback}`. Must be grounded in the active A-set, verifiable against the done-when, its amplitude set by G4 (not by confidence). Feedback writes cells only when amplitude = +1 and the pattern is durable.

**W-signal (write-back):** the reinforcement pulse that moves the brain through phase space. `{trigger: "G4 passed + durable pattern", new_cell, new_index_line, tags[], lateral[], type: positive|negative|null}`.

**D-signal (decay/hygiene):** the forgetting mechanism that prevents saturation on stale weights. Rosenblatt: *"forgetting is greatest for temporally remote events and negligible for recent ones… values decay in proportion to their distance from the most recent reinforcement."*
- *Trigger:* a cell names a file/flag/endpoint that no longer exists · contradicts a stronger, more recent cell · proved wrong at G4.
- *Action:* mark volatile · delete · update · merge into survivor (record `"supersedes old X"`).
- *Volatile marker:*
  ```markdown
  > ⚠ POSSIBLY STALE — verify before acting. This cell names [file/endpoint/flag].
  > Confirm it still exists; delete or update if not.
  ```

---

## §8 · OPERATING DISCIPLINE — HYGIENE

**Consolidation (gamma maintenance).** Alpha generates duplicates over time (every G5 that skips the index check adds one). When the same rule appears in N cells, merge into one authoritative cell and delete the rest.
```
1. grep INDEX for near-duplicate hooks    4. Union all tags onto the survivor
2. Load candidates, compare bodies         5. Note "supersedes B-NNN, B-NNN" in Why
3. Merge into one cell, delete the others  6. Remove deleted index lines
```
*Triggers:* a new write-back looks familiar (check first) · INDEX exceeds ~30 lines (scan for dup hooks) · a skill cites 4+ similar-looking cells.

**Conflict reconciliation.** *"The stronger, repeated, more recent signal wins; record the resolution so the contradiction does not resurface."* Signal strength, strongest first: (1) explicit, repeated, recent across multiple cells → winner · (2) explicit recent in one cell → strong · (3) implicit code/git convention → medium · (4) old, unreinforced cell → weak · (5) silence → weakest, overrides nothing. Mark the loser superseded: `"Supersedes [[B-NNN]] — old rule X; updated because Y."`

**Reference discipline (long-term potentiation).** Long source docs live in `references/`; their cell is *only a recall handle*.
```
1. Convert/copy source to references/<slug>.md
2. Small B-cell: tags [reference, @arch/@project:<x>]; hook = what it is + when to use;
   body = brain path + provenance + when-to-load
3. Append one INDEX line (@reference + domain tags)   4. Verify the file is readable
```
**Never paste a full document into a cell** — a cell holding a 50-page spec is a saturation bomb, not a handle.

**Per-machine boundary (only portable signal syncs).** The brain is shared across machines through one git repo, but **only durable, portable signal rides it** — cells, index, references, skills, and the transient handoff note. Everything below is per-machine local state that must **never** sync; raw sync would leak credentials/paths or break the peer machine:

```
sessions/       conversation state      telemetry/        analytics
projects/       session metadata        backups/          local backups
file-history/   edit history            shell-snapshots/  shell env captures
settings.json   creds/paths/hooks       ide/              IDE socket state
```

The first three are read *locally, read-only* and distilled into a host-tagged block in the handoff note — that distilled block is the only trace that travels. Enforce this in code, not discipline: the sync effectors carry a blacklist guard that aborts (or silently unstages) any of these paths staged for the shared repo. Adding a new per-machine dir means updating the guard, not just the prose.

**Anti-patterns (pathological configurations):**

| Anti-pattern | Pathology | Consequence |
|---|---|---|
| Knowledge in CLAUDE.md | S-layer bloat — router becomes store | Session cost rises; nothing selective |
| Auto-loading all cells | Saturation — all A-units active at once | Performance degrades; signal drowns |
| Facts in skills | R-layer stores what A should own | Skill diverges from cell; two truths |
| Long docs in cells | A-unit overload | Cell unscannable; reference discipline collapses |
| Speculative pre-built skills | Unused R-units, no reinforcement history | Misfire or never fire; cost, no benefit |
| No G5 write-back | Zero reinforcement | Brain never grows; re-derivation every session |
| Stale cells kept | Decayed weights retained | G1 activates wrong cells; wrong output |

---

## §9 · SKILLS — REFLEX VS. PROGRAM

Rosenblatt's Part IV covered **program-learning perceptrons** — systems that execute learned *sequences*, not just single classifications. The skill taxonomy is exactly this distinction.

- **Passive convention → cell only (reflex).** "Never commit secrets." "Use identity X." Always-on rules applied automatically whenever the relevant cells are active. No skill needed.
- **Task-shaped procedure → skill (program).** "Run the deploy." "Do the commit ritual." "Ingest a reference." A *sequence* of operations bundling several cells into one motion. Rosenblatt: *"a program-controlled perceptron could direct its attention successively to different parts of the field in a systematic order."* A skill directs the R-unit's attention across A-units in order.

**Design:** *thin* (steps + citations, no duplicated facts — when a cell changes the skill still works) · *specific activation* (the description is the A→skill coupling; name exactly when it fires and when it does not — vague descriptions over- or under-fire) · *procedural, not declarative* (skills say what to do; cells say what is true).

| Skill | Fires when | Cites |
|---|---|---|
| `safe-commit` | before any commit/PR/push | @git cells |
| `iac-guard` | editing IaC / before `apply` | @infra cells |
| `status-log` | after deploy/infra event | @infra, @workflow |
| `task-gates` | ambiguous/high-blast task | gate protocol |
| `reference-ingest` | adding a long doc | @reference discipline |
| `brain-consolidate` | >~30 cells or dup hooks | @meta cells |

---

## §10 · SELECTIVE ATTENTION

> *"In a complex field with more than one trained stimulus present, rather than giving a conflicting mixture of responses, the perceptron picks a single familiar object and responds to it to the exclusion of everything else — suppressing A-unit activity for the others."*

When a context holds multiple tasks — many open files, many implied goals — the agent picks the single task most strongly matched by the S-signal and works it exclusively, suppressing the rest. **This is why G2 SCOPE exists:** the fence is the attention mechanism. It names which stimulus the R-unit answers to and inhibits everything outside it. Without G2, the R-unit gets conflicting A-signals from several tasks and emits mixed, incoherent output.

```
G0 fire on primary task → G1 activate its domain (others suppressed by scope)
→ G2 fence ("touch src/auth only; not src/billing, src/ui")
→ G3 attend exclusively to src/auth → G4 observable tests only src/auth
```

---

## §11 · TOKEN ECONOMICS

The saturation theorem, in token terms.

| Mechanism | Basis | Effect |
|---|---|---|
| Protocol loaded once/session | fixed genetic structure | ~800 tok/session, not per task |
| Index grepped, not loaded | classify before activation | only ~3–10 matched lines enter |
| Only matched cells load | subthreshold units silent | 2–5 cells × ~300 = ~600–1500 tok |
| References deferred | potentiation on demand | ~5000-tok doc stays out unless needed |
| G1 cache hit | learned-weight reuse | saves ~2000–5000 tok per re-derived insight |
| Intent shorthand | S-unit pre-loading | saves 1–3 clarification turns |
| Saturation (anti-pattern) | all A-units active | ~30,000 tok/session; performance degrades |

**The economics compound.** A 50-cell brain loading 3 cells/task costs about what a 5-cell brain loading 3 costs. Expressiveness grows; per-task cost stays flat. Selective recall is the only regime in which the brain scales indefinitely without degrading.

---

## §12 · ONE-SCREEN SUMMARY

- **Protocol** (tiny, always loaded) — the genetic structure: RECALL before reasoning, WRITE-BACK after learning.
- **Index** — the interaction-matrix register: a cheap tagged map, grepped per task, never auto-loaded.
- **Cells** — A-units, one fact each, loaded only when tag-sum meets threshold. Tags = excitatory weights; scope = inhibitory; `Related:` = lateral couplings; `ref_path` = deferred potentiation.
- **References** — long-term store, reached through small handle cells, loaded only on document detail.
- **Skills** — R-unit motor programs: thin, cite cells, never duplicate facts.
- **Gate Loop** — Intent (transduce) → Recall (grep) → Scope (attention fence) → Build (emit) → Verify (error-correct) → Learn (write back).

Facts in cells. Documents in references. Procedures in skills. Nothing duplicated. Recall selective. Growth by reinforcement. Cheap because saturation is architecturally prevented. Stand it up once, globally, and every project inherits the same perceptron.

---

## APPENDIX A · NOTATION

*(after Rosenblatt, Appendix A, CAL VG-1196-G-8)*

| Symbol | Rosenblatt | Brain mapping |
|---|---|---|
| S-unit (sᵢ) | sensory transducer | user prompt + G0 INTENT |
| A-unit (aᵢ) | association element | B-NNN cell |
| R-unit (rᵢ) | response emitter | agent output / skill |
| λᵢ | activity state | cell loaded (1) or not (0) |
| θ | threshold | tag-match count to activate |
| cᵢⱼ / vᵢⱼ | connection / its value | @tag coupling / tag-match strength |
| Nₐ / N* | A-units / active A-units | cells in brain / cells loaded this task |
| δ / μ | reinforcement increment / decay | new cell at G5 / stale-cell aging |
| θ-servo | threshold servo | G2 SCOPE fence |
| α / γ system | unbounded / conservative reinforcement | write-back / consolidation-dedup |
| phase space | all memory states | all INDEX configurations |
| interaction matrix | all coupling coefficients | all cells + tags |
| convergence | finite-iteration guarantee | G1 finds the cell if it exists |
| saturation | degradation at capacity | full-brain load |
| back-coupling | R→A feedback | G5 write-back |
| selective attention | A-unit suppression | G2 fence + inhibitory tags |

---

## APPENDIX B · CHECKLIST

**Every non-trivial task:** G0 goal + why · G1 grep INDEX, load only matches · G2 touch/not-touch/branch/done-when · G3 build against fence · G4 verify the observable · G5 consider write-back.

**Before writing a cell:** durable? · reusable? · non-redundant? · not codebase-native?

**Periodic hygiene:** near-duplicate hooks → consolidate · cells naming stale files → mark volatile/delete · @project cells now used widely → promote.

**Never:** knowledge in CLAUDE.md · auto-load all cells · duplicate facts into skills · paste full docs into cells · pre-build speculative skills · skip G1.

---

*Rosenblatt, Frank. "Principles of Neurodynamics: Perceptrons and the Theory of Brain Mechanisms."*
*Cornell Aeronautical Laboratory, Report VG-1196-G-8. Buffalo, NY: 18 March 1961.*
*Transformation: Brain of Brains architecture, read through Rosenblatt's neurodynamic framework. Open reference architecture.*
