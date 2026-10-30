# sf-dev

Toolkit for **development orgs** (sandboxes and scratch orgs): generate code and metadata, create and delete records, and deploy. It writes to the org, so do not point it at production. It depends on [`sf-core`](../sf-core/README.md), which provides the setup, auth, read-only tooling (SOQL, retrieve, manifests, logs), MCP servers and deploy-safety gate.

## Install

```text
/plugin install sf-dev@gla_salesforce
```

## Skills (17)

- **Code and data (5)** — `platform-apex-generate`, `platform-apex-test` (generate, run and fix tests), `platform-apex-anonymous-run`, `platform-data-manage`, `automation-flow-generate`.
- **Deploy and org lifecycle (2)** — `platform-metadata-deploy` (deploy, validate, destructive deploy) and `dx-org-manage` (scratch orgs and source-tracking reset).
- **Declarative metadata (9)** — `platform-custom-object-generate` (+ validation rules), `platform-custom-field-generate`, `platform-custom-config-generate` (custom metadata types and settings), `platform-value-set-generate`, `platform-ui-generate` (tabs, apps, list views), `platform-flexipage-generate`, `platform-report-generate` (+ report types), `platform-permission-set-generate`, `platform-sharing-generate` (OWD and sharing rules).
- **Orchestration (1)** — `platform-lightning-app-coordinate`.

Production deploys (validate / quick deploy) are intentionally not part of this plugin.
