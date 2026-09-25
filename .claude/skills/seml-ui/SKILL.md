---
name: seml-ui
description: "Rules and validated findings for the UI layer of a Semarchy `.seml` data product. Invoke BEFORE creating or editing a form, a business view or a transition, a collection view, a display card, a stepper, an action set, the application navigation, the model diagram or the branding images — and whenever a tab is missing or duplicated, a queue looks suspiciously short, a collection fails to open, or an icon renders wrong. Several findings here are build-green and runtime-broken."
---

# Authoring the screens of a Semarchy data product

Everything here was validated against the CLI build, the embedded schemas or a running application.

Entities, attributes and rules are the sibling skill `/seml-model`. SemQL expressions are `/semql-suggest`. Deploying and loading are `/platform-ops`.

- **Navigation DOMAINS — `cfg["app"]["domains"]`** (engine capability). The default drawer is one
  "Master data" folder plus a "Reference data" subfolder, which stops reading as soon as a model carries
  **two unrelated subject areas**: everything lands in one folder and the drawer no longer says what the
  application is about. `domains` replaces that layout with several top-level sections:
  `[{_name, label, entities:[…], icon?, color?, description?, subfolders:[{_name,label,entities,icon?,color?}]}]`
  — unknown entity names raise. Give each domain its own colour and a description, and set
  `nav_labels_visible: True` so the section titles actually show. Stewardship and Administration stay
  transverse: splitting the work queues per domain fragments them for no benefit. Where a model carries
  two hierarchy architectures (trees as DATA vs trees as MODEL), the domain split is also the honest
  place to draw that line.
- **A filtered browse action on a HIERARCHICAL business view shows only the ROOTS of the tree**
  (runtime-validated, and the nastiest kind of defect: nothing fails, the queue is just quietly
  wrong). A hierarchical view carries `rootFilter: <parentRole>.<pk> IS NULL`, and a
  `BrowseBusinessViewAction`'s `appliedFilter` is **ANDed** with it. Measured: a "customers with no
  link" worklist over 5 matching records displayed **1** — the only one that happened to be a root of
  the account structure. This hits **every** filtered stewardship action (worklists, inline matching
  review, `SDWE`/`SAWE` error views) on any entity listed in `cfg["hierarchy_views"]`. Fix: point
  those actions at a **flat companion view** — same collection, same form, `enableHierarchyView:
  false`, **no `rootFilter`**, and drop the self-recursion transition (it only exists to draw the
  tree). The engine now emits `<E>WorklistBusinessView` automatically for any hierarchical entity that
  carries worklists, error views or an inline matching review. Reflex when a queue looks suspiciously
  short: compare its count against the same condition run through a `NamedQuery`, which has no root
  filter — a named query with the candidate conditions as `VALUE` columns is the cheapest way to see,
  row by row, what each expression really evaluates to.
- **Data-quality WORKLISTS — `cfg["app"]["worklists"]`** (engine capability). The generic shape of "here is
  the queue of records a steward must still deal with": each entry emits a named `BusinessEntityBuiltInFilter`
  on the entity's business view **and** a `BrowseBusinessViewAction` in the Data-stewardship folder pointing
  at it. `[{entity, _name, label, condition (SemQL), description?, icon?, color?, nav?}]`; **`nav: False`**
  emits the filter only, for a browsing aid that is not a queue. Distinguish the *kinds* of missing data
  rather than lumping them: "no identifier supplied" and "an identifier that resolves to nothing" are
  different corrections and belong in different worklists.
- **Form-field label icons & markdown fields (build- AND runtime-validated, CLI 2.0.0).** Every form
  field type accepts a **`labelProperties`** block (schema-confirmed, distinct from
  `componentProperties`): `icon` (image URL, e.g. `images://mdi/email.svg` — used **instead of** the
  label), `iconColor` (e.g. `md:blue`), `isVisible` (label visibility, `'0'`/`'1'` +
  `isVisibleValueInterpreter: LITERAL`), `alignment`, `color`, `isFloating`. **Runtime layout rule:
  an icon field COLLAPSES the label column** — the icon renders directly against the value, ignoring
  the tab's `labelWidth` and `alignment`, so mixing icon fields and text-label fields in the same
  row grid breaks the column alignment. Use icons only in rows composed **entirely** of icon fields
  (e.g. an Email | Mobile contact row, a full-width address line); keep text labels everywhere else.
  **Doctrine (user-validated): do not over-use icons in place of labels** — only where the meaning is
  instantly obvious to everyone (email, phone, address pin); statuses, codes, dates and LOV values
  keep their label; in doubt, keep the label. Full guidelines: generation prompt § UI.
  And **`MarkdownField` is a real field type**: `useCustomLabel`+`label`, `dataType: STRING`, SemQL
  `value`. **Its renderer accepts inline HTML (runtime-validated)** — a computed **HTML `<table>`
  with inline styles** (border-collapse, per-row `border-bottom`, cell padding) renders correctly,
  which plain markdown pipes cannot style (they render border-less). Composed `lookup … end` blocks
  (24 in one expression) and `CHR(10)` both work in the value. Pattern: a consent/status matrix
  (rows × columns of `CASE WHEN lookup count … end > 0`) in the record's form, with the field label
  hidden (`isVisible: '0'`) so the section title carries the heading and the table takes full width.
- **Embedded address map in a form** (`ObjectField`, `componentProperties.sourceType: url`) — two traps,
  both runtime-validated. **(1) Embed a MAP, not a maps website**: the keyless **Google Maps embed**
  `'https://maps.google.com/maps?q=' || <free-text address, spaces TRANSLATEd to '+'> || '&z=15&output=embed'`
  renders the bare map alone; the Nominatim UI page (`nominatim.openstreetmap.org/ui/search.html?q=…`)
  renders its ENTIRE search page in the slot (header, search box, result panel — defect seen in app), and
  the OSM `export/embed.html` endpoint accepts only coordinates, unusable from address attributes.
  **(2) Put the address fields BESIDE the map, not above it**: the engine's `form_layout` section entry
  accepts `(label, {"columns": [[field, …], [field, …]]})` — one row of COLUMN containers (e.g. the stacked
  street/postal/city/country fields in the left column, the `ObjectField` map alone in the right) instead
  of full-width field rows pushing the map down.
- **Drawing a graph from SemQL — a DOT diagram passed in a URL** (runtime-validated). Same pattern as the
  embedded map above: an `ObjectField`/`MarkdownField` whose value is a URL built in SemQL, drawn by a public
  service (`https://quickchart.io/graphviz?graph=digraph%7B…%7D`), so nothing has to be hosted — which
  matters in SaaS. Three rules learned the hard way: **(1) a node's IDENTITY in DOT is its quoted string**, so
  everything variable (dates, counts, status) must go into a **separate node statement**
  (`"<code>"[label="…"];`) rather than into the identity — otherwise the same node drawn once as a parent
  and once as a child becomes two boxes and the tree falls apart. Emit that statement from the edge where the
  node is the CHILD: exactly once per node. **(2) Sanitize everything that reaches the URL** —
  `TRANSLATE(REGEXP_REPLACE(<expr>, '[^A-Za-z0-9 ._-]', '', 'g'), ' ', '+')` (spaces last: a query string
  decodes `+` back to a space, which restores the words in the drawn label), and pre-encode every DOT
  metacharacter (`%22` `"` · `%3B` `;` · `%5B%5D` `[]` · `%3D` `=` · `%2C` `,` · `%7B%7D` `{}` · `-%3E`
  `->` · `%5Cn` for a label line break). **(3) Budget the attribute length** — the whole graph travels in the
  URL, so size the target attribute against the largest expected structure (~120 chars per edge with a label;
  a 24-edge tree lands near 2.8 kB) and prefer short business codes as identities. Use `rankdir=TB` and a
  small `fontsize` for anything deeper than two levels.
- **Form-field widget overrides — `cfg["field_widgets"] = {entity: {attr: <widget>}}`, where `<widget>` is
  either a field-type string or a dict** `{"type": <FieldType>, plus any field property: dataType, label,
  useCustomLabel, componentProperties…}`. The dict form is what a `Binary` attribute needs (see next bullet).
  Getting this wrong produces a form element with **no real `_type`**, and the build dies with the unhelpful
  **`Cannot read properties of undefined (reading '_type')` in `FormValidator.exploreFormElements`** — that
  error signature always means *a form element whose `_type` is missing or is not a field-type string*.
  `cfg["computed_fields"]` entries likewise accept `componentProperties` (and `dataType`), needed for an
  `ObjectField` that embeds a URL (map, viewer).
- **Binary attributes** (`dataType: Binary`; images, logos, scanned documents): form widget by use —
  an **image** → `ImageField` (`dataType: BINARY`); a **document/file** (PDF…) → `ObjectField` with
  `componentProperties: {sourceType: content, sourceTypeValueInterpreter: LITERAL, objectHeight: '<px>'}`.
  To use a binary attribute as the **display-card avatar**: `avatarImageValue: <attr>`,
  `avatarImageValueInterpreter: SEMQL`, `avatarImageSourceType: CONTENT` (the avatar has **no** fallback
  field). Load/read via REST as **base64 strings** in the Persistable map (round-trips to raw bytes;
  verify by decoding a `GET …/query/{DL}/{E}/GD` value — PNG/PDF magic bytes).
- **Enrichers/validations must be wired into the entry stepper's `FormStep`** to run during authoring
  (`formStepEnrichers[]` / `formStepValidations[]`; no `_type`). Triggers: `executedOnFormOpen` /
  `executedOnDataChange` = **proactive**, `executedOnButtonClick` = **manual** (enrichers only). Policy:
  SemQL enrichers → proactive; **external plugin/REST enrichers → manual** (avoid hammering the service);
  validations → proactive (open + change). `formStepValidations` needs `validationType` (`CHECK` for
  SemQL) + `validator`. The engine's `stepper()` applies this automatically.
- **Complete steppers of business objects with child/grandchild creation** (not reference data, not
  lookups). Owned children = references with `deletePropagation: CASCADE` whose `toEntity` is the parent
  (recurse for grandchildren); self-references add one nested level (sub-sites, subsidiaries); `RESTRICT`
  to-one lookups are **never** child steps. Tree is `CollectionStep → FormStep (+ sibling child
  CollectionSteps)`; a **`FormStep` has no `childSteps`** — child collections are siblings of the form.
  A child `CollectionStep` binds via **`oneToManyReference`** and needs the full `enableChild*` /
  `manyToMany*` property set; step `_name`s unique per stepper. Engine builds it for `cfg["core"]` when
  `cfg["stepper_children"]` is on. Details in the generation prompt (§ Complete steppers with child
  creation).
- **A `BusinessViewTransition` of type `EMBEDDED_COLLECTION` requires a `formCollection`** (build-validated:
  `Missing key 'formCollection' referencing a FormCollection`) — it renders THROUGH a `FormCollection` of the
  parent form. For a plain child-collection tab that needs no form-embedded collection, use
  `transitionType: FULLY_QUALIFIED` with the tab property set (pluralLabel + displayFormTabAs + iconUrl +
  visibleInFormTabCondition), which is what the engine's `bv_transitions` shorthand emits.
- **`BusinessViewEntityReference` resolves ONLY the root BVE of the SAME business view** (build-validated):
  a nested BVE is not addressable (`Unknown reference`), and a ref into another view fails with
  `Illegal reference: … is not a child of <this view>`. Consequences: unlimited-depth recursion works only
  when the recursive level IS the view's root entity; a tree alternating two entities (A→B→A…, e.g.
  container→edge→node) **cannot recurse** — the only workaround is a chain of INLINE `BusinessViewEntity`
  levels, one per depth (each inline level renders exactly one depth). Also note: a transition
  `transitionPath` may be a **dotted path through a to-one role onto a to-many role**
  (`ChildNode.ParentPlacements` builds), and transition `filter`s cannot correlate with the parent record.
- **Hierarchical Business View needs the self-transition flagged into the tree, not just the view flag**:
  `enableHierarchyView: true` shows no child level unless the downward self-reference is a
  `BusinessViewTransition` (`transitionPath` = the reference's `fromRoleName`, `transitionType:
  FULLY_QUALIFIED`, target = same entity) **that also sets `enableHierarchy: any <fromRoleName> has (1=1)`
  + `enableHierarchyInterpreter: SEMQL`** (the "appear in hierarchy" flag — a SemQL condition; omitting it
  keeps the branch OUT of the tree, so it looks empty). Add `transitionVisible: '0'`/`LITERAL` so children
  sit directly under the parent, and a `rootFilter: <parentRole>.<pk> IS NULL`. **For unlimited depth,
  point the transition's `businessViewEntity` at the root level via `BusinessViewEntityReference`
  (`ref: <BVpackage>.<BVName>.<rootBveName>`) rather than an inline copy — an inline `BusinessViewEntity`
  renders only ONE child level.** The engine auto-adds all this for self-referencing entities in
  `hierarchy_views`. Details in the generation prompt (§ Recursive hierarchies).
- **Where a tab belongs — business view for related records, form for attribute groups** (user-validated
  doctrine, engine-enforced; fixes two build-green defects reported from 2026-08 generations: the **same
  child rendered under two tabs** — a form-collection tab *and* a business-view transition tab, same label,
  same content — and **tabs with neither label nor icon**). Decide ownership before laying out any UI:
  | The tab shows | Owner | Emitted from |
  |---|---|---|
  | **related records** — children, memberships, lineage/history, where-used (i.e. a **transition**) | the **business view** | `cfg["bv_transitions"]`, `lineage_views`, `hierarchy_views` |
  | **a group of the record's own attributes** (split a long form instead of scrolling vertically) | the **form** | `cfg["form_layout"]` sections / extra `FormTab`s |
  | **an embedded component** — dashboard, map, custom widget, computed matrix, or a table explicitly wanted inside the form | the **form**, on **explicit request only** | `cfg["computed_fields"]`, `cfg["form_collections"]` |
  Rules: (1) a to-many relationship reaches the record detail as a **transition**, never as a duplicate
  form-collection tab — the engine **fail-fasts** when the same `path` is declared in both
  `cfg["form_collections"]` and `cfg["bv_transitions"]`, and when two transitions of one business-view
  level open the same path; (2) **no bare tabs**: the tab label comes from **`pluralLabel`** (not `label`),
  and the engine now emits `pluralLabel`+`displayFormTabAs`+`displayTransitionNodeAs`+`iconUrl`
  (target entity's icon, or `icon`) + `visibleInFormTabCondition` **by default** for every transition —
  `tab: True` is no longer needed and re-declaring these properties is pointless; (3) the single opt-out is
  **`tab: False`**, for when the form was explicitly asked to own that tab; (4) **`tab_condition`** overrides
  the default `any <path> has (1=1)` (needed for a dotted path) or omits it with `False` — but note that a
  **TO-ONE `transitionPath` is rejected outright, whatever the condition** (build-validated, CLI 2.0.0): a
  `BusinessViewTransition` whose path is the to-one role of a reference (`Order` -> its `Customer`) fails with
  `invalid qualified name: <Role>, your expression must contains an array`, and `tab_condition: False` does
  NOT rescue it — the constraint is on the path itself, which must be to-MANY. Reach the parent from a child
  through the reference FIELD on the form, not through a transition; (5) a form
  tab holding **only** a collection competes with a transition tab, a collection **embedded in a section
  next to attribute fields** does not (it renders inline — the supported way to put a child table inside an
  attribute tab). The engine's end-of-run **`Tab check`** (`cfg["tab_check"]`, on by default) reports
  duplicate / unlabelled / icon-less / bare tabs across the generated UI **and** hand-authored forms and
  business views passed through `extra_objects` — treat any finding as a generation defect.
- **Coloured collection cells — `componentProperties` on a table column** (schema-confirmed in the CLI
  mappers, build-validated CLI 2.0.0). A `TextColumn`/`DateColumn`/`ChipsColumn`/`CheckboxColumn` accepts
  **`textColor`** and **`cellBackgroundColor`**, each with its own `…ValueInterpreter` (`LITERAL` by
  default, **`SEMQL`** for a computed colour). So a status column colours itself from its own value:
  `{"componentProperties": {"textColor": "CASE WHEN Status = 'DELIVERED' THEN '#1b5e20' … END",
  "textColorValueInterpreter": "SEMQL", "cellBackgroundColor": "CASE WHEN … END",
  "cellBackgroundColorValueInterpreter": "SEMQL"}}` — the readable way to show a fulfilment status, a
  consent answer or a loyalty tier in an embedded child table. `ChipsColumn` additionally takes
  `chipMaxWidth` / `useAvatarInChip` / `useSecondaryTextInChip`, and `TextColumn` a `displayFormat`.
  (Emitted today through `cfg["extra_objects"]` — there is no per-column colour key in the engine yet.)
- **Non-flat forms**: beyond ~6 attributes, split into meaningful **business sections**
  (identification / characteristics / origin / pricing / lifecycle…), not mechanical chunks; isolate
  technical/derived attributes in their own section; `CheckboxField` without `textAlign`.
- **Reference display text must be String-typed** (`DisplayCard.primaryTextExpression`, and any
  entity label expression used to render a `ReferenceField`). When a foreign key is **unmatched**, the
  render query falls back to the **raw FK code (text)**, so the display expression is emitted as a
  `CASE` mixing the display value with that text. If the display attribute is numeric/date
  (`LongInteger`, `Decimal`, `Date`…), PostgreSQL rejects the mixed-type `CASE` at **runtime** —
  **builds green, then fails on form/collection open with `BadSqlGrammarException`** (`… then
  BID.<code> else ACCST.<numericAttr> …`, text vs bigint). Fix: point `primaryTextExpression` at a
  **String** attribute (the name/label), or a SemQL text expression / explicit cast — never a bare
  numeric/date attribute. **Watch the fallback on an entity with no text attribute at all** — a pure
  relationship entity (two references + a flag + a validity period) has none, so the first-attribute
  fallback lands on a **Boolean**, which is the same runtime crash. That is exactly the case for
  `cfg["display_text"]`: name the association by its two sides
  (`<RefRole>.<Name> || ' / ' || <OtherRole>.<Label>`, each traversal ending on a scalar). The engine's `display_card()` prefers a String attribute automatically, and
  **counts a LOV-typed attribute as text** (its `dataType` is the LOVType QN but it is stored as a STRING, so
  a type/category label is a perfectly good primary text — excluding it silently demotes the card to the next
  free-text attribute, typically a note or an account number). When the best display text is conditional
  (a party that is either a person or an organization, so the name lives in different attributes), set
  **`cfg["display_text"] = {entity: "<SemQL text expression>"}`** rather than accepting the first-attribute
  fallback — a card pointing at `OrgName1` renders **blank** for every person record. Two more display-card
  levers the engine emits (schema-confirmed): **`cfg["display_secondary"]`** = {entity: SemQL text
  expression} → `secondaryTextExpression` (same String-typed/COALESCE rules as the primary), and
  **`cfg["avatar_expression"]`** = {entity: SemQL expression returning an image URL} → a **conditional
  avatar** (`SEMQL` + `URL`, e.g. a `CASE` over a type/status code mapping to `images://<Lib>/<file>.svg`);
  runtime-checked: golden collections load fine with it deployed (a 2026-08-21 "collection fails to
  load" report against it turned out to be a user-confirmed false alarm — when a render breakage is
  suspected, get the exact app error BEFORE reverting a feature). Keep the CASE total (`ELSE <default
  url>`) — the avatar has no fallback — and remember it wins over `avatar_attribute` when both are set.
- **`BusinessView` has two MANDATORY properties that are easy to forget**: **`enableHierarchyView`** and
  **`useHierarchyOnOpen`** (both boolean). Omitting them fails the build with a bare
  `Missing property "enableHierarchyView"` / `"useHierarchyOnOpen"` (type: SCHEMA) — so any hand-written or
  newly generated business view (a stewardship / matching-review view, for instance) must set them even when
  the view is not hierarchical (`false`/`false`).
- **Record lineage applies to `ID_MATCHED` entities too, not only `FUZZY_MATCHED`.** An ID-matched entity has
  master and source records exactly like a fuzzy one — only the *match* metadata (confidence, suggested merges)
  is fuzzy-specific. Restricting the lineage transitions to fuzzy entities silently leaves an ID-matched hub
  without its `MasterRecords` / `SourceRecords` / history tabs. The engine's `cfg["lineage_views"]` accepts both.
- **Diagram**: every reference appears in **exactly one** `referenceLink`; final check
  `references = referenceLinks`. A `referenceLink` carries **only** `reference` (+ optional
  `bendPoints`) — endpoints are resolved from the reference's `fromEntity`/`toEntity`.
  `fromEntityShape`/`toEntityShape` are **obsolete/deprecated — never emit them**. Every participating
  entity needs an `EntityShape`. **A SELF-reference's link MUST carry `bendPoints`** — with an empty
  list both endpoints sit on the same shape and the loop renders collapsed/illegible; route at least
  two bend points into the empty space beside the entity's shape (the engine now emits them
  automatically for any reference whose `fromEntity` = `toEntity`).
- **Diagram layout — size-aware**: `EntityShape` height grows with **attribute count**, width with the
  **longest attribute label**; a uniform grid overlaps. Isolate tall hub entities (fuzzy master,
  15–20+ attributes) in their own vertical band (left "hub"/star layout, nothing above/below); give
  wide entities (long labels) extra horizontal clearance; place each entity next to its real
  neighbours and keep loosely-connected ones out of the link crossfire; space ~480–600 px
  (columns) / ~450–520 px (rows), more for large entities; route self-loop `bendPoints` into empty
  space. Rebuild + reload the diagram tab to confirm no overlaps.
- **Application**: root folder → per-domain subfolders (core entities directly, reference data in a
  subfolder); consistent colors/icons per folder; readable shades on a white background.
- **App branding images (logo / avatar / cover SVGs) — design for the slot, runtime-validated.** Two traps
  that build green and only show in the running app: **(1) SVG gradients are NOT rendered** in the app's
  image slots — any `fill="url(#…)"`/`stroke="url(#…)"` falls back to **BLACK** (a black logo circle, a black
  cover panel). Use **flat fills only** (layer solid colors with `opacity` for depth); avoid `defs`/`url()`
  references entirely. **(2) Each slot has its own geometry and CENTER-CROPS the image** — design to the
  slot, not to a generic banner: the nav-drawer **cover** slot is ≈ 250×110 (≈ 2.2:1), so a 4:1 banner loses
  both edges (a left-anchored title renders truncated, e.g. "…rovider Hub"); keep a cover near **2.2:1**
  with all critical content (title, subtitle) **centered in the middle ~55% of the width**. **Logo and
  avatar** may be cropped to a **circle** in headers: use a compact square (~96–128 px) with the motif
  centered and generous margins, no text. After changing assets, re-zip (Python `zipfile`, see Platform CLI
  guide) and re-import the library — images are runtime artifacts, no model redeploy needed.
- **Data stewardship navigation** (error views + duplicate/matching review): own `FolderNavGroup` +
  root `FolderAction`. A `BrowseBusinessViewAction` chooses the data view via **`viewType`** ∈
  {`GD`,`GDWE`,`MD`,`SD`,`SDWE`,`SA`,`SAWE`} — **error views = `SDWE` for matching entities**
  (fuzzy/ID-matched) and **`SAWE` for basic entities** (reuse the existing business view; `viewType`
  only switches the view). **`appliedFilter` is a REFERENCE to a named
  `BusinessEntityBuiltInFilter`** (`{_name,label,semQlCondition,visible}` on the view's
  `rootBusinessViewEntity.businessEntityBuiltInFilters`), **not** a raw SemQL string (inline → "Unknown
  reference"); reference it by QN `<BV>.<rootBve>.<filterName>`. **Native match metadata** (SemQL-usable
  in collection-column `value`/`semQlCondition`/filters): golden views expose `ConfidenceScore`,
  `MastersCount`, `ConfirmationStatus` (`CONFIRMED`/`NOT_CONFIRMED`/`WAS_CONFIRMED`/`PARTIALLY_CONFIRMED`),
  `IsConfirmed`, `HasSuggestedMerge`, `SuggestedMergeID`, `SuggestedMergeMastersCount`; master/source
  views expose `MatchGroupID`, `SourceID`, `PublisherID`, `Gold_<PK>`. These built-ins are **rejected in
  `sortExpression`** and `MatchGroupID` is **not** in `GD`. **They are also reserved as attribute NAMES**:
  naming a user attribute `ConfidenceScore` (validated), and by extension any of these built-ins, fails the
  build with `The name '<X>' is a reserved name` — use a distinct name (e.g. `MatchConfidence`). **Boolean-flag trap: `HasSuggestedMerge` and
  `IsConfirmed` are physically boolean** (docs call them 0/1) — never filter with `= 1` (builds but fails
  at runtime with `BadSqlGrammarException`) or `= true` (fails at build — SemQL has no boolean literal);
  filter through a companion attribute instead (`SuggestedMergeID IS NOT NULL` for suggestions, the string
  `ConfirmationStatus = 'PARTIALLY_CONFIRMED'` for suspect duplicates). Build- & runtime-validated review
  filters: *Suggestions* = `SuggestedMergeID IS NOT NULL`; *Suspect duplicates* = `MastersCount > 1 and
  ConfirmationStatus = 'PARTIALLY_CONFIRMED'`. These views apply to **fuzzy** entities only. **Do not add a
  "Duplicate Manager" nav action** — there is no such app-action and a `GD` browse action just duplicates
  the entity's normal golden view; the native Duplicate Manager opens *from within* that standard view
  (and the matching-review views). So the stewardship group holds only the **Errors** + **Matching review**
  folders (error views = `SDWE` matching / `SAWE` basic). **The engine emits the Matching-review folder
  automatically** (validated): list fuzzy entities in `cfg["app"]["matching_review"]` → each gets two named
  `BusinessEntityBuiltInFilter`s (`SuggestedMerges`/`SuspectDuplicates`) on its `<E>BusinessView`
  `rootBusinessViewEntity`, and the app gains a **Stewardship** `FolderNavGroup` + root folder with a
  **Matching review** subfolder of `BrowseBusinessViewAction`s (`viewType: GD` + `appliedFilter` → the named
  filter, referenced by QN `<E>BusinessView.<E>.<filterName>`). Non-fuzzy entities in the list are skipped;
  an empty/absent list emits no stewardship group. Labels overridable via `app.stewardship_label` /
  `app.matching_review_label`.
- **Record-lineage & history navigation = `BusinessViewTransition`s with reserved `transitionPath`s**
  (`transitionType: FULLY_QUALIFIED`, `businessViewEntity` points back to the **same entity** with its
  **default** collection/form/card — a golden-only stewardship collection is invalid on `MD`/`SD`/history).
  Reserved paths: **`MasterRecords`** (golden→master), **`SourceRecords`** (master→source, nested under
  MasterRecords), **`MasterHistoryRecords`** (master→master history, nested under MasterRecords),
  **`GoldenHistoryRecords`** (golden→golden history, sibling of MasterRecords). Give each `filter: 1 = 1`
  + `displayFormTabAs: ICON_LABEL` + `useCustomLabel: true` with **both `label` and `pluralLabel`** (the
  tab label comes from `pluralLabel` — omit it and the lineage tabs show no label); unique `_name`s **per
  business view** (prefix per occurrence). **Every transition renders a detail tab** and
  `displayFormTabAs` has no "hidden" value (`ICON`/`ICON_LABEL`/`LABEL` only), so **every** transition
  carries `displayFormTabAs`+`iconUrl`+`pluralLabel`+`visibleInFormTabCondition` — no exception, no
  "absorbed, left bare" case (that earlier guidance is withdrawn: it produced tabs with neither label nor
  icon, and duplicated tabs when a form-collection tab covered the same path — see § *Where a tab
  belongs*). The engine emits the four properties by default. **Add
  the block on the matching entity at every level of every BV** (root AND nested targets). History
  transitions need the data historized: `GoldenHistoryRecords`→`historizeGolden: true`,
  `MasterHistoryRecords`→`historizeMaster: true` (extra storage); app `historyBrowsingEnabled: true`
  (+ optional `historyBrowsingRequiredRole`) enables history browsing; history records from the next load
  on, not retroactively. Pattern reference: CorporateFinance accelerator `Counterparty` BV. **The engine now
  emits this automatically** (validated): list a fuzzy entity in `cfg["lineage_views"]` → it gets
  `MasterRecords` (with `SourceRecords` nested), and `MasterHistoryRecords`/`GoldenHistoryRecords` are added
  from the entity's `historizeMaster`/`historizeGolden` flags. Default tab labels (Master/Source/Master
  history/Golden history) are overridable per entity via `cfg["lineage_labels"]` (`{ename:{transitionName:label}}`);
  app history browsing via `cfg["app"]["history_browsing"]` (+ `history_browsing_role`).
