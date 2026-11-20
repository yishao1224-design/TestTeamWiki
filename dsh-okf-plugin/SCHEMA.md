---
type: Schema
title: Bundle Schema & Conventions
description: Authoritative conventions for this OKF bundle — types, tags, frontmatter templates, lifecycle rules.
status: stable
lifecycle: active
updated: 2026-11-20
wiki_name: "dsh-okf-plugin"
code_commit: 95b0615aefbbd4d1e16040efdd37d1e1884e5ed0
code_synced: 2026-11-20
---

# Bundle Schema & Conventions

*(Starter schema written by `okf.py init`. Flesh out the Domain, type
registry, and tag taxonomy on first real ingestion — see the full
template in the knowledgeAgent repo's SCHEMA.md.)*

# Domain

*(one paragraph: what this knowledge base covers and for whom)*

# Type registry

| type | lives in | meaning |
|------|----------|---------|
| `Concept` | `concepts/` | An idea, definition, method, or explanation |
| `Entity` | `entities/` | A concrete nameable thing (person, org, system, dataset, tool) |
| `Comparison` | `comparisons/` | Structured X-vs-Y |
| `Query` | `queries/` | A filed answer to a real question |
| `Source` | `sources/` | Immutable raw capture |
| `Schema` | root | This file only |

Add new types here before first use. Lint enforces this table: an
unlisted `type` is a warning; a known `type` in the wrong directory is
an error (`archive/` is exempt).

# Page frontmatter template (all types)

Shared by every page type. Set `type` and the directory to match the
registry above — **do not default to `type: Concept`** just because
this example shows it.

```yaml
---
type: Concept            # match to directory — see type registry above
title: Human Readable Title
description: One sentence for index/previews.
status: draft            # OKF v0.2: draft | stable | deprecated
lifecycle: draft         # draft | active | needs_review | deprecated | archived
confidence: medium       # low | medium | high
created: 2026-11-20
updated: 2026-11-20
stale_after: 2026-11-20     # required once lifecycle: active
tags: []
sources:
  - resource: /sources/2026-11-20-example.md   # capture, bundle-relative
    id: example                             # cite in the body as [example]
    title: Example capture
  - resource: ../../src/thing.py            # live repo file, doc-relative
    id: thing-impl
    title: thing.py implementation
    last_modified: 2026-11-20
    blob: 0000000000aa                      # pin: okf.py drift --hash <file>
generated:               # who produced this content, stamped at write time
  by: agent/version      # actor: human:<id> | <tool>/<version> | process:<id>
  at: 2026-11-20
verified:                # append one entry per real re-verification
  - by: human:you        # only for a real check — never on freshly written text
    at: 2026-11-20
---
```

`status` is the OKF v0.2 standard field; `lifecycle` is this bundle's
five-state extension and `status` is derived from it (`active` and
`needs_review` are both `stable`; `archived` is `deprecated`). Lint
errors if the two disagree.

`generated` says what produced the page; `verified` says who confirmed
it. Stamp `generated` whenever you write, and `verified` only after a
real re-check — an agent verifying its own fresh output records nothing.

Cite sources by `id` in the body — `…as the README states [example]` —
not with a `# Citations` section, which v0.2 replaced with `sources`.
Pin `blob:` only on re-readable local files; `okf.py drift` rechecks
those and lint reports the ones that moved.

Only `type: Concept` pages describing **domain** knowledge (business
rules, processes, system behavior) use the domain-concept body template
(`## Definition`, `## Key Behaviors`/`## Invariants`,
`## Related Concepts` — see the full template in the knowledgeAgent
repo's SCHEMA.md). `Entity`/`Comparison`/`Query` pages and
method/meta `Concept` pages keep free structure instead.

# Tag taxonomy

- `unverified` — claims not yet checked against a second source
- `contested` — sources actively disagree
