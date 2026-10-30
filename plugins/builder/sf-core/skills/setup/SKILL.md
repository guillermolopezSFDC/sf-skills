---
name: setup
description: Validate the local Salesforce development environment — scan required tools (SF CLI, Code Analyzer, Node, NPM, Git, MCP, Source Tracking) with 🔴/🟡/🟢 status and offer guided install/update.
allowed-tools:
  - Bash
  - Read
---

Run the tool prerequisite scan and render an actionable status report. On-demand only — it does not run automatically on session start.

## Phase 1: Scan

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/sf-context check-tools
```

The output is a JSON object with a `tools` array (plus a `diagnostic` block on any critical failure).

**The banner is painted for you — do not reproduce it.** When `check-tools` runs, the plugin paints the framed **"Ready to build on Salesforce?"** banner deterministically on the visible channel (one status row per tool, the footer verdict, and the wayfinding footer), exactly like the SessionStart banner. Read the JSON for your own understanding, but do **NOT** reproduce, redraw, or re-render the banner — add only a short read of what it means for the user, then go to Phase 2.

The painted banner looks like this (illustrative — the version/message text in each row comes from the JSON):

```
──────────────────────────────────────────────────────────────
 Ready to build on Salesforce?   checking your toolchain…
──────────────────────────────────────────────────────────────
 🟢 Salesforce CLI             v2.144.6
 🟢 Code Analyzer              v5.14.0 · JIT, auto-installs on first use
 🟢 Node.js                    v22.11.0 LTS
 🟢 NPM                        v10.9.0
 🟢 Git                        v2.50.1
 🟢 Salesforce MCP (config)    .mcp.json + proxy present
 🟢 Salesforce MCP (endpoint)  org instance reachable
 ℹ️  Salesforce MCP (process)   confirm with /mcp or /doctor
 🟢 Source Tracking            enabled
──────────────────────────────────────────────────────────────
 ✓ toolchain ready
```

Status dots: 🔴 `critical` (missing or below minimum), 🟡 `warn` (outdated, non-LTS, or misconfigured), 🟢 `ok`, ℹ️ `info` (a note that can't be auto-verified). A setup is "all green" when there are no 🔴 or 🟡 rows; ℹ️ rows never count against it. The footer verdict is ` ✓ toolchain ready` when all green, or ` ⚠ <N> need attention · <M> ready` otherwise.

**If the banner did not paint** (an older Claude Code build, or a paint fallback), do **not** hand-render it from the JSON. Print it with the deterministic renderer:

```bash
${CLAUDE_PLUGIN_ROOT}/scripts/sf-context readiness-banner
```

**Deterministic results — do NOT override a failure.** The JSON report is the authoritative result. If a tool reports 🔴/🟡, report it as-is; never re-run the check a different way (PowerShell, raw shell probe) and present the result as 🟢. When the report includes a `diagnostic` block (attached on any critical failure), surface it — it carries platform, active shell, working directory, plugin root, and the resolved executable paths, and is secret-free by design.

**MCP is reported as three distinct rows — never inferred from one another:** `config` (is `.mcp.json` + `sf-mcp-proxy.bundled.js` present?), `endpoint` (is the org instance URL reachable — a connectivity proxy), and `process` (informational: this script cannot see the MCP subprocess Claude Code owns, so confirm with `/mcp` or `/doctor`).

### Tools checked

| Tool | Minimum requirement | Verification |
|---|---|---|
| Salesforce CLI | Present, and on the latest release | `sf --version` (🟡 when an update is available) |
| Code Analyzer plugin | Installed **or** JIT-registered | `sf plugins inspect @salesforce/plugin-code-analyzer`, falling back to the CLI's `oclif.jitPlugins` registry |
| Node.js | >= 18 (even/LTS) | `node --version` |
| NPM | >= 3.10 | `npm --version` |
| Git | Must be present | `git --version` |
| Salesforce MCP | config + endpoint + process rows (see above) | `.mcp.json` check, HTTP probe, `/mcp` |
| Source Tracking | Enabled for connected org | `sf project deploy preview` |

The Code Analyzer plugin is a JIT ("just-in-time") plugin — registered ≠ installed. If it is registered but not yet physically installed, `check-tools` reports it 🟢 with a note that it auto-installs on first `sf code-analyzer` run. Only a plugin that is neither installed nor JIT-registered is 🔴.

## Phase 2: Install / Update

**If all green:** confirm setup is complete — the user is ready to develop.

**If any 🔴/🟡 items exist:** offer options:

```
Some tools need attention. What would you like to do?

  [1] Fix all items
  [2] Choose which items to fix
  [3] Skip for now
```

For each item the user wants to fix, show the command below for their OS and ask them to confirm before running it — do **not** run install commands automatically.

| Item | Command |
|---|---|
| Salesforce CLI — install | `npm install --global @salesforce/cli` (or macOS: `brew install sf`) |
| Salesforce CLI — update | `sf update` |
| Code Analyzer — install | `sf plugins install @salesforce/plugin-code-analyzer` |
| Code Analyzer — update | `sf plugins update @salesforce/plugin-code-analyzer` |
| Node.js — install / upgrade | macOS: `nvm install --lts && nvm use --lts` or `brew install node`; Windows: download the LTS from https://nodejs.org |
| NPM — update | `npm install --global npm@latest` |
| Git — install | macOS: `xcode-select --install` or `brew install git`; Windows: https://git-scm.com |
| Source Tracking — enable | `sf org enable tracking --target-org <alias>` |
| Salesforce MCP — misconfigured | reload the plugin with `/reload-plugins` |

## Notes

- After installing a tool that modifies PATH (Node.js, SF CLI), the user may need to restart Claude Code for the change to take effect.
- Source Tracking requires a connected org — if none is configured, run `/sf-core:login` first.
- For org authentication issues (expired session, wrong org, INVALID_SESSION_ID), run `/sf-core:login`.
- An outdated SF CLI is 🟡, not 🟢: readiness means *latest*.
- For a quick org/project banner rather than a prerequisite scan, run `/sf-core:status`.
