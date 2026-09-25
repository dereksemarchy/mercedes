# Contributing

**The contribution guide lives in
[`ai-model-development/CONTRIBUTING.md`](../ai-model-development/CONTRIBUTING.md).**

This file exists only so GitHub finds one: it looks for `CONTRIBUTING.md` at the repository root, in
`.github/` and in `docs/`, and all the workshop documentation is deliberately kept together under
[`ai-model-development/`](../ai-model-development/).

## What you need to know before opening a pull request

This repository is a **template**. Teams clone it, build their own model in **their own** repository,
and send back only what is reusable. So a pull request to `main` may touch **tooling paths only**:

```
CLAUDE.md
README.md, ai-model-development/**
.claude/skills/**
```

Never `src/`, `output/`, `input/` or `resources/` — those hold client content, and a PR carrying them
will be refused whatever it also contains.

Three rules decide whether a contribution is accepted:

1. **Generic.** No client name, id or data. State the correct setup, the symptom it fixes, and how it
   was validated (build, schema, or a running platform).
2. **Validated, not assumed.** The CLI build and the running platform outrank the online documentation
   and any prior assumption — including one already written in this repository. A newer finding that
   contradicts an older note replaces it; leaving both is worse than either.
3. **In the right place.** A pattern every future model should get goes in the engine, a `.seml`
   finding goes in the skill for its task, a platform behaviour goes in the Platform CLI guide. See
   the routing table in
   [`ai-model-development/ARCHITECTURE.md`](../ai-model-development/ARCHITECTURE.md) § 8.

The full procedure — bootstrapping a workspace, staying current with the template, and the exact
branch/validation sequence for a pull request — is in the guide linked above.
