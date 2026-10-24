---
name: semql-suggest
description: "Write, fix or review a SemQL expression for a Semarchy .seml model — the Claude Code equivalent of Copilot's \"@semql /suggest\". Use whenever a SemQL string is being authored or debugged (enricher expression, validation/matcher condition, business-view filter or rootFilter, sortExpression, stepper referenceFilter, display-card expression, named-query/search-form filter, fuzzy-lookup score)."
---

# SemQL expression authoring

SemQL is the expression language of the Semarchy Data Platform. Expressions are plain strings inside
`.seml` (YAML) files. **SemQL is not SQL** — write the simplest expression that satisfies the request
and do not import SQL reflexes (COALESCE/NVL, TRIM, defensive null handling) unless explicitly asked.

## 1. Ground the expression in the real model — do not guess attribute names

The `semql` MCP server (see
[ai-model-development/tools/semql-mcp-bridge](../../../ai-model-development/tools/semql-mcp-bridge/README.md))
proxies the Semarchy extension's own model index. **Requires VS Code open on the workspace.**

1. `mcp__semql__semql_list_entities` — the fully qualified entity names.
2. `mcp__semql__semql_get_entity_context` — what is actually reachable *at the location the
   expression will live in*:
   - `entityQualifiedName` — e.g. `MyModel.entities.Customer.Customer`
   - `componentType` — `entityAttribute` · `businessView` · `enricher` · `validation` · `stepper` ·
     `displayCard` · `application` · `modelPrivGrant` · `dupsManager` · `actionSet` · `namedQuery` ·
     `searchForm` · `matcher` · `survivorshipRule` · `form` · `collectionView`
   - `property` — `expression`, `condition`, `filter`, `rootFilter`, `sortExpression`,
     `referenceFilter`… Dotted paths for nested SEML properties: `binning.expression`,
     `pluginInputs.expression`, `column.expression`, `sourceIdGeneration.idExpression`.
   - `navigationPath` (optional) — attribute names to walk through REF attributes.

Infer `componentType`/`property` from what the user is editing (the open file and the surrounding
YAML key are usually decisive); ask only when there is genuinely no indication. Use **only** the
attributes the tool returns.

If the MCP server is unavailable (VS Code closed, model not yet in `src/`), say so and fall back to
reading the `.seml` files directly — but flag that the attribute list is unverified.

## 2. Syntax

```
Name1 = 'Hello'                              -- attribute
Address.City = 'Paris'                       -- to-one (REF) navigation
any Orders have (Status = 'ACTIVE')          -- to-many (ARRAY_REF) in a condition
all Items have (Price > 0)
any Orders have (any Lines have (Qty > 10))  -- nested

lookup first Name from Items end             -- aggregation over a collection
lookup count from Orders where Status = 'ACTIVE' end
lookup string_agg Name separated by ', ' from Items end
lookup min Price from Orders end
```

**`has` and `have` are interchangeable** in `any`/`all` blocks (both build- and runtime-validated:
the engine emits `any <role> has (1=1)` for hierarchy flags; grant filters written with
`any <role> have (…)` build and filter correctly) — pick one per model for readability.

**Navigation root shifts inside `lookup … end` and `any/all … have (…)`** — attributes resolve
against the *target* entity of the collection, so do not repeat the collection name:

```
CORRECT     lookup first Person.FirstName from PersonProduct end
INCORRECT   lookup first PersonProduct.Person.FirstName from PersonProduct end
```

**`ParentRecord`** reaches the enclosing entity from inside those blocks. It is a built-in
pseudocolumn — never returned by the tool, always available. Do not confuse it with a back-reference
REF attribute: inside a lookup on `PersonProduct`, `Product.Name` navigates the foreign key, whereas
`ParentRecord.Name` is the outer record.

```
lookup string_agg ParentRecord.ProductName from PersonProduct end
any Orders have (ParentRecord.CustomerName = Name)
```

Literals: single-quoted strings (`Name = 'Hello'`), bare numerics (`Price > 42`),
`IS NULL` / `IS NOT NULL`. **There is no `true`/`false` keyword in SemQL** — using one fails the build.

## 3. Entry point type

`semql_get_entity_context` returns an `entryPointType`; respect it.

| Type | Expected shape |
|---|---|
| `CONDITION` | boolean expression — `Status = 'ACTIVE' and Amount > 0` |
| `EXPRESSION` | value expression — `FirstName \|\| ' ' \|\| LastName` |
| `ORDER_BY` | comma-separated clauses, optional `ASC`/`DESC` + `NULLS FIRST`/`NULLS LAST`; `lookup … end` allowed |
| `SELECT`, `ATTRIBUTE`, `MANY_PATH` | specialised — follow the aliases returned by the tool |

## 4. Context aliases

Some locations expose aliases instead of bare attributes. Always use the exact names the tool returns.

| Location | Aliases | Example |
|---|---|---|
| `matcher` / `condition` | `Record1`, `Record2` (same entity) | `Record1.LastName = Record2.LastName and Record1.FirstName = Record2.FirstName` |
| `stepper` / `referenceFilter` | `Record` (source), `Referenced` (reference target) | `Referenced.Status = 'Active' and Referenced.Country = Record.Country` |
| `stepper` / `manyToManyPickerFilter` | `Record` (parent), `Referenced` (target) | `Referenced.Status = 'Active'` |
| `FuzzyLookupRule` (`lookupScoreExpression`, `binningCondition`, `sortExpression`) | `Record`, `Referenced` | see the workshop rules below |
| `searchForm` / `namedQuery` | bind variables, `:` prefixed | `ValueStatus = :SEARCH_PARAM_VALUE_STATUS` |

Bind variables come back in a `bindVariables` map keyed by component qualified name — use the exact
names returned, not the `SEARCH_PARAM_*` / `QUERY_PARAM_*` naming convention.

## 5. Workshop rules that override the generic guidance

These are validated against the CLI build and the running platform in this workshop, and they win
over both the vendor tooling's advice and the online docs (the findings below, and `/seml-model`).

- **Match-metadata boolean flags — do not compare them.** `HasSuggestedMerge` and `IsConfirmed` are
  physically **boolean** columns. `= 1` builds green then fails at runtime with
  `BadSqlGrammarException`; `= true` fails at build. Filter through a companion attribute instead:
  suggestions → `SuggestedMergeID IS NOT NULL`; suspect duplicates → `MastersCount > 1 and
  ConfirmationStatus = 'PARTIALLY_CONFIRMED'`. *(The Semarchy assistant's own prompt teaches the form
  `HasSuggestedMerge = '1'` — quoted, which is not the `= 1` we proved broken, but it is unverified
  here. Prefer the companion-attribute form.)*
- **Built-in match metadata is rejected in `sortExpression`**, and `MatchGroupID` does not exist in
  `GD`. Golden views expose `ConfidenceScore`, `MastersCount`, `ConfirmationStatus`, `IsConfirmed`,
  `HasSuggestedMerge`, `SuggestedMergeID`, `SuggestedMergeMastersCount`; master/source views expose
  `MatchGroupID`, `SourceID`, `PublisherID`, `Gold_<PK>`.
- **`FuzzyLookupRule` requires the alias prefix.** A bare attribute name fails the build with
  `invalid qualified name … can't access <attr>`; write `Record.<Attr>` / `Referenced.<Attr>`.
  `sortExpression` is plain SemQL — no SQL `desc`. Self-exclusion belongs in `binningCondition`
  (`Referenced.<pk> <> Record.<pk>`).
- **Hierarchy transitions** need `enableHierarchy: any <fromRoleName> has (1=1)` with
  `enableHierarchyInterpreter: SEMQL`, plus a `rootFilter` of the form `<parentRole>.<pk> IS NULL`.
- **Display text must be String-typed.** A `primaryTextExpression` pointing at a numeric, date or
  boolean attribute builds green and crashes at render (`BadSqlGrammarException`, text-vs-bigint) via
  the unmatched-FK fallback. Point at a String attribute or cast explicitly. A LOV-typed attribute
  counts as text.
- **An enriched value must belong to its target attribute's domain** — a SemQL enricher feeding a
  LOV-typed attribute must produce a value present in the LOV.
- **`lookup … end` composes** (build-validated): the terminated block embeds in larger expressions —
  comparisons (`lookup count from Edges end <= 1`), OR-combinations, several lookups in one condition.
  The failures come from SQL reflexes: `LOOKUP COUNT(X) FROM R WHERE …` (function-style aggregate,
  missing `end`) dies with `no viable alternative` / `missing T_END`.
- **Enrichers can target a reference FK**: `attributeName: FID_<toRoleName>` builds (validated) — use it
  to derive a reference from other attributes (rule-key resolution, materialized parent). Runtime
  re-pointing behavior still to confirm on a deployed model.
- **BASIC entities reject `POST_CONSO`/`PRE_POST` validation scopes** (validations AND reference
  `validationScope`) — always `PRE_CONSO` on BASIC.
- **Niladic functions require parentheses**: `CURRENT_DATE()` builds; bare `CURRENT_DATE` is parsed
  as an attribute qualified name and fails (`invalid qualified name … can't access CURRENT_DATE`).
  The CLI bundle registers it as a 0-argument function. Same expectation for `SYSDATE()` etc.
- **Date arithmetic on PostgreSQL — no `ADD_MONTHS`; `MAKE_INTERVAL` takes all 7 arguments**
  (build-validated, CLI 2.0): `ADD_MONTHS(...)` fails with `function ADD_MONTHS does not exist`, and
  `MAKE_INTERVAL(2)` fails with `expects 7 to 7 parameters`. "Today + 2 years" as a Date:
  `SEM_CAST_DATE(CURRENT_DATE() + MAKE_INTERVAL(2, 0, 0, 0, 0, 0, 0))` (years, months, weeks, days,
  hours, mins, secs). Runtime value not yet checked on a load.
- **User-defined Boolean attributes cannot be predicated bare NOR compared to a literal**
  (build-validated): `where MyFlag` and `MyFlag OR OtherFlag` fail parse (`no viable alternative`),
  and there is no boolean literal. The working form is **`SEM_CAST_STRING(MyFlag) = 'true'`** — in
  plain conditions and inside `lookup … where …` alike. To **produce** a boolean value (SemQL
  enricher targeting a Boolean attribute), wrap with **`SEM_CAST_BOOLEAN(…)`** over a string, e.g.
  `SEM_CAST_BOOLEAN(CASE WHEN lookup count from X end > 0 THEN 'true' ELSE 'false' END)` — a bare
  comparison is NOT a valid top-level expression (`mismatched input '>' expecting <EOF>`), but
  comparisons inside `CASE WHEN` parse fine. Full cast family: `SEM_CAST_STRING/BOOLEAN/INTEGER/
  NUMERIC/DATE/TIMESTAMP/UUID`.
- **Regex replace is `REGEXP_REPLACE(str, pattern, replacement)`** — `SEM_REGEXP_REPLACE` does NOT
  exist (build error `function SEM_REGEXP_REPLACE does not exist`), even though the string appears in
  the CLI bundle. Validated string toolbox: `REGEXP_REPLACE`, `TRANSLATE`, `REPLACE`,
  `SEM_SUBSTRING`, `RTRIM/LTRIM/UPPER`, `METAPHONE(x, n)`, `SEM_EDIT_DISTANCE_SIMILARITY(a, b)`
  (0-100), `SEM_NORMALIZE`.

## 6. Deliver

Give the expression, one line on why it is shaped that way, and the exact `.seml` property it goes
into. Name any attribute you could not confirm through the tools. Then build:

```bash
sem-design dm model build --root-folder "src/<Model>"
```

The build is authoritative for syntax and references — but note that several of the traps above are
**runtime** failures the build will not catch.

## Validated findings from this workshop

Measured against the CLI build or a running platform. Where one contradicts the online
documentation or your prior assumption, it wins.

- **Matching two records on a CHILD attribute ("share ANY email/phone"): nested `any … have` reaching the
  other alias through `ParentRecord`** (build-validated, CLI 2.0, with negative controls; runtime still to
  confirm). A bare `Record2` inside `any Record1.<role> have (…)` or inside a lookup's `where` is unreachable
  (`can't access Record2`) — but `ParentRecord.Record2` is: `ParentRecord` climbs back to the matcher context.
  Each nesting level shifts `ParentRecord` one level up, so the inner block compares to the outer child row:
  `any Record1.IndividualContext.EmailIndividualContext have (any ParentRecord.Record2.IndividualContext.EmailIndividualContext
  have (Email.EmailAddress = ParentRecord.Email.EmailAddress))` — multi-hop role paths are fine, and a wrong
  attribute on either side fails the build (the check is real). This supersedes the older
  `(lookup min … from Record1.… end) = (lookup min … from Record2.… end)` workaround, which compares ONE
  representative value, not "any shared". **Do not reach for `usingMatchOn` for this**: its `matchOnExpression`
  accepts only a DIRECT to-many role of the matched entity (`IndividualContext` builds,
  `IndividualContext.EmailIndividualContext` fails `can't access …`), and once set, `Record1`/`Record2` become
  the CHILD rows (the parent's attributes move to `Record1.<toRole>.<attr>`; bare `Record1.FirstName` fails) —
  it adds child-level pair matching without enabling anything the nested form cannot already express. Either
  way, the child rows must already be certified when the matcher runs (children loaded in the same batch as
  the parent masters may read empty — see `/platform-ops`).

- **SemQL has NO token-sort / word-reordering function — get order-insensitive name matching from the MATCH
  RULES, not from a normalized key** (build-validated). Cross-source person matching keeps hitting the same
  wall: source A publishes `FAMILY Given`, source B publishes `Given Family`, so a single normalized full-name
  key never collides. The instinct is to sort the tokens inside the normalization enricher — there is no
  function for it (no `SORT`, no array split/sort/join in the SemQL catalog). Working pattern: normalize the
  parts *separately* into two technical attributes (`NormalizedFamilyName`, `NormalizedGivenName`, each
  transliterated + folded + uppercased), then add a **swapped** match rule alongside the direct one
  (`Record1.NormalizedFamilyName = Record2.NormalizedGivenName AND Record1.NormalizedGivenName =
  Record2.NormalizedFamilyName`). Score the swapped rule so it needs a corroborating attribute (birth year,
  club) to reach the merge threshold — a bare swap is exactly how two different people get merged. Bonus: the
  normalized values stay readable next to the published name for a steward reviewing the merge, which a
  re-ordered string does not.
- **Transliteration before folding: letters that do not DECOMPOSE get DELETED by a fold** (build-validated;
  the defect this prevents was observed in real data). `SEM_NORMALIZE` / accent-folding handles `é→e`, `ü→u`
  — but `Æ`, `Ø`, `Å`, `ß` have no decomposition, so a fold-then-strip pipeline (`REGEXP_REPLACE(..., '[^A-Z
  ]', ' ')`) silently *removes* them: `SKJÆRBÆK` → `SKJRBK`, and two spellings of one name become two people.
  Always map the non-decomposing letters EXPLICITLY first, then fold, then upper, then strip:
  `REGEXP_REPLACE` chains for `[Ææ]→AE`, `[Øø]→O`, `[Åå]→AA`, `[Ää]→AE`, `[Öö]→O`, `[Üü]→UE`, `[ß]→SS`,
  wrapped in `SEM_NORMALIZE(...)`, then `UPPER`, then the non-alpha strip, then `REGEXP_REPLACE(x,' +',' ')`
  to collapse the gaps. Applies to any Nordic/German/Turkish source data, and it builds green either way —
  only the match results tell you it went wrong.
- **`METAPHONE` takes TWO arguments** (build-validated): `METAPHONE(<text>, <max_output_length>)` — the
  PostgreSQL signature. `METAPHONE(x)` fails the build with *"function METAPHONE expects 2 to 2 parameters
  but got 1"*. Use 8 as a sane length. (Also documented in the `semql-suggest` skill's function list.)
- **SemQL cannot touch a Boolean attribute directly — cast it** (build- AND runtime-validated): a bare boolean
  predicate (`where MyFlag`, `MyFlag OR OtherFlag`) fails parse with `no viable alternative`, and SemQL has
  no boolean literal. Filter with **`SEM_CAST_STRING(MyFlag) = 'true'`** (works in validations and inside
  `lookup … where …`); feed a Boolean attribute from an enricher with **`SEM_CAST_BOOLEAN('true'|'false')`**,
  e.g. `SEM_CAST_BOOLEAN(CASE WHEN lookup count from X end > 0 THEN 'true' ELSE 'false' END)` — a bare
  comparison is not a valid top-level enricher expression (`mismatched input '>' expecting <EOF>`) but is
  fine inside `CASE WHEN`. Cast family: `SEM_CAST_STRING/BOOLEAN/INTEGER/NUMERIC/DATE/TIMESTAMP/UUID`.
  Regex is **`REGEXP_REPLACE`** (`SEM_REGEXP_REPLACE` does not exist despite appearing in the CLI bundle).
  Runtime-confirmed on a deployed model: `SEM_CAST_BOOLEAN(CASE …)` feeds Boolean attributes correctly and
  `SEM_CAST_STRING(flag) = 'true'` filters correctly inside lookups.
- **Ordering and indenting a tree from SemQL — materialize ONE absolute path, not a level plus a sort key**
  (build- and runtime-validated). An aggregate expression forbids a nested lookup, so a `string_agg` over an
  edge/placement entity cannot compute each line's depth for itself; a **stored** attribute, however, is
  accepted **both** inside the aggregate expression and in the lookup's `order by`. So materialize, with an
  enricher **on the edge entity** (where lookups are allowed at the top level), the node's **absolute path
  from the root**: `COALESCE(<gt-grandparent> || '/', '') || … || COALESCE(<parent> || '/', '') || <own code>`,
  the ancestors resolved at fixed depths (`lookup max <parentCode> from <upPath(k)> … end`). One string then
  carries everything: sorting it **alphabetically** yields depth-first pre-order (a parent's path is a strict
  prefix of its children's, and `/` sorts before any letter), and the **depth is the separator count** —
  `LENGTH(p) - LENGTH(TRANSLATE(p, '/', ''))` (`TRANSLATE` with an empty `to` deletes the character), which
  feeds `REPEAT('    ', …)` for the indentation. Do **not** store a numeric level alongside it, and do not
  build a shifted fixed-slot sort key: both are redundant, and the slot construction is unreadable and breaks
  the moment the depth cap changes. Caveats to document on the attribute: it is computed over **all** edges,
  **not as of a date**, and siblings order by **code**, not by any sort-order attribute.
- **Comparing two structures (or any two collections) - materialize a delimited SET, then test containment**
  (build- and runtime-validated, platform 2.0). SemQL cannot correlate two collections, so "does this row of A
  exist in B?" has no direct form. It has an indirect one: store on each container a **delimited set** of its
  members' keys (`'|' || string_agg(<key>) separated by '|' || '|'`), then ask
  `STRPOS(<other container>.<set>, '|' || <key> || '|') > 0` from inside a lookup over the members - the
  wrapping delimiters are what make it an exact membership test instead of a substring accident. Three things
  it lives or dies by: **(1) the key must be the SHARED identity, not the local occurrence** - comparing two
  trees on a per-tree node code reports every line as different (measured: 0 matches between two hierarchies
  whose node codes embed the hierarchy), so materialize a second path built from the underlying business
  object's code; **(2) the correlation back to the current record must be a TO-ONE path from the lookup
  target** (`<child> -> <container> -> <peer> -> <set>` builds - two hops are fine), so the diff is markable
  only in the direction where such a path exists; the reverse side needs the pairing recorded on both
  containers, while the COUNTS stay exact either way (|B| - |A inter B|, the intersection being identical
  measured from either side when the key is unique per container); **(3) a lookup NESTED inside another
  lookup's expression breaks the outer context** (`invalid qualified name: <outer attr>`), so the peer's set
  must be a STORED attribute, never a computed one. Comparing full paths and comparing bare keys is what
  separates *moved* from *added/removed*: two sets, one extra attribute, and the diff classifies itself.
- **A SemQL lookup CANNOT correlate with the current record, and its source must be a ROLE PATH**
  (build-validated, CLI 2.0.0 — the finding that decides every "is X its own ancestor / is this a duplicate
  of me" rule). Inside `lookup … where …` *every* name resolves against the lookup **TARGET**, and there is
  no alias for the outer record: a bare outer attribute (`NodeID`), `Record.<attr>` and `<Entity>.<attr>` all
  fail with `invalid qualified name … can't access …` (the outer path is silently prefixed onto the target
  path). The `Record.` / `Referenced.` aliases exist **only** in the `recordReference` context of a
  `FuzzyLookupRule` (score, `binningCondition`, `referenceFilter`) — do not reach for them elsewhere.
  Likewise `lookup … from <Entity> end` is rejected (`can't access <Entity>`): a lookup ranges over a
  **navigation path**, never over all records of an entity. Consequences to plan for, not to fight:
  **(1)** a cycle cannot be detected by comparing an ancestor's id with the current record's — count the
  **length of the ancestor chain** instead (§ *Fixed-depth hierarchy navigation* in the generation prompt);
  **(2)** a "unique across the entity" rule (at most one active default/primary/preferred row) is not a
  validation at all — materialize the condition into a **NULL-free token attribute** with an enricher and put
  a **`UniqueKey`** on it.
- **The LINEAGE paths are valid lookup sources — `lookup … from MasterRecords end` computes SOURCE COVERAGE from a
  GOLDEN context** (build-validated, CLI 2.0.0; runtime to confirm on a deploy). The reserved to-many paths the
  lineage transitions use (`MasterRecords`, and `SourceRecords` from a master level) are navigable by SemQL like
  any role, and the master-level built-ins resolve **inside** the block — so `PublisherID` tells you *which
  systems feed a golden record*. That is how to answer "records known only from source X" **dynamically**,
  instead of the usual proxy heuristic on an attribute only X fills (which mislabels every record where that
  attribute merely happens to be empty):
  `lookup count from MasterRecords where PublisherID = 'X' end > 0 and lookup count from MasterRecords where PublisherID <> 'X' end = 0`.
  Validated in a `BusinessEntityBuiltInFilter.semQlCondition` **and** in a collection-column `value` — e.g. a
  *contributing sources* column
  `lookup string_agg (PublisherID) separated by ', ' from MasterRecords order by PublisherID end`, which shows
  the steward *why* a record is in the queue. Both really are validated at build: a wrong attribute inside the
  block fails with `invalid qualified name: MasterRecords.<x>`. Applies to any matched entity (FUZZY_MATCHED or
  ID_MATCHED), and the population re-evaluates itself on every load — no stored flag to maintain.
  **Also usable in a `NamedQuery`** (build-validated, CLI 2.0): an `ObjectProperty` `type: MULTI_RECORDS`,
  `semQlExpression: MasterRecords`, whose `targetObject` is a `QueryObject` on the **same entity** exposing
  `PublisherID`, `SourceID` and the entity's attributes — returns the golden record's masters (runtime to confirm).
  **Scope limit, build-validated: the lineage arrays exist ONLY in the VIEW contexts.** The CLI injects them
  through `addMastersLineage`, called by the GD / GE / GH4B view builders alone — so `MasterRecords` is absent
  from the MD and SD contexts, and it is rejected outright in a **DisplayCard** expression and in a
  **SemQLEnricher** (`invalid qualified name: MasterRecords, can't access MasterRecords`). Consequences:
  a coverage signal can be a business-view filter or a collection column, but **never an icon, a form field or a
  stored attribute** by this route — see the stamp-and-survive pattern in `/seml-model` — and putting such a lookup on the
  entity's DEFAULT display card would break the lineage tabs, the Duplicate Manager and the stepper, which all
  render masters and sources through that same card.
  Engine support for the safe shape: **`cfg["browse_cards"] = {entity: {avatar (SemQL -> image URL),
  secondary?, primary?, description?, columns?}}`** emits a SECOND card `<E>BrowseDisplayCard` plus a
  `<E>BrowseCollectionView`, and wires them to the entity's own `<E>BusinessView` **only** — the flat worklist
  view (which also serves the `SDWE`/`SAWE` error views, i.e. source records) and every other consumer keep the
  default card. Its `columns` land on that browse collection, where golden-context expressions are legal;
  **`cfg["extra_columns"] = {entity: [col, …]}`** appends columns to the DEFAULT collection instead and must
  therefore stay context-safe (plain attributes, or the master-level built-ins `PublisherID`/`SourceID`, which
  read empty on a golden row).
- **SemQL `lookup` is a block with a mandatory `end`, and the block COMPOSES** (build-validated):
  `lookup <agg> [<expr>] from <ChildRole> where <cond> end` — e.g. `lookup count from Edges end`,
  `lookup sum Amount from Lines where Status = 'OPEN' end`. SQL-style `LOOKUP COUNT(X) FROM R WHERE …`
  (parenthesized aggregate, no `end`) fails with `no viable alternative` / `missing T_END`. Once terminated
  with `end`, the block embeds in a larger expression: comparisons (`lookup count from R end <= 1`),
  OR-combinations, and multi-lookup conditions all build. Inside the block, attributes resolve against the
  TARGET entity (do not repeat the role name). **The aggregate `<expr>` itself composes too** (build-validated,
  CLI 2.0.0): a full `CASE WHEN … THEN … END` block — string concatenations inside its branches included —
  is accepted as the lookup expression (`lookup max CASE WHEN Status = 'X' THEN '…' || SEM_CAST_STRING(D) END
  from Role where <cond> end`); the parser separates the CASE's `END` from the lookup's `end`. This is what
  makes a **one-lookup-per-cell status matrix** possible (each cell computes its styled chip + date INSIDE
  the lookup, `COALESCE(…, '<not collected>')` outside) instead of 2–3 lookups per cell. **Niladic functions need their parentheses**:
  `CURRENT_DATE()` builds, bare `CURRENT_DATE` is parsed as an attribute and fails.
- **SemQL functions = the PostgreSQL dialect catalog embedded in the CLI, not Oracle's** (build-validated
  v1.11.0). `CURRENT_DATE`, `SYSDATE` (parsed as qualified names → `invalid qualified name`) and
  `ADD_MONTHS` (`function does not exist`) are all **rejected**; the catalog lives in the CLI bundle
  (`out/cli.js`, searchable) and is PostgreSQL-shaped. Validated working: **`NOW()`**,
  **`MAKE_INTERVAL(y, m, w, d, h, min, s)`** (7 args, e.g. *age ≥ 18* =
  `DateOfBirth <= NOW() - MAKE_INTERVAL(18, 0, 0, 0, 0, 0, 0)`), and `ASCII`/`SUBSTR`/`TRANSLATE`/`MOD`/
  `INITCAP`/`NULLIF` — enough to write a **pure-SemQL NPI/Luhn check-digit validation** (see the ProviderMDM
  `NpiFormat` pattern: digits-only via `TRANSLATE(x,'0123456789','') = ''` + positional `ASCII(SUBSTR(...))-48`
  arithmetic; validated at build AND at load — an invalid check digit blocks the record pre-conso). When a
  SemQL function is uncertain, grep the CLI bundle for its name before guessing Oracle equivalents.
