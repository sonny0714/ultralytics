# Agent Rules

How the agent works. Project-independent — a rule that cannot be stated without naming a specific
project, server, dataset, or experiment belongs in [`project.md`](project.md), and a rule about how
code or docs are *written* belongs in [`code_style.md`](code_style.md).

- 🔴 **HARD RULE — every command runs ① plan → ② verify & revise the plan → ③ execute → ④ review what landed.** No `Edit` / `Write` / launch / push before a written, verified plan; no completion claim before the post-execution review. Depth scales with blast radius, the phases never do. Full spec: [Plan → Verify → Execute → Review](#plan--verify--execute--review-hard-rule--every-command-4-phases) below.
- **Code & doc style** (naming, structure, comments, markdown, where a new `.md` goes, refactoring workflow, operational style): see [`code_style.md`](code_style.md). This file does not duplicate those rules.
- **Server access & user-specific rules** — `md_files/users/{username}/user.md`: which servers are active, their containers/GPUs, and (via `access.yaml`) ssh user/key/port/mount paths. **Read this before any ssh / `docker exec` / launch** — do not guess hostnames or container names.
- **`md_files/users/{username}/access.yaml` is the connection + secrets map** — one file per user, no username wrapper (the directory names the user). Read it directly for ssh (`servers.{srv}`: ssh_user / ip / port / ssh_key / source_mnt_path / alias_dir) and per-project secrets (`git_projects.{project}`: wandb_api_key / wandb_entity). Never ask the user for a key/host that is already in this file.
- **Currently launched experiments** — `md_files/users/{username}/run/_index.md`: the single Active table of long-running remote processes (what runs where, running vs ended). **Consult it before assigning, relaunching, or reporting on a run.** Tracking format, artifact collection on completion, and the `finished` completion marker are all in [`code_style.md`](code_style.md) §12.3 / §12.8; run_id readback goes through the `run_scan` skill.
- **Project-specific guide** — `md_files/project.md`: **read it once at session start**. It is deliberately NOT `@import`ed by the project-root `CLAUDE.md` (too large to re-inject every turn), so nothing loads it automatically.
- **Before creating ANY `.md`, pick its home** from the decision table in [`code_style.md`](code_style.md) §9.3. The trees are not interchangeable: `memory/` (permanent canon) · `plans/` (disposable plan **and** handoff) · `refactoring/` (code-migration topic) · `users/{user}/` (**`user.md` · `run/` only**).
- **Server performance / GPU tiers / launch load-balancing** (if the project tracks multiple servers): always consult `md_files/memory/server.md` before assigning a training/launch across servers (perf ranking, hardware tiers, load-balancing rules, restart gotchas). It is the canonical source; `user.md` → Active Servers holds only the dynamic state.
- **Paper figure style** (if the project produces paper figures): always consult `md_files/memory/eval_figure_guide.md` before creating or editing any paper figure — whole-figure golden ratio 1:1.618, minimal outer margins, unified axis/tick/legend font sizes. It is the canonical source for figure geometry.
- **When adding rules**: code / doc style → `code_style.md`; project-specific architecture or conventions → `project.md`; agent behavior and workflow → this file.
- **This file must be written in English only** — Korean content belongs in `project.md` or user-specific files.

# Behavior Summary

- Respond in Korean
- **Plan → Verify → Execute → Review on every command** (hard rule, section below) — the plan, its verification verdict, and the post-execution review are *visible in the response*, never internal-only
- **Report with a status label + layer + plan position** — every finding opens `[FIXED]` / `[OPEN]` / `[UNRECOVERABLE]` / `[NEEDS-CONFIRM]`, names whether it was data / code / config, and multi-phase work restates its phase list + `k/n` + next gate ([§Reporting Style](#reporting-style--status-label-first-no-ambiguous-nouns))
- Don't over-assume; explain code step-by-step by function if long
- When instructed "do without asking", execute automatically without permission prompts
- When user input contains `http://` before filenames (e.g. `http://xxx.py`, `http://learning.md`), strip the `http://` and treat as plain filename
- **No AI attribution in git** — a commit / PR / tag never carries a `Co-Authored-By` assistant trailer or a "Generated with …" line (hard rule, [§Git Attribution](#git-attribution--no-ai-co-author-line-hard-rule) below; overrides the harness default)
- **Execution over design Q&A** — default to writing code; minimize design-question round-trips. If the session feels slow, tell the user to start a fresh one.
- **User stepped away / AFK** — keep coding to completion without asking for opinions. Anything needing later review or discussion goes in the final response; if it must survive the session, write it as a plan under `md_files/plans/`.

## Answer Consistency for Decision Questions

For decision-type questions — multi-technique comparison, implementation-scope calls, baseline selection, architecture choices — do the research **before** the first answer, not after a follow-up forces a revision.

- Consult the codebase and relevant project docs before answering. Don't answer decision questions from general knowledge alone.
- When confidence is low, say so explicitly ("confidence low, checking X") instead of guessing.
- If a later answer contradicts an earlier one, **explicitly flag the correction** ("prior answer was wrong in respect X"). Don't silently swap conclusions — that erodes trust.
- Persist finalized decisions in `project.md` or a dedicated guide doc, so the same question recurring months later produces the same answer.

## Reporting Style — status label first, no ambiguous nouns

Every report of a finding, a blocker, or progress answers three questions **before** any narrative: **what state is it in, what exactly was wrong, and where that leaves the plan.** A sentence like "domain X was in an unregenerable state" fails all three — the reader cannot tell whether data was lost, whether the code was at fault, or whether it is fixed now.

### Outcome first, process second — never interleaved

Open with **what now works and what now does not, in deliverable terms** — the table, the figure, the run, the number the user actually wants. Only after that comes how it was found, what was tried, and which file was edited. A report that opens mid-investigation forces the reader to reconstruct the verdict from the narrative, which is the reader doing the agent's job.

- **Two lists, named separately**: what is possible now, and what is not. Never one blended paragraph.
- **A status label attaches to a deliverable, not to an intermediate.** A lost debug log, a missing console dump, a stale cache is not "data loss" — say what it costs at the deliverable level ("the table still regenerates; only its second-path cross-check is gone"). Grading an intermediate with deliverable vocabulary inflates a non-event into an alarm.
- If the outcome is "nothing was lost", **say that first**, before explaining what was investigated.

### Every finding opens with a status label

Emit one of these verbatim, before describing anything:

| Label | Means |
|-------|-------|
| `[FIXED]` | found and repaired this turn; evidence quoted |
| `[OPEN]` | found, not repaired — what blocks it is named |
| `[UNRECOVERABLE]` | cannot be repaired; the lost input and its consequence are named |
| `[NEEDS-CONFIRM]` | repaired under the agent's own call; the user must ratify it |

### A finding names its layer and its evidence

Never report "X was broken / missing / impossible" on its own. Name **which layer** — the reader's next action differs completely per layer:

| Layer | State it as |
|-------|-------------|
| **data** | which file or dir, on which server, lost vs never existed |
| **code** | which `file:line`, which condition (e.g. `required=True`) |
| **config** | which key, which value, in which yaml |

Then one line each for **how it surfaced** (the command and its actual output) and **what changed** (the edit, or "nothing").

- **Never leave "did it stop or did it fall back?" implicit.** If execution refused to start, say so. If a fallback swallowed the problem, say what the fallback substituted. Silence reads as the wrong one of the two.
- **Repaired ≠ whole.** When a fix restores one property but not another, name survivor and casualty separately — "regeneration restored; cross-checking permanently impossible", not "handled".

### Multi-phase work reports its position every time

Any task with a written plan restates, in **every** response: the full phase list **with a few-word gloss on each phase**, the current phase, its progress as `k/n` against a named denominator, and the next gate.

- **A bare identifier is not a phase list.** `A ✅ B ✅ C ✅ D ◀ 2/8 E F G` says nothing — every entry carries what it *does* (`C logging granularity ✅ · D runs_map for 8 domains ◀ 2/8 · E census`). The reader must never have to open the plan file to learn what the current letter means.
- A fraction with no list does not locate the work; a list with no fraction does not show the distance left.
- When the phase order changes, show the **new** order and say which phase moved and why — a silently reordered list reads as the original plan.

Prose paraphrase of progress ("made good progress on the wiring") is not a report. Progress is a fraction against a denominator, or it is not progress.

## Judgment-Bound Work — diagnosis, options, trade-offs *first*

For prose (a sentence, an abstract, a section) and other judgment-bound work, the bottleneck is not *editing the file* but **judging which structure is logically right**. Returning a single polished new version each turn forces the user to find the weakness themselves every round (one version → reads weak → "fix it" → still weak → …); one abstract actually took ~6 hours that way.

**When revising prose, ship the new version together with:**

| Element | Content |
|---------|---------|
| **① Diagnosis** | *why* the current version is weak, **structurally**: given–new break, duplication, level jump, dangling reference, tautology, echo ("one ___ per X" repetition). Name the defect — not "this reads better" |
| **② Options** | 2–3 concrete candidates (A/B/C) as *actual sentence text* |
| **③ Trade-offs** | what each option gains/loses (word count, clarity, emphasis, duplication, echo) |
| **④ Recommendation** | which option, and why — decisively |

Why: the real work of prose revision is *judgment*, not editing. If the user must discover the weakness each turn, the loop is slow; with diagnosis+options+trade-offs, the user only *judges* and the loop converges fast.

Scope (vs **Execution over design Q&A** above): this applies to **judgment-bound work** (abstract/intro/sentence structure — the bottleneck is which structure is right). **Execution-bound work** (code edits — what to do is obvious, execution is the bottleneck) stays execution-first. Classify the task first, then pick the mode.

Don't: silently commit a superficially smoother single version and stop (= the cause of the 6-hour loop). Exception: trivially obvious fixes — apply directly.

**Rule:** judgment-bound revision is not "fix and show" but "lay out the why, the options, and the trade-offs, and ask for judgment" — the moment you return only one version, the user is demoted to a weakness detector.

## Workflow Hints (tool-use)

These apply when *operating* on code, not when *writing* it — the writing rules are in [`code_style.md`](code_style.md).

| Rule | Why |
|------|-----|
| **Blast-radius before edit** | Before modifying a symbol, read its full body and **output the blast radius**: grep every call site, the objects/config it is injected into, and sibling locations of the same pattern. Each item is either in the change or marked out-of-scope (with a reason). Never patch a literal you only saw in one place — the same pattern likely exists elsewhere. |
| **Verify after refactor** | After extracting helpers, renaming, or changing units/conversions, regenerate any downstream artifacts and compare to the previous output before declaring done. |
| **Never reuse a display-truncated value** | When you slice a value for a debug print (`str(v)[:60]`, `key[:8]+'…'`, `head -c`), that sliced string is a THROWAWAY for the terminal — never copy it back as the real key / secret / token / path / id. Read the full value from source when you use it; truncate only a separate display copy. A truncated API key / hash / mount path silently fails auth (401) or mis-resolves while the log still *looks* fine — and the reflex is to blame the source ("the key is invalid") instead of your own slice. Verify a suspect secret at its own edge (a raw `curl` to the auth endpoint) before declaring it bad. |

## Tool Output Frugality

Every byte a tool prints lands in the conversation context and is re-sent on every subsequent turn. Probes that look "cheap" (`ls -la`, `find`, `cat large_file`) compound fast. Default to the **minimum data needed to answer the question**.

| Bad | Why it bloats | Better |
|-----|---------------|--------|
| `ls -la some/dir/` when you only need "does X exist?" | Dumps every entry with size/date/perm metadata | `[ -e some/dir/X ] && echo yes \|\| echo no` |
| `ls -la projects/` to count files | One long line per file × N files | `find projects/ -maxdepth 1 -name '*.jsonl' \| wc -l` |
| `cat large_file.log` to find one line | Whole file enters context | `grep -n PATTERN large_file.log` (or `Read` with `offset`/`limit`) |
| `find /` / `ls -R` exploratory dumps | Output grows with directory size, unbounded | Targeted `Glob` / `Grep` with a specific pattern |
| Re-running a command "just to check" after a successful edit | Duplicates output already in context | Trust the tool result; only re-run when state actually changed |

Rule of thumb: before running any listing/dump command, ask "what single fact do I need?" and pick a command that prints just that fact. The same applies to `Read` — pass `offset`/`limit` for large files instead of reading the whole file.

## Plan → Verify → Execute → Review (HARD RULE — every command, 4 phases)

**Every user command runs four phases, in this order, with no exception**: ① build a plan → ② verify and revise that plan → ③ execute the verified plan → ④ review what actually landed. This is a **hard rule**. It is not waived for a "small" task, for a one-line follow-up, or for a task whose answer looks obvious. Only the **depth** of each phase scales (see *Depth scaling* below) — a phase is never skipped, and two phases are never collapsed into one message-breath.

Two failures this exists to kill, one at each end. At the front: read a little, start editing, patch whatever breaks — edits resting on an unverified assumption. At the back: finish the last edit and call it done — a change that was never read back, so the **user** finds the defect instead of the agent. The plan is verified before it runs (②) and the result is verified after it runs (④); neither substitutes for the other.

| Phase | What it produces | Gate to the next phase |
|-------|------------------|------------------------|
| **① PLAN** | a written plan in the response — goal, cited reads, steps, checks | the plan exists as text, and nothing has been mutated yet |
| **② VERIFY & REVISE** | a per-row verdict on the V-checklist + the revised plan | every row PASS — revise until they are |
| **③ EXECUTE** | the change itself, step by step | every planned step attempted, or the deviation routed back to ② |
| **④ REVIEW** | a per-row verdict on the R-checklist + the plan-parity report | Definition of Done (code) / Completion Checklist (all) |

For a code task the four phases expand to the fixed order **deep read (cited) → flow-chart the algorithm & structure (from the real code) → verify the plan → confirm → implement → flow-chart parity check → run tests → read back the result**; the flow-chart and unit steps are detailed in **Flow-Chart & Unit Gates** below.

### Phase ① — Plan

- **No state-mutating tool call may precede the plan.** Read-only probes (`Read`, `Grep`, `Glob`, `git log`, `test -e`) are *how the plan is built* and are always allowed. `Edit` / `Write` / a launch / `git commit|push` / a delete / a remote `docker` op are **not** — those wait for phase ③.
- A code / ops plan must satisfy **Definition of Ready** below (cited code read, contract trace, test plan, steps, flow-chart). That list is the plan's content spec — do not restate it here.
- A non-code plan (a question, a doc edit, an analysis, a readout) still names one line each: **what will be read**, **what will change or be claimed**, **how it will be checked**.
- Multi-step work also gets a `TodoWrite` list; work that must survive the session gets `md_files/plans/{slug}.md` ([`code_style.md`](code_style.md) §9.3).
- For judgment-bound work (prose, design calls) the plan artifact **is** the diagnosis / options / trade-offs table of *Judgment-Bound Work* above.

### Phase ② — Verify & revise the plan (the step that is most often skipped)

Verification is a **separate adversarial pass over the plan just written** — assume it is wrong and try to break it. Answer every row explicitly; a row that cannot be answered is a **FAIL**, never a pass.

| # | Check | The mistake it catches |
|----|-------|------------------------|
| **V1** | Every sub-item of the request maps to ≥1 plan step — enumerate them | a silently dropped sub-item |
| **V2** | Re-open each cited `file:line`; symbol / signature / config key is exactly as quoted | a key or signature quoted from memory that does not exist |
| **V3** | Every assumption is listed and marked verified / unverified; an unverified one is verified now or the plan branches on it | "I assumed X" that only fails at runtime |
| **V4** | The blast radius is closed — each call site / sibling is in the plan or out-of-scope with a reason | a half-applied rename |
| **V5** | Every step names how its result will be checked | a step called done with no evidence |
| **V6** | Every irreversible act (delete, overwrite, kill, launch, push) is named, and its precondition probe is a step | destroyed data, a wrong launch |
| **V7** | The plan contradicts no project canon (auto-recall memory, `project.md`, `code_style.md`, an active `refactoring.md`) — name what was consulted | re-opening a settled decision |

- Emit **both** outputs: the per-row verdict **and** the revised plan (or `revision none — reason`). A pass that finds nothing and never says why is not a verification pass.
- Then **confirm with the user** before executing. Scale to blast radius: a trivial reversible edit needs no round-trip. When the user is **AFK** or said **"do without asking"**, still print the plan and the verdict, then proceed without waiting — **only the wait is waived, never the phase** (this is how the rule stays consistent with *Execution over design Q&A* / *AFK* above).
- A launch adds the **Experiment Launch Config Gate** rows to this checklist.

### Phase ③ — Execute

- Execute **the verified plan**, step by step — not a remembered version of it, and not an improved one invented mid-edit.
- **A surprise returns you to phase ②, not to an inline patch.** A failing command, a missing file, a differing signature, an impossible step: stop, state what the plan got wrong, re-verify, present the revision, then resume. Patching in place *is* the implement-then-fix-on-failure loop this rule forbids.
- After **two** re-entries on the same task, stop and escalate to the user with what the plan keeps getting wrong — do not silently re-plan a third time.
- Executing the last step is **not** finishing. Phase ④ is mandatory before any completion claim.

### Phase ④ — Review the execution (did it land, and only it?)

Review runs on **fresh evidence** — re-read the file, re-run the command, re-grep the symbol. "I remember writing it correctly" is not evidence, and an edit tool reporting success only proves a string was replaced, not that the right string in the right place now does the right thing. Answer every row explicitly; an unanswerable row is a **FAIL**.

| # | Check | The mistake it catches |
|----|-------|------------------------|
| **R1** | Re-read the user's **original message verbatim**; every sub-item is shipped or reported unmet | drift away from the request over a long turn |
| **R2** | Plan parity — each planned step marked `done` / `changed (reason)` / `dropped (reason)` | a step quietly abandoned mid-run |
| **R3** | Read back what **actually landed** (`git diff`, re-read the edited region) — not what was intended | an edit applied at the wrong site, or half-applied |
| **R4** | No collateral change — nothing outside the plan's files/lines moved; no debug leftovers, no stray rename | an unintended edit riding along unnoticed |
| **R5** | Re-grep every changed symbol; each call site / sibling / doc reference is consistent **now** | a blast radius closed on paper but not in the tree |
| **R6** | Behavior evidence from a real run — quote the output line and the exit code | "should work" shipped as done |
| **R7** | Coupled artifacts regenerated and compared (templates, indices, generated outputs, downstream numbers) | the source updated, its generated copy left stale |

- Anything R1–R7 flags is **fixed in this same turn and then re-reviewed** — or stated explicitly as unmet / **unverified** in the response. Noticing a defect and not reporting it is the worst possible outcome of this phase.
- A fix made during phase ④ re-enters phase ④; it does not get to skip its own review.
- For code, phase ④ is closed by **Definition of Done** (all 5 lines) plus the **Completion Checklist**.

### Depth scaling (compress a phase — never skip it)

- **Read-only question** — ① what will be read + what will be claimed (1–2 lines) · ② V1 + V2 on the cited sources · ③ answer · ④ R1 + R3 (re-read the source before asserting it)
- **Small reversible edit** — ① steps + cited reads · ② full V-checklist, no confirmation wait · ③ edit · ④ R1–R5 + one parity line
- **Code change / launch / irreversible op** — ① Definition of Ready in full · ② full V-checklist + user confirmation · ③ implement · ④ R1–R7 + Definition of Done in full

### Violations (any one of these means the rule was broken)

- An `Edit` / `Write` / launch / push whose plan was never written down.
- A plan and its execution in one breath, with no verification verdict between them.
- "Too small to need a plan" — depth scales, the phase does not.
- Repairing a mid-execution failure inline instead of returning to phase ②.
- A completion claim with no phase ④, or a phase ④ that restates intent instead of re-reading what landed.
- A verification / review section whose entire content is "it looks fine".

## Flow-Chart & Unit Gates

These four steps make implementation smooth — apply them on every implement / develop / modify task, scaled to blast radius.

1. **Flow-chart from the real code (algorithm + structure)** — before changing anything, read the actual implementation on disk and draw two flows *from the code*, not from memory or the prompt's description:
	- **Algorithm flow** — the ordered operations / data transforms the code performs (inputs → each step → output).
	- **Structure flow** — the call / composition graph (caller → callee → injected objects); the same seams as the Blast-radius list.
	- Cite `file:line` at each node. A flow drawn from a remembered description instead of the code as written is not acceptable.
2. **Confirm before implementing** (= the phase ② confirmation, not a second one) — present the two flows as the plan and get the user's confirmation before writing code. This is the Ready gate made explicit; it is **not** open-ended design Q&A (still minimize round-trips — show one concrete flow to confirm, not a menu of options to debate). Scale to blast radius: a trivial, reversible edit needs no round-trip. When the user is **AFK** or has said **"do without asking"**, still *produce* the flow but do not block on a reply — state it in the response (and, if it must survive the session, in `md_files/plans/{slug}.md`) and proceed (honors **Execution over design Q&A** / **AFK** above; the analysis phase still happens, only the wait for confirmation is waived).
3. **Flow-chart parity check after implementing** — re-derive both flows *from the implemented code* and diff them node-by-node against the confirmed plan flow. Every divergence is reconciled or called out explicitly. "Looks done" is not parity — the post-implementation flow must match the pre-implementation flow you confirmed.
4. **Unit check at every seam — especially environments** — when the change touches a unit-bearing quantity, fix the expected unit at its **source** and confirm **each consumer reads it in that unit**. This bites hardest at **environment boundaries**: a quantity stamped in one unit (e.g. simulated time in ns or μs) but consumed for processing — deadline, expiry, rate, frame timing — in another (e.g. ms). A shape / `out.shape` smoke type-checks and passes while the number is off by 1000×, so the env "runs" but its reward / timing is silently wrong. Verify the unit at each boundary the value crosses, not at each line. Which layer uses which unit is **project-specific** — see [`project.md`](project.md).

## Experiment Launch Config Gate

Before launching ANY eval / experiment batch, verify **every launch coordinate against its canonical source**, not from memory and not from a sibling batch's script. The recurring failure mode is silently copying a prior batch's config and changing one axis wrong — a run that "works" (rc=0) but measures the wrong thing. Each row below is verified with a `file:line` or run-index citation **before** the smoke, and named in the plan:

| Coordinate | What to verify (against canon, cited) | Silent-failure if wrong |
|------------|----------------------------------------|-------------------------|
| **env / app token** | the EXACT variant token — evaluator, scene, backend. A one-token diff can swap the metric entirely | the run passes and the number is a different quantity |
| **seed × cell set** | the exact seed→cell mapping from the canonical source, NOT assumed `0..N`. If the seed reshuffles the split, the SAME index is a DIFFERENT cell per seed. Confirm the largest index is in range | wrong cells scored; an out-of-range index fails one seed only |
| **data / trace style** | the exact style(s) the target table uses, not the full grid | extra/missing styles shift the pooled mean |
| **algorithmic config params** | every algorithmic knob from the canonical **yaml** — NEVER override. Override only speed knobs | off-metric while shape checks pass (the §Unit gate at the env boundary) |
| **model id per (method, seed)** | resolved per seed from the canonical run-id map, and **present on each launch server** (verify `test -d`) | a missing or stale weight silently loaded |
| **readout isolation** | the aggregator reads ONLY this batch's cells. Container clocks can run behind the host, so run-dir NAMES are unreliable — isolate by config `mtime` or an explicit since-filter, and confirm pre-existing same-evaluator runs don't contaminate | old runs pooled into the new number |
| **launch container** | on EVERY target server: each of your existing containers is proven live-or-idle by CPU + output advance, the idle/zombie ones are removed, and this launch gets a **new dedicated** container (§Dedicated Launch Container) | a co-tenant's run is killed by a reuse/restart, or the batch silently shares a GPU with an invisible tenant |

Then **smoke ONE cell per method-class** (rc=0 + correct output schema + index range valid) in a GPU-verified container, and only then do the single parallel launch. This gate is the launch-time counterpart of the code-time **Definition of Ready** below. The project's concrete coordinate values live in [`project.md`](project.md).

## Experiment Artifact Pipeline

Getting a number from a run store into a paper float is **never one script**. Each stage hands off to the next **through a file**, and each stage has its own `.py`. A single script that walks from run store to a finished table is the failure this section prevents: nothing in the middle is inspectable, so a wrong number surfaces only in the PDF.

| Stage | Input → Output | Owns |
|-------|----------------|------|
| 1 | run store (`run_id`) → `cells.csv` **+ `runs_map.csv`** | the ONLY stage that touches run ids |
| 2 | `cells.csv` → the experiment record's tables | version-invariant |
| 3 | `cells.csv` → one submission's floats | **one `.py` per version** |

- **Stage 1 must emit `runs_map.csv`** (the cell coordinates plus `run_id` and its path). A domain whose stage-1 script does not record which runs it consumed has no provenance: its numbers cannot be re-derived and its runs cannot be protected from a cleanup.
- **Stages 2 and 3 never see a `run_id`.** They read the CSV. A table generator that globs a run store means the stages have collapsed back into one.
- **Figures take the same path as tables, and are the higher risk.** A table's cells can be diffed after the fact; a figure's numbers live inside the image. A figure that does not go through stage 1's CSV has no way to be checked at all.

### Getting the number into the `.tex` — the paste is the failure

A stage that PRINTS its table for a human to paste is **not wired**, however correct its
numbers are: the CSV can change and the float will not. "Regenerated it, looks fine" then
means the generator ran, not that the paper moved.

| Rule | Detail |
|------|--------|
| **Stage 2/3 writes the file** | The generator's output goes to the `.tex` on disk — never to stdout for a human to copy. A `--out`/splice path is part of the wiring, not a convenience |
| **Every generated float carries a provenance header** | First lines name the generator and say do-not-hand-edit. A float with no such header is unwired until proven otherwise |
| **A hand-maintained float is a declared exception** | It says so in its own header AND is listed where the domains are listed. An undeclared hand-maintained table is indistinguishable from a broken generator |
| **Splice, don't rewrite** | When a generator owns only some rows of a shared float, it replaces a marked band and leaves the rest — and it offers a `--check` mode that reports what WOULD change without writing |
| **Search before writing a generator** | The emitter often already exists and only lacks a destination (a `print()` of the full table body). Wiring that is minutes; a second generator beside it is a permanent divergence |

### Stage 3 targets exactly one version, and never a frozen one

| Rule | Detail |
|------|--------|
| **One `.py`/invocation per version** | Stage 3 is version-scoped by definition. The version comes from an explicit variable with **no default** — a fallback silently writes the previous venue's tree |
| **A submitted version is frozen** | Regenerating it is a content change to a document already sent. Later numbers land in the NEXT version's directory; the frozen one is only ever read |
| **Verify by diffing the two versions** | The new version starts as a copy of the frozen one, so the diff after regeneration must equal the intended change exactly. Anything else in that diff is an unplanned edit |
| **Scope is every float, not the convenient ones** | Tables a driver happens to splice and tables it does not are the same obligation. A float left out is a float that silently keeps last version's number |

### Storage is 1:1, consumption is N:M

- **A run has exactly one home**, keyed by its **launch topic** — what was run, not what it feeds.
- **`topic` is never a table name.** The same run legitimately feeds several tables and figures; naming its directory after one of them is wrong the moment a second one uses it.
- **Never partition run storage by consumer.** "Every run behind table X" is answered by that domain's `runs_map`, not by `ls` — with genuine N:M no directory tree can provide that view, and forcing one causes either duplication or mislabeling.

### Cleaning up a run store

| Rule | Detail |
|------|--------|
| **Whitelist = union of every `runs_map`** | A run absent from every map is unused |
| **Gate before any move/delete** | EVERY table and figure must ALREADY have a stage-1 script emitting `runs_map`. One missing map = a needed run is invisible and gets deleted |
| **Census spans every server first** | A needed artifact may exist only on the box that produced it. Collect, then prune |
| **Move, verify, then delete** | Move survivors; regenerate every affected table and figure to byte-identical output; only then delete the remainder |
| **Preserve `mtime` on the move** | Readout isolation keys on config `mtime` (container clocks run behind, so run-dir names are unreliable) — a move that resets it breaks that gate silently |
| **"Missing" is a separate axis** | `runs_map` answers what exists and is used. Missing = the cells a table renders as placeholders. Report both |

### Verification is planned per stage, before execution

A multi-stage restructure states, for **each** stage and **before starting it**: what gets regenerated, what it is diffed against, and which result gates the next stage. "Move now, check at the end" is not a plan — by then the evidence needed to tell a bug from a legitimate data change is gone. Byte-identical regeneration of the affected floats is the default gate; when a value legitimately changes, the change is stated and re-approved rather than absorbed silently.

## Evaluation Metric Discipline

How evaluation / comparison results are *taken* — applies to every results table, plot, or benchmark. The goal is one apples-to-apples axis; a table mixing definitions or units is worse than no table.

| Rule | Detail |
|------|--------|
| **One canonical metric set** | Take metrics through the project's single canonical metric definition (the one row-builder / metric spec). Never hand-roll per-table columns. |
| **No ad-hoc columns** | Render exactly the canonical set — do not add diagnostics outside the spec, and do not silently drop a canonical metric. If a metric is not in the spec, it does not go in the comparison table. |
| **Same evaluator for every row** | Route every method/variant through the *identical* eval path so the metric set, its definition, its units, and the pipeline match across all rows. Never mix an internal objective for one row with an environment-realized number for another — that drift makes the table meaningless. |
| **State the unit & normalization** | Report at the agreed unit and say it explicitly. Extensive quantities (totals) are divided to the agreed per-entity basis; intensive ones (%, rates) are already per-entity — do not divide them again. |
| **Keep metric families whole** | When a metric family has several variants, keep all canonical variants and be ready to say what each measures. They look redundant only in the trivial case; they diverge exactly where the result is interesting. |

- The concrete metric set, the canonical row-builder, and the evaluator entry point are **project-specific** — see [`project.md`](project.md).

## Definition of Ready

This is the **content spec for phase ①** on a code task; phase ② then verifies what it produced. Before writing or changing any code, produce a plan that **proves you read the code at depth**. "I'll check X while coding" is not allowed — read it now and cite it. No implementation starts until the plan contains, with concrete citations (not intentions):

1. **Code read (cited)** — for every function you will call/modify and every config key you will set: quote its definition at `file:line` (exact signature, the exact schema key, return type). If you have not read it, you cannot plan around it. *This is what catches "set a config key not in the parser" or "call a signature that changed" at plan time, not at runtime.*
2. **Contract trace** — the path the change flows through (caller → callee → injected objects), each seam named = the Blast-radius list.
3. **Test plan** — the concrete test cases by level decided **before** implementing: which unit smokes, and the ONE yaml-config e2e (real config loader → entry point) that is the completion gate ([`code_style.md`](code_style.md) §8.2). Name the test file, the method/env, and the assertion. **Test files live under `{project_root}/execution/test_codes/`**, and the plan's Steps include an explicit "write the planned tests there → run them" verification step — a plan that produces no runnable test file under `execution/test_codes/` is not Ready.
4. **Steps** — implementation broken into verifiable steps.
5. **Flow-chart confirmed** — the **algorithm flow** (item 2's Contract trace already supplies the structure flow), drawn from the real code, and the combined flow **confirmed by the user** before implementation begins (per **Flow-Chart & Unit Gates** above; AFK / "do without asking" → record and proceed). If units are in scope, the expected unit at each seam is named here too.

When a plan is built from delegated/subagent findings, the citations must still be present — accepting a "0 issues / all KEEP" verdict without the `file:line` evidence behind it is not a Ready plan.

## Definition of Done

This is the **closing gate of phase ④** on a code task. Do **not** say "done / implemented" unless the **Done-report** below (all 5 lines) is filled. If it cannot be filled, state **"unverified"** explicitly — never substitute a weaker check and call it done.

1. **e2e (yaml-config path)** — ran the change through the **real entry point** (config loader → runner), driven by the **actual yaml config**, at the **minimal agent-scale e2e** (full-scale e2e is the **user's**, never the agent's — see Hard rules). Capture the **full output** (no `grep`/`head` filtering — that is how real errors get missed) and quote the exit code.
2. **behavior evidence** — quote the line(s) in that output where the asserted behavior actually appears (a number, a log row), not "tests pass".
3. **integration** — paste the call-site grep for every changed symbol and whether each was updated (the Blast-radius list above, now closed).
4. **flow parity** — re-derived the algorithm + structure flow from the **implemented** code and diffed it node-by-node against the confirmed plan flow; quote each divergence and its reconciliation, or state "identical" (Flow-Chart & Unit Gates step 3).
5. **unit check (if units apply)** — for every unit-bearing value the change touches, state the unit at its source and confirm each consumer reads it in that unit. If no units are in scope, say "no units".

Hard rules:
- `py_compile` / import / `out.shape ==` smoke is **not** e2e. "Tests green" alone is never a completion claim.
- **Full-scale e2e / training runs are ALWAYS the user's**, never the agent's (experiments run on GPU servers — [`code_style.md`](code_style.md) §12.5). The agent's completion gate runs **only the minimal-scale e2e**.
- The minimal e2e overrides scale knobs so it still exercises **≥ 2 iterations × ≥ 2 updates per iteration** — `1×1` misses iteration-boundary and update-accumulation bugs, while full-scale wastes the user's GPU. The exact override **keys/values are project-specific** → [`project.md`](project.md).
- A hand-built config object is a unit smoke, **not** the gate — only a run through the real config loader counts. How that test is built: [`code_style.md`](code_style.md) §8.2.
- e2e runs **in the container** (real deps), per [`code_style.md`](code_style.md) §12.5. If not yet cleared to run it, report **unverified** and ask — do not claim done.

## Completion Checklist

After completing any task (code modification, question response, doc change), verify:

1. **Phase ④ review ran** — R1–R7 answered on fresh evidence (re-read / re-run / re-grep), each planned step reported `done` / `changed (reason)` / `dropped (reason)`, and any mid-run deviation went back through phase ②
2. All items in the request are addressed — do not skip sub-items
3. Modified code files are consistent with each other
4. If a function signature changed, all call sites are updated
5. For code changes, the **Definition of Done** gate above is met (yaml-config e2e ≥ 1 update with behavior evidence) or **unverified** is stated explicitly — never "tests pass" alone. Non-code tasks: run any available checks
6. **Flow parity + units** — implemented-code flow matches the confirmed plan flow, and every unit-bearing seam was checked (Flow-Chart & Unit Gates steps 3–4)
7. Alias/setup files are synced if `md_files/users/` or command files were modified

### Category checks (apply the ones that fit the task)

- **Code change** — comments English only; no docstrings / multi-line comment blocks; naming and function-length rules of [`code_style.md`](code_style.md) §1 / §11 hold
- **Refactoring topic** — the topic's `refactoring.md` was read at session start; phase progress note updated; diffed against `backup/` before moving to the next phase
- **Backend / subprocess test** — container restarted before the test so no external subprocess survives as zombie/orphan; none left behind after it
- **Cross-server work** — went through a project skill, not a raw `ssh ... 'git pull'` / `ssh ... 'docker exec'` chain
- **Path mentioned in code or docs** — no hardcoded host/container mount path; referred to by `access.yaml` field name instead ([`code_style.md`](code_style.md) §12.6)

Anything the checklist flags is fixed on the spot, or stated explicitly as unmet in the final response.

## Alias Dependency

Cross-server git / docker / rsync / run-scan workflows are driven by scripts from the alias project.

- alias path: `md_files/users/{username}/access.yaml` → `servers.{server}.alias_dir`
	- Server detection: `hostname -I` for the IP, `$SSH_CONNECTION` for the port → matched against the ip/port entries in `access.yaml`
- **These workflows are unavailable without the alias project.**
- New environment setup: `git clone` then `./init.sh -t all` → requires an edited `configuration.yaml` → `configuration.sh` is generated
- Bootstrap a bare machine (no `configuration.yaml`): `./init.sh -b` → bashrc aliases + base container on the current server only
- `configuration.sh` is gitignored (it holds personal server info)

## Project Skills (use these instead of raw git/docker/rsync)

Project-level Skills wrap the repetitive cross-server workflows. **Prefer the Skill over a raw `ssh ... 'git pull'` or `ssh ... 'docker start'`** — the skill handles server detection, ownership, agent forwarding, and the known-host warts the raw command will trip on.

| Skill | Use case | Invoke it when … |
|-------|----------|------------------|
| **`git`** | pull / clone -f / push_all on the parent project or one submodule (`-s`), across servers | any cross-server git sync ("pull on server X", "apply this commit elsewhere too") |
| **`docker`** | container lifecycle — start / stop / image pull, local or remote | any remote docker control ("restart the container on X", "pull the image on Y") |
| **`artifact_sync`** | rsync experiment artifacts between servers (collect runs / distribute models) | any training-artifact sync between servers |
| **`run_scan`** | find run dirs across servers — run_id readback, completion status, wait-until-done | any run_id collection or "did it finish?" question |
| **`submodule`** | backup/restore + convert between tracked files and a git submodule | splitting a subtree out, or folding one back in |
| **`monitor`** | multi-server CPU/GPU dashboard in one tmux session | checking load before assigning a launch |

Hand-rolled SSH chains hit known-host prompts, agent-forwarding gaps, and a missing `chown` after docker writes. Inside the project repo the Skill names carry no plugin prefix — confirm the available Skills in the session's system-reminder list before invoking.

### These rule files are generated — sync rule

`md_files/agent.md` and `md_files/code_style.md` are generated from the templates in
`{alias_dir}/core/setup/templates/agent/` by **`update_md.sh`** (`./update_md.sh -p <project>`),
which overwrites both files unconditionally. The other generated files (`CLAUDE.md`, `user.md`,
`run/_index.md`, hooks) come from `setup.sh`.

When editing either of these two files, do BOTH (without being asked):

1. Edit the project file for immediate effect.
2. Edit the matching template in `core/setup/templates/agent/` so other projects inherit the change.
3. `diff` the two to confirm they are identical — a template that differs from the project file
	 means the next `update_md.sh` run will silently revert the edit.

## Dedicated Launch Container (every launch, EVERY server)

An experiment is **never** launched into whatever container happens to be up. This holds on **every** server the launch touches — no server is exempt, including the one you are sitting on. Every launch first runs these three steps on **each** target server, each named in the launch plan with its evidence:

| Step | Action | Evidence that closes it |
|------|--------|-------------------------|
| **① probe** | prove whether each of **your** existing containers holds a live run | CPU% **and** output advance (below) — `pgrep` alone never closes this |
| **② remove** | no live run (idle / zombie / stopped) → `docker rm -f <container>` | `docker ps -a` no longer lists the name |
| **③ create** | build a **new** container dedicated to this launch and run there | create cmd carries `--shm-size` + `--memory-swappiness=0` ([`code_style.md`](code_style.md) §12.1); the name lands in `run/_index.md` |

**Scope** — probe and remove only containers **owned by this user** (the `_{PROJECT_USER}` suffix). Another user's container on a shared box is never touched, idle or not.

**① A PID is not a run.** `pgrep -af` proves a process object exists, nothing more. A zombie (`STAT Z`), a stopped proc (`T`), a proc blocked on a dead GPU (NVML error), and a hung dataloader all keep the PID alive while nothing advances. Sample **twice, ≥ 30 s apart**, and require CPU **and** output to move:

| Signal | Probe | Reads as "no run" |
|--------|-------|-------------------|
| CPU | `docker stats --no-stream <container>` | `CPUPerc` ≈ 0 % in both samples |
| proc state | `docker exec <c> ps -eo pid,stat,pcpu,etime,args` | `Z` / `T` state, or every proc ≈ 0 % CPU |
| GPU tenant | host `nvidia-smi --query-compute-apps=pid,used_memory --format=csv` | the container's PIDs absent (GPU run only) |
| output | log growth (`wc -l` on the §12.2 path) or newest run-dir mtime | unchanged across the two samples |

- **`nvidia-smi` never decides this on its own — always read CPU with it.** A hung or crashed-but-unreaped proc keeps its GPU memory allocated, so the container still appears in `--query-compute-apps` (and the GPU still looks "used") while nothing advances. GPU memory present + CPU flat = **not a run**.
- CPU flat **or** output not advancing = **nothing to protect** → step ②. A **zombie / stopped process is an explicit remove case**, not a reason to hesitate.
- Both advancing = a real tenant: **never** `docker rm` / `docker restart` / broad `pkill` it — that kills the run. Leave it and still give the new launch its own container (step ③).
- State the verdict per container (`live` / `idle` / `zombie`) in the launch plan. "Looks idle" is not a verdict; two timestamped samples are.

**③ Why new, never reuse.** A reused container carries the previous tenant's env, `/dev/shm` leftovers, and possibly a stale GPU cgroup, and it mixes two owners' processes — exactly what makes the *next* launch's step-① probe unreadable. **One launch = one container = one owner.**

- One GPU, standard name: `{alias_dir}/core/docker/docker.sh -i <image> -t <server> -g <gpu_id>` creates just `{image}_{gpu_id}_{PROJECT_USER}` (`-g` also skips the `_test` container). **Without `-g` it recreates that server's whole per-GPU set** from `gpu_available_list` — not what one launch wants.
- Beside a protected tenant on the same GPU: add `-n <tag>` → `{image}_{gpu_id}{tag}_{PROJECT_USER}` (`-g 0 -n b` → `{image}_0b_{PROJECT_USER}`).
- Remote server → the **`docker` skill**, never a raw `ssh ... docker` chain.
- Record the actual container in the launch block's `Where` column ([`code_style.md`](code_style.md) §12.3), so the next launch's step ① knows what it is inspecting.

### GPU-Broken Container With a Live Run

A container can lose GPU access while an experiment is still running inside it: inside-container `nvidia-smi` shows **`Failed to initialize NVML: Unknown Error`** even though the host `nvidia-smi` is fine (a driver/cgroup refresh cut the container's GPU device cgroup). `docker restart <container>` re-acquires the GPU — **but restart kills every process in that container**, so it is only safe when no run you care about is inside.

| Container state | Action |
|-----------------|--------|
| No live/valuable run inside | `docker restart <container>` to recover the GPU, then relaunch as needed. |
| An experiment is still running inside | **Do NOT restart** — it would kill the run. Leave the broken container alone and spin up a **new** container on a working GPU, then launch there. |

Steps for the "live run" case:

1. **Verify the run is actually progressing** — run the step-① probe above (CPU% **and** output advance, two samples ≥ 30 s apart); `pgrep` alone does not decide it. A container that lost GPU access may already be dead or looping errors; if so it drops to the first row and restart is fine.
2. **Do not restart the broken container** — a still-advancing run is not worth losing; keep it until it finishes or is confirmed dead.
3. **Create a fresh container on a working GPU** — pick a healthy slot (`memory/server.md` GPU tiers / load-balancing), then create through `docker.sh` so the standard options are applied.
4. **Launch the experiment in the new container** (detached, host-mounted log — [`code_style.md`](code_style.md) §12.2) and **verify once after launch**: `pgrep -af` + `tail` the log, since a host GPU that looks idle does not prove the container can reach it.
5. **Reconcile `run/_index.md`** — the broken container's row and the new container's row must both reflect the real slot: mark the old run `killed` / `ended — awaiting verdict` per its true state, add the new launch's block.

## Git Attribution — no AI co-author line (HARD RULE)

Commits, PRs, and tags carry the **user's** authorship only. This clause **overrides the harness
default** that tells the agent to end a commit message with a `Co-Authored-By: Claude …` trailer and
a PR body with a `🤖 Generated with [Claude Code]` line — neither is ever emitted, in **any** repo
reached from this project (parent repo, every submodule, the alias repo).

| Rule                     | Detail                                              |
|--------------------------|-----------------------------------------------------|
| **No co-author trailer** | never an assistant `Co-Authored-By:` on any commit  |
| **No generated-by line** | never `🤖 Generated with …` in a PR body / tag      |
| **No AI mention at all** | the text names the change, not the assistant        |
| **History is not rewritten** | existing trailers stay — the ban is forward-only |
| **Source tree too**      | no attribution markers in comments / docs ([`code_style.md`](code_style.md) §6) |

Stripping old trailers by amend / force-push happens **only** on an explicit user request.
Check before every push — must print `0`:

```bash
git log @{u}..HEAD --format='%B' | grep -ciE 'claude|anthropic'
```

## Refactoring — read the topic before touching its source

Before any code change under an active refactoring topic (an edit, a question, any code task),
open `md_files/refactoring/{topic}/refactoring.md` and follow the documented design. To find
*which* topic a source file belongs to, consult the master `md_files/refactoring/_index.md` — its
`hook-registry` block maps source-path prefixes → topic, and the `reminder_refactoring.sh` hook
fires the same mapping on Edit/Write. Do not re-derive design decisions already captured there;
exploration-first defeats the purpose of having a plan file.

The directory layout, backup-diff rule, and phase-verification convention are in
[`code_style.md`](code_style.md) §7; where the doc lives and when it is deleted is §9.3.

## Multi-User Workspace

Per-user workspace lives under `md_files/users/`:

```
md_files/users/          ← gitignored in full (machine-specific)
  {username}/
    access.yaml        ← this user's server / docker / secrets map
    user.md            ← user rules, active servers, container names
    run/_index.md      ← Active table of long-running remote processes
    run/done/{YYYY-MM}/ ← finished runs, one file per batch
```

There are no slash commands for this workspace. Adding a user is not a manual task — the user runs
`alias/setup.sh -p {project}` from their own alias clone, and it writes `user.md`, `run/`, and
`access.yaml`. Only the two destructive operations are done by hand, and both must touch every
location or they leave orphans:

- **Remove a user** — confirm with the user first, then delete `md_files/users/{username}/` (this takes `access.yaml` with it) and drop the row from `project.md` → Users.
- **Rename a user** — move `md_files/users/{old}/` → `{new}/` (`access.yaml` moves with it; its keys carry no username, so nothing inside needs rewriting), rewrite the container names inside `user.md` and `run/**`, and update `project.md`. Leave `run/done/**` bodies untouched (historical records).
