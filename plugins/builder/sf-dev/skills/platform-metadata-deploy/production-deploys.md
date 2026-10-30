# Production deploys (validate → quick-deploy)

Production is never deployed with `sf project deploy start`. It is a two-step promotion: a **server-side validation** produces a job ID, then a **quick deploy** promotes that validated job without re-running tests, behind an explicit confirmation.

## P1. Validate against production

Production validations REQUIRE a test level. Use `RunLocalTests` by default; switch to `RunSpecifiedTests --tests <ClassName>...` only if the user has explicitly listed tests.

```bash
sf project deploy validate --target-org <alias> --json [scope flags] --test-level RunLocalTests
```

Scope flags — use exactly one: `--source-dir <path>`, `--metadata <Type:Name>`, or `--manifest manifest/package.xml`.

The response returns a **job ID** (`result.id`) valid for **10 days**. Persist it **with the target's immutable identity** — the org ID and instance URL from `sf org display`, not just the alias — so the quick deploy can prove the validation belongs to the org it is about to promote to:

```bash
mkdir -p .sfdx
ORG_JSON=$(sf org display --target-org <alias> --json)
ORG_ID=$(printf '%s' "$ORG_JSON" | jq -r '.result.id')
INSTANCE_URL=$(printf '%s' "$ORG_JSON" | jq -r '.result.instanceUrl')
printf '{"jobId":"%s","createdAt":"%s","targetOrg":"%s","orgId":"%s","instanceUrl":"%s","testLevel":"RunLocalTests"}\n' \
  "<id>" "<iso8601>" "<alias>" "$ORG_ID" "$INSTANCE_URL" > .sfdx/last-validation.json
```

On validation failure, triage from `result.details.componentFailures` and `result.details.runTestResult.failures` — surface the top component errors and test failures with a scoped fix direction, then STOP. Never use `--ignore-errors`/`--ignore-warnings` during validation.

## P2. Gate the quick deploy strictly

Before promoting, verify all five:
1. **Target is production** — re-confirm via step 1a's classification.
2. **A validation exists** — read `.sfdx/last-validation.json`, or ask for the job ID, or fall back to `--use-most-recent`. Resolve `--use-most-recent` to a concrete job ID **now** (do not carry it forward as an unresolved flag), so the checks below run against a known job.
3. **Validation belongs to THIS target** — re-run `sf org display --target-org <alias> --json` and require the current `result.id` (org ID) to equal the `orgId` persisted in `.sfdx/last-validation.json`. This closes the stale-cache hazard: a cached job ID resolves its org from the deploy cache, so without this check the banner can show org B while the job promotes to org A. On any mismatch — or if the persisted record lacks an `orgId`/`instanceUrl` to compare — treat it as a **hard stop** and re-validate (P1) against the intended target. Never promote a validation whose target you cannot prove.
4. **Validation is fresh** — explicit `--job-id` ≤ 10 days; a `--use-most-recent` resolution ≤ 3 days. If stale, re-validate (P1) first.
5. **Explicit user confirmation** — print the banner below and require a literal "yes". Populate every banner field from the verified validation record and the just-fetched org data — never from unverified input. Never auto-confirm.

```text
┌─ PRODUCTION DEPLOY ─────────────────────────────┐
│ Org alias:      <alias>                         │
│ Instance:       <instanceUrl>                   │
│ Edition:        <edition>                       │
│ Validation ID:  <jobId>                         │
│ Validated:      <createdAt> (X days ago)        │
│ Components:     <componentCount> queued         │
│ Tests:          <run>/<passed>/<failed>         │
└─────────────────────────────────────────────────┘
Confirm deploy to PRODUCTION? (yes/no)
```

`sf org display --json` does not return the edition. Read `<edition>` from the Organization object's `OrganizationType` (e.g. `Enterprise Edition`):

```bash
sf data query --query "SELECT OrganizationType FROM Organization" --target-org <alias> --json | jq -r '.result.records[0].OrganizationType'
```

## P3. Promote and audit

The literal "yes" from P2's banner is the authorization. In the **sf-core plugin** context, a PreToolUse hook (`sf-deploy-gate`) blocks a bare production `deploy quick` and only permits it when the command carries the machine-readable confirmation marker `CONFIRM_PROD=1`. Emit that marker to transmit the confirmation the user just gave — it is **not** an independent bypass, so add it **only after** the explicit "yes", never before:

```bash
# plugin context: CONFIRM_PROD=1 relays P2's confirmation to the safety hook
CONFIRM_PROD=1 sf project deploy quick --job-id <validation-id> --target-org <alias> --wait 30 --json
```

Outside the plugin (bare CLI, headless/MCP) there is no such hook — run the same command without the marker.

**Quick deploy starts a new deployment with its own job ID** — it does not reuse the validation ID. Capture the response, extract the new ID, and use *that* for polling, reporting, and the audit filename; keep the validation ID separately for traceability. Stop if the response carries no deployment ID:

```bash
QUICK_RESULT=$(CONFIRM_PROD=1 sf project deploy quick --job-id <validation-id> --target-org <alias> --wait 30 --json)
QUICK_DEPLOY_ID=$(printf '%s' "$QUICK_RESULT" | jq -r '.result.id')
[ -n "$QUICK_DEPLOY_ID" ] && [ "$QUICK_DEPLOY_ID" != "null" ] || { echo "No quick-deploy ID returned — do not report the validation ID as the deploy; investigate."; exit 1; }
```

Quick deploy promotes the validated components and does NOT re-run tests. Persist the report **for the quick-deploy operation** (not the validation) for the audit trail, then guide post-deploy verification (smoke-test critical paths, watch Setup → Deployment Status, prepare a rollback plan if anything regressed):

```bash
mkdir -p .sfdx/deploy-history
sf project deploy report --job-id "$QUICK_DEPLOY_ID" --target-org <alias> --json > ".sfdx/deploy-history/${QUICK_DEPLOY_ID}.json"
```

If quick deploy times out or stays in progress, poll with the **quick-deploy ID**, not the validation ID.

Adding `CONFIRM_PROD=1` after an explicit confirmation is the sanctioned path — it is not "working around" the gate. But never override a denial the user has **not** confirmed, and never override one raised for a different reason — a target mismatch (P2 check 3), a destructive deploy, or an unknown/unresolved org. Surface those denials and stop.

---
