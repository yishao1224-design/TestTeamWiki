---
type: Source
title: "package.json at 95b0615 (verbatim)"
description: "The bundle manifest: name, exports, published file list, npm scripts, the dsh.bundle.patch and dsh.client declarations, and dev dependencies."
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/package.json
tags: []
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Capture: package.json

Verbatim copy of `package.json` at commit `95b0615aefbbd4d1e16040efdd37d1e1884e5ed0`.

```json
{
  "name": "@local/dsh-okf",
  "version": "0.1.0",
  "private": true,
  "description": "DeepSeek Harness plugin for OKF knowledge bundles: fetch from git / Azure DevOps, preview, attach a bundle to a chat session, and manage a team's project wikis from the Team Wiki sidebar page",
  "type": "module",
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js"
  },
  "files": [
    "lib/index.js",
    "lib/client.js",
    "lib/check-pr.js",
    "examples/wiki-auto-merge.yml",
    "examples/wiki-publish-to-ado.yml",
    "templates/hub",
    "cordis.patch.yml",
    "README.md"
  ],
  "scripts": {
    "build": "node build.mjs",
    "typecheck": "tsc --noEmit",
    "smoke": "node scripts/smoke.mjs && node scripts/team-smoke.mjs && node scripts/hub-smoke.mjs"
  },
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    },
    "client": {
      "platform": "web",
      "inject": [
        "@deepseek-ai/dsh-client-locale",
        "@deepseek-ai/dsh-client-ui-renderer",
        "@deepseek-ai/dsh-client-ui-conversation"
      ]
    }
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "@types/react": "^18.3.12",
    "react-dom": "^18.3.1",
    "react": "^18.3.1",
    "tsdown": "^0.22.2",
    "typescript": "^5.6.3"
  },
  "license": "MIT"
}
```

Notes for readers of this capture:

- `exports["."]` is the Host entry (`lib/index.js`); `exports["./client"]` is the browser artifact the client loader fetches.
- `dsh.client.platform` must be `web`; `build.mjs` refuses to build otherwise.
- There are no runtime dependencies: React is a dev dependency because the shell seeds it into the module table.
