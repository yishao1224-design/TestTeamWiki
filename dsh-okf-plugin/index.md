---
okf_version: "0.2"
---

# Bundle Index

* [Bundle Schema & Conventions](SCHEMA.md) - Authoritative conventions for this OKF bundle — types, tags, frontmatter templates, lifecycle rules.

# Concepts

* [Architecture of the dsh-okf Plugin](concepts/architecture.md) - The two halves of the plugin, the components inside each, how they talk to each other, and where state lives on disk.
* [Build, Run, Test and Deploy](concepts/build-run-test-deploy.md) - The exact commands to install, type-check, build, smoke-test, harness-check and preview the plugin, and how it is installed and updated in DSH.
* [Conventions, Pitfalls and Open Questions](concepts/conventions-and-pitfalls.md) - The coding and workflow conventions this repository follows, the traps a maintainer should know, and what the authors themselves flag as unverified.
* [Data Model and Entities](concepts/data-model.md) - The records the plugin stores — bundle frontmatter, library.json, team.json, work records and wire DTOs — and the invariants that hold them together.
* [Host Routes and the Bundle Library](concepts/host-routes-and-library.md) - The Host entry's twenty-three /api/okf routes, error mapping, the library.json store, and the git and bundle-reading modules behind it.
* [Publishing to the Azure DevOps Wiki](concepts/publish-pipeline.md) - How a project bundle is rendered into published/Projects pages for an Azure DevOps wiki, and how a merge mirrors them there.
* [Review State and Auto-Merge](concepts/review-and-auto-merge.md) - How a page counts as reviewed, how an agent edit resets it, and the eligibility rule that decides whether a wiki pull request may merge without a person.
* [Team Wiki Service](concepts/team-wiki-service.md) - The team.ts Host service that lists a hub's project wikis, runs maintenance rounds, opens pull requests and bootstraps or syncs wikis from code.
* [The Browser Client](concepts/browser-client.md) - The plugin's browser half — its slots, one store, the chip/dialog/team page components, the safe markdown renderer and the inline style table.
* [The OKF Toolchain (okf.py) and the Hub Template](concepts/okf-toolchain.md) - The stdlib-only Python toolchain the hub carries — init, lint, index, links, log, stats, drift — and the template files Set up the hub writes.
* [dsh-okf Plugin — Overview](concepts/overview.md) - What the dsh-okf DeepSeek Harness plugin is for, who uses it, and the main flows it supports.

# Entities

* [DeepSeek Harness (DSH)](entities/deepseek-harness.md) - The host application the plugin runs inside — its plugin bundles, Host services, client slots and workspace services.
* [Team Wiki Hub Repository](entities/team-wiki-hub.md) - The single git repository that holds every project's OKF bundle, the AGENTS.md registry, the shared toolchain and the published pages.

# Sources

* [build.mjs at 95b0615 (verbatim)](sources/2026-11-20-build-script.md) - The build script that emits lib/index.js, lib/check-pr.js and lib/client.js, and the guards it enforces on each artifact.
* [cordis.patch.yml at 95b0615 (verbatim)](sources/2026-11-20-cordis-patch.md) - The one-row patch that mounts the plugin, plus the commented list of optional config keys.
* [dsh-okf-plugin README at 95b0615 (verbatim)](sources/2026-11-20-readme.md) - The project README as captured from commit 95b0615: purpose, setup, use, Team Wiki flows, config, layout, build and test, and the author's own 'not verified yet' list.
* [examples/wiki-auto-merge.yml at 95b0615 (verbatim)](sources/2026-11-20-wiki-auto-merge-workflow.md) - The hub CI workflow that re-checks eligibility with the base branch's check-pr.js and okf.py lint before enabling GitHub auto-merge.
* [examples/wiki-publish-to-ado.yml at 95b0615 (verbatim)](sources/2026-11-20-wiki-publish-ado-workflow.md) - The hub workflow that mirrors published/Projects/ into an Azure DevOps wiki's Projects/ folder after a merge, refusing any other change.
* [package.json at 95b0615 (verbatim)](sources/2026-11-20-package-manifest.md) - The bundle manifest: name, exports, published file list, npm scripts, the dsh.bundle.patch and dsh.client declarations, and dev dependencies.
* [templates/hub/AGENTS.md at 95b0615 (verbatim)](sources/2026-11-20-hub-agents-template.md) - The hub skeleton's AGENTS.md: rules for agents and the empty project registry table that Set up the hub writes.
* [templates/hub/scripts/okf.py key regions (excerpt)](sources/2026-11-20-okf-py-toolchain.md) - Excerpts of the shared toolchain that ships with the hub: its subcommands, the status/lifecycle mapping, and the lint rules that give the bundle its conventions.
* [tsconfig.json at 95b0615 (verbatim)](sources/2026-11-20-tsconfig.md) - TypeScript settings: ES2022 target, bundler resolution, react-jsx, strict flags including exactOptionalPropertyTypes, no emit, src only.
