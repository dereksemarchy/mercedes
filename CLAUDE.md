# CLAUDE.md — Semarchy SDP data product design workshop

A **portable, reusable** guide for designing any Semarchy Data Platform ("SDP") model in this
workshop — from a business need to a `.seml` model that **builds** with the CLI. This file references
only **stable instructions** — not the context files of a given project (client inputs, specification,
target model).

> Convention: each project brings its own context files (see "Per-project layout" below). The
> instructions here do not change from one project to the next.

## LLM cost policy — Opus 5 by default, Fable 5 on justification only (MANDATORY gate)

This workspace defaults to **Opus 5** (`.claude/settings.json` → `"model": "opus"`) to control costs.
**Fable 5 is reserved** for situations that genuinely require it: a **live client demo**, or a
**genuinely complex implementation** (e.g. a full model generation with many interacting constraints).

**Gate — apply at the FIRST response of every session, ALWAYS declare, never stay silent.** Start your
first response with a one-line declaration of the model that is actually serving you, whatever it is —
`Model in use: <model id>.`, written in the language the user is speaking. Read it from your system prompt (*"You are powered by the model
named …"*); if you have any doubt, the authority is the session transcript
(`~/.claude/projects/<project-slug>/<session-id>.jsonl` → `"model"` on the `assistant` entries).
Then:
- **on Fable**, and the user has not already justified it in this session → remind the policy in one
  line, then **ask the user to confirm** (e.g. via AskUserQuestion): *stay on Fable (justified case:
  live demo / complex implementation)* or *switch with `/model opus`* — and **wait for the answer
  before starting the substantive work**. **Never self-justify the case on the user's behalf**
  ("this looks like a complex generation, so I proceed" is exactly the defect — incident 2026-08-20:
  the agent declared Fable correctly, then decided alone that the task qualified and continued). The
  choice belongs to the user; asking one confirmation question is not "blocking the work". Once the
  user has confirmed either way, do not raise it again for the rest of the session;
- **on Opus or any other model** → the declaration line alone, no reminder, no comment.

**Why the declaration is unconditional** (incident 2026-08-20, session `b022a810`): the previous wording
only spoke up *on Fable*, which made the gate blind in the opposite direction. The user had deliberately
selected **Fable** in the model menu, the session was nonetheless served by **Opus 5** from its very first
assistant turn (all turns `"model":"claude-opus-5"` in the transcript, zero trace of Fable), and nothing
signalled the discrepancy — the silent-on-Opus rule guaranteed it stayed invisible. Harness-side settings
can move the session off the chosen model on their own: **"Switch models when a message is flagged"** and
the composer's **Auto** mode both do. Declaring the serving model every session surfaces such a switch
immediately, in both directions.

**The system-prompt model line can go STALE after an in-session `/model` switch** (incident
2026-08-21, session `57a54dd8`): the user ran `/model claude-fable-5` mid-session, every subsequent
transcript entry was served by Fable, yet the agent's system prompt kept saying *"You are powered by
the model named Opus 5"* — so the (correctly unconditional) declaration was WRONG and contradicted the
user's model menu, which was right. Rule: the declaration still comes from the system prompt, but **if a
`/model` switch happened in the session, or the user challenges the declared model, verify against the
transcript before repeating the claim** (`grep '"model"' <session>.jsonl` on the assistant entries —
count per model and look at the switch timestamps). Never insist on the system-prompt value against a
transcript that disagrees.

**You cannot switch the model yourself** — only the user can, via the model menu, `/model` or
`claude --model`. So a mismatch between the menu and the declared model is a harness/settings matter to
raise with the user, never something you caused or can fix; never claim to have changed the model.

---

## Repeatable process (need → model → build)

1. **Scope / specify** the need using the MDM methodology (§ Stable instructions). Produce a
   **specification sheet** in `output/` from the provided template.
2. **Validate the AI hypotheses — MANDATORY gate, never skipped.** The specification is not `Validated`
   and **nothing is generated** while a line remains at `AI suggestion — pending validation`. Protocol:
   [methodology §9.1](ai-model-development/methodology/mdm-semarchy-discovery-methodology.md) — count the
   pending suggestions, then **≤ 5 → one batch list** with recommended defaults; **> 5 → interactive, one
   suggestion at a time, waiting for each answer**. Always submit them **grouped by coherent theme** in
   dependency order (structuring assumptions → sources → roles → *one group per entity* → relationships →
   reference data → one group per data-quality family → processes → views/screens/dashboards), with per item
   *what / why / recommended default / impact if wrong / alternatives*. Flip each line and append to the
   decision log (§10.3 of the template) as you go. Per-group bulk accept is allowed; a global "accept
   everything" is not (log it as *accepted without item-level review* if insisted on).
3. **Pick a reference model** that fits (§ Reference models) as a `.seml` style guide.
4. **Generate the `.seml`** into `src/<ModelName>/` following the generation prompt and the
   conventions below (the example generator is the starting point).
5. **Build and iterate** until `Build successful` — CLI errors are authoritative. The command
   depends on the project's `cfg["seml_version"]` (§ Stable conventions — *SEML version dialects*):
   ```bash
   sem-design data-product build --data-product-folder "src/<ModelName>"   # seml_version 2.0
   sem-design dm model build --root-folder "src/<ModelName>"               # seml_version 1.11 (legacy)
   ```
6. **Validate like the IDE** (schemas + cross-references) — optional but recommended.
7. **Deploy & load — only on explicit request, writes confirmed one by one.** Before the deploy,
   **control the target data location with the user** (new DL on a free datasource / upgrade of the
   same data product / conflict with another one — PLATFORM-CLI.md § Pre-deploy guardrail). Order
   (validated end to end): deploy the model (per-version deploy verb, § `sem-design` CLI — never
   while an integration batch runs), import the image library, create roles / API keys, deploy the
   REST clients (`sem dm admin
   rest-clients set`), then load the demo data (dry-run first; three-phase loader). Before the FIRST
   write, **ask which API key to use** — consent to an action is not consent to a credential
   (PLATFORM-CLI.md § Authorization policy). Keep a per-project runbook in `resources/DEPLOYMENT.md`.
8. **Resynchronize the sources of truth — the closing step whenever `src/` was touched directly**
   (by you or by the user, assisted or by hand). Direct `.seml` edits are either **overwritten** by the
   next generation or **unrebuildable** by it, and a green build hides both. So at the end of a session
   on a model: run the **`/sync-model`** skill — it **asks first whether the manual changes are finished
   and whether it may synchronize** (the run is expensive and worthless while edits are in flight),
   then starts from the mechanical gap analysis
   (`python ai-model-development/scripts/drift-report.py --generator output/generate-<Model>.py`) and
   folds each finding back into the right layer: model definition (`output/generate-<Model>.py`) /
   hand-authored fragment (`model_sources/` + `extra_objects`) / **engine** when the pattern is generic /
   deletion when it is stale — plus the **specification sheet and its §10.3 decision log** for anything
   that changed functional truth. Ends by proving convergence: drift report back to zero, then a green
   build. Never run it without the confirmation, and never on a generator that does not run. It is a
   **user-triggered** step: after direct `src/` work, offer it in one line and let the user decide —
   fast demo edits (global instructions § *Direct model edits*) deliberately stay outside the loop, and
   this is how they come back in later, if and when the user asks.
9. **Self-learn** — feed anything new you discovered back into the tooling (see next section) before
   considering the task done.

---

## Self-learning — keep the generation tooling current (STANDING PRINCIPLE)

This ecosystem is **self-improving**: every project teaches it something. Treat the capture of new
knowledge as part of *finishing* a task, not an optional extra. **Whenever you encounter a new object
type, a new attribute, a working (or failing) configuration, a reserved value/enum, a build/schema
constraint, a platform/runtime behaviour, or a "builds green but breaks at deploy/runtime" gotcha that
is not already documented — consolidate it, generically, into the generation tooling in the same
session, before wrapping up.**

- **Authority order (verify, don't assume):** the **CLI `build` + the embedded draft-07 JSON schemas**
  and the **running platform** (deploy / loads / render) are authoritative — above your prior
  assumptions *and above the online docs* (docs describe intent; they were wrong here about
  `HasSuggestedMerge`/`IsConfirmed` being `0/1` when the column is boolean, and about lineage not being
  a transition). When something surprises you, confirm it against the schema/CLI/runtime and record
  **what was validated and how**.
- **Where to write it — pick the most automatic target that fits, and usually more than one:**
  | Learning | Update |
  |---|---|
  | A pattern future models should emit **automatically** | [`ai-model-development/scripts/seml_engine.py`](ai-model-development/scripts/seml_engine.py) — teach the generator so it never has to be hand-added again (**preferred**) |
  | A generation rule / how-to / gotcha | [`ai-model-development/prompt/dxp-model-development-prompt.md`](ai-model-development/prompt/dxp-model-development-prompt.md) |
  | A durable `.seml` finding about the DATA MODEL layer | **`.claude/skills/seml-model/SKILL.md`** |
| A durable `.seml` finding about the UI layer | **`.claude/skills/seml-ui/SKILL.md`** |
| A SemQL expression rule | **`.claude/skills/semql-suggest/SKILL.md`** |
| A deploy / load / runtime-diagnosis behaviour | **`.claude/skills/platform-ops/SKILL.md`** |
| A rule about how the WORKSHOP itself works (dialects, one model per workspace, where files live) | **this file** → § *Workshop rules that always apply* |
  | Platform / CLI behaviour (deploy, data loads, roles, REST clients, batches…) | [`ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md`](ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md) |
- **Where a finding does NOT go: this file.** `CLAUDE.md` is reloaded in full at every session, so a
  bullet added here is paid for in every context window of every future session, whether or not the
  subject ever comes up. **The only thing a new finding earns here is ONE INDEX LINE, and only if the
  defect is SILENT** — build green (or no error at all) and the runtime wrong. Anything a failure
  message would have told you belongs in its skill and nowhere else. Applied on 2026-08-27, this rule
  took the file from ~33 000 to ~10 000 tokens; without it, it grows back.
- **How to write it:** **generic** — no client/project data, names, or ids; state the **correct `.seml`
  (or command) setup**, the **symptom it fixes**, and mark it **validated** (build and/or runtime).
  Prefer teaching the engine *and* leaving a human-readable note in the prompt / this file.
- **Deduplicate & self-correct:** update the existing entry rather than adding a duplicate; **delete or
  fix** any guidance later proven wrong (do not leave a stale note standing). A guideline that was
  authoritative when written can be invalidated by a new schema/CLI/platform finding — the new finding wins.
- **Packaging note:** these four artifacts are what ships to other teams. Keeping them current *is* the
  product — a fix that lives only in one project's `src/` but not in the tooling is a fix that will be
  re-discovered the hard way by the next team.

---

## Stable instructions (reusable across projects)

Everything reusable is grouped under **`ai-model-development/`** (hub):
`prompt/` · `methodology/` · `use-case-prompts/` · `reference-models/` · `scripts/` · `cli/` · `tools/`.
The hub's **technical architecture** (workflow, component contracts, invariants, self-learning loop —
the Product & R&D transfer document) is
[ai-model-development/ARCHITECTURE.md](ai-model-development/ARCHITECTURE.md); keep it current when a
component's contract changes.

### Semarchy model generation — `ai-model-development/prompt/`
- [ai-model-development/prompt/dxp-model-development-prompt.md](ai-model-development/prompt/dxp-model-development-prompt.md)
  — rules for generating a Design XP model: entities/types, references, enrichers, validations,
  forms, collection/business views, steppers, duplicate managers, survivorship, workflows,
  application navigation, diagram, grants, build. **Read this first for any generation.**
- [ai-model-development/prompt/README.md](ai-model-development/prompt/README.md) — prompt execution guide.
- [ai-model-development/use-case-prompts/](ai-model-development/use-case-prompts/) — reusable business
  context examples (bookstore, clinical-trial, sap-product-360).

### MDM scoping / specification methodology — `ai-model-development/methodology/`
- [ai-model-development/methodology/MDM-ProjectScoping-MultiAgent.md](ai-model-development/methodology/MDM-ProjectScoping-MultiAgent.md)
  — multi-agent system: Module A (strategic scoping) + Module B (Semarchy discovery & specification).
- [ai-model-development/methodology/mdm-semarchy-discovery-methodology.md](ai-model-development/methodology/mdm-semarchy-discovery-methodology.md)
  — 10 discovery phases, the **Semarchy glossary** to use strictly, per-phase questions, and known
  limitations not to present as settled.
- [ai-model-development/methodology/semarchy-implementation-specification-template.md](ai-model-development/methodology/semarchy-implementation-specification-template.md)
  — mandatory structure of the specification sheet (sections 0–10, a status on every line:
  Confirmed / AI suggestion — to validate / Open).

### Agent instructions applied to `.seml` files
- The **authoritative** rules for editing `src/**/*.seml` are consolidated **in this file** (the
  online documentation sources and CLI policy below, and § *Stable conventions & decisions*) and in
  the generation prompt ([dxp-model-development-prompt.md](ai-model-development/prompt/dxp-model-development-prompt.md)).
  Honor them whenever editing `.seml`.

### Online documentation (authoritative)
- **<https://docs.semarchy.com/llms.txt>** — LLM-oriented index of the full documentation; use it to
  navigate the docs and to find the **correct `.seml` attributes** for each object type.
- **<https://github.com/semarchy/sdp-accelerators>** — example models (entities, relationships,
  applications, forms, collections, steppers). **Browse on the web only — do not clone.**
- **<https://support.semarchy.com/support/solutions/43000375116>** — knowledge base for specific
  troubleshooting.
- **`sem-design` CLI — authorized commands: build, deploy, auth login.** The verbs depend on the CLI
  version, which PINS the `.seml` format (§ Stable conventions — *SEML version dialects*). **Build**
  (read-only compile — run freely, it is the authority on the model): `sem-design data-product build
  --data-product-folder <root>` (2.0) / `sem-design dm model build --root-folder <root>` (1.11).
  **Deploy** (outward-facing write — confirm the action, the auth method AND the **target data
  location** first: inspect it and walk the user through new-DL / upgrade-of-same-product /
  conflict-with-another-product, PLATFORM-CLI.md § Authorization policy + § Pre-deploy guardrail;
  never deploy while an integration batch is running): `sem-design
  data-product deploy` (2.0: `--data-product-folder … --instance-url https://<tenant>` — no `/dm`
  suffix — **`--overwrite-deployment-url https://<tenant>/dpc` — ALWAYS set it** (runtime-validated):
  during the current transition the CLI otherwise falls back to the legacy `/dm` deployment route,
  which still works but **disturbs the target `dpc` component**; a future platform well-known page
  will route the CLI transparently, until then the overwrite is required (accepts the base `…/dpc`
  form or the full endpoint URL). **504 gotcha (runtime-validated): the `/dpc` deploy may return
  `Gateway Timeout` while the deployment actually SUCCEEDED asynchronously** — before any retry,
  check `sem dm data-locations get <DL>`: `deploymentDate` advanced + `DL_READY` = success (data
  preserved on upgrade). Observed on **10 consecutive upgrade deploys** of one data product, so treat the
  504 as the normal outcome of an upgrade, not an incident. **Check the STATUS, not just the date, and
  WAIT**: an immediate `get` can return `DEPLOYING_ME_AND_JOBS` with the OLD `deploymentDate` —
  the deploy is still running server-side and a retry would collide with it. Poll until ready:
  `until sem dm data-locations get <DL> --json | grep -q '"status": "DL_READY"'; do sleep 15; done`
  — `--data-source … --data-location …`; auth = stored session or
  `--client-id`/`--client-secret`, **no API key**) / `sem-design dm data-location deploy`
  (1.11 flags, `--api-key` deprecated in 2.0).
  **`sem-design auth login --url <tenant>`** (2.0, runtime-validated) starts a browser device flow:
  run it in the background, relay the printed `…/device?user_code=…` URL to the developer, wait for
  approval — the session (shared `~/.sem/config.yaml`) then authenticates deploys AND the REST
  loads/query APIs (`Authorization: Bearer` via `sem auth token`) with no secret collected. Do not
  run other `sem-design` commands.
- **`sem` Platform CLI is authorized** for platform-side administration (concepts the model build does
  not handle: REST clients, roles, users, role mapping, API keys, data sources…). **Read/inspect
  commands (`list`, `get`, `status`, `rest-clients test`) may be run freely; write commands (`set`,
  `set-all`, `delete`, `assign`/`unassign`, `create`, `import`, `deploy`) are outward-facing and
  require explicit user confirmation first.** **Credentials are a user-supplied prerequisite: never
  search the filesystem, environment, CLI config stores or other workspaces for an API key or secret —
  ask the user for the key before any deployment/load, and stop at generation if none is provided.**
  Full authorization policy, conventions (no trailing
  slash on `--url`, etc.) and concept catalogue:
  [ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md](ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md).

### Workshop start-up
- [TUTORIAL.md](ai-model-development/TUTORIAL.md) + [tutorial/](ai-model-development/tutorial/) — team tutorial (installation, auth & security,
  workspace tour, manual authoring, fast generation path, prompt recipes, platform-side boundary).

## Reference models (`.seml` style guides)

**Primary reference — use by default:**
[**`ai-model-development/reference-models/BusinessPartner360/`**](ai-model-development/reference-models/BusinessPartner360/README.md)
— a complete, build-validated Supplier / Business-Partner 360° model **bundled with the tool** that
exercises the broadest set of configurations (all three entity types; match & merge + full
survivorship; recursive hierarchies; record-lineage & history transitions; `Binary` attributes; REST
client enrichers; steppers with child creation; workflows; data-stewardship navigation; a demo dataset
+ two-phase loader). **Read it first** as the style guide for any generation — its
[README](ai-model-development/reference-models/BusinessPartner360/README.md) is a pattern catalogue with
file pointers. Build it standalone to inspect it:
`sem-design dm model build --root-folder "ai-model-development/reference-models/BusinessPartner360/src/BusinessPartner360"`
(**HierarchyHub is in the 2.0 dialect** since 2026-08-26 — the installed CLI is 2.0 and rejects a 1.11
tree, so build it with `sem-design data-product build --data-product-folder … --force`, the force being
the known named-query validator defect. **BusinessPartner360 is still 1.11**; regenerating either in the
other dialect is one `seml_version` flip of its generator).

**Complementary references — only when a pattern is not covered by BusinessPartner360:** browse the
GitHub accelerators **[`semarchy/sdp-accelerators`](https://github.com/semarchy/sdp-accelerators)**
(web only — do not clone), e.g. `CorporateFinance` / `FinanceKYC` (match & merge, lineage), `Healthcare`,
`ManufacturingAssets` (BASIC-entity UI), and others for sector patterns.

To confirm an object's exact `.seml` attributes, cross-check `docs.semarchy.com/llms.txt`, the bundled
reference model, the accelerators above, and (offline) the schemas embedded in the installed CLI (see
Tooling).

## Tooling

- **`sem-design` CLI** (installed, on the PATH). The `build` command is authoritative (schema +
  validation + reference resolution). **A single model under `src/`** — see § *Stable conventions*
  (structuring constraint shared by the CLI and the IDE extension). The bundle `…/@semarchy/sem-design/out/cli.js` also contains
  the **seml↔json mappers** and the **42 draft-07 JSON schemas** (useful to learn the exact fields of
  an object type or to unblock a schema error). A copy of the install archives (Design CLI + Platform
  CLI, all OS) is provided under
  [ai-model-development/cli/](ai-model-development/cli/) (`DesignCLI/semarchy-sem-design-1.11.0.tgz`, `PlatformCLI/`).
- **`sem` Platform CLI** — talks to a **running platform** to deploy/administer runtime artifacts that
  the model build does *not* handle (REST clients, roles, users, role mapping, API keys, data sources,
  image libraries, jobs…). `--url <base>` **without a trailing slash** (a trailing `/` produces
  `//dm/...` → `404`); auth with `--api-key`/`$SEM_API_KEY` or client credentials. Reads/`get` are
  safe; **writes (`set`/`delete`/`assign`/…) are outward-facing — confirm with the user first**.
  **Operating guide** (authorization policy, conventions, per-concept command catalogue):
  [ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md](ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md)
  (binary `sem.exe`; run `sem <command> --help` for the command tree).
- **SemQL model awareness for the agent — `ai-model-development/tools/semql-mcp-bridge/`** (validated
  on extension 1.11.0).
  The Semarchy IDE extension contributes two **Language Model Tools** (`semql_list_entities`,
  `semql_get_entity_context`: the attributes, navigation, entry-point type and context aliases valid at
  a given `componentType`+`property`) **and** a chat participant `@semql /suggest`. Only the *tools* are
  reachable programmatically — `vscode.lm.tools` / `vscode.lm.invokeTool` are public APIs and the
  extension registers the tools **unconditionally**, whereas a **chat participant is invocable only by a
  chat host (Copilot Chat) and has no callable API** (don't look for a workaround). The bridge is a
  dependency-free VS Code extension that serves MCP over loopback and forwards to `vscode.lm`; install
  with `install.ps1` (Windows) or `install.sh` (macOS/Linux) from that folder, then
  `claude mcp add --transport http --scope local semql http://127.0.0.1:39847/mcp`. It requires VS Code
  open on the workspace and indexes **only the workspace's own `src/` model** (empty `src/` →
  `{"entities":[]}`; the bundled reference model is not indexed — open it as its own workspace). The
  `/suggest` behaviour itself is just a prompt over those tools: it lives in the **`semql-suggest` skill**
  (`.claude/skills/semql-suggest/`), which carries the SemQL syntax rules **plus this workshop's
  runtime-validated corrections** to them.
- **Reusable scripts** — `ai-model-development/scripts/`:
  | Script | Role | Usage |
  |---|---|---|
  | [scripts/seml_engine.py](ai-model-development/scripts/seml_engine.py) | **Generic emission engine**. `generate(model, cfg, extra_workflows=…)` turns a model definition + config into the full `.seml` tree (entities, references, LOVs, UI, diagram, app, model jobs, workflows, publishers, grants). No project data inside. Every built-in label comes from the `_LABELS` catalogue in the **design language** (`cfg["design_language"]`, mandatory); `check_language(lang, paths)` reports mixed-language labels and non-ASCII technical names. **`cfg["seml_version"]` ("1.11" \| "2.0") is mandatory too** — it selects the `_DIALECTS` emission entry (§ Stable conventions — *SEML version dialects*). | imported by a model-definition file |
  | [scripts/generate-model.example.py](ai-model-development/scripts/generate-model.example.py) | **Example model definition** (DemoPOV). Holds only `build_model()` + `CFG` + any project-specific workflow, then calls the engine. **Copy it to `output/generate-<Model>.py` to start a new project** — a project's model definition is project content and must not live under `ai-model-development/`, which ships to other teams. Requires `pyyaml`. | `python output/generate-<Model>.py` |
  | [scripts/validate-seml-schemas.js](ai-model-development/scripts/validate-seml-schemas.js) | Validates each `.seml` against the extension's schemas (what the IDE sees). Requires `ajv@6`. | `node ai-model-development/scripts/validate-seml-schemas.js src/<Model>` |
  | [scripts/check-xrefs.js](ai-model-development/scripts/check-xrefs.js) | Checks that every qualified-name reference resolves. | `node ai-model-development/scripts/check-xrefs.js src/<Model>` |
  | [scripts/drift-report.py](ai-model-development/scripts/drift-report.py) | **Generator ↔ `src/` gap analysis** — regenerates the model definition into a throw-away tree outside the workspace (real `__main__` entry point, `cfg["src"]` redirected) and compares it **semantically** with the live tree: *only in src* / *only in the regeneration* / *modified*, property by property. Exit `0` in sync, `1` drift, `2` analysis failed. `--out`/`--json` for the report, `--keep` to inspect the regeneration. Mechanical half of the **`/sync-model`** resynchronization skill. | `python ai-model-development/scripts/drift-report.py --generator output/generate-<Model>.py` |

  One-off prerequisites: `pip install pyyaml` and, for schema validation, `npm install` run **inside
  [`ai-model-development/scripts/`](ai-model-development/scripts/)** — that folder owns the tooling's
  `package.json`/`package-lock.json` (currently `ajv@6`), and Node resolves modules from the *script's*
  directory upward, so the validators work from any cwd. The `sem-design` CLI must be installed (the
  scripts locate it automatically, or via the `SEM_DESIGN_PKG` environment variable).
  **Guardrail — tool dependencies live ONLY under `ai-model-development/`** (validated after a root
  install had to be undone): never run `npm i`/`pip install -t` or create a `package.json` at the
  **workspace root** or in the project folders (`input/`, `output/`, `src/`, `resources/`) — the root
  belongs to the client project, a stray root `package.json` makes Node resolution cwd-dependent, and
  the files pollute every project cloned from the template. A new JS/Python dependency for a shared
  script goes in `ai-model-development/scripts/package.json` (committed lockfile, gitignored
  `node_modules/`); a truly new tool gets its own subfolder under `ai-model-development/` (like
  `tools/semql-mcp-bridge/`), never the root.

---

## Per-project layout (generic)

| Folder | Content (project-specific, not listed here) |
|---|---|
| `input/` | Elements received from the client (specs, data exports/samples, workshop notes). |
| `output/` | Everything a generation run produces **for this project**: the **specification sheet**, the **generation brief** (`GenerationBrief-<Project>.md`), and the **model-definition script** `generate-<Model>.py` that emits `src/`. The script is hand-written **source** — the source of truth for the model, not a derived artifact — so never treat this folder as scratch space to be cleaned. It imports the shared engine from `ai-model-development/scripts/`. |
| `src/<ModelName>/` | The generated Semarchy model / data product (`.seml`) — **exactly one per workspace** (product constraint: CLI + IDE extension). Standard folders: `entities/`, `references/`, `list_of_values/`, `publishers/`, `workflows/`, `model_diagrams/`, `applications/`, `complex_types/`, `named_queries/`, plus per dialect: `certification_jobs/` + `data_product_priv_grants/` + `dynamic_variables/` (2.0) or `model_jobs/` + `model_priv_grants/` + `model_variables/` (1.11). |
| `resources/image-libraries/<shortname>/` | Avatar / logo / cover and application icons. |
| `resources/RestClient/` | Deployable platform **REST client** JSON (one file per client, filename = client name = the enricher `pluginId`). Deployed with the Platform CLI, not with the model build. |
| `resources/platform-administration/` | Other platform admin payloads (API keys, data sources…). |
| `resources/data/` | Demo/seed **datasets** (one JSON per entity, numbered = load order) + a loader for the REST loads API. Loaded into a data location, not built. |

## Stable conventions & decisions (reusable knowledge)

The detailed `.seml` findings — 86 of them, build- or runtime-validated — live in **task-scoped
skills**, loaded when the task calls for them instead of sitting in every context window:

| Skill | Invoke it before |
|---|---|
| **`/seml-model`** | creating or editing an entity, attribute, reference, matcher, survivorship rule, enricher, validation, unique key, LOV, grant, retention policy or workflow |
| **`/seml-ui`** | touching a form, business view, transition, collection, display card, stepper, action, navigation, diagram or branding image |
| **`/semql-suggest`** | writing or debugging any SemQL expression |
| **`/platform-ops`** | any deploy or data load, and any batch / queue / empty-value diagnosis |
| **`/sync-model`** | folding direct `src/` edits back into the generator and the specification |

**This is a rule, not a suggestion: invoke the matching skill BEFORE writing, not after a failure.**
Most of what they carry is invisible to the build — a green build proves nothing about the runtime.

### Silent traps — the ones you would never think to look up

Each of these produces a **green build and a wrong runtime**. The one-liner is here so the trap is
known to exist; the skill carries the diagnosis and the fix.

- **Generator ↔ `src/` drift is silent and cumulative — a green build proves nothing about reproducibility** → `/sync-model`
- **The matcher's merge threshold is EXCLUSIVE — a rule scoring exactly it does NOT merge** → `/seml-model`
- **A filtered browse action on a HIERARCHICAL business view lists only the tree's ROOTS** → `/seml-ui`
- **Model-level role flags (`integrationWsRole`…) vanish on regeneration unless declared — loads then 403** → `/seml-model`
- **`EnricherCache.pluginIdOrRestClientName` is never checked at build — a typo means an empty cache** → `/seml-model`
- **`--force` buys a build, never a runtime: a rejected stepper is DROPPED and its action disappears** → `/platform-ops`
- **A POST_CONSO enricher on a BASIC entity NEVER runs — values stay empty** → `/seml-model`
- **A PRE_CONSO validation on a matched entity cannot bind its own PK — it fails for every record** → `/seml-model`
- **Folding DELETES the letters that do not decompose (Æ, Ø, Å, ß) — transliterate first** → `/semql-suggest`
- **`CUSTOM_RANKING … DESC` ranks NULLS FIRST — a null-valued master beats every real value** → `/seml-model`
- **A child's PRE_CONSO enricher navigating UP computes empty when parent and child share a batch** → `/platform-ops`
- **A POST_CONSO enricher walking CHILDREN computes empty on the first load** → `/platform-ops`
- **A persist MERGES, positional ids renumber on insert, and deleting goldens leaves the sources** → `/platform-ops`
- **An enricher that cannot evaluate SUSPENDS the queue — every later entity is silently empty** → `/platform-ops`
- **The default certification order is wrong as soon as an entity looks up reference data** → `/platform-ops`
- **A non-String display text builds green and crashes the collection on open** → `/seml-ui`
- **Record lineage applies to ID_MATCHED too — restrict it to fuzzy and the tabs vanish silently** → `/seml-ui`
- **SVG gradients render BLACK and every image slot centre-crops** → `/seml-ui`

### Workshop rules that always apply

- **SEML version dialects — "models → data products" (extension/CLI ≥ 2.0, build- & runtime-validated).**
  The installed CLI/extension **pins the `.seml` format**: a 1.11 CLI rejects a 2.0 tree (`no model
  found`) and a 2.0 CLI rejects a 1.11 tree (`no data product profile found`) — even through its
  legacy `dm` verbs. The whole 1.11→2.0 delta is **nominal** (all 39 embedded schemas otherwise
  byte-identical; the compiled artifact differs only by QN segments, same `dataHubModelSchemaVersion`):
  | | v1.11 | v2.0 |
  |---|---|---|
  | Root file / `_type` | `<Pkg>.Model.seml` / `Model` | `<Pkg>.DataProductProfile.seml` / `DataProductProfile` |
  | Jobs | `model_jobs/` / `ModelJob` | `certification_jobs/` / `CertificationJob` |
  | Grants | `model_priv_grants/` / `ModelPrivGrant` | `data_product_priv_grants/` / `DataProductPrivGrant` |
  | Variables | `model_variables/` / `ModelVariable` | `dynamic_variables/` / `DynamicVariable` |
  **Unchanged**: `ModelDiagram`/`model_diagrams/`, `EntityPrivGrant`, inner `ModelJobParam`/`ModelJobTask`/
  `modelTasks`, all 153 other types. The engine emits per **`cfg["seml_version"]`** (**mandatory**, like
  `design_language` — it raises rather than guess) from its `_DIALECTS` table, and **transposes
  `extra_workflows`/`extra_objects`** (QN folder segments, the four `_type`s, filename suffixes) so
  `model_sources/` fragments stay **dialect-neutral** — never duplicate a fragment or a reference-model
  tree per version. Migrating an existing `src/` = the extension command *Migrate the elements…*
  (rewrites `_type`/filenames/folders only) **plus** a global find/replace of the three folder segments
  (`_package` + cross-refs like `integrationJob`) — the build errors name the exact replacement.
  Retiring a version = deleting its `_DIALECTS` entry and the legacy mentions in the docs.
- **One model per workspace — structuring constraint of the product, not a CLI quirk.** The workshop, the
  **Semarchy IDE extension** and the **CLIs** all handle a **single** model under `src/`. A second model there
  breaks the build (*"multiple model file found in the workspace (type: VALIDATION)"*, for either model — the
  build resolves the workspace from the parent of `--root-folder`, so `--root-folder` does not isolate it) and
  the extension. A scope covering several independent models = several workspaces; plan it at project setup.
  (This is why the bundled reference model sits under its own parent, `reference-models/<Name>/src/<Name>/`,
  and can be built without disturbing the project's `src/`.) **Corollary for the reference model's own
  generator: its `SRC` must point at that own tree, never at `<workspace>/src/`.** Running a generator whose
  `SRC` is `<workspace>/src/<Pkg>` while the project's model is already there drops a second model into the
  workspace and breaks **both** builds (*"multiple model file found in the workspace"*) — verified the hard way.
- **Design language — one model, one language for the user-facing layer** (structuring assumption,
  validated at specification time in **§0.1/§0.2** of the spec template, *before* generating). Two layers,
  two rules: **technical names** (`_name`, `physicalName`/`physicalTableName`/`physicalColName`,
  `toRolePhysicalName`, `roleName`, LOV codes, SemQL) are **always English/ASCII/unaccented** — they become
  SQL identifiers and REST payload keys; **user-facing labels** (`label`, `pluralLabel`, `description`,
  `documentation`, `validationLabel`, LOV value labels, form/section/tab, action, workflow, application and
  folder labels) are **100% in the chosen design language** — accented labels build fine (validated).
  Mixing the two languages is the most-reported generation defect and **the CLI build stays green on it**,
  so nothing catches it downstream. `cfg["design_language"]` is **mandatory** in the engine (it raises
  rather than guess); all its built-in labels come from the `_LABELS` catalogue, overridable per project
  with `cfg["labels"]`, and `generate()` ends with a heuristic `check_language()` report (MDM loanwords —
  *data*, *record*, *master*, *golden*, *steward* — deliberately not treated as English markers, so
  "Data Steward" in a French model is not flagged). When lifting a snippet from BusinessPartner360 or an
  accelerator (both English), **translate the labels — never copy them**; client-imposed glossary terms
  are a stated exception, not a mix.
- **Source of truth = the specification sheet**, and only once its **hypothesis validation gate** is closed
  (§ *Repeatable process*, step 2 — `Overall status = Validated`, zero `AI suggestion — pending validation`).
  Define the data model explicitly in the model definition file (`build_model()`), do not read a compiled
  `model.json`.
- **Never hand-write `out/model.json`** — it is an artifact produced by `build`.
- **The generator writes files, it never prunes** — when an entity/reference is renamed or removed from the
  model definition, the engine emits the new files but leaves the old ones in place; the stale `.seml`
  (e.g. `entities/<Old>/`, `references/<Old>.Reference.seml`) then fails the build with `Unknown reference`
  or lingers as a ghost object. After removing/renaming an entity or reference, **delete its stale folder/file
  under `src/`** before rebuilding.
- **Project content never lives under `ai-model-development/`.** That folder *is* the product shipped to other
  teams (prompt, methodology, engine, reference model); anything specific to one client belongs in the project
  folders — `input/`, `output/`, `src/`, `resources/`. Concretely: a project's **model-definition script goes to
  `output/generate-<Model>.py`**, not to `scripts/` next to the engine. Two consequences to accept knowingly:
  `output/` is gitignored in this template, so the project's source of truth is **not versioned in the workshop
  repo** (version it in the client's own repo); and the script must reach the engine explicitly, e.g.
  `sys.path.insert(0, os.path.join(ROOT, "ai-model-development", "scripts"))`. The **bundled reference model's**
  own generator is the exception that proves the rule: it is part of the product, so it stays under
  `ai-model-development/`, and it must write into its own `reference-models/<Name>/src/` tree.
- **Hand-authored `.seml` belongs OUTSIDE `src/` — the `model_sources/` pattern.** Some objects are a deliberate
  design that no generic rule derives: a workflow's branch topology, a two-column form with an embedded
  collection, a cross-referenced named query, a size-tuned diagram layout, a deep navigation tree. Writing them
  by hand *inside* `src/` guarantees the silent drift indexed above. Keep them as `.seml` fragments in a sibling
  `model_sources/` folder, have the model definition `load_source()` them and pass them through
  `extra_workflows=` / `extra_objects` (extra objects are written **last**, so they also override a generated
  object of the same path). The generator then still produces the complete model from a clean tree, the YAML
  stays readable and diffable as YAML, and `src/` remains 100% generated output. Document the *why* per file in
  `model_sources/README.md`.
- **Strict Semarchy vocabulary**: Fuzzy-matched / Basic / ID-matched entity, Role, LOV,
  Survivorship, Match & Merge, Business View (root entity + transitions), Action Set vs Workflow.
- **SaaS scope rule for capability assessments**: Semarchy SaaS blocks database access — any pattern
  needing custom SQL (recursive `SqlFunction`, DB views, triggers) counts as **absent in SaaS** and may
  only be cited as a *self-hosted* workaround. Model-level patterns (enrichers incl. `FID_<role>`
  targets, validations, association entities, loads API) are SaaS-compatible.

## When the IDE reports errors but the build is green

The `build` (authoritative) and the schema validation can diverge from a stale extension display.
Reflexes: `Developer: Reload Window`, re-run `build`, then run `validate-seml-schemas.js` and
`check-xrefs.js` on the model folder. Ask for the **exact** error text (file + line) before fixing.
