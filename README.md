# AI DATA PRODUCT GENERATION

AI-assisted generation of **deployable Semarchy Data Platform (SDP) data products** — from a
business need, through scoping and specification, to a build-validated `.seml` model ready to deploy.

This repository is a **portable, reusable workshop**: an AI assistant (Claude Code / Codex in VS Code)
drives generation against a curated reference model and a build-authoritative CLI, following a set of
stable conventions that improve over time.

## Start here → [`ai-model-development/TUTORIAL.md`](ai-model-development/TUTORIAL.md)

The tutorial is the entry point (installation, security, workspace tour, manual authoring, the fast
generation path, prompt recipes, and what is out of scope). New teammates should read it first.

## What's in the box

| Path | What it is |
|---|---|
| [`ai-model-development/TUTORIAL.md`](ai-model-development/TUTORIAL.md) + [`ai-model-development/tutorial/`](ai-model-development/tutorial/) | Team tutorial (8 navigable pages). |
| [`ai-model-development/ARCHITECTURE.md`](ai-model-development/ARCHITECTURE.md) | **Technical architecture** — workflow, component contracts, invariants, self-learning loop. The transfer document for Product & R&D. |
| [`CLAUDE.md`](CLAUDE.md) | The assistant's authoritative operating rules, the **self-learning** principle, and an index of the traps that leave a build green and a runtime wrong. Deliberately small — it is reloaded in full every session. |
| [`.claude/skills/`](.claude/skills/) | Five task-scoped skills carrying the validated `.seml`, SemQL and platform findings — loaded only when the task calls for them. Part of the tool, even though Claude Code requires them at this path. |
| [`ai-model-development/`](ai-model-development/) | The reusable **hub**: generation `prompt/`, scoping `methodology/`, `use-case-prompts/`, bundled `reference-models/`, `scripts/` (engine + validators), `cli/` (Design + Platform CLIs, **both 1.11 and 2.0 Design CLI archives**), `tools/` (SemQL MCP bridge). |
| [`ai-model-development/reference-models/BusinessPartner360/`](ai-model-development/reference-models/BusinessPartner360/README.md) | **Primary reference model** — a complete, build-green example exercising the broadest set of configurations. [`HierarchyHub`](ai-model-development/reference-models/HierarchyHub/README.md) complements it (generic hierarchy metamodel). |
| [`ai-model-development/tools/semql-mcp-bridge/`](ai-model-development/tools/semql-mcp-bridge/README.md) | MCP bridge exposing the IDE extension's SemQL model-awareness tools to the assistant (+ the `semql-suggest` skill). |
| `src/` · `resources/` · `input/` · `output/` | Per-project working folders (the delivered project is the BusinessPartner360 demo). |

## Prerequisites (summary — full details in the tutorial)

- **VS Code** + an AI assistant extension (Claude Code / Codex) + the **Semarchy** extension.
- **Node.js** (≥ 18) and **Python 3** with `pyyaml`.
- **`sem-design` CLI** (model build, authoritative) and the **`sem` Platform CLI** (runtime/platform).

**SEML has two dialects and the installed `sem-design` CLI pins the format** ("models" 1.11 vs
"data products" 2.0 — no cross-version compatibility; both install archives are bundled under
`ai-model-development/cli/DesignCLI/`). The generation engine emits either from the same model
definition via `cfg["seml_version"]`. The committed reference trees are in the **1.11** dialect.

Smoke test (green build proves the toolchain works) — pick the line matching your installed CLI
(`sem-design --version`):

```bash
# CLI 1.11.x (matches the committed reference trees)
sem-design dm model build --root-folder "ai-model-development/reference-models/BusinessPartner360/src/BusinessPartner360"
# CLI 2.0.x — 1.11 trees do NOT build on it ("no data product profile found"): smoke-test on a
# project model under src/, or regenerate a reference model in 2.0 (one seml_version flip)
sem-design data-product build --data-product-folder "src/<Model>"
```

## macOS users

The tooling is **cross-platform by design** (audited 2026-08-19; the validators' package locator is
Unix-aware since then). Everything installs the same as on Windows/Linux, with three mac-specific
notes:

- **Design CLI**: the bundled npm archives are pure JavaScript — `npm i -g …/DesignCLI/<version>.tgz`
  works as-is.
- **`sem` Platform CLI**: use the bundled mac archives
  (`ai-model-development/cli/PlatformCLI/sem_mac_arm64.tar.gz` or `sem_mac_x86_64.tar.gz`): untar,
  `chmod +x sem`, and if Gatekeeper quarantines the unsigned binary, clear it with
  `xattr -d com.apple.quarantine ./sem`.
- **SemQL MCP bridge**: install with `./install.sh` (the `.ps1` is the Windows variant).

Python/Node scripts, the generation engine, the loaders and the docs' commands are OS-neutral
(stdlib + bash-style). First-run checklist and details: [tutorial page 1](ai-model-development/tutorial/01-installation-and-context.md).

## Security

- **Never commit secrets** (API keys, client secrets, tokens). Pass them via `$SEM_API_KEY` /
  `$SEM_CLIENT_ID` / `$SEM_CLIENT_SECRET`. See [tutorial page 2](ai-model-development/tutorial/02-configuration-and-security.md).
- Connecting to a platform is only needed to **deploy** and administer runtime artifacts; **building a
  model is fully offline**.

## Ownership

Internal Semarchy tooling. Contributions should keep the model **build-green** and, per the
self-learning principle in `CLAUDE.md`, fold any newly validated object/configuration back into the
generation tooling.
