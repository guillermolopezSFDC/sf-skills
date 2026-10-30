# sf-core

Shared foundation for working with Salesforce from Claude Code. It is safe against **any** org, including production: nothing here writes to an org. The write-capable toolkit lives in [`sf-dev`](../sf-dev/README.md), which depends on this plugin.

## Install

```text
/plugin marketplace add guillermolopezSFDC/sf-skills
/plugin install sf-core@gla_salesforce
```

Prerequisites: Claude Code, Node.js LTS (the bundled language servers run under `node`), the Salesforce CLI (`sf`) and Python 3.8+ (hooks). Run `/sf-core:setup` to check them.

## Skills (12)

| Skill | What it does |
|---|---|
| `setup` | Scans the toolchain (SF CLI, Code Analyzer, Node, NPM, Git, MCP, Source Tracking) and offers guided install/update |
| `status` | Shows the connected org, project and next step |
| `login` / `logout` | Authenticate or disconnect an org |
| `set-default` | Set the default target org |
| `platform-lsp-integrate` | Use the bundled Apex and SOQL language servers through MCP |
| `platform-metadata-api-context-get` | Metadata API context (types, fields, structure) |
| `platform-data-and-tooling-api-context-get` | Data and Tooling API context |
| `platform-soql-query` | Author and optimize SOQL/SOSL |
| `platform-metadata-retrieve` | Retrieve metadata from an org (read-only) |
| `platform-manifest-generate` | Generate `package.xml` manifests |
| `platform-apex-logs-debug` | Analyze debug logs and governor-limit issues |

## What else is in the box

- **Agent** — `salesforce-dev`, the primary Salesforce agent for DX projects (skills first, then SF CLI, then API).
- **MCP servers** — `salesforce-api-context`, `salesforce-metadata-experts` and `salesforce-lsp`.
- **Hooks** — org-context detection on session start and a deploy-safety gate on `sf project deploy` / `delete`.

See [docs/configuration.md](docs/configuration.md) for UI modes and guard rails.
