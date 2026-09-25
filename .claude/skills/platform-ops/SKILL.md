---
name: platform-ops
description: "Everything that happens once the model leaves the build: deploying a data product, loading data through the REST loads API, and diagnosing what went wrong. Invoke BEFORE any deploy or data load, and whenever a batch suspends, a queue stalls, a deploy returns 504 or 500, computed values come back empty after a load, or records are rejected en masse. The authorization policy and the command catalogue live in ai-model-development/cli/PlatformCLI/PLATFORM-CLI.md; this skill carries the runtime-measured behaviours that document does not."
---

# Deploying, loading and diagnosing a Semarchy data location

Every finding here was measured on a running platform. Deploys and loads are outward-facing actions: confirm the action, the credential AND the target data location with the user first (PLATFORM-CLI.md § Authorization policy).

The model-side rules are `/seml-model` and `/seml-ui`.

- **`--force` buys a BUILD, never a runtime behaviour — and a stepper the build rejects is DROPPED
  silently, taking its action out of the application** (runtime-validated, the clearest demonstration of
  what forcing really does). Measured: `canFinishSubLevel: false` on a stepper's ROOT step, tried as a way
  to offer a read-only journey with no *Finish* and therefore no save. The CLI refuses it (*"The step <X>
  needs canFinishSubLevel to be 'true' because it is the root step"*); `--force` builds and deploys it
  cleanly, the data location returns `DL_READY` — and **the action simply no longer appears** in the app.
  No error, no warning, nothing in the logs to look at. So distinguish the two kinds of forced validation
  before spending a deploy on one: a **defective validator** (the named-query bind-variable case, where the
  runtime demonstrably supports what the build rejects) is worth forcing; a **structural rule the runtime
  enforces too** is not — forcing only moves the refusal from build time, where it is explicit, to runtime,
  where it is invisible. A root step must always be able to finish; make a journey read-only by other means
  (a browsing form field, `authoringMode: force-browsing`) rather than by removing its exit. **Reading a forced
  build: `--force` still PRINTS every validation error, then ends with `Build successful`** — so scan the output
  for `(type: VALIDATION)` lines, never trust the last line alone.
- **A child's PRE_CONSO enricher that navigates UP to its parent computes empty when both are loaded in the
  SAME batch** (runtime-validated) — the mirror image of the child-walking rule below. The parent is not
  certified when the child's pre-consolidation phase runs, so `lookup … from <ParentRole>.… end` returns
  nothing; build green, enricher correct, every derived value null and no error raised. Fix: load the child
  in a **later batch** than its parent (verified: 0/100 resolved in a shared batch, 100/100 after the same
  rows were re-persisted once the parent was certified). Plan loads as **one batch per level of derivation**,
  not just reference → masters → children.
- **Post-consolidation enrichers whose lookups walk CHILDREN compute empty on first load** (runtime-validated):
  the parent is certified in its own batch before the children exist, so derived helpers (counters, worst-of
  statuses, primary-code lookups) come out null. Fix = a **phase-3 re-certification load**: after the children
  integrate, re-persist the parent records (FULL payloads — a persist MERGES, so an omitted attribute keeps
  its stored value; see the loading-hazards bullet) and
  submit again; batches on one queue run sequentially, so submitting right after phase 2 is safe. The demo
  loader implements it via a `"phase": 3` spec flag.
- **Three loading hazards that compound into one another, and cost a demo data location a full afternoon**
  (runtime-measured, platform 2.0). **(1) A persist MERGES with the record already stored under that id** -
  an attribute you omit KEEPS its previous value, it is not cleared. To blank a value, send the key with an
  explicit `null`; sending "the record without that field" changes nothing. Measured on root placements
  written onto ids that previously carried a child: every root silently inherited the old parent.
  **(2) Surrogate ids built POSITIONALLY in a demo dataset (a counter over a list) renumber themselves the
  moment rows are INSERTED rather than appended** - the next load then writes new content onto ids holding
  something else, and (1) turns that into a silent hybrid record rather than an error. Append new data at the
  END of the generation order, or derive ids from a stable business key. **(3) A HARD_DELETE of the GOLDEN
  records does not remove the SOURCE records**, so it is not a clean slate: the next persist merges with the
  surviving source record and the ghost values come straight back. What makes the trio expensive is that the
  damage surfaces far from its cause - as mass failures of ancestor-walking rules (depth, cycle), because a
  rejected row never enters the golden table, its children can then no longer resolve their chain, and one
  bad parent cascades down a whole branch and survives every reload. Diagnosis that works: the **PERSIST-time
  validation probe** (`persistMode: NEVER` + `responsePayload: SUMMARY_AND_RECORDS`, then CANCEL) returns
  `failedValidations` per record AND the merged `recordValues`, which is where the stale attribute becomes
  visible. Tell-tale: a ROOT failing a DEPTH rule - a root cannot be too deep, so its stored parent is not
  what the payload says.
- **Loads API value formats** (runtime-validated): `Timestamp` attributes accept ISO 8601 with milliseconds
  and `Z` (`2026-06-12T09:15:00.000Z`) — both `…T09:15:00` and `… 09:15:00` are rejected with `Unable to
  parse value … as a date`. `Date` accepts plain `YYYY-MM-DD`.
- **An enricher that cannot evaluate at certification SUSPENDS the integration batch — and a suspended queue
  silently starves every LATER certification task** (runtime-validated). The classic cause is a **form-only
  enricher left at a certification scope**: an enricher written for a stepper/form (it reads a field the user
  is editing, and is wired through `formStepEnrichers`) must carry **`enricherExecutionScope: NONE`** — at
  `PRE_CONSO`/`POST_CONSO` it also runs inside `INGEST_ALL`, where that field is null on every record, and it
  can fail there. The damage is **not local**: the batch suspends on that task, the queue stops, and every
  enricher of every entity certified *after* it never runs. The symptom is misleading — an unrelated entity's
  computed attributes come back **empty** while the build is green, the deploy is fine and the load reports
  `PERSISTED` everywhere. **Reflex on an empty computed attribute: check the queue and the batch BEFORE the
  expression** — `sem dm admin execution-engine clusters queues get Q0.Default --cluster-name <DL>` (a
  `SUSPENDED` status or a non-zero `runningJobs` that never moves) and `integration-batches`, whose
  `currentTask.name` **names the offending enricher**. Note that `integration-batches` can answer HTTP 500
  for a while after a cancel, so read the queue rather than the batch list when it does.
- **An opaque 500 on deploy has TWO causes, and only one of them leaves the data location broken**
  (both runtime-validated). **(a) An un-submitted LOAD still `RUNNING`** — created and never submitted or
  cancelled, typically by an ad-hoc probe script — blocks the deploy with `Problem during import Internal
  Server Error`, while the DL stays `DL_READY` and the integration queue reads empty. Nothing is damaged:
  cancel the load (`POST /dm/api/rest/loads/{DL}/{id}` with `{"action":"CANCEL"}`, or the CLI) and redeploy.
  **Reflex: the queue being idle does NOT mean the data location is free — list the LOADS and look for
  `RUNNING`**, which is a different object from a batch. Any script that creates a load must cancel it in a
  `finally`, exactly as the demo loader does. **(b) A deploy attempted while an integration BATCH is still
  running** fails the same way and DOES leave the data location in `MAINTENANCE` (runtime-validated). Recovery: wait until the batches are `FINISHED`
  (`sem dm data-locations integration-batches`), redeploy (the fresh deploy succeeds on top), then
  `sem dm data-locations set-status <DL> DL_READY --force`. Always let the loader's submitted batches finish
  before redeploying.
- **Certification order is the order of the job's `modelTasks`, and the default (declaration order) is
  WRONG as soon as a core entity looks up reference data** (runtime-reasoned, build-validated; the engine
  builds the tasks from the `build_model()` entity order, i.e. `core` then `reference_ent`). A task reads
  the golden tables **as the earlier tasks left them**, so an entity certified before its lookup targets
  sees them EMPTY on the first load: a display name derived from a referenced business object comes out
  null, a data-driven `FID_<role>` stays unresolved — and it silently settles on the *next* load, which is
  the worst kind of demo defect. Declare the dependency order explicitly with
  **`cfg["job_order"] = [entity, …]`** (listed entities first in that order, the rest behind them; unknown
  names raise). Rule of thumb: **reference/lookup entities first, then the hub, then the association
  entities**. This does NOT fix same-entity dependencies (a rule walking its own ancestors) — those need a
  **second pass** that re-persists the records in FULL once the batch is integrated (§ *Post-consolidation
  enrichers whose lookups walk CHILDREN…*).
- **A rejected record drags down the records whose MANDATORY reference points at it** (runtime-validated,
  platform 2.0). A validation failure is not isolated: the children whose required FK targets the rejected
  record cannot resolve it and are rejected too. Measured on a hierarchy model: 3 seeded rule breaches
  produced **5** records in error (1 parent + its 2 children + the 2 direct breaches). Consequences:
  when seeding demo violations, put them on **leaves** unless the cascade is the point, and never write
  "expect N records in error" from the number of broken rules — count the owned children too.
- **Cross-entity ordering is not enough: same-entity and later-entity dependencies need a SECOND PASS**
  (runtime-validated). `cfg["job_order"]` fixes the order between entities, but a rule aggregating a
  record's own children (the parent certifies first) or an expression walking its own entity's ancestors
  still sees an incomplete picture on the first load - derived values come out empty and aggregate rules
  pass when they should fail. Fix: re-persist the affected entities in **FULL** once the batch is
  integrated - full because a persist MERGES, so an attribute you omit keeps its stored value (see the
  loading-hazards bullet). Batches on one queue run sequentially, so the second load can be submitted
  immediately. Measured: an ancestor-inheritance expression and a per-record allocation total both only
  settled on pass 2. Ship the loader with that pass built in, and a flag to disable it - running without
  it is the clearest way to *show* the behaviour.
- **Fuzzy self-reference (hierarchy on a FUZZY_MATCHED entity) loads in two phases like any fuzzy FK**
  (runtime-validated): the parent FK needs the parent's **golden id**, so phase 1 loads all masters
  *without* `FID_<ParentRole>`, integrate, map `SourceID → golden`, then phase 2 **re-persists the full
  child records** (same `SourceID`+`PublisherID`) with `FID_<ParentRole>` set to the parent's golden id.
  Re-persist the **complete** record, not just the FK — a partial re-persist creates a new master version
  whose null attributes can survive into the golden (default `PREFERRED_PUBLISHER` with skip-nulls false).
- **Reference a FUZZY parent by its SOURCE key in a load — `SourceID_<Role>` + `PublisherID_<Role>`**
  (runtime-validated). This is the scalable alternative to `FID_<Role>`: the pair resolves to the parent's
  golden record at integration, so parents and children need no golden-id round trip and can even share one
  load. It matters because the **query API is hard-capped at 100 records and ignores every paging/filter
  parameter** (`size`/`offset`/`limit`/`pageSize`/`orderBy`/`filter` all return the same first 100 rows) —
  which makes the documented `SourceID → Gold_<PK>` remapping **impossible above 100 masters**. Confirmation
  that the source-key form is the supported one: a fuzzy entity's master view returns its own reference
  columns as `SourceID_<Role>`/`PublisherID_<Role>`. Also note **`ID_MATCHED` entities require `PublisherID`
  on persist** in addition to their PK — they carry per-publisher masters exactly like fuzzy ones.
- **Data ingestion (REST loads API)**: `POST {base}/dm/api/rest/loads/{DL}` — `CREATE_LOAD` →
  `PERSIST_DATA` (per entity) → `SUBMIT` (`INGEST_ALL`). Needs a role with **`integrationWsRole: true`**
  in the model grants (a `semarchyConnect`/`semarchyAdmin` key is refused — it's a model privilege).
  Records: references as **`FID_<Role>`** = target id; FUZZY parent → `SourceID`+`PublisherID`,
  ID_MATCHED → its PK, BASIC → its id; complex types **flattened to dotted keys** (`PrimaryAddress.City`),
  native-typed values; `persistMode: ALWAYS` for masters. **Multivalued attributes**: the loads API
  **rejects a JSON array** (`"expecting STRING but got [...]"`) and enforces the attribute `length` —
  load a **single scalar string** per record (multiple values via REST need a longer delimited
  attribute or a child-table mechanism, not an array). **Query REST API caveat** (`GET
  …/query/{DL}/{E}/{GD|MD|…}`): returns `{records}` **capped at ~100 rows**, ignores `filter`/`offset`
  query params, and the golden payload does **not** expose native match metadata (`MastersCount`,
  `ConfidenceScore`); to enumerate/confirm merges use the app UI (golden view + Duplicate Manager) or
  compare **source-vs-golden counts** (a golden count below the loaded source count = merges happened).
  **Two-phase**: children of a FUZZY parent need
  its generated golden id → load masters, integrate, map `SourceID→golden` via `/query/{DL}/{E}/MD`
  (`Gold_<PK>`), then load children. A **suspended integration batch blocks redeploy/drop** — clear it
  with `sem dm admin execution-engine clusters queues cancel Q0.Default --cluster-name <DL> --force` (per-DL;
  the `admin` segment is part of the path, and it usually takes **two** calls — the first drains the *queued*
  job, the second kills the *running* one; check `… queues get Q0.Default` until `runningJobs: 0`). External REST
  enrichers run at integration and suspend the batch if unreachable → set them `enricherExecutionScope:
  NONE` and pre-fill outputs in the payload. Full procedure: PlatformCLI guide (§ Load data) and
  [resources/data/README.md](resources/data/README.md).
