---
name: seml-model
description: "Rules and validated findings for the DATA MODEL layer of a Semarchy `.seml` data product. Invoke BEFORE creating or editing an entity, an attribute, a reference, a matcher, a survivorship or ID rule, an enricher, a validation, a unique key, a LOV, a publisher, a privilege grant, a retention policy or a workflow — and whenever a computed attribute comes out empty, a record is rejected, or a merge does not happen. Most of what it contains is build- or runtime-validated against the platform and is invisible to the build."
---

# Authoring the data model of a Semarchy data product

Everything here was validated against the CLI build, the embedded schemas or a running platform. Where a finding contradicts the online documentation, the finding wins — the docs describe intent.

Screens, forms and views are the sibling skill `/seml-ui`. SemQL expressions are `/semql-suggest`. Loading, deploying and diagnosing a data location are `/platform-ops`.

- **No label translation in the solution today** — a **multi-language application** (the same user-facing label
  served in several languages per user) is **not implementable** in the current version; it is announced for a
  future one. Never offer it as an option, and never design a workaround (translation entity, per-language LOV
  duplication) unless the client explicitly commissions a custom design. Consequence: the design language
  rule (workshop rules, `CLAUDE.md`) is a **one-shot decision** — changing it later means re-labelling the whole model by hand.
- **Survivorship — 7 consolidation strategies** (CLI-confirmed v1.11.0, not just 2): `PREFERRED_PUBLISHER`,
  `CUSTOM_RANKING`, `LONGEST_VALUE`, `SHORTEST_VALUE`, `LARGEST_VALUE`, `SMALLEST_VALUE`, `MOST_FREQUENT_VALUE`;
  two rule kinds (Standard + ID); `overrideStrategy` ∈ {`NO_OVERRIDE`, `UNTIL_CONSOLIDATED_VAL_CHANGE`,
  `UNTIL_NEXT_USER_CHANGE`, `ALWAYS_AUTHORED_FROM_MDM`, `WITH_DECAY`}. Enums to trust: `entityType`
  (BASIC/ID_MATCHED/FUZZY_MATCHED), id `mode` (MANUAL/SEQUENCE/SEMQL/UUID), `deletePropagation`
  (RESTRICT/CASCADE/NULLIFY), scopes (NONE/PRE_CONSO/POST_CONSO/PRE_POST).
- **The matcher's merge threshold is EXCLUSIVE — a rule scoring EXACTLY `mergeThreshold` does NOT merge**
  (runtime-validated, platform 2.0, and invisible at build time). Measured: two source records whose
  normalized names were **byte-identical**, same country, matched by a rule scoring **90** against
  `mergeThreshold: 90` — they stayed two golden records; raising that one rule to **95** merged them on
  the next load, with nothing else changed. So write every rule that must merge **strictly above** the
  threshold, and reserve the exact-threshold score for a rule you want to surface as a *suspect duplicate*.
  Diagnosis tip: if only the highest-scoring rule (typically an exact-identifier rule at 100) ever
  produces merges while similarity rules never do, suspect this before suspecting the similarity function
  — count golden vs master records (`/query/{DL}/{E}/GD` vs `/MD`), which is the cheapest signal.
- **Never predict a FUZZY entity's population from the business key — the similarity rules merge what the key
  keeps apart** (runtime-measured). Announcing "N records will land in this queue" by grouping the source file
  on its identifier (DUNS, VAT, registration number) ignores every non-identifier match rule, and the rules
  that fire hardest are precisely the ones on records where the identifier is MISSING: measured on a
  provider-vs-CRM dataset, a company predicted as *known only from the provider* (its identifier appeared in no
  other source) merged with two internal masters carrying **no identifier at all** but the same name — the
  name+country rule scored 95 against a 90 threshold. The golden count was right, the prediction was wrong.
  So derive an expected population from the **match rules** (identifier rules AND similarity rules, against the
  threshold), or state the count as unverified until the load runs. Cheapest check afterwards: group the master
  view (`/query/{DL}/{E}/MD`) by `Gold_<PK>` and read each group's publishers — it shows exactly which records
  merged and why the count differs.
- **`Record` is a reserved alias in SemQL: a to-one role named `ParentRecord` fails the build**
  (`invalid qualified name … no parent context available to use ParentRecord`), while `ParentCustomer`,
  `ParentNode` and `ParentCompany` all parse. `Record1`/`Record2` are the matcher aliases and
  `Record.`/`Referenced.` the fuzzy-lookup ones, so avoid `Record` anywhere in a role name — it is not
  caught by the reserved-*attribute*-name check and only shows up when an expression navigates the role.
- **A `DynamicVariable` is resolved ONCE, AT LOGIN — never use it as an interactive filter input**
  (user-confirmed product behaviour). It IS the only object that enters the global SemQL bind-variable
  registry, so `:VariableName` is accepted **everywhere SemQL is validated** — business-view
  `rootFilter` and **transition filters**, named-query **child navigation filters**, enrichers,
  validations (build-validated). That reach makes it tempting for "let the user choose a value and
  filter the whole tree by it" — and that is exactly the trap: the value is refreshed only when
  the session opens, so nothing a user does while browsing can change it. Legitimate uses: **session
  context** (who is connected, their scope, their region) for row-level security and personalised
  defaults. Illegitimate: any as-of date, threshold or selector the user is meant to move.
  Corollary, corrected 2026-08-25 — **an API parameter DOES propagate into sub-levels**: a
  named-query parameter works in a child navigation `filter` at RUNTIME (the 2.0 build validator wrongly
  rejects it; `--force` gets past it). What has no browse-time parameter is the **application**: search
  filters only its own level, transition filters are static, no form field runs client code, and the
  Application schema has no variable. So an as-of view is reachable through the API today, and remains a
  genuine product gap inside the app.
- **Getting per-publisher provenance ONTO the golden record — stamp on the master, aggregate by survivorship**
  (build- AND runtime-validated, platform 2.0). The lineage arrays are unreachable outside view contexts (`/semql-suggest`, lineage paths),
  so "is this record fed by source X?" cannot be looked up from the golden. It can be *materialized*, exactly and
  structurally, in three pieces: **(1)** `PublisherID` **IS** readable in a `PRE_CONSO` SemQLEnricher, which runs
  on each master — stamp a one-character flag per source family (`CASE WHEN PublisherID = 'X' THEN 'Y' ELSE 'N'
  END`, one expression per flag, all in one enricher); **(2)** give each flag a **single-attribute
  `LARGEST_VALUE`** survivorship rule (`consolidationSkipNulls: false`, `overrideStrategy: NO_OVERRIDE`) — over
  `'Y'`/`'N'` a MAX is a logical OR, so one contributing record is enough to flip the consolidated answer;
  **(3)** read the flags anywhere, including a display card, a form field or a validation, since they are now
  plain atomic attributes. Measured on 14 goldens / 27 masters: zero disagreement between the consolidated flags
  and the real contributing publishers. Use `String(1)` `'Y'`/`'N'`, **not Boolean** (SemQL cannot touch a
  boolean directly, and MAX over a text flag is the whole trick). Two things to state on the attribute's
  documentation: it is populated **only at load time** (a redeploy alone shows empty flags — plan a reload), and
  **on a master it describes that single record**, so any label built on it ("known only from X") is a lie
  outside the golden context — which is why the card carrying it must be a dedicated one.
- **Survivorship rule constraints (build-validated)**: **an attribute may be listed in at most ONE
  survivorship rule** — listing it both in the default rule's explicit `attributes` and in a
  per-attribute rule fails the build with `Conflicts with rule <X>` (remove it from the default
  rule's list; unlisted attributes fall to the default rule anyway). **`PREFERRED_PUBLISHER` is reserved for the
  single `defaultRule: true` rule and is the only strategy that may span MULTIPLE attributes** — every
  value-based rule (`MOST_FREQUENT_VALUE`, `LONGEST_VALUE`, `SHORTEST_VALUE`, `LARGEST_VALUE`,
  `SMALLEST_VALUE`) must be **single-attribute** (else "Not allowed for default rule with multiple
  attributes"). The `PREFERRED_PUBLISHER` default rule needs an **explicit `attributes` list** and
  **`consolidationSkipNulls: false`** — `skipNulls: true` on a default multi-attribute rule is rejected
  with that same "Not allowed for default rule with multiple attributes" message. **`consolidationSkipNulls: true`
  is rejected by the value strategies too** — not just `MOST_FREQUENT_VALUE` but also `LONGEST_VALUE`,
  `SHORTEST_VALUE`, `LARGEST_VALUE`, `SMALLEST_VALUE` ("… does not support skip nulls"); these strategies
  already **ignore nulls inherently** (SQL `MAX`/length semantics), so leave `skipNulls: false` and they
  still return the non-null value. length/size strategies (`LONGEST_VALUE`…) are **rejected on `Binary`**
  attributes. **Cross-publisher per-attribute survivorship pattern** (e.g. header from publisher A,
  execution from publisher B, where B's masters are null on A's header attrs): since only ONE
  `PREFERRED_PUBLISHER` rule is allowed, rank the **default** rule to the publisher that owns most
  attributes (B, skip-nulls false), then give each attribute owned by the *other* publisher (A — null on
  B) a **single-attribute `LARGEST_VALUE`/`LONGEST_VALUE`** rule (skip-nulls false, null-ignoring) so A's
  value survives — you cannot use a second `PREFERRED_PUBLISHER` rule for the A-group. **Attributes not listed in any
  rule fall to the default rule** — so a value present on only *some* masters of a group can be lost
  (default `PREFERRED_PUBLISHER` with `consolidationSkipNulls: false` takes the top-ranked master's
  value even if null). Fix: either load the value onto the preferred (or every) master of the group, or
  add a dedicated single-attribute rule with a null-skipping strategy that the datatype accepts.
- **Model-level role flags live on `ModelPrivGrant`, and only the model build can grant them**
  (schema-confirmed, 5 properties): **`integrationWsRole`** (the role the **REST loads API** accepts — without
  it every ingestion call is refused, and it cannot be granted platform-side because it is a *model*
  privilege), **`dataAdminRole`**, **`allowApiPublishingAsUser`**, `allowDataQualityDocumentation`,
  `allowEnrichmentDocumentation`. `roleName` may differ from the file `_name` (e.g. `_name: SemAdmin` /
  `roleName: semarchyAdmin`). **Recommended default in EVERY model — the `SemAdmin` grant**
  (user-validated convention, present in BusinessPartner360, HierarchyHub and MasterVendorIndex): a grant
  with `roleName: semarchyAdmin`, `integrationWsRole: true`, `dataAdminRole: true` and READ_WRITE on all
  entities, because developers and sandbox API keys usually carry the built-in `semarchyAdmin` role —
  without this grant their keys are refused by the loads API even though they administer the platform.
  Keep a dedicated `IntegrationService`-style grant alongside it for production service keys. The engine
  passes all five flags through from `cfg["grants"]` entries
  (`{"role": …, "roleName"?: …, "integrationWsRole": True, …}`) — **it used to drop them silently**, so a
  model whose loads worked could lose ingestion access on the next regeneration with a green build. Declare
  the flag in the model definition, never by hand-editing the generated grant. **POV/demo convention
  (runtime-validated):** when a single admin API key drives both deploy and loads, the grant mapped to
  the admin platform role (`roleName: semarchyAdmin`) must itself carry `integrationWsRole: true` —
  adding platform roles to the key does NOT help (403 `access to integration service must be granted
  in Model Privileges`), and the fix is a model redeploy, so set it from the first generation.
- **A role's access is per entity, not per grant** — the common shape is *write on my own sub-domain, **READ**
  on the hub entity and on reference data* (a manager of one role-type must see the party its rows hang from
  without being able to edit it, and must see reference data without owning it). One `ModelPrivGrant` holds one
  `EntityPrivGrant` per entity, each with its own `defaultAccessType` + `create/delete/removeAllowed`. In the
  engine: `access` is the grant default and **`entity_access = {entity: "READ"|"READ_WRITE"}`** overrides it per
  entity. Modelling such a role as a single blanket `READ_WRITE` over-grants it; splitting it into two grants
  for the same role is not possible (one grant per role name).
- **Never emit YAML anchors/aliases in `.seml`** (`&id001` / `*id001`). A generator that reuses the *same*
  nested object in two places (a form step appearing at the stepper root **and** under a self-hierarchy level,
  a shared property block) makes PyYAML serialize the second occurrence as an alias. `sem-design build` and the
  extension's schema validation both accept it, so nothing fails — but a `.seml` file is a **source** artifact
  that is read, diffed and hand-edited, and an alias makes the second occurrence invisible while betting on
  every downstream consumer implementing anchors. Fix at the dumper level, not per call site:
  `class _Dumper(yaml.SafeDumper): def ignore_aliases(self, data): return True` (the engine does this).
- **Compound attributes (addresses…) — flatten to atomic attributes grouped in a form section.** The
  engine emits `ComplexType` files (`model["complexTypes"]`) but has **no entity-level complex-attribute
  emission** (every entity attribute is an `AtomicAttribute`), and the bundled reference model flattens
  its addresses the same way (zero `ComplexAttribute` in BusinessPartner360). So when a specification
  says "compound/complex address", implement it as flat attributes (`AddressStreet`, `PostalCode`,
  `City`, `Country`) grouped in an "Address" form section — identical on screen — and record the
  flattening as an implementation note in the spec's decision log. (Validated: CustomerB2C, 2.0 build green.)
- **Entity-type choice**: single source → `BASIC`; shared key across sources → `ID_MATCHED`; no common
  key (similarity matching) → `FUZZY_MATCHED`. A fuzzy entity carries both a `matcher` **and**
  survivorship rules (Standard + ID), and usually a `DupsManager`.
- **`physicalName` ≤ 25 characters**; suffix it if it is a reserved SQL word. **And unique across the
  model (build-validated):** two references (or entities) whose generated name truncates to the same
  25 chars fail the build with `physicalName conflicts with <file>` — long reference names like
  `<Entity>HasIssuingState` / `<Entity>HasIssuingCountry` are the classic collision; pass an explicit
  `physicalName`/`physicalTableName`. The engine now **fail-fasts at generation** on both collisions.
- **Record types**: author as **GOLDEN** (Create/Edit); fuzzy import as **MASTER** (so it goes through
  Match & Merge). BASIC entities import as GOLDEN.
- **REST client enrichers — two layers, kept in sync**: a `RestClientEnricher` in the model
  (`pluginId` = platform client name; `pluginInputs.inputName` / `pluginOutputs.outputName` must match
  the client's `requestInputs`/`responseOutputs`) **plus** a platform **REST client** deployed
  separately (`resources/RestClient/<name>.json`, via `sem dm admin rest-clients set`). The model
  build never deploys the client. Client JSON keys: `requestUrl` with `${input}` variables,
  `requestHeaders` (newline-separated string), `requestBodyFormat`/`requestBody`, `responseOutputs[].valueSelector`
  (JSONPath), `onAbsentValueBehavior`/`onDataTypeMismatchBehavior` ∈ {`USE_DEFAULT_VALUE`,`FAIL`}; no
  `authentication` block for public APIs (auth goes in `requestHeaders`). Full procedure in the
  generation prompt (§ REST client enrichers).
- **`EnricherCache.pluginIdOrRestClientName` must be the REST client name, byte for byte** — the same string
  as the enricher's `pluginId` and as `resources/RestClient/<name>.json`. There is **no prefix convention**
  (no `C…` for "client"): the name is resolved as-is, the build **never checks it** (the property is a free
  string, not a cross-reference), and a mismatch is a pure runtime no-op — the cache table is created and
  stays empty while every call goes back out to the service. Whenever an enricher cache is added, re-read the
  three spellings side by side. A stale/prefixed value is a classic copy-paste inheritance from another model.
- **Description vs documentation — where each is accepted (schema-validated).** Enrich model objects with
  business/source context by populating `description` (and `documentation` where allowed). Entities,
  atomic attributes (incl. PK), references, LOV types **and** LOV values accept **both** `description` and
  `documentation` (the engine mirrors one into the other). **Enrichers, validations and `DisplayCard` accept `description`
  but NOT `documentation`** — emitting `documentation` on a `SemQLEnricher`/`SemQLValidation` fails the build
  (`Property documentation is not allowed`); emit `description` only there. **`Application` accepts a top-level
  `description` but NOT a top-level `documentation`** (app documentation goes inside
  `documentationConfig.documentation`). **`ModelJob` accepts `description` but NOT `label`.**
  **`Workflow` accepts NEITHER `description` nor `documentation` — only `label`**, so a workflow's
  human-readable documentation must live in the workflow/step/transition `label`s. The engine now emits a
  friendly `description` for the Application, each ModelJob (`cfg["job_meta"]`), enrichers and
  validations, and mirrors both fields where allowed.
- **Audience of description/documentation text — write for the reader you actually have.** These strings
  surface in the app UI, generated docs and catalogue, so match the intended audience. For a business-user
  deliverable: plain language, no product/tooling jargon (avoid "MDM", "golden record", "master", "publisher",
  "survivorship", "fuzzy", raw source API column names) and no project/process meta ("POV/POC", RFI/section
  citations, "(stub)"); name the source **system** in business terms (Salesforce, IFS Cloud) rather than its
  raw field ids. The engine's built-in default labels are business-friendly ("Contributing sources" / "Source
  data" / "Source history" / "Change history" for lineage tabs; "Record id" for the id column; "Default value
  selection" / "Identity" for the fallback survivorship rules; plain-language matching-review filter text).
- **The documented model IS the semantic layer — document everything; never reify model metadata as
  entities by default** (user-validated doctrine; defect observed in the 2026-08 benchmark runs). The
  `description`/`documentation` fields of the data product root, entities, attributes, references, LOVs
  (type AND values), enrichers/validations and jobs are what the **MCP-server layer exposes to AI agents
  as the semantic model** — an agent reasons with THESE fields, so all business knowledge belongs there,
  exhaustively: meaning, sources, units, **calculation rules** ("net invoiced revenue = gross booked −
  cancelled − credit notes, in USD") and **reasoning/usage rules** ("rank customers at group level").
  Corollary: do **NOT** create entities whose rows describe the model or instruct its consumers
  (`BusinessTerm`, `Glossary`, `Instruction`/`AIInstruction`, `FieldClassification`, taxonomy…) unless
  the client **explicitly** requires that content to be managed as runtime data (authored/versioned/
  approved by business users in the app) — a pitch narrative describing a "semantic/meaning/instructions
  layer" is a documentation requirement, not an entity list; surface any such entity as its own gate
  hypothesis with embedded documentation as the recommended default, never inside a bulk-accepted entity
  group. Engine support: `cfg["description"]` (data-product root description) and an end-of-run
  `check_documentation()` report (missing or label-only descriptions — the build is green either way);
  treat its findings as generation defects. **Complement (never duplicate) the functional description
  with a final `AI agent instructions:` section when a plausible misuse exists** — canonical ENGLISH
  marker (stable for the MCP layer, like `[PII]`), content in the design language, emitted via the
  engine helper `with_ai_instructions(desc, instructions)`. Trigger families (full catalogue in the
  generation prompt § Descriptions and Documentation): LOV codes driving calculations/filters;
  hierarchies and their rollup rules; amounts/units/currency conversions; temporal validity and
  allocations; technical/derived attributes (do not reason on or re-derive them); golden vs
  master/source records. Not on every field — only where wrong usage is plausible.
- **On a BASIC entity, a POST_CONSO ENRICHER never runs - build green, values silently empty**
  (runtime-validated, platform 2.0). A basic entity has no consolidation phase, so the scope has nothing to
  hook onto; unlike the validation case below the CLI does NOT refuse it, the deploy is clean, the load
  reports `PERSISTED` everywhere and the attribute simply stays null. Measured on an enricher aggregating an
  entity's children into a stored key: empty on every record until the scope became `PRE_CONSO`, then correct
  on the next load with nothing else touched. Reflex when a computed attribute is empty on a BASIC entity:
  read its SCOPE before suspecting the expression or the queue. Corollary: a `PRE_CONSO` child aggregate still
  reads the golden tables as the earlier tasks left them, so it needs the second pass to see children loaded
  in the same batch.
- **BASIC entities accept only `PRE_CONSO` validation scopes** (build-validated): a `SemQLValidation`
  with `validationScope: POST_CONSO` (or `PRE_POST`) and a `Reference` with `validationScope: POST_CONSO`
  both fail the build with *"post-consolidation validation is not supported for BASIC entities"* — there is
  no consolidation phase on a basic entity. Emit `PRE_CONSO` everywhere on BASIC; keep POST_CONSO for
  ID_MATCHED/FUZZY_MATCHED only.
- **A PRE_CONSO SemQL validation on a MATCHED entity (FUZZY_MATCHED or ID_MATCHED) cannot reference
  the entity's own PK attribute** (runtime-validated, platform 2.0; builds green): the PK does not bind
  to the loaded value at that stage, so the rule fails for EVERY record — a 17-char VIN failed
  `LENGTH(VIN) = 17`, and even `VIN IS NULL OR LENGTH(VIN) = 17` failed identically (neither branch
  true: the PK is neither NULL nor the payload value; the persist response exposes the source PK as
  `Gold_<PK>`). Non-PK attributes in the same rule set validate fine. Fix: express the rule on a non-PK
  attribute or drop it (the attribute's declared length already bounds the value). The engine now
  **fail-fasts at generation** on this pattern. Diagnostic technique that isolated it (validated):
  run the validation at PERSIST time via the loads API — `optionsPerEntity.<E>.validations` takes
  **objects `{"validationName": …, "validationType": "CHECK"}`** (not bare strings) — with
  `persistMode: NEVER` + `responsePayload: SUMMARY_AND_RECORDS` on a probe record + a control record,
  then CANCEL the load: per-record `failedValidations` come back synchronously, no integration needed.
- **`CUSTOM_RANKING` survivorship: `<expr> DESC` ranks NULLS FIRST (PostgreSQL default) — a master with a
  NULL ranking value beats every real value** (runtime-validated: a tier ranked by `LoyaltyPoints DESC`
  surfaced the points-less CRM master's value on a golden holding 8120 points; builds green, wrong data).
  Always wrap the ranking expression: `COALESCE(<attr>, 0) DESC` (or a sentinel below/above the real range
  for ASC). Same reflex for any survivorship/sort expression over an attribute that only SOME publishers
  feed. Fix = regenerate + redeploy + re-submit a full load (survivorship applies at certification).
- **A `Publisher.code` must not be a SQL reserved word** (build-validated): `code: REF` fails with
  `code is a reserved key word for POSTGRESQL` — pick `REFDATA`-style codes.
- **`Timestamp` attributes require a `precision`** like Integer/LongInteger (same "The precision attribute is
  required" build error; fractional-seconds digits, 6 is a safe default — the engine now defaults it).
  **A `String` attribute's `length` is capped at 4000** (build-validated: "The length must be between in 1 and
  4000") - size any rendered or aggregated text against that ceiling, and budget it explicitly (characters per
  element x the largest expected set) whenever the value grows with the data, such as a URL-encoded drawing or
  a delimited key set.
  Reserved physical-name instances to remember: **`RESULT`** and **`ORDER`** are rejected for POSTGRESQL
  (`physicalName`/`physicalTableName is a reserved key word`) — rename them (`SCREEN_RESULT`, and an `Order`
  entity needs an explicit `physicalTableName` such as `SALES_ORDER`; the default derived from the entity
  name lands straight on the reserved word).
- **`UniqueKey` — business uniqueness beyond the PK** (build-validated, CLI 2.0.0). Standalone object in
  `entities/<E>/unique_keys/`, `_package: <PKG>.entities.<E>.unique_keys`; **required**: `_name`, `_package`,
  `_type`, `entity`, `label`, `validationLabel`, `validationScope`, `keyAttributes` (≥ 1). A `KeyAttribute` is
  `{attribute: <FULL attribute QN>}` — **an attribute reference, with no expression form**, so a *conditional*
  uniqueness needs the enriched-token detour of the correlation rule (`/semql-suggest`). Make that token **NULL-free** (a constant for the rows that
  must be unique, the row's own business code otherwise): whether the platform collides NULL keys is a runtime
  behaviour you would otherwise be betting on, and a null-free token is correct either way. The engine emits
  them from **`cfg["unique_keys"] = {entity: [{_name, attributes:[…], label?, validationLabel?, scope?,
  description?}]}`**. Reflex: an entity whose spec says "unique business code" usually has **nothing** enforcing
  it — the PK is the surrogate id, not the code.
- **`NamedQuery` — the structure/export API object** (build-validated, CLI 2.0.0). Lives in `named_queries/`
  at the model root. Required: `_name`, `_package`, `_type`, `active`, `rootQueryObject` (the **full QN**
  `<PKG>.named_queries.<NQ>.<queryObject _name>`); optional `description`, `defaultLimit`, `filter`, `sort`.
  `queryObjects[]` = `{_type: QueryObject, _name, entity, description?, objectProperties[]}`; each
  `objectProperty` = `{_type: ObjectProperty, _name, type: VALUE|SINGLE_RECORD|MULTI_RECORDS,
  semQlExpression?, targetObject?, filter?, sort?}`. Two traps: **`ObjectProperty` accepts NO `description`**
  (`Property description is not allowed` — like `documentation` on an enricher; only the QueryObject and the
  NamedQuery take one), and a **`bindingName` must match `^[A-Z][A-Z_$0-9]*`** (upper snake) — it is then
  referenced in the query's SemQL as **`:BINDING_NAME`** (`$`/`#`/bare all fail). **A deployed named query is served on its OWN route**
  (runtime-validated, platform 2.0) — NOT on the entity query API (`/dm/api/rest/query/{DL}/{E}/{GD|MD|…}`
  takes an *entity* + a data-access view type and returns 404/400 for a query name):
  **`GET {base}/dm/api/rest/named-query/{DL}/{QueryName}/{viewType}`** (`GD`) returns `{records:[…]}`, and
  **`…/named-query-count/{DL}/{QueryName}/{viewType}`** returns `{"count": N}`. **Parameters are passed as
  QUERY-STRING params under the `QueryParameter`'s `_name`** (e.g. `?hierarchyCode=X`) — *not* under its
  `bindingName`, which is only the SemQL-side token (`:HIERARCHY_CODE`). No extra grant flag is needed
  (`allowApiPublishingAsUser` is unrelated). **To discover any data-location API surface**, fetch
  **`GET {base}/dm/api/rest/api-docs?domain=data&data-location={DL}`** — an OpenAPI 3.0 descriptor that
  accepts the Bearer token and, despite the name, is served as **YAML** (parse it with `yaml.safe_load`,
  `json.loads` fails). The `api-ui/index.html` page itself rejects a Bearer (browser session only), so go
  to `api-docs`, never to the UI. Give an optional parameter
  **no `defaultValue`**: an empty-string default makes the idiomatic `:P IS NULL or <col> = :P` guard false and
  the query returns zero rows. The engine has no NamedQuery emission — ship one as a `model_sources/` fragment
  through `extra_objects` (`path = ["named_queries", "<X>.NamedQuery.seml"]`).
- **A SemQL enricher can target a reference FK via `attributeName: FID_<toRoleName>`** (build-validated;
  runtime behavior — actual FK re-pointing at certification — still to confirm on a deployed model). This
  enables data-driven references: deriving a reference key from other attributes/navigation (rule-table
  resolution, materialized parent links), with broken-reference errors then acting as a data-driven check.
- **Enriched value must fit the target attribute's domain**: an enricher feeding a **LOV-typed**
  attribute must produce a value in the LOV, else `lovValidationScope` rejects the record. External
  code systems (ISO/registry/source enums) rarely match a curated LOV — reconcile by (1) capturing the
  raw value + a SemQL map → curated LOV, (2) making the LOV the source's code system, or (3) switching
  the attribute to free-text (drop the orphan LOV). Never point an external-source output straight at a
  curated LOV. Details in the generation prompt (§ Enriched value vs. target attribute domain).
- **Fuzzy Lookup Rules** (rank/filter a REFERENCE's candidate targets by similarity — *not* a merge; the
  right tool to establish a relationship, e.g. a self-referencing parent, without deduplicating). Object
  `_type: FuzzyLookupRule`, one standalone file per entity under `entities/<E>/fuzzy_lookup_rules/`
  (`_package …fuzzy_lookup_rules`). Required: `entity`, `reference` (the reference resolved by the lookup),
  `lookupScoreExpression` (SemQL → integer 0-100 per candidate), `label`, `_name`, `_package`, `_type`.
  Optional: `binningCondition` (SemQL pre-filter of candidates), `sortExpression` (tie-break). **Build-validated
  SemQL context (`recordReference`): the expressions compare two aliases — `Record` = the current record being
  authored, `Referenced` = each candidate target** (e.g. `Record.ClientCode`, `Referenced.ClientCode`). A **bare
  attribute name** (no `Record.`/`Referenced.` prefix) fails the build with `invalid qualified name … can't
  access <attr>`; `sortExpression` is a plain SemQL expression (NO SQL `desc`); references are navigated by role
  (`Record.<RefRole>.<Pk>`). Score returns an integer 0-100 (a constant like `100` also builds, as a placeholder).
  Put self- and duplicate-exclusion in `binningCondition` (e.g. `Referenced.<pk> <> Record.<pk>`). Validated
  example (SYSTRA `ParentProjectLookup`): `LEAST(100, (CASE WHEN Record.AccountingStructure.Code IS NOT NULL AND
  Referenced.AccountingStructure.Code = Record.AccountingStructure.Code THEN 45 ELSE 0 END) + (CASE WHEN
  Record.ClientCode IS NOT NULL AND Referenced.ClientCode = Record.ClientCode THEN 27 ELSE 0 END) + …)`. **The rule is NOT automatic on the browse view —
  it surfaces only when WIRED to the reference field in a STEPPER's form step** via `referenceFormFieldConfigs[]`
  on the `FormStep`: `{fdnFormField: <QN of the ReferenceField>, referencePickerType: AUTO_COMPLETE|COLLECTION|NONE,
  enableUserSort: true, fuzzyLookup: {rule: <FuzzyLookupRule QN>, minScore: <int>}}` (all three of
  `fdnFormField`/`referencePickerType`/`enableUserSort` are **required**). `fdnFormField` **references a
  `ReferenceField`** by its full form QN following the generated layout path
  `<Form>.<tab>.Main.Sec{n}.S{n}Row{m}.<FDN_field>` (and must belong to the same form as the step's `formTab`).
  At runtime the steward then sees the ranked candidate parents in that field's picker (Edit/stepper), not on
  the golden collection. **Picker behaviour (runtime-validated): `referencePickerType: AUTO_COMPLETE` searches
  the WHOLE reference set — `binningCondition` only bounds the fuzzy *scoring*, NOT the browsable list**, so to
  actually restrict what the picker DISPLAYS add **`referenceFilter`** on the field config (same `Record`/`Referenced`
  context; e.g. `Referenced.ClientCode = Record.ClientCode`). To let the steward SEE the match attributes, use
  `referencePickerType: COLLECTION` with a focused **`referencePickerCollectionView`** (ordered columns).
  **Build-green / deploy-NPE trap: a `COLLECTION` picker MUST have a `referencePickerCollectionView`** — omit it and
  the model builds but the DEPLOY fails with `NullPointerException` in `AuthoringFormFactory.createRefPickerConfig`.
  Always point one: a focused view, or the target entity's default `<Target>CollectionView`. **For a cross-entity
  reference (e.g. Project→Bid) the focused picker view AND the FDN reference columns must be built on the TARGET
  entity, not the form's entity** (the engine resolves the target from the reference). The engine emits the rule,
  the stepper wiring, `referenceFilter`, and the picker collection view from `cfg["fuzzy_lookups"]` =
  `[{entity,_name,reference,label,lookupScore,fdnField?,minScore?,pickerType?,referenceFilter?,pickerColumns?,binning?,sort?,description?}]`.
  **`fdnField` is optional** — it defaults to the generated `FDN_<toRole>` reference field, and the engine
  resolves it to the field's **full form QN** automatically (walking the generated form layout), so a bare
  short name is never emitted. Override `fdnField` (bare field name) only for a non-default field.
- **PII / sensitive-attribute tagging — no native flag; use a documentation marker + attribute-grant
  masking (validated).** The `.seml` schema has **no first-class `sensitive`/`PII`/`classification`
  attribute property** (confirmed against the CLI's embedded schemas). Native *semantic tagging* of
  attributes is a **Semarchy xDG (governance catalogue)** capability, separate from the SDP data product build.
  So flag PII **in the model** by writing a canonical marker (e.g. `[PII]`) into the attribute's
  `description`/`documentation` (visible in the model tree, generated model doc and catalogue exports), and
  **enforce** it via **attribute privilege grants** (`attributePrivGrants: {accessType: DENIED, attribute}`
  on the `EntityPrivGrant` — "no access"/"masked" both map to `DENIED` at the model layer; value-masking is a
  platform concern). Drive **both** from a **single PII registry** so identification and enforcement never
  diverge. The engine does this: `cfg["pii_attributes"] = {entity: [attr,…]}` prefixes `cfg["pii_tag"]`
  (default `[PII]`) onto each listed attribute's documentation; the project's grant helper then auto-adds
  those attributes to the `DENIED` set for every role that only **READs** the entity (a consumer, not the
  owner/steward), **except the integration role** (`integrationWsRole: true` — its loads must still write PII;
  attribute grants can restrict what a role writes). Person-PII cited by a spec but **absent from the source
  extracts** is not modelled (stay faithful to the data; note it as a limitation).
- **`DataRetentionPolicy` — the type enum is `DEFAULT` | `FOREVER` | `NEVER` | `PERIOD`** (there is **no**
  `DURATION`), across all four families: `history*`, `deletions*`, `sourceData*`, `sourceErrors*`, each a
  `<family>Type` / `<family>Duration` / `<family>Unit` triplet. Per-entity overrides go in
  **`entityRetentions[]`** (`{entity, + the same four triplets}`), and the schema demands the **complete set on
  every entry** — a partial override fails with `Missing property "sourceErrorsType"`. So a per-entity entry
  must restate what differs *and* inherit the rest (the engine does this from `cfg["retention_policy"]["entities"]`).
- **"Browse as of date" is a technical-audit feature, not time-variant data viewing** (user-clarified,
  runtime-relevant): history browsing shows records *as stored* at a past date. It does NOT render a
  structure (e.g. a hierarchy) "as of date D" per business **effective dates** — past or future; that
  need requires effective-dated association entities + date filters, and the two needs must never be
  presented as one capability. Related: a native **"Browse as graph" action exists** (records +
  relationships as an explorable graph) but it is **not configurable at design time** — the user picks
  which transitions to expand at runtime; no pre-defined graph views.
- **Workflows**: trigger from an Action Set (`StartWorkflowEntityAction` → `Start`) for a
  user-initiated process; `StartFromCompleteBatchEvent` (bound to an integration `ModelJob`) for a
  process triggered automatically at the end of a load batch. A `SubmitAutomation` requires a
  `ModelJob` (`integrationJob`). `DataRecordOutput.rootEntity` points to the workflow's `RootEntity`
  in `dataScope`, not the entity itself; valid cardinalities: `SINGLE_RECORD` / `MULTI_RECORD`.
  **`UserTaskTransition.behaviorOnTransition` accepts exactly two values: `KEEP_DATA` and
  `REVERT_DATA`** (build-validated, CLI 1.11.0). There is no `DISCARD_DATA` / `CANCEL_DATA` — the
  "reject and throw the edits away" branch is `REVERT_DATA`, not a discard verb.
- **Workflows are for genuine authoring/approval processes — not for things with a dedicated native UI.**
  In particular **do NOT model duplicate/match-merge stewardship as a `DATA_AUTHORING` workflow** over
  the authoring stepper: it is handled by the **native Duplicate Manager** (the entity `DupsManager` +
  the `ReviewAndConfirm` / `MergeOrSplit` / `ReviewSuggestions` action-set actions the engine generates).
  A dup-review workflow is redundant and not the supported path. Reserve `DATA_AUTHORING` + stepper for
  create/edit flows (onboarding, change requests); use the native form where one exists.
- **Workflow palette** (use varied patterns in the reference model): start events
  (`StartFromSelectionEvent`/`EmptySelection`/`SelectedParent`/`StartEvent`/`FromCompleteBatchEvent`),
  `UserTask` (`taskType` DATA_AUTHORING/VALIDATION/GENERIC), `ParallelBlock`+`Branch`, `Router`+`Route`,
  and **5 automations**: `SubmitAutomation`, `SetWorkflowMetadataAutomation`, `SendEmailAutomation`,
  `CallProcedureAutomation` (needs a `SqlFunction`), `DeleteDataAutomation`. **To run validation rules at
  a workflow point use `transitionValidations` on a `UserTaskTransition`** (`{validationType: CHECK,
  validator, onTransitionFinish: BLOCK|SKIP}`) — there is no standalone "validate" automation.
  `SetWorkflowMetadata` expressions run in the **workflow** context (can't reference entity attributes).
  Details in the generation prompt (§ Workflows).
