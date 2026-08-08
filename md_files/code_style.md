# Code & Doc Style

Conventions for the artifacts this repository produces — source, documentation, and the way both are run and stored. Project-agnostic: every rule here says *how* something is written, never *what* this project is. A rule that cannot be stated without naming a specific project, server, dataset, or experiment belongs in [`project.md`](project.md) instead.

Sibling to [`agent.md`](agent.md) (how the agent works — plan/verify/execute/review, reporting, launch procedure) and [`project.md`](project.md) (this project's architecture, and its concrete instances of the rules below). Each rule has exactly one home; the other two files cross-reference it rather than restating it.

## 1. Naming Conventions

- `n_` prefix for counts and repetitions: `n_envs`, `n_critics`, `n_blocks`, `n_gradient_step_per_update`. Never `num_`. When porting external code that uses `num_`, rename at the boundary.
- Private helpers and attributes use a single leading underscore. Reserve double underscore for true name-mangling needs only.
- Renames are made directly, not via aliases or `# old_name = new_name` shims. Update every call site in the same change. Backwards-compatibility shims are discouraged unless an external consumer outside this repository depends on the old name.
- When a name carries a tier or version (e.g. `set_task` → `set_task_set`), include the full hierarchy in the new name rather than overloading the old one. Same for chain-level renames (`v5_*` → `v6_*`) — apply across the whole chain in one commit, not file by file.
- **Abstraction-level suffixes** for sub-component attributes (when a project splits primitives by abstraction level — e.g. `layer/` vs `block/` vs `module/`):

	| suffix | for | example |
	|--------|-----|---------|
	| `_layer` | a layer-level sub-component (raw-tensor signature) | `self.in_proj_layer`, `self.norm_layer` |
	| `_layers` | a list of layer-level sub-components | `self.cell_layers`, `self.mlp_layers` |
	| `_block` | a block-level sub-component (structured signature) | `self.encoder_block` |
	| `_fn` | a method / callable / function attribute ONLY | `loss_fn`, `encoder_fn`, `apply_fn` |

	`_fn` is reserved for callables (often auto-registered by a framework); do not apply it to a list of sub-modules.
- **Single canonical name across abstraction levels** for the same concept. If `LayerX` uses `is_output_layer`, then `BlockY` that wraps `LayerX` must NOT introduce a synonym like `mark_last_layer` for the same flag — pick one canonical word and propagate.
- **No cryptic abbreviations — spell role/scope prefixes in full.** Use `critic_`, `actor_`, `encoder_`, `decoder_` (and `ctx_` for context — the one sanctioned short form); never single-letter prefixes like `c_`, `a_`, `e_`. The worst case is a short prefix that means two things — `c_` reading as *both* `critic_` and `context_` makes every `c_*` ambiguous. Applies to locals, params, dict keys, and struct fields, not just public API. Even a code variable that mirrors a paper symbol (`a_j` in an equation) is spelled out (`action_j`); keep the symbol only in the comment.
- **One canonical name per concept, end to end.** A concept keeps the same name through every layer (storage → sample → consumer) so there is no per-seam remap (`cur_action` everywhere, never `action` → `cur_action` at the boundary). Map a foreign vocabulary (an external library's terms) to the canonical name once, at the single seam where it enters — not scattered across consumers.
- **Distinct concepts get distinct names — never overload one name for two.** When two coexist, name them apart (raw next state `next_state` vs the decoder's K-future forecast `next_state_target`). Reusing one name — or one prefix for two scopes — is a bug magnet.
- **The external-API edge is the one naming exception.** Where code mirrors an external library (gym `obs`/`action`, a framework's field names), keep that vocabulary *only* in the boundary signature; everything inside uses the canonical name. The visible foreign term then signals "this is the external edge," which aids recognition.

## 2. Module Structure

- Choose between inheritance and composition by the universal-vs-optional rule: *universal* concerns (must be present in every instance) belong in an inheritance chain; *optional* concerns (only some leaves need them) are injected via composition.
- Inheritance chains are linear — no multi-inheritance. A sibling that does not fit the chain (e.g. orchestrators that hold other instances) is a separate top-level class deriving from the same root, not a mix-in.
- Single source of truth: any constant used by ≥ 2 call sites is defined once at module top, not duplicated inline. A literal appearing in 2+ places must be hoisted to a named constant.
- Extract helpers when 3+ call sites share the same multi-line pattern. Helpers make the contract explicit and prevent per-callsite drift.
- Prefer one class or one cohesive group of helpers per file. When a file grows past ~500 lines, split by sub-topic.

## 3. Interfaces, Arguments, and Dependency Injection

Two-tier convention for `__init__` / call signatures:

| Layer character | Style | Example |
|-----------------|-------|---------|
| Entry-point / application | **Receives whole `config` object** | top-level orchestrators, factories |
| Library / reusable component | **Receives only specific scalars / shapes / objects** | numerical kernels, replay buffers |

- Entry points construct expensive shared resources (vec_env, models, etc.) and **inject** them into library components. Library components must not reach back into `config.*` namespaces.
- Resources that own randomness, mutable state, or external handles are passed in by reference, not re-created inside the receiver.
- Factories belong at the entry-point layer. Library code expects the resource already built.

## 4. Config Schema

- Configuration belongs in YAML (or equivalent structured file), not in code. Code reads the schema; code does not redefine defaults.
- Schema is grouped by scope (e.g. `env.*`, `model.*`, `training.*`). When a field is used by exactly one consumer, it lives in that consumer's section — no shared global bag.
- Validation happens at the boundary where the config is first consumed (entry point), not deep inside the call tree.
- Field renames bump every YAML file in the same change; no compat shim layer.

## 5. Error Handling

- Fail fast at boundaries: validate inputs where they enter the system (entry point, public API) and raise `RuntimeError` / `ValueError` immediately on contract violation.
- Internal helpers trust their callers — do not re-validate the same input five layers deep. Trust internal code and framework guarantees.
- Do **not** add fallbacks for cases that cannot happen. If a code path is unreachable given the construction contract, do not write defensive code for it.
- When a precondition is required (e.g. "a manager must be attached before step"), check it explicitly with a clear `RuntimeError("manager not attached…")` rather than letting an `AttributeError` surface deeper.

### 5.1 No unrequested fallback (🔴 HARD RULE)

**Never write a fallback that the user did not ask for.** A fallback substitutes a second behavior when the first one fails, and in doing so it *deletes the failure*: the run continues, the exit code is `0`, the log looks ordinary, and the artifact is silently wrong. The bug then surfaces days later as a number nobody can explain. Fail **loudly, at the point of failure** — that is the default, and it is not negotiable for convenience.

The bullet above bans fallbacks on **unreachable** paths; this rule extends it to **reachable** ones. "But what if the file is missing / the key is absent / the import fails?" is precisely the case that must raise, not degrade.

The ban is on the **behavior**, not on any keyword — any construct where a missing or failing primary path still yields a usable-looking value is a fallback:

| Fallback shape | Write instead |
|-----------------------------------------|------------------------------------------|
| `except Exception: x = default`          | catch the one expected type, or let it raise |
| `d.get(key, default)` for a required key | `d[key]`                                 |
| `getattr(o, "a", default)` on a contract attr | `o.a`                               |
| `x = cfg.lr or 3e-4`                     | read the value the schema guarantees     |
| `if not path.exists(): return`           | `raise FileNotFoundError(path)`          |
| primary source empty → try a second one  | pick one source; make the other explicit |
| `try: import x / except: x = None`       | import it; declare the dependency        |
| a retry loop that gives up and continues | give up and **raise**                    |
| `cmd 2>/dev/null \|\| true`, `\|\| echo default` | let the rc propagate (§12.9)      |

**The three narrow exceptions** — each is declared in the plan before it is written, never discovered in review:

- **The user asked for it.** Then it is a requirement, not a fallback, and it is named as such in the plan.
- **A genuinely optional config field.** Its default lives in the **yaml schema** (§4) and is read as an ordinary value — nothing failed, so nothing was substituted.
- **A declared compat seam** — one boundary that must span two versions/platforms, isolated in a single function, carrying a removal trigger (§10).

**When a fallback is authorized it carries all three of:** *narrow* (one named exception type or one named condition — never bare `except`), *loud* (logs at WARNING or above the moment it engages, naming the primary that failed **and** the substitute used), and *bounded* (a written removal trigger if it is temporary).

- A fallback found in review is **removed**, not documented. Adding a comment explaining a silent fallback keeps the defect and adds prose.
- **Making a failure loud must not make every run fail.** An exit code answers *"did the requested work happen"*, not *"was every input perfect"*. Where a fan-out legitimately skips inputs (a box that is down, a source holding nothing), name those in a WARN list and keep the exit code for real failures — an always-1 exit is ignored exactly as fast as an always-0 one. Check the callers' real invocation before promoting a printed error to a non-zero exit.
- The reporting counterpart — never leaving "did it stop, or did it fall back?" implicit in a report — is [`agent.md`](agent.md) → Reporting Style.

## 6. Comments and Docstrings

- Default to writing no comments. Comments document *why*, never *what*. Well-named identifiers should make *what* obvious.
- When a comment is needed (hidden constraint, subtle invariant, workaround for a specific bug, behavior that would surprise a reader), keep it to one line. Multi-paragraph docstrings and multi-line comment blocks are not used.
- Never reference the current task, PR, or caller in comments ("used by X", "added for the Y flow"). That information belongs in the PR description.
- **No attribution markers in source** — not a per-user tag (`# usertag: {user}`), not an AI-authored marker. Who asked for a change and who wrote it are recorded by the commit, never by the file. See [`agent.md`](agent.md) → Git Attribution for the commit-side half of this rule.
- Comments in code are English. Korean belongs in `md_files/` docs and per-user notes only.
- **Sources that ship to outsiders** (a submitted/arXiv `.tex` tree, a released header) carry no workflow records, verification TODOs, rationale blocks, or internal repo references — the reader is not the team.

### 6.1 Comment register — no assistant fingerprints (🔴 HARD RULE)

A comment reads as **the engineer who owns this code, writing to the next engineer who will change it**. Every giveaway below comes from writing in some *other* register instead: teaching an audience, or narrating an editing session. The ban is on the register, not on any one word — a comment that merely avoids the listed strings while still addressing a student or recounting an edit is the same violation.

This applies to **every** comment in **every** source language (`#`, `//`, `/* */`, docstrings, YAML comments) and to generated files that ship as source. It is not waived for tests, scripts, or throwaway probes.

| # | Never appears | Why it is a giveaway |
|---|---------------|----------------------|
| 1 | **Assistant / tool attribution** — `by Claude`, `AI-generated`, `Copilot`, `ChatGPT`, `co-authored` | Authorship is the commit's job. Extends the attribution-marker rule above |
| 2 | **Session or request references** — `(user directive)`, `as requested`, `as discussed`, `per your ask`, `user request` | Records *who asked*, which no reader of the code can use. Same defect as the task/PR ban above |
| 3 | **Edit-event narration** — `FIXED:`, `UPDATED:`, `NEW:`, `changed from X to Y`, `we now use …`, `previously this did …` | A changelog inside the file; git already holds it, and it rots on the next edit |
| 4 | **Conversational address** — `Let's …`, `we'll …`, `you can see …`, `feel free to`, `don't worry`, `Note that we` | Addresses a student. Code comments have no second person |
| 5 | **Self-praise / enthusiasm** — `elegantly`, `cleanly handles`, `nice trick`, `Great!`, `simply just` | Grades the code instead of explaining it |
| 6 | **Emoji or decorative glyphs** in a comment body | Never appears in engineer-written source; instantly reads as generated |
| 7 | **Restating the code** — `# increment i`, `# loop over the items`, `# return the result` | §6's *why-not-what* rule; padding that AI adds by default |
| 8 | **Unanchored hedging** — `should work`, `might need adjusting`, `for now` with no removal trigger | Ships uncertainty as a permanent artifact (§10 requires a written trigger) |
| 9 | **Tutorial scaffolding** — `Step 1/2/3` on a trivial sequence, `# ===== SECTION =====` banners in a short file, a docstring on every one-line helper | Structures the file for a reader being walked through it |

- **Rows 3 and 9 have a narrow legitimate form.** Describing *runtime* state that an earlier run produced ("leftover from a failed prior run") is not edit narration — it tells the reader what they may find on disk. Numbered steps are fine when the order is itself the contract and getting it wrong is destructive (a backup-then-delete sequence); they are not fine as decoration on ordinary code.
- **Vendored / third-party trees are out of scope** (`reference_codes/`, an upstream library copied in). Never rewrite an upstream comment to satisfy this rule — that creates a diff against upstream for no behavioral gain.
- **Check what you wrote before committing**, not later: re-read each comment and ask whether an engineer who never saw the conversation would write that sentence. If the answer needs the conversation, delete the sentence.

## 7. Refactoring Workflow

- Each refactoring lives under its own directory (`md_files/refactoring/{topic}/`) with:
	- `refactoring.md` — plan, decisions, start/end dates, progress notes
	- `_index.md` — one-glance status + source-path scope + links to coupled topics
	- `backup/` — original source mirror for diff reference (read-only, not executable)
- `md_files/refactoring/_index.md` is the **master index**: a table of active/completed topics plus a `hook-registry` block mapping source-path prefixes → topic. Read it to find which topic a given source file belongs to. The `reminder_refactoring.sh` hook parses the same block to fire a "read refactoring.md first" reminder when an edited source path matches.
- Read the topic's `refactoring.md` before any code change under an active topic. The plan is authoritative — do not re-derive design decisions that were already captured.
- Verify each phase against `backup/` before moving to the next phase. After completion, **delete the topic directory** (`git rm -r md_files/refactoring/{topic}/`) — git history is the archive, no separate archive copy — and drop the topic from the master `_index.md` (table + hook-registry).
- Refactorings progress in numbered phases (Phase 1, Phase 2, …). Each phase ends with a verification step (tests pass, output reproduced) before the next begins.

### 7.1 Backup-code review discipline

Backup files are the **intent reference**, not the implementation template. When porting from a backup:

| Do | Don't |
|----|-------|
| Identify the **intended mechanism** the backup expresses (e.g. TBPTT chunked stop_gradient, hidden carry, stabilizer placement). List each before writing new code. | Skip backup mechanisms because the new design "looks cleaner" without them — that drops design intent silently. |
| Cross-check each mechanism against current standards (papers, official repos). Keep what is validated; replace what is outdated. | Trust backup comments at face value (e.g. "Stabilizer to replace LayerNorm" when the construct does not actually stabilize). Verify the comment matches the code. |
| Re-implement the mechanism in current standards (new APIs, modern conventions). | Blind-carry attributes, kwargs, or options whose purpose you cannot articulate (e.g. legacy `return_natural_value` flag). Drop or justify each carry-over. |
| When deferring a backup mechanism to backlog, surface the trade-off explicitly in the design doc — do not defer silently. | Defer mechanisms to "backlog" as a quiet way to ship a smaller diff — the rest of the team will assume the backup behavior is preserved. |

## 8. Testing Style

- Tests are organized by topic, not by mechanical correspondence to source files. A topic-level test file covers one concern across multiple source modules.
- Three levels: **smoke** (compile / import / one step), **unit** (single function or class in isolation), **integration** (multi-module end-to-end). Each level has its own directory or naming prefix.
- After any refactor, regenerate downstream artifacts (numbers, plots, schemas) and compare against the previous output. "Tests pass" is not enough — feature correctness needs the artifact to match.
- Integration tests hit real dependencies (real database, real subprocess) where the risk of mock/prod divergence is non-trivial. Mocks are reserved for genuinely external systems that cannot be exercised in CI.

### 8.1 Cross-module alignment (mandatory in every test task)

Testing a module in isolation is **not enough** — an isolated pass hides interface drift (signature / shape / unit / contract / lifecycle mismatch) at the seam with the module's real collaborators. Running the coupled modules **together** is what performs the *alignment* work: it forces every seam the new code touches to agree.

| Rule | Detail |
|------|--------|
| **Wire in the real collaborators** | When a test exercises module X, also construct and run the modules X actually composes with (its callers and callees), not stubs. Drive them through one shared entry path so their contracts meet. |
| **Trace the coupling first** | Before writing the test, grep the new/changed symbol's call sites and the objects it is injected into. Each coupled module is either in the test or explicitly justified as out of scope. |
| **One end-to-end path per change** | Every behavior change ships at least one test that runs the change through its full coupled path (e.g. manager → env → vec_env → runner step), not only a unit assertion. See [`agent.md`](agent.md) → Definition of Done (the update-≥1 e2e gate). |
| **Align at the boundary, fail loud** | If a coupled module disagrees (wrong shape / missing key / stale signature), the combined test must surface it as a hard failure — never adapt the test to paper over the drift. Fix the seam, then re-run the combined test. |

- The deliverable of a test task is **alignment**, not a green unit. A change is "tested" only once it has been run together with the modules it is wired into and they agree.

### 8.2 YAML-config-driven e2e (the completion gate)

A test that builds its config by hand (a `SimpleNamespace` / dict / virtual-config helper) exercises the modules but **not the config path production uses**. The real entry point reads the **yaml** through the project's config loader, validates the schema, and injects it. A hand-built config silently diverges from that schema — a renamed/missing yaml field, a validation rule, or an injection seam passes the test yet breaks the real run. This is the §8.1 drift hole at the config boundary.

| Rule | Detail |
|------|--------|
| **e2e loads the real yaml** | The completion-gate test runs through the project's config loader on the **actual yaml**, not a hand-assembled config object. The loader's name and entry module are project-specific — see [`project.md`](project.md). |
| **Override only speed knobs** | To make it short, override `n_iteration` / `n_env` / `n_sample` / `max_steps` via the real override path. Never fork a parallel mini-yaml (it drifts); never touch algorithmic knobs. |
| **Through the entry point** | Drive the `runner` entry point, not a test-local `build_*` shortcut, so parse → validate → inject → run is exercised end to end. |
| **Module/virtual tests stay** | Fast `SimpleNamespace` smokes still isolate logic — keep them, but they are **unit**, not the e2e gate, and not a completion claim. |

This section owns **how the gate test is built**. *When* that test lets you claim done, and what the
done-report must contain, is [`agent.md`](agent.md) → Definition of Done — do not restate either half
in the other file.

## 9. Documentation

### 9.1 Documentation sync

- When a structural decision changes (inheritance chain, module split, naming scheme), update the project-level doc (`project.md` or the relevant guide) **in the same change**. The doc and the code must not drift across commits.
- Cross-references between docs use relative markdown links, not absolute paths.
- On move or delete, **update every reference** — including paths written inside code comments and docstrings, not just markdown links.

**Cross-reference direction — a back-reference is allowed, a mutual restatement is not.** Two
documents may point at each other, because a reader can enter from either side and must be able to
reach the other half. What must never happen is both sides saying the *same* thing. Test any A↔B
link pair against this:

| Shape | Meaning | Verdict |
|-------|---------|---------|
| **complementary** | each side states its OWN half and links to the other half | ✅ round-trip always yields new information |
| **duplicated** | both sides state the same conclusion | 🚫 the link is a wasted trip, and one copy goes stale on the next edit |
| **mutually deferred** | both sides say "the rule lives over there" | 🚫 the rule then lives nowhere |

- The usual complementary split is **rule ↔ value**: this file (or `agent.md`) states the rule and
	says the concrete value is project-specific; `project.md` states the value and cites the rule it
	instantiates. That is a type conversion, not a cycle.
- Before adding a back-reference, name which half each side owns. If you cannot, the content belongs
	in one file only and the other gets a pointer with no restatement.

### 9.2 Markdown writing style

| Category       | Guideline                                                              |
|----------------|------------------------------------------------------------------------|
| **Structure**  | Use clear heading hierarchy (H1 → H2 → H3), group content by category |
| **Readability**| Prefer tables for comparisons/specs, use lists for sequential items    |
| **Conciseness**| One idea per bullet, no filler — keep sentences short and direct       |
| **Formatting** | Bold (`**key**`) for key terms, backticks for code/paths/commands      |
| **Lists**      | Use hyphens (`-`) consistently, tab-indent for sub-levels              |
| **Code**       | Always specify language in fenced blocks (` ```python `, ` ```bash `) |
| **Links**      | Cross-reference related docs with relative paths                       |
| **Whitespace** | One blank line between sections, no trailing whitespace                |
| **No path hardcode** | Never hardcode host/container mount paths in markdown — cite the field name instead (§12.6) |

Tables are the primary format for rules and specs, so they carry their own rules — these keep a table readable in both raw source and rendered views:

| Rule | Detail |
|------|--------|
| **Pad cells with spaces** | Align columns in source: `\| Short  \|` not `\| Short\|`. Improves raw-markdown readability |
| **Keep cells ≤ 60 chars** | Long content degrades readability — move to prose or split into sub-tables |
| **Use alignment markers** | Left `:---`, center `:---:`, right `---:` — apply when data type benefits (numbers right, labels left) |
| **Match separator to header width** | Pad dashes to match the widest cell: `\|--------\|` not `\|--\|` |
| **Avoid 4+ columns** | Wide tables overflow in narrow viewports (IDE sidebar, mobile). Split or restructure |
| **No merged concepts per cell** | One idea per cell — use separate rows instead of cramming with commas |

### 9.3 Where a new `.md` goes

Before creating any `.md`, answer one question first: **does this document disappear when the
work is done?** Permanent reference and disposable plan live in different trees; filing a plan
in the permanent tree — or a canonical decision in the disposable one — is the failure this
table prevents.

| Kind | Location | Lifetime |
|------|----------|----------|
| **memory** — reference / design / verdict that outlives the task | `md_files/memory/{category}/` | **permanent** |
| **plan · handoff** — an execution plan that ends | `md_files/plans/{slug}.md` | **deleted** on completion or abandonment |
| **refactoring** — a code-migration topic | `md_files/refactoring/{topic}/` | directory **deleted** on completion (§7) |
| **run record** — experiment launch / verdict | `md_files/users/{user}/run/` | Active → `done/{YYYY-MM}/` (§12.3) |

🚫 **`md_files/users/{user}/` holds `user.md` and `run/` — nothing else.** Never create audit
reports, plans, handoffs, design notes, or audit dictionaries there, and never scatter them at the
`md_files/` root. A per-user scratch or staging file is **not** part of this layout.

**memory — `md_files/memory/`.** Only what outlives the task: reference docs, design canon, and
verdicts that must produce the same answer when the question recurs months later.

| Rule | Detail |
|------|--------|
| **Classification** | Group into category directories. **≥2 files → a directory** (drop the prefix from the filename); **a singleton stays flat** with its project-root prefix (never a one-file directory) |
| **Index it** | Every new file gets a row in the `project.md` memory index — a memory file missing from that index is a file nobody finds |
| **Title must match content** | The H1 states what the body actually is. If the document is a procedure rather than an analysis, the title says procedure |
| **When an audit closes** | Keep only the **verdict**; delete the process record (sentence dictionaries, convergence narratives, completed edit logs). Git history is the archive |
| **vs auto-recall memory** | A decision that compresses to one line → auto-recall memory. Anything needing tables, procedures, or `file:line` evidence → a `md_files/memory/` file, with the auto-recall entry pointing at it |

**plan · handoff — `md_files/plans/`.** A handoff is not a separate kind — "an execution plan
another session picks up" *is* a plan. Same tree, same rules.

| Rule | Detail |
|------|--------|
| **Location** | `md_files/plans/{slug}.md`. A multi-file plan gets `md_files/plans/{slug}/` |
| **Naming** | `{slug}` is short snake_case with **no date prefix/suffix** (git records the date). Revisions overwrite the same file — no `_v2` / `_v3` siblings |
| **Header** | First lines carry the status (in progress / done / dropped) plus a one-line goal, so status is readable without opening the body |
| **Delete on completion** | When the work is done (or dropped), **delete the file**. No completion log, no "done" ledger, no archive copy |
| **Move surviving conclusions out first** | Anything that must outlive the plan (a convention, a canonical decision, a residual open item) goes into `project.md` / `code_style.md` / memory **before** the plan is deleted |
| **Verify status against reality** | Judge "not started / done" by probing the code (`test -e`, grep for the symbol), not by trusting the document's own status line |

**Before creating any of them:** check whether an existing document absorbs it — a new section in
the right file beats a new file almost every time — and on creation wire the inbound reference
immediately (`project.md` index, parent-doc link, or auto-recall memory pointer).

## 10. Cleanup Discipline

- Remove dead code at the moment it becomes dead — do not leave commented-out blocks. Git history is the archive.
- Avoid backwards-compat hacks like renaming unused `_vars`, re-exporting removed types, or adding `// removed` comments. If something is unused, delete it.
- Temporary debug instrumentation has a written removal trigger (a date, a condition, or "remove after Y is verified"). Without a trigger, do not commit it.
- Build cache and bytecode (`__pycache__`, `.pyc`, etc.) never enter version control. Generation is suppressed at runtime by environment flags, not by `.gitignore` alone.

## 11. Functional / Stylistic Preferences

- Prefer pure functions for compute-heavy code. Side effects (logging, I/O, RNG advance) are pushed to the edges.
- Random state is class-owned: every class that samples randomness takes a `seed` in `__init__` and owns its own generator (`numpy.random.default_rng(seed)` / JAX `PRNGKey` threaded via `jax.random.split`). No shared default RNG, no `_resolve_rng`-style fallback. (See [`project.md`](project.md) → RNG Ownership.)
- Pythonic idioms over clever ones: comprehensions, dataclasses, context managers, `pathlib`. Avoid unnecessary abstraction layers that exist only to be "extensible".
- One idea per line; one responsibility per function. When a function body grows past ~30 lines or three distinct phases, split it.

## 12. Operational Style

Conventions for how code is *run*, built, and distributed across machines. Specific names (containers, servers, mount paths) live in the user's `user.md` (auto-generated from `access.yaml`). This section is the general operational style — substitute the actual names from `user.md` when applying.

### 12.1 Container Discipline

- Every Python invocation, build, and test runs **inside** a container. The host shell orchestrates (ssh, file inspection); it never runs project code directly.
- Each `docker exec` for Python sets `-e PYTHONDONTWRITEBYTECODE=1` so `__pycache__` / `.pyc` never appear in the project tree.
- Docker writes default to root ownership. After any Docker-side write that produces host-visible files, restore ownership to the host user: `docker exec -w <workdir> <container> chown -R $(stat -c '%u:%g' .) <target>`.
- Container roles are two:
	- **Lightweight / CPU base** — file ops, general terminal commands, project-independent scripts. Cheap to restart.
	- **Project-specific GPU** — Python execution, builds, GPU work. Pre-installed with the project's stack.
- Container *names* follow a per-user pattern (`base_<user>`, `<image>_test_<user>`, `<image>_<gpu_id>_<user>`). See your `user.md` → "Project Docker Containers" / "Active Servers" for the actual names.
- Do not chain commands with `&&` inside a single `docker exec`. Use one `docker exec` per command — clearer error attribution, predictable shell semantics.
- **Create containers through `{alias_dir}/core/docker/docker.sh`, not a hand-run `docker run`.** The script applies `--init`, `--shm-size`, and `--memory-swappiness` for you; a hand-run command is exactly where those get dropped. Remote server → the `docker` skill ([`agent.md`](agent.md) → Project Skills).

**Shared memory.** GPU containers (non-base) are always created with **`--shm-size=<90 % of host RAM>`**, auto-detected per server. The Docker default `/dev/shm=64MB` crashes DataLoader workers ("Bus error: DataLoader worker killed"). The 90 % ceiling scales with the box (a 128 GB host → ~115 g, a 64 GB host → ~57 g) and is lazy-allocated, so it is a limit and not a reservation. **Never** create a non-base container without it; a CPU-only base image is exempt.

**Swap.** The real guarantee is that **a GPU server carries no swap device** — keep `/proc/swaps` empty on every training box. That is the only mechanism that holds on every host.

- `--memory-swappiness=0` is the secondary, per-container guard: it disables swapping of the container's anonymous pages. Use it — **not** `--memory-swap == --memory`, which imposes a memory cap and would OOM-kill a long run on a DataLoader spike (the same failure `--shm-size` guards against). `--memory-swappiness=0` imposes no cap, so it never triggers an artificial OOM.
- **It only takes effect on cgroup v1 hosts.** cgroup v2 replaced the swappiness ratio with absolute limits (`memory.swap.max`), so docker discards the flag and prints `Your kernel does not support memory swappiness capabilities …`. That warning does **not** mean swap is off — it means the container is left free to swap at the host default. The split is per OS release, never per image (Ubuntu 20.04 / systemd 245 → v1; 22.04+ / systemd 249 → v2). `docker.sh` reads it in the per-server preflight (`stat -fc %T /sys/fs/cgroup`) and attaches the flag only on v1; a v2 host that *also* has a swap device gets a `[WARN]`, since that is the one combination where runs can silently swap.

**Base container.** If `base_{PROJECT_USER}` is not running, start it before any file-level container work:

```bash
docker ps -a --format '{{.Names}} {{.Status}}' | grep base_{PROJECT_USER}   # check
docker start base_{PROJECT_USER}                                            # if Exited
{alias_dir}/core/docker/docker.sh -i base -t <server>                       # if absent
```

### 12.2 Long-Running Processes (SSH-Decoupled)

- Any command expected to outlive the SSH connection is launched **detached** (`docker exec -d ...`). Foreground SSH-bound commands die on disconnect; `tmux` and SSH keepalive are not substitutes for detached launch on real long runs.
- Output goes to a host-mounted log file via shell redirection:
	- Path pattern (host): `{source_mnt_path}/artifacts/tee_log/{project}/{topic}/{YYYYMMDD-HHMMSS}.log`
	- Path pattern (container): `{target_mnt_path}/artifacts/tee_log/{project}/{topic}/{YYYYMMDD-HHMMSS}.log` (typically `/root/{basename}/artifacts/tee_log/...`)
	- **`artifacts/` lives ONE LEVEL ABOVE the project root** — a sibling of the `{project}/` directory (next to `datasets/`), not inside it. `{source_mnt_path}` / `{target_mnt_path}` is the **parent of all projects** (= `default_source_mnt_path`), not the project root. See §12.6.
	- **Always use the absolute path.** Never write the log as a relative `tee_log/...` — scripts run with cwd = project root, so a relative path resolves *inside* the project root (wrong). The mounted host dir avoids growing container storage only when it resolves to the parent.
	- **`artifacts/` groups all host-side bulk run outputs**, each namespaced by project, so they never bloat the project tree / git (it is above the repo root → inherently outside git):

		| subdir | holds |
		|--------|-------|
		| `artifacts/tee_log/{project}/{topic}/` | detached-run logs |
		| `artifacts/runs/{project}/{run_id}/` | raw training save (weights / checkpoints / config / logs) |
		| `artifacts/wandb_logs/{project}/` | wandb run store |

	- Any in-repo `outputs/` (e.g. promoted models, analysis figures) keeps only **refined** artifacts, never raw runs. Bulk-save dirs are auto-created at write time (`os.makedirs(exist_ok=True)` / framework default), so the grouping appears on first run with no manual setup.
	- `{topic}` is a short snake_case slug describing the run kind. One run = one log file. Never write logs ungrouped directly under `{project}/`.
	- Filename is the launch timestamp — re-runs never collide; ordering is obvious from the name.
- Use `python3 -u` for unbuffered output (real-time tail).
- Status checks are on demand, not by polling — `pgrep -f <script>` for liveness, `tail -N <log>` for last lines.

### 12.3 Active Run Tracking

- The user's `run/_index.md` under `## Active` is **one H3 block per launch (batch)**, and each block carries a **per-method table** so that *which method ran where, under which run_id, with which status* is readable at a glance — never buried in prose `>` lines. A block has two parts:
	- **Header line** — `### <slug> (<seed/variant>) — <Started KST>` + one meta line with the batch-shared fields: `commit`, wandb `project`, `log topic`, and gate/note. These are shared by every method in the batch, so they live once in the header, not per row.
	- **Method table** — one row per method, so co-tenant methods are distinguished:

		| column | content |
		|---------|---------|
		| Method | the method name (one row each — never a comma-list) |
		| Where | server · GPU · container — the actual slot (e.g. `<server> g1 <container>`), never a size label like `heavy/light` |
		| run_id | the **on-disk save-dir name** — the dir under `artifacts/runs/{project}/{run_id}/`. **NOT the wandb run name.** Minted at launch from timestamp+seed+random, so it is unknown until the run dir exists — fill it by readback (below), `—` until then |
		| wandb | the wandb **run name**, deterministic from config and therefore known at launch time. `—` when wandb is off (e.g. eval). project (header) + this locates the wandb run |
		| Status | `running` · `ended — awaiting verdict` (proc gone, readout pending) · `done` · `killed` — per method, since methods finish independently |

	The functions that mint the run_id and the wandb name are project-specific — see [`project.md`](project.md).

- **run_id ≠ wandb name — do not conflate them.** The wandb name is deterministic from config and knowable at launch; the folder run_id is minted at runtime and is *not* predictable. Fill the column by **reading it back right after launch** (`run_scan`, or the newest dir under `artifacts/runs/{project}/`, or the `run_id` field inside the run's own config dump). Never copy the wandb name into the run_id column as a placeholder.
- Rows stay thin because the batch-shared fields are hoisted to the header and everything else is reconstructable from conventions: log path from §12.2 (`artifacts/tee_log/{project}/{topic}/{ts}.log`), liveness from `pgrep -af '<pattern>'` inside the container — the launch argv carries the per-method flags, so the method→slot map is verifiable rather than guessed. If a launch genuinely needs longer notes (relaunch lineage, fix history), create its done-file (below) at launch time and link it from the header line.
- Lifecycle is two stages:
	- **Active** — the batch's H3 block. On a method finishing, flip that method's Status in place (`ended — awaiting verdict` until judged); the block stays until the whole batch is judged.
	- **Done** — once the batch is judged (or confirmed dead/killed), move the block to `run/done/{YYYY-MM}/{YYYY-MM-DD}_{slug}.md` (date = finish/confirm date; launch date if unknown) with a short result readout, and delete the block. The monthly directories ARE the archive — `md_files/users/` is gitignored in full, so these records live only on the machine that wrote them; copy anything that must survive that machine into `md_files/memory/` or the project's `outputs/results/`.
- **Artifact collection on completion (remote launch)** — a run launched on a non-hub server leaves its raw save in that server's above-root `artifacts/runs/{project}/{run_id}/`; before the Active block moves to Done, **collect to the hub** (`sync_hub=true` server) into the hub's in-repo `outputs/models/{topic}/` (via `artifact_sync -a collect`, narrowed to the exact dirs with `-r/-R` and `-x weights` — never a hand-rolled rsync). Collection scope depends on the run kind: a **training run** is pulled as the whole raw `{run_id}/` dir (weights are the artifact); an **adaptation / eval run** is pulled as its **complete logging set** (config + every log file + stdout) **excluding weights and `check_points/`** — adapted weights are never reloaded, but the raw logs must stay re-processable, so collecting only a subset of the logs is not allowed. Then write the judged readout / aggregates under `outputs/results/{topic}/` and record the run_id ↔ method mapping there. The done-file readout links both locations. Topic layout and promoted-path details are project-specific — see the project's `project.md` → Artifacts / Outputs Layout.
- The agent is not a daemon — status is checked only when prompted; nothing auto-polls.

### 12.4 Cross-Host Operations

- Cross-server git / docker / rsync / run-discovery ops always go through the project skills — the roster and what each covers is in [`agent.md`](agent.md) → Project Skills. Hand-rolled `ssh ... 'git pull'` / `ssh ... 'docker exec'` / `ssh ... ls` chains are forbidden: they trip on known-host prompts, agent-forwarding gaps, and missing post-write `chown`.
- A per-batch collector script that re-hardcodes server IPs / ports / keys is the same violation in slower form — those coordinates already live in `md_files/users/{user}/access.yaml`, and the skills read them.
- Short interactive foreground sessions may use `ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=120` for keepalive. Not a substitute for `docker exec -d` on actual long runs — keepalive keeps the pipe alive but the process still dies if the SSH client is killed.

### 12.5 Tests vs Experiments

- **Tests** (smoke, unit, integration) run on the development host's container — quick, interruptable, no long log expected. Must be idempotent and self-cleaning.
- **Experiments** (training, multi-day runs, distributed measurements) run on dedicated GPU servers, never on the development host. A crash on the dev host should never kill a multi-day experiment.
- The specific dev host vs GPU server matrix is per-user — see your `user.md` → "Active Servers" (allow_push / sync_hub / GPUs columns). The `sync_hub=true` server is for push and result collection, not training load.
- Pre-flight before any training: verify the objective actually separates candidates at the production horizon, on a non-learning baseline. Training against a flat objective wastes GPU time. The project's concrete pre-flight is in [`project.md`](project.md).

### 12.6 Path Mapping

- Source-mount and container-mount paths are per-user / per-server. Never hardcode them in code or docs — cite the field location and let setup substitute.
- Refer to `md_files/users/{user}/access.yaml` fields (the file carries no username wrapper):
	- Host source root: `{source_mnt_path}` = `servers.{server}.source_mnt_path`
	- Container target root: `{target_mnt_path}` = `docker_images.{image}.target_mnt_path`
	- Alias / setup root: `{alias_dir}` = `servers.{server}.alias_dir`
- Default container mount rule: `{source_mnt_path}` → `/root/{basename_of_source_mnt_path}` (defined in docker image config).
- Subdirectories below the project root (`{source_mnt_path}/{project}/...`) are identical between host and container; only the prefix differs.

### 12.7 Entry-Point Bootstrap

- Project entry-point scripts (run / test) share a single bootstrap that handles `sys.path`, `PYTHONDONTWRITEBYTECODE`, stdout buffering, and framework env flags (e.g. XLA / CUDA prealloc). Individual scripts never hand-roll `sys.path.insert` or `os.environ[...] = ...` for these — they import from the bootstrap.
- Pure-Python entry uses `python3 -m <pkg.module>` over direct file paths, so the project's package boundary is honored.
- Build cache (`__pycache__`, `.pyc`), shared-memory leftovers (`/dev/shm/*`), and tmp logs are cleaned proactively after crashes — small cleanup scripts ship with the project.

### 12.8 Run Completion Marker (`finished`)

**Every run-producing entry point must write a `finished` marker into its run dir as its
last successful act.** A run with no marker is not complete — no exceptions, no substitute
signal.

| Rule | Detail |
|------|--------|
| **Where** | `{run_dir}/finished`, written by the runner itself on its success path |
| **What** | small JSON: `run_id`, `status`, `finished_at` (ISO with offset), stage, plus counters that make the run auditable |
| **When** | only after the run reaches its documented end — never at launch, never from a wrapper script |
| **Never on failure** | a crashed / killed / early-returned run leaves NO marker; that absence is the signal |
| **One helper** | written by a single shared helper, not re-implemented per entry point |

- **Do not judge completion by an incidental artifact.** A checkpoint dir, a weights file, or
	a log file can be created at launch, left empty by a stage that produces none, or written
	mid-run — each has silently mis-reported completion. Only the explicit marker means finished.
- Every consumer — collectors, aggregators, the `run/_index.md` Status column, `run_scan -d file:finished` —
	tests the marker, never a derived heuristic.
- When a new entry point is added, its marker call ships in the same change. An entry point that
	can produce a run dir but never writes a marker is an incomplete implementation.

### 12.9 Shell scripting gotchas

Silent failures, not bash errors — each has bitten a multi-server script here before. A
"stops after the first iteration" or "only the first server got processed" symptom is almost
always one of these.

| Trap | Symptom | Fix |
|------|---------|-----|
| **`while read` loop + `ssh` inside** | Loop exits after the first iteration. ssh inherits the loop's stdin (fd 0), consumes the rest of the input, and the next `read` hits EOF. | Isolate the loop's stdin on fd 3: `while IFS= read -r x <&3; do … done 3<<< "$list"`. Also call ssh **without `-t`** for non-interactive commands (a forced pty re-grabs fd 0 even with `< /dev/null`). |
| **`ssh -t` for batch commands** | Same stdin-consumption symptom; the pty also breaks `< /dev/null` redirection. | Drop `-t`. Use `ssh -n` or the fd-3 isolation above. Keep `-t` only for genuinely interactive remote sessions. |
| **Prefix-matching an IP to answer "is this server me?"** | The wrong server is detected when one IP is a string prefix of another (`192.168.1.1` matches `192.168.1.10`), or when a wildcard pattern matches a docker bridge interface. | Match every address from `hostname -I` **exactly**, or through an octet-aware wildcard matcher. Most-specific pattern wins on ties. |
| **Unset `${VAR[@]}` under `set -u`** | A standalone copy of a shared helper dies with `unbound variable` because a list the helper expects is only defined in the full environment. | Guard with `[ -n "${VAR+x}" ]` before iterating, or pre-merge the data so the standalone never sees a missing variable. |

## 13. Standard Pattern Research

When implementing a component that has a well-established standard pattern in the field — RNN/LSTM/GRU encoders, transformers, attention, normalization layers, PEFT (LoRA / Adapter / O-LoRA), positional encoding, diffusion noise schedules, optimizer schedules, etc. — **research the modern standard BEFORE writing code**. Default to the most-cited modern reference; do not invent.

| Component class | Default reference to check |
|-----------------|----------------------------|
| RNN encoder / cell | PyTorch `nn.GRU`/`nn.LSTM` defaults, paper's official repo, HF Transformers RNN heads |
| Transformer / attention | Vaswani 2017 + HF Transformers `BertLayer` / LLaMA decoder layer |
| Normalization | LayerNorm (Ba 2016) for general; RMSNorm (Zhang & Sennrich 2019) for LLM-style pre-norm |
| PEFT (LoRA / Adapter) | HuggingFace `peft` package defaults; LoRA paper (Hu et al. 2021); O-LoRA repo (cmnfriend) |
| Loss / scheduler | PyTorch / Optax defaults; original paper appendix |

### 13.1 Research depth — full structure, not fragments

Reading a single docstring or a snippet is NOT research. Before writing the implementation:

1. **Read the full reference encoder/decoder block diagram** (paper §3 + official repo's main module file). Sketch the full pipeline (every layer in order — `embed → pos_enc → N × {norm → MHA → +res → norm → FFN → +res} → final_norm → head` for transformer). Compare your sketch against the paper figure.
2. **Identify every named sub-component** (positional encoding, residual connection, normalization placement, dropout placement, mask types). Each one is either in your implementation or explicitly justified as omitted.
3. **For each sub-component, check its hyperparameter defaults** in the reference (FFN ratio = 4× for BERT, kernel sizes 1/3/5/… for Inception_Block_V1, etc.).
4. **Architecture-specific PEFT target** — LoRA is applied at different places per architecture (RNN = head, Transformer = attention Q/K/V/O, VAE = latent heads). Decide explicitly; do not default to one global rule.

### 13.2 Reference-comparing smoke tests — beyond shape

A passing `out.shape == (B, S, D)` smoke does NOT verify the implementation. Add semantic checks that would fail if a standard mechanism were omitted:

| Mechanism | Failing-on-omission smoke |
|-----------|--------------------------|
| Positional encoding | Zero input + run forward: outputs at different positions must differ |
| Causal mask | Change a future input; outputs at earlier positions must stay identical |
| TBPTT detach | Loss on step k must have zero gradient wrt input at step < detach boundary |
| Dropout (`deterministic=False`) | Run twice with the same RNG seed — outputs must differ (vs deterministic=True identical) |
| PEFT scope | Build the block with PEFT enabled; flatten params; assert the adapter leaves appear **only** under the architecture-specific scope chosen in 13.1 |
| Hidden carry IN | Passing nonzero carry must change output vs default zero carry |
| Multi-head split | Verify `hidden_dim % n_heads == 0` rejection on bad config |

Run these smokes on every block, not just shape sanity.

### 13.3 Portability to a tracing / JIT framework

Eager-mode patterns (PyTorch, NumPy) do not always trace. The rules below are stated for JAX,
the most common case; the same three failure classes appear in any tracing framework:

- **Dynamic shape under `jax.jit`** is forbidden. Any reshape whose dimensions depend on JAX-traced values (e.g. `length // dynamic_period`) breaks under jit. Either make the schedule static (config kwarg) or use `lax.fori_loop` over a padded representation.
- **Python control flow over traced values** fails (e.g. `if period > 0` on a tracer). Use `lax.cond` / `lax.switch`.
- **`int(tracer)` / `numpy(tracer)`** fails. If you genuinely need Python ints (e.g. for reshape), accept them as static config kwargs.

When porting a PyTorch reference, run the block under `jax.jit(block.apply)` as part of the smoke. JIT-failure means real-training-failure.

### 13.4 General rules

- **Pick one or two reference implementations** (LLaMA decoder, HF BertLayer, official paper repo) and match their structural decisions when uncertain.
- **Default hyperparameter values come from the reference** (e.g. `lora_alpha=8` from HF PEFT). Do not invent defaults.
- **State the reference in the docstring** with URL so future readers can verify. Example: `# Pre-norm RMSNorm — LLaMA / Gemma convention (Zhang & Sennrich 2019, arXiv:1910.07467).`
- **When the backup diverges from the standard**, prefer the standard unless the divergence is documented and justified.

This rule exists to prevent the "clean reinvention" failure mode — where a new implementation looks neat but silently loses standard mechanisms (stabilization, gradient flow, common finetuning entry points).

### 13.5 Fidelity-claim gate

Do not claim an implementation is "faithful to / matches / reproduces the reference" until you have **diffed the implemented step against the named reference line by line** and recorded that diff as the basis.

| Do | Don't |
|----|-------|
| Cite the reference at `file:line` and walk each operation against your code (e.g. "both grads summed into one optimizer step — `<reference>.py:608-644`"). | Claim "coupled / faithful / standard" from the shape of the code or a remembered description. |
| When the mechanism differs (2 steps vs 1 summed step, detach vs through-grad, per-batch vs same-batch), make the diff explicit and reconcile before claiming parity. | Ship a structurally different step under the reference's name — a "coupled" update that is actually two separate updates is a silent algorithm swap. |

This is the §7.1 backup-review discipline applied to fidelity claims: the reference is the spec, and "matches the reference" is a verified diff, not an assertion.

## 14. Derived Keys, Categorical Data, and Plots

Three failure classes that pass every type and shape check while producing a wrong artifact.

| Rule | Detail |
|------|--------|
| **Force explicit ordering for grouped/categorical data** | When grouping by a known fixed set of categories, always pass the complete expected list to the library (`order=`, `reindex(...)`, …) so a missing bucket cannot silently shift the remaining positions. |
| **Verify label ↔ data alignment** | Many plotting libraries place categorical data at integer positions `0..N-1`, not at data values. When setting ticks/labels manually on such an axis, confirm the positions actually map to the intended values. |
| **Sanitize keys derived from other keys** | A log / dashboard / output key built by transforming an existing key inherits its structure — a `/` namespaces a panel section, a suffix carries a unit. Inspect the real values crossing the seam, not the shape: a slash left in a derived key silently mis-groups the panel (nested or empty section) while type and shape checks pass green. Enumerate the actual source keys — every variant, not one example — before naming the derived key. |
