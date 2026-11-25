# Bundle Update Log

## 2026-11-25
* **Ingestion**: [templates/hub/scripts/okf.py key regions at da10648 (excerpt)](/sources/2026-11-25-okf-py-toolchain.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Creation**: [Page Stages — OKF Status and the Derived Review States](/concepts/page-stages.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [DeepSeek Harness (DSH)](/entities/deepseek-harness.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Team Wiki Service](/concepts/team-wiki-service.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Publishing to the Azure DevOps Wiki](/concepts/publish-pipeline.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [dsh-okf Plugin — Overview](/concepts/overview.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [The OKF Toolchain (okf.py) and the Hub Template](/concepts/okf-toolchain.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Data Model and Entities](/concepts/data-model.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Build, Run, Test and Deploy](/concepts/build-run-test-deploy.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [The Browser Client](/concepts/browser-client.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Azure DevOps Hubs and Pull Requests](/concepts/azure-devops-pull-requests.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Automation Tasks and the Team Wiki Tools](/concepts/automation-tasks.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Bundle Schema & Conventions](/SCHEMA.md) — dsh-okf-plugin: adds 1 page, updates 15 pages, captures 1 source
* **Update**: [Bundle Schema & Conventions](/SCHEMA.md) — status is the only state field: no lifecycle in the template, review stages derived, needs-review in the tag taxonomy
* **Migration**: dropped the retired lifecycle key from every page and SCHEMA.md with okf.py migrate; the 2026-11-20 captures keep it (captures are immutable)
* **Creation**: [Page Stages — OKF Status and the Derived Review States](/concepts/page-stages.md) — dsh-okf-plugin: sync wiki to da10648 (drop the lifecycle key, read the stage off OKF status)
* **Ingestion**: [templates/hub/scripts/okf.py key regions at da10648 (excerpt)](/sources/2026-11-25-okf-py-toolchain.md) — capture the status-only toolchain, TOOLCHAIN_VERSION, migrate and the new lint rules
* **Update**: [The OKF Toolchain (okf.py) and the Hub Template](/concepts/okf-toolchain.md) — migrate, TOOLCHAIN_VERSION, status-only lint rules, index badges, ensureToolchain
* **Update**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — reviewed means stage active; the stamp and Mark reviewed drop the lifecycle key and the needs-review tag
* **Update**: [Team Wiki Service](/concepts/team-wiki-service.md) — sameRemote reuse, locked-folder fallback round, ensureToolchain at round open and at Open pull request
* **Update**: [Data Model and Entities](/concepts/data-model.md) — status is the only state field, a derived stage on the wire records, the retired key
* **Update**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — add host/stage.ts and the reply-link prefix in knowledge.ts
* **Update**: [The Browser Client](/concepts/browser-client.md) — catalog filter and state badges read stage, "needs review", the Mark reviewed hint
* **Update**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — no lifecycle key, reply links, toolchain provisioning, a locked workspace folder
* **Update**: [Publishing to the Azure DevOps Wiki](/concepts/publish-pipeline.md) — skip by derived stage, "Due for re-review" reads stage needs_review
* **Update**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — summarize counts stages, the reply-link rule in the prompt context
* **Update**: [Automation Tasks and the Team Wiki Tools](/concepts/automation-tasks.md) — wikiAt prefers an open round, which may sit in a fallback folder
* **Update**: [dsh-okf Plugin — Overview](/concepts/overview.md) — derived stages, and each round brings the hub's toolchain current
* **Update**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — the toolchain the plugin writes, the draft rule without a lifecycle key
* **Sync**: the wiki reflects https://github.com/yishao1224-design/dsh-okf-plugin at `code_commit: da10648a182b38b410d22f952d8e1d973d8825f0` (was b71e34fb1c).

## 2026-11-23
* **Creation**: [Automation Tasks and the Team Wiki Tools](/concepts/automation-tasks.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [DeepSeek Harness (DSH)](/entities/deepseek-harness.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Team Wiki Service](/concepts/team-wiki-service.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [dsh-okf Plugin — Overview](/concepts/overview.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Data Model and Entities](/concepts/data-model.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Build, Run, Test and Deploy](/concepts/build-run-test-deploy.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Update**: [Bundle Schema & Conventions](/SCHEMA.md) — dsh-okf-plugin: sync wiki to b71e34f (Team Wiki tools, log.md commit)
* **Sync**: the wiki reflects https://github.com/yishao1224-design/dsh-okf-plugin at `code_commit: b71e34fb1c76d88af680440531fda3c5a9c75c34` (was ec5a774e59).
* **Creation**: [Azure DevOps Hubs and Pull Requests](/concepts/azure-devops-pull-requests.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Team Wiki Service](/concepts/team-wiki-service.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [dsh-okf Plugin — Overview](/concepts/overview.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [The OKF Toolchain (okf.py) and the Hub Template](/concepts/okf-toolchain.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Data Model and Entities](/concepts/data-model.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Build, Run, Test and Deploy](/concepts/build-run-test-deploy.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [The Browser Client](/concepts/browser-client.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — dsh-okf-plugin: adds 1 page, updates 11 pages
* **Update**: [Bundle Schema & Conventions](/SCHEMA.md) — dsh-okf-plugin: adds 1 page, updates 11 pages

## 2026-11-21
* **Creation**: [Azure DevOps Hubs and Pull Requests](/concepts/azure-devops-pull-requests.md) — Document Azure Repos hubs: the PullHost shape, the AzureDevOps REST client, auto-complete and the build-validation guard
* **Update**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — Add host/azure.ts and the shared PullHost shape
* **Update**: [Build, Run, Test and Deploy](/concepts/build-run-test-deploy.md) — Add ado-smoke.mjs and the Azure Pipelines guard to the hub setup steps
* **Update**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — Azure DevOps failures as github-* errors; auto-complete waits on branch policies
* **Update**: [Data Model and Entities](/concepts/data-model.md) — A maintenance round's pull request may live on GitHub or Azure DevOps
* **Update**: [dsh-okf Plugin — Overview](/concepts/overview.md) — Azure DevOps pull requests and auto-complete
* **Update**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — The Azure DevOps client also throws GitHubError
* **Update**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — check-pr's Azure Pipelines output and the build-validation guard
* **Update**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — Azure Repos hubs and the build-validation guard
* **Update**: [Team Wiki Service](/concepts/team-wiki-service.md) — Pull requests on GitHub or Azure DevOps, plus auto-complete
* **Update**: [The Browser Client](/concepts/browser-client.md) — Provider-specific token warning and a host-neutral review link
* **Update**: [The OKF Toolchain (okf.py) and the Hub Template](/concepts/okf-toolchain.md) — Set up the hub now writes the Azure auto-complete workflow

## 2026-11-20
* Update: [Team Wiki Service](/concepts/team-wiki-service.md) — Sync from code is the one code button; a pending wiki's card opens its workspace and its missing pull request
* **Ingestion**: [examples/wiki-publish-to-ado.yml at 95b0615 (verbatim)](/sources/2026-11-20-wiki-publish-ado-workflow.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [examples/wiki-auto-merge.yml at 95b0615 (verbatim)](/sources/2026-11-20-wiki-auto-merge-workflow.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [tsconfig.json at 95b0615 (verbatim)](/sources/2026-11-20-tsconfig.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [dsh-okf-plugin README at 95b0615 (verbatim)](/sources/2026-11-20-readme.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [package.json at 95b0615 (verbatim)](/sources/2026-11-20-package-manifest.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [templates/hub/scripts/okf.py key regions (excerpt)](/sources/2026-11-20-okf-py-toolchain.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [templates/hub/AGENTS.md at 95b0615 (verbatim)](/sources/2026-11-20-hub-agents-template.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [cordis.patch.yml at 95b0615 (verbatim)](/sources/2026-11-20-cordis-patch.md) — Add the dsh-okf-plugin project wiki
* **Ingestion**: [build.mjs at 95b0615 (verbatim)](/sources/2026-11-20-build-script.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Team Wiki Hub Repository](/entities/team-wiki-hub.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [DeepSeek Harness (DSH)](/entities/deepseek-harness.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Team Wiki Service](/concepts/team-wiki-service.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Review State and Auto-Merge](/concepts/review-and-auto-merge.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Publishing to the Azure DevOps Wiki](/concepts/publish-pipeline.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [dsh-okf Plugin — Overview](/concepts/overview.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [The OKF Toolchain (okf.py) and the Hub Template](/concepts/okf-toolchain.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Host Routes and the Bundle Library](/concepts/host-routes-and-library.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Data Model and Entities](/concepts/data-model.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Conventions, Pitfalls and Open Questions](/concepts/conventions-and-pitfalls.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Build, Run, Test and Deploy](/concepts/build-run-test-deploy.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [The Browser Client](/concepts/browser-client.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Architecture of the dsh-okf Plugin](/concepts/architecture.md) — Add the dsh-okf-plugin project wiki
* **Update**: [Bundle Schema & Conventions](/SCHEMA.md) — Add the dsh-okf-plugin project wiki
* **Creation**: [Bundle Schema & Conventions](/SCHEMA.md) — Add the dsh-okf-plugin project wiki
* **Initialization**: Bundle skeleton created by `okf.py init`.
