---
name: platform-metadata-deploy
description: "Deploy metadata to Salesforce orgs (primarily sandboxes and scratch orgs; production only as an explicit, confirmed exception) with sf CLI v2: validate or dry-run, deploy, report status, handle test levels and failures, and run destructive deploys (destructiveChanges.xml) to remove components. Trigger on deploying, validating, or deleting metadata from an org. DO NOT TRIGGER for writing Apex (use platform-apex-generate), creating scratch orgs (use dx-org-manage)."
metadata:
  version: "1.2"
  domains: ["Platform", "Developer Experience"]
  relatedSkills:
    - "agentforce-generate"
    - "agentforce-test"
    - "automation-flow-generate"
    - "experience-lwc-generate"
    - "integration-connectivity-connected-app-configure"
    - "integration-connectivity-generate"
    - "platform-apex-test"
    - "platform-custom-object-generate"
    - "platform-data-manage"
  cliTools:
    - tool: ["curl"]
      semver: ">=7.0.0"
    - tool: ["git"]
      semver: ">=2.0.0"
    - tool: ["jq"]
      semver: ">=1.7.0"
    - tool: ["sf"]
      semver: ">=2.0.0"
---

# platform-metadata-deploy: Comprehensive Salesforce DevOps Automation

Use this skill when the user needs **deployment orchestration**: dry-run validation, targeted or manifest-based deploys, CI/CD workflow advice, scratch-org management, failure triage, or safe rollout sequencing for Salesforce metadata.

## When This Skill Owns the Task

Use `platform-metadata-deploy` when the work involves:
- `sf project deploy start`, `report`, or retrieval workflows
- release sequencing across objects, permission sets, Apex, and Flows
- CI/CD gates, test-level selection, or deployment reports
- troubleshooting deployment failures and dependency ordering

Delegate elsewhere when the user is:
- authoring Apex code (this skill only deploys it)
- authoring LWC components → experience-lwc-generate
- creating custom objects or fields → platform-custom-object-generate
- building Flows → automation-flow-generate
- doing org data operations → platform-data-manage
- authoring or testing Agentforce agents → agentforce-generate

---

## Critical Operating Rules

- Use **`sf` CLI v2 only**.
- On non-source-tracking orgs, deploy/retrieve commands require an explicit scope such as `--source-dir`, `--metadata`, or `--manifest`.
- Prefer **`--dry-run` first** before real deploys.
- **Production is an exception path, not the default.** Never run `sf project deploy start` against a production org. Only if the user explicitly asks for a production deploy, follow [production-deploys.md](production-deploys.md) (server-side validation, then quick deploy behind an explicit confirmation). If the target is unknown, ask first.
- For Flows, deploy safely and activate only after validation.
- Keep test-data creation guidance delegated to **`platform-data-manage`** after metadata is validated or deployed.

### Default deployment order
| Phase | Metadata |
|---|---|
| 1 | Custom objects / fields |
| 2 | Permission sets |
| 3 | Apex |
| 4 | Flows as Draft |
| 5 | Flow activation / post-verify |

This ordering prevents many dependency and FLS failures.

---

## Required Context to Gather First

Ask for or infer:
- target org alias and environment type
- deployment scope: source-dir, metadata list, or manifest
- whether this is validate-only, deploy, retrieve, or CI/CD guidance
- required test level and rollback expectations
- whether special metadata types are involved (Flow, permission sets, agents, packages)

Preflight checks:
```bash
sf --version
sf org list --skip-connection-status
sf org display --target-org <alias> --json
test -f sfdx-project.json
```

---

## Recommended Workflow

### 1. Preflight
Confirm auth, repo shape, package directories, and target scope.

### 1a. Classify the target org (routes prod vs non-prod)

Before any deploy, classify the target from its `sf org list` record — the path forks on whether the org is **production**. Do NOT classify from `sf org display`: it reports only a plain My Domain and omits the sandbox/scratch/trial fields, so a Sandbox-or-Demo Org (SDO) is misclassified as production (forcedotcom/sf-skills#356). The `sf org list` record carries the real classification booleans. Pass `--skip-connection-status` so this stays a local read that does not refresh an OAuth token per authenticated org (forcedotcom/sf-skills#349):

```bash
sf org list --skip-connection-status --json
```

Find the record whose `alias`, `username`, or `orgId` matches the target, then classify from that record (first match wins):
- **Scratch** — `isScratch` is `true`.
- **Sandbox** — `isSandbox` is `true`, or the `instanceUrl` contains `--` (the My Domain sandbox marker) or `test.salesforce.com`.
- **Trial / Developer Edition** — `trailExpirationDate` (the CLI's spelling; also accept `trialExpirationDate`) is a date in the **future**, or the `instanceUrl` host contains `orgfarm-`, `.develop.my.salesforce.com`, `.pc-rnd.`, or `.scratch.my.salesforce.com`. A past date does not count — the date is cached at login and never refreshed, so a converted trial keeps a stale past date and is not a trial.

Anything else is **production** — including a **Dev Hub** (`isDevHub` is `true`) that shows none of the above signals, since Dev Hub is normally enabled in the customer's own production org. If no record matches or the record has no usable info (no `instanceUrl` and none of the flags), treat it as **unknown** and ask the user before proceeding, rather than assuming production or non-production.

- **Non-production** → the dry-run + deploy path (steps 2–6).
- **Production** → only on the user's explicit request, via [production-deploys.md](production-deploys.md). Never run `sf project deploy start` against production.

### 2. Validate first (non-production)
```bash
sf project deploy start --dry-run --source-dir force-app --target-org <alias> --wait 30 --json
```
Use manifest- or metadata-scoped validation when the change set is targeted.

### 3. If validation succeeds, offer the next safe workflow
After a successful validation, guide the user to the correct next action:
1. deploy now
2. assign permission sets
3. create test data via platform-data-manage
4. run tests / smoke checks
5. orchestrate multiple post-deploy steps in order

### 4. Deploy the smallest correct scope
```bash
# source-dir deploy
sf project deploy start --source-dir force-app --target-org <alias> --wait 30 --json

# manifest deploy
sf project deploy start --manifest manifest/package.xml --target-org <alias> --test-level RunLocalTests --wait 30 --json

# manifest deploy with Spring '26 relevant-test selection
sf project deploy start --manifest manifest/package.xml --target-org <alias> --test-level RunRelevantTests --wait 30 --json
```

### 5. Verify
```bash
sf project deploy report --job-id <job-id> --target-org <alias> --json
```
Then verify tests, Flow state, permission assignments, and smoke-test behavior.

### 6. Report clearly
Summarize what deployed, what failed, what was skipped, and what the next safe action is.

Output template: [references/deployment-report-template.md](references/deployment-report-template.md)

---

## High-Signal Failure Patterns

| Error / symptom | Likely cause | Default fix direction |
|---|---|---|
| `FIELD_CUSTOM_VALIDATION_EXCEPTION` | validation rule or bad test data | adjust data or rule timing |
| `INVALID_CROSS_REFERENCE_KEY` | missing dependency | include referenced metadata first |
| `CANNOT_INSERT_UPDATE_ACTIVATE_ENTITY` | trigger / Flow / validation side effect | inspect automation stack and failing logic |
| tests fail during deploy | broken code or fragile tests | run targeted tests, fix root cause, revalidate |
| field/object not found in permset | wrong order | deploy objects/fields before permission sets |
| Flow invalid / version conflict | dependency or activation problem | deploy as Draft, verify, then activate |

Full workflows: [references/orchestration.md](references/orchestration.md), [references/trigger-deployment-safety.md](references/trigger-deployment-safety.md)

---

## CI/CD Guidance

Default pipeline shape:
1. authenticate
2. validate repo / org state
3. static analysis
4. dry-run deploy
5. tests + coverage gates
6. deploy
7. verify + notify

- When org policy and release risk allow it, consider `--test-level RunRelevantTests` for Apex-heavy deployments.
- Pair this with modern Apex test annotations such as `@IsTest(testFor=...)` and `@IsTest(isCritical=true)`.

Static analysis now uses **Code Analyzer v5** (`sf code-analyzer`), not retired `sf scanner`.

Deep reference: [references/deployment-workflows.md](references/deployment-workflows.md)

---

## Agentforce Deployment Note

Use this skill to orchestrate **deployment/publish sequencing** around agents, but use the agent-specific skill for authoring decisions:
- agentforce-generate for `.agent` authoring, Agent Builder, Prompt Builder, and metadata config

For full agent DevOps details, including `Agent:` pseudo metadata, publish/activate, and sync-between-orgs, see:
- [references/agent-deployment-guide.md](references/agent-deployment-guide.md)

---

## Cross-Skill Integration

| Need | Delegate to | Reason |
|---|---|---|
| custom object / field creation | platform-custom-object-generate | define objects and fields before deploy |
| run tests after deployment | platform-apex-test | post-deploy test execution and coverage |
| Flow creation / repair | automation-flow-generate | Flow authoring and activation guidance |
| test data or seed records | platform-data-manage | describe-first data setup and cleanup |
| Agent authoring and publish readiness | agentforce-generate | agent-specific correctness |

---

## Reference Map

### Start here
- [references/orchestration.md](references/orchestration.md)
- [references/deployment-workflows.md](references/deployment-workflows.md)
- [references/deployment-report-template.md](references/deployment-report-template.md)

### Specialized deployment safety
- [references/trigger-deployment-safety.md](references/trigger-deployment-safety.md)
- [references/agent-deployment-guide.md](references/agent-deployment-guide.md)
- [references/deploy.sh](references/deploy.sh)

### Asset templates
- [assets/package.xml](assets/package.xml) — manifest template covering common metadata types
- [assets/destructiveChanges.xml](assets/destructiveChanges.xml) — template for removing metadata from target orgs

---

## Score Guide

| Score | Meaning |
|---|---|
| 90+ | strong deployment plan and execution guidance |
| 75–89 | good deploy guidance with minor review items |
| 60–74 | partial coverage of deployment risk |
| < 60 | insufficient confidence; tighten plan before rollout |

---

## Completion Format

```text
Deployment goal: <validate / deploy / retrieve / pipeline>
Target org: <alias>
Scope: <source-dir / metadata / manifest>
Result: <passed / failed / partial>
Key findings: <errors, ordering, tests, skipped items>
Next step: <safe follow-up action>
```

## Additional guides

- [Production deploys (validate → quick deploy)](production-deploys.md)
- [Destructive deploy (destructiveChanges.xml)](destructive-deploy.md)
