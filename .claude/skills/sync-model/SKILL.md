---
name: sync-model
description: "Resynchronize hand-made changes in src/ back into the model definition (output/generate-<Model>.py, model_sources/, the engine) and into the specification sheet. Starts from a semantic gap analysis between src/ and a throw-away regeneration. Use at the END of a working session on a model, or whenever direct .seml edits (yours or the user's, assisted or manual) must become reproducible again — \"resync the model\", \"fold my edits back into the generator\", \"is src/ still reproducible?\", \"update the spec from the model\"."
---

# Resynchronize src/ → generator + specification

The engine **writes files, it never prunes**, and it only writes what the model definition declares.
So every direct edit under `src/` is one of two silent defects: a **hand-edit** the next generator run
**overwrites**, or a **hand-added object** the generator **cannot rebuild**. Neither shows up in a green
build; both only surface at runtime, or the day someone regenerates. This skill closes that loop: gap
analysis → fold back into the source of truth → prove convergence.

Two sources of truth are involved, and both must end up consistent with `src/`:

| Layer | File | Holds |
|---|---|---|
| Model definition | `output/generate-<Model>.py` (`build_model()` + `CFG`) | everything the engine can derive |
| Hand-authored fragments | `model_sources/*.seml` + `extra_objects` / `extra_workflows` | deliberate designs no generic rule derives |
| Generic capability | `ai-model-development/scripts/seml_engine.py` | patterns EVERY future model should get |
| Functional truth | `output/Specification-<Project>.md` | what the business agreed to, + the decision log §10.3 |

## 0. Confirmation gate — ALWAYS first, before any analysis

The run is expensive (a full regeneration, a property-level diff, then write-backs across two or three
files). It is also worthless if edits are still in flight. So **ask before doing anything**, with
`AskUserQuestion`, and wait:

1. *"Are your manual changes to the model finished — can I resynchronize now?"*
   → `Yes, sync now` (recommended) · `Not finished — I will tell you` · `Gap analysis only, no write-back`
2. Only if the workspace holds several candidates, or nothing obvious: *"Which model definition is the
   source of truth?"* (list the `output/generate-*.py` found).

Never infer the answer from context, never start "just the analysis" to save a round trip — the analysis
*is* the expensive half. If the user picked **Gap analysis only**, stop after §2 and hand over the report.

## 1. Locate the pieces (one line of output, no ceremony)

- generator: `output/generate-<Model>.py` — **if there is none**, the model was authored by hand: say so
  and stop. There is no source of truth to fold back into; offer either to keep authoring in `src/`
  (accepting non-reproducibility) or to write a model definition from scratch as a separate task.
- model tree: `src/<Model>` (exactly one per workspace).
- specification: `output/Specification-*.md` (may legitimately be absent — demo/POV models).
- `model_sources/` — existing fragments and their `README.md` (documented exceptions: do not re-flag them).

## 2. Gap analysis — mechanical, authoritative

```bash
python ai-model-development/scripts/drift-report.py \
  --generator output/generate-<Model>.py \
  --out <scratchpad>/drift.md --json <scratchpad>/drift.json
```

It runs the generator through its **real entry point** into a throw-away tree outside the workspace
(never a second model tree inside it) and compares **semantically** — objects and properties, not bytes.
Exit code: `0` in sync · `1` drift · `2` the analysis could not run (a broken generator: fix it first,
a drift report against a failed run is meaningless).

Read the report from the file (grep per section) rather than pasting it whole — it can be long.
Three classes come back:

| Class | Means | Default reading |
|---|---|---|
| **Only in src** | the generator does not rebuild it | a hand-added object **or** a stale file left by a rename — **a defect until proven cosmetic** |
| **Only in the regeneration** | absent from `src/` | hand-deleted in `src/`, or a capability added to the generator/engine since the last run |
| **Modified** | same object, different properties | a hand-edit the next run overwrites |

Two complementary provenance signals, cheap, worth using: `git status --short src/` + `git diff src/`
when `src/` is tracked (in this template it is gitignored, in a client repo it usually is not), and your
own transcript for the edits made through you in this session. Provenance decides *intent*; the drift
report decides *facts*. Where they disagree, the report wins.

## 3. Classify every finding, then get one approval

Each finding lands in exactly one bucket:

| Bucket | When | Write-back |
|---|---|---|
| **A · model definition** | the engine already has a `cfg` key or model-definition shape for it (an attribute, a label, a widget override, a grant flag, a display card, a LOV value, a lineage view…) | edit `output/generate-<Model>.py` |
| **B · hand-authored fragment** | a deliberate design no generic rule derives: form layout topology, workflow branches, a tuned diagram layout, a deep navigation tree, a cross-referenced named query | move the `.seml` to `model_sources/<name>.seml`, load it via `load_source()` and pass it through `extra_objects` / `extra_workflows`; document the *why* in `model_sources/README.md` |
| **C · engine capability** | the pattern is generic — any future model should get it automatically | teach `seml_engine.py` (new `cfg` key + `_LABELS` entries for any new wording), then the one-line note in the generation prompt / CLAUDE.md per the self-learning policy |
| **D · stale / accidental** | a leftover folder or file from a rename, a scratch object nobody wants, an edit the user disowns | **delete from `src/`** — outward-facing enough to need its own explicit confirmation, listed file by file |

Rules that decide the hard cases:
- **B over A when the shape is a design, not a parameter.** Bending the model definition into
  reproducing a bespoke layout is how generators become unreadable. If it took human judgement to lay
  out, it is a fragment.
- **C over A when the fix is generic.** A trap fixed only in one project's generator is a trap the next
  team re-discovers; that is the self-learning policy, not a nice-to-have.
- **A schema-mandatory property present in `src/` and absent from the regeneration is never cosmetic** —
  it looks exactly like a formatting omission in a diff and comes back as a build failure later.
- **`extra_objects` are written last**, so a fragment legitimately overrides a generated object of the
  same path — that is the supported way to keep a hand-tuned object *and* a clean regeneration.

Present the classification as **one compact table** (finding → bucket → target file), ask for the go
once, and include the D deletions explicitly in that go. Then apply.

## 4. Specification write-back

Only findings that change **functional truth** belong in the spec — a new attribute, a new entity or
reference, a changed validation/enrichment rule, a new LOV value, a matching/survivorship change, a new
screen or view, a changed business label. Pure technical or layout drift does not.

| Finding | Spec section |
|---|---|
| entity / entity type | §3.1 |
| attribute (incl. PII marker) | §3.2 |
| reference, relationship entity | §3.3 / §3.4 |
| reference data, LOV values | §3.5 |
| enricher · validation · matching · survivorship | §4.1 · §4.2 · §4.3 · §4.4 |
| workflow, process | §5.1 / §5.2 |
| business view, transition, action | §6.1 / §6.2 / §6.3 |
| form, collection, display card | §7.1 / §7.2 |
| branding, image library | §8 |
| role, grant | §2 |

For each: update the line **and its status** (a decision taken in the model is `Confirmed`, not an AI
suggestion), then append one entry to the **§10.3 decision log**: date (absolute), what changed, that it
came from a direct model edit resynchronized on that date, and who decided it. Add a §10.4 validation
history line for the batch.

**Do not reopen the hypothesis-validation gate** for these entries: they are decisions the user already
made in the model, not new assumptions. If a finding turns out to be a *question* rather than a decision
(you cannot tell what business rule an edit encodes), do not invent one — list it under §10.1 Open
questions and say so in the final report.

## 5. Prove convergence — the step that makes the sync real

1. Re-run the drift report. Expect **exit 0**. Anything left must be a *documented* exception in
   `model_sources/README.md`; a leftover you cannot explain means the write-back is incomplete.
2. Run the generator for real (it is now idempotent), so `src/` is 100% generated output again.
3. Build — authoritative, per the project's `cfg["seml_version"]`:
   `sem-design data-product build --data-product-folder "src/<Model>"` (2.0) /
   `sem-design dm model build --root-folder "src/<Model>"` (1.11). Green, or fix and repeat.
4. Optional IDE parity: `validate-seml-schemas.js` + `check-xrefs.js`.

If the model is **already deployed**, the resync itself changes nothing on the platform. Say plainly
whether a redeploy is needed (any change to entities/attributes/rules) — and do not deploy: that is its
own confirmed, outward-facing action with its own pre-deploy guardrail.

## 6. Final report (short, factual)

- one line per finding: what moved where (generator / fragment / engine / deleted);
- what the specification now records, and the decision-log entries added;
- the documented exceptions that remain, and why;
- build status, and whether a redeploy is pending;
- one sentence stating the end state: **`src/` is reproducible from `output/generate-<Model>.py` again**
  — or exactly which parts are not, if the user declined a write-back.

## Guardrails

- Nothing is deleted from `src/` without an explicit, file-by-file go.
- The gate in §0 is not optional and is not merged into the work: no analysis before the answer.
- Never hand-edit `out/model.json` or treat it as an input; never read the compiled model to reconstruct
  the definition.
- The throw-away regeneration lives **outside** the workspace — a second model tree inside it breaks
  every build (*multiple model file found in the workspace*).
- Demo-mode edits (global instructions: fast direct `.seml` edits for a demo) are deliberately outside
  this loop. This skill is how they get folded back in later — when the user asks for it.

## Why this matters — the drift is silent

- **Generator ↔ `src/` drift is silent and cumulative — verify a regeneration before trusting it.** Because
  the engine never prunes and only writes what the model definition declares, **anything hand-added to `src/`
  after a run survives invisibly** (a grant file, a workflow, a widget, a fixed icon URL) and **anything
  hand-*edited* is overwritten** on the next run. Neither shows up in a green build, and the damage is
  runtime-only: a hand-added `ModelPrivGrant` disappears, a hand-fixed `label`/`iconUrl` reverts, a
  hand-tuned form field is flattened. Reflex before touching a generator — and the closing step of any
  session that edited `src/` directly (§ *Repeatable process*, step 8): **run the gap analysis**, which is
  now tooled rather than manual:
  ```bash
  python ai-model-development/scripts/drift-report.py --generator output/generate-<Model>.py
  # exit 0 = in sync · 1 = drift · 2 = the analysis could not run (fix the generator first)
  ```
  It regenerates through the generator's **real entry point** into a throw-away tree **outside** the
  workspace and compares **semantically**, classifying every finding as *only in src* (hand-added, or stale
  after a rename — **a defect until proven cosmetic**), *only in the regeneration* (hand-deleted, or a new
  generator/engine capability) or *modified* (a hand-edit the next run overwrites). The full write-back
  loop — classification into model definition / `model_sources/` fragment / engine / deletion, the
  specification + §10.3 decision-log update, and the convergence proof — is the **`/sync-model`** skill
  ([.claude/skills/sync-model/SKILL.md](.claude/skills/sync-model/SKILL.md)), which asks for confirmation
  before spending the run. Two method notes the script embodies (do not re-implement them by hand):
  compare **semantically, not byte for byte** (key order and formatting differ harmlessly; a textual diff
  drowns the findings, and a schema-**mandatory** property missing from the regeneration looks exactly like
  a cosmetic omission until it becomes a build failure), and **drive the regeneration through the real
  `__main__` entry point** (a harness that rebuilds the `engine.generate` argument list by hand silently
  drops `extra_workflows` / `extra_objects` and reports drift that does not exist).
