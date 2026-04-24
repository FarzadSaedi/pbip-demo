# `.github/` — GitHub Configuration Folder Guide

This document explains the **`.github/` folder** in this repository: what lives there, what each file does, and the **recommended approach when starting a new project**.

---

## 1. Purpose

`.github/` is the **standardised, GitHub-specific configuration folder**. GitHub automatically picks up files from this folder for:

- **Workflows** — CI/CD pipelines under `workflows/*.yml`.
- **Copilot instructions** — `copilot-instructions.md` is read by GitHub Copilot in VS Code.
- (Optional, not yet present) **Issue/PR templates, CODEOWNERS, dependabot config, security policy**, etc.

In this repo it does two jobs:
1. **Define automated pipelines** (deploy + BPA).
2. **Teach Copilot the house style** for editing TMDL.

---

## 2. Folder contents

```
.github/
├── copilot-instructions.md         ← House style for GitHub Copilot
└── workflows/
    ├── bpa.yml                     ← Best Practice Analyzer pipeline
    └── deploy.yml                  ← Deployment to Microsoft Fabric pipeline
```

### 2.1 `copilot-instructions.md`
A Markdown file that **GitHub Copilot in VS Code automatically reads** when working in this repository. It tells Copilot:
- Who it is (a Power BI semantic model developer).
- The folder layout convention (`src/`, model/report folders).
- TMDL syntax rules — how to declare descriptions (`///`), where measures vs columns go, format strings, multi-line DAX, etc.
- Power Query / M commenting and step-naming conventions.
- A "COMPANY context" section used as background when generating descriptions.

This file is the difference between Copilot writing "this is a column" and Copilot writing a meaningful, business-aware description in your team’s tone of voice.

### 2.2 `workflows/deploy.yml`
- **Trigger**: manual `workflow_dispatch` with two inputs:
  - `workspace` (string) — the target Fabric workspace name.
  - `environment` (choice) — `DEV` or `PRD`.
- **Runs on**: `ubuntu-latest`.
- **Steps**: checkout → set up Python 3.12 → `pip install fabric-cicd` → run `python ./src/deploy.py --spn-auth true ...`.
- **Secrets used**: `FABRIC_CLIENT_ID`, `FABRIC_CLIENT_SECRET`, `FABRIC_TENANT_ID`.

It is intentionally manual; promotion to PRD should be a deliberate human action.

### 2.3 `workflows/bpa.yml`
- **Trigger**: every Pull Request to `main` that touches `*.SemanticModel/**` or `*.Report/**`, plus manual `workflow_dispatch`. (A `push` trigger is present but commented out.)
- **Runs on**: `windows-latest` (Tabular Editor is a .NET Windows tool).
- **Steps**: checkout → run `.\.bpa\bpa.ps1` twice (once for models, once for reports).

It is the **merge gate** — see [BPA-GUIDE.md](BPA-GUIDE.md) for details on the rules.

---

## 3. How `.github/` ties everything together

```
PR opened ─► bpa.yml runs ─► fails on rule violations ─► you fix ─► PR merged
                              │
                              ▼
Manual trigger ─► deploy.yml runs ─► fabric-cicd publishes src/ to Fabric workspace
                              │
                              └── uses FABRIC_* secrets to authenticate as the SPN

Developer in VS Code ─► Copilot reads copilot-instructions.md ─► generates TMDL in house style
```

---

## 4. Recommended approach when starting a new project

### Step 1 — Decide your **branching and environment strategy** before writing workflows

| Option | When to pick it |
|---|---|
| **Single `main` + manual PRD deploy** *(the demo)* | Small team, one product. Easy to start. |
| **`develop` → `main`** with auto-deploy DEV / manual PRD | Several developers, parallel work. |
| **Trunk-based** with feature flags + per-PR preview workspaces | Mature team, very high change cadence. |

Your branch model drives the workflow triggers.

### Step 2 — Use **GitHub Environments**, not bare repository secrets
Create one GitHub Environment per Fabric environment (`dev`, `prd`):
1. **Settings → Environments → New environment**.
2. Add the `FABRIC_*` secrets **per environment** (so PRD secrets are isolated).
3. Add **required reviewers** to the `prd` environment, so deploys to PRD require an approval click.
4. Reference the environment in `deploy.yml`:
   ```yaml
   jobs:
     deploy:
       environment: ${{ github.event.inputs.environment }}
   ```

This single change is the biggest production-readiness improvement you can make to the demo.

### Step 3 — Replace client secrets with **OIDC federated credentials** (optional but recommended)
Instead of storing a long-lived `FABRIC_CLIENT_SECRET`, configure the App Registration with **federated credentials** for GitHub Actions and use `azure/login@v2` in the workflow. No secrets in GitHub at all. See: https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect

### Step 4 — Add the standard GitHub config files
Beyond what the demo ships, every serious project should add:

| File | Purpose |
|---|---|
| `.github/CODEOWNERS` | Auto-request review from data engineers on `src/*.SemanticModel/**` and from BI analysts on `src/*.Report/**`. |
| `.github/pull_request_template.md` | Reminds authors to run BPA locally, link the work item, summarise model changes. |
| `.github/ISSUE_TEMPLATE/*.yml` | Templates for "bug in report", "new measure request", "new report request". |
| `.github/dependabot.yml` | Keeps `actions/checkout`, `actions/setup-python`, `fabric-cicd` versions current. |
| `.github/SECURITY.md` | Where to report security issues responsibly. |
| `.github/workflows/lint-yaml.yml` | Optional: lints workflow YAML. |

### Step 5 — Pin action versions
The demo uses `actions/checkout@v4` and `actions/setup-python@v2`. Always pin a major version (or even a SHA for max safety) so an upstream release can’t suddenly break your CI. Renovate / Dependabot can bump these via PR.

### Step 6 — Customise `copilot-instructions.md` for your team
- Replace every `COMPANY` placeholder with your real company / project context.
- Document your **measure naming convention** (e.g. `[Sales Amount]` not `[Sum of Sales]`).
- Document your **format string defaults** (currency, percentage, integer).
- Document your **DAX style** (variable casing, indentation, comment style).
- Add any **forbidden patterns** (e.g. "don’t use IFERROR", "don’t use bi-directional relationships").

### Step 7 — Wire branch protection
- **Settings → Branches → Add rule for `main`**:
  - Require pull request reviews (1 or 2).
  - Require status checks to pass: select **`bpa`**.
  - Require branches to be up to date.
  - Restrict who can push directly.

Without branch protection, the BPA workflow is advisory only.

---

## 5. Things to avoid in `.github/`

- ❌ Hard-coding workspace names or tenant IDs in workflow YAML — pass them as inputs or environment variables.
- ❌ Storing the SPN client secret in plaintext in the YAML — always use `secrets.<NAME>`.
- ❌ Using `pull_request_target` unless you fully understand the security implications (it gives untrusted PR code access to secrets).
- ❌ Triggering deploys on `pull_request` from forks — they don’t get secret access anyway and you don’t want strangers deploying to your PRD workspace.
- ❌ Over-fitting `copilot-instructions.md` to one developer’s style — treat it like shared code; review changes via PR.
- ❌ Unbounded `concurrency` settings — at minimum, cancel-in-progress for the same PR to avoid wasted CI minutes.

---

## 6. Recommendations for new projects

1. **Start by splitting the demo workflow into two**: one that runs on PR (lint/BPA only) and one that runs on a manual trigger (deploy). The demo already does this — keep it that way.
2. **One workflow file per concern.** Don’t cram lint, deploy, and release notes into a single YAML.
3. **Use a reusable workflow** (`workflow_call`) for the deploy step if you have multiple repos doing the same thing — write it once, call it many times.
4. **Add a `concurrency:` block** to `deploy.yml` keyed on the environment so you cannot accidentally run two PRD deploys in parallel.
5. **Write status checks back to PRs** from non-CI processes (e.g., from a data refresh test) using the GitHub Checks API.
6. **Tag every successful PRD deploy** in Git automatically (`prd-2026.04.24-1`). Makes rollback a checkout-and-redeploy.
7. **Cache `_tools/`** in the BPA workflow with `actions/cache` to speed up runs.
8. **Run BPA on `push` to `main`** too (re-enable the commented trigger), so a force-push or admin merge can’t bypass the gate.
9. **Add a manual "rollback" workflow** that takes a Git tag and re-runs the deploy from that tag.
10. **Document your workflows in the README** with badges and a one-paragraph description each. New joiners shouldn’t have to read YAML to understand the pipeline.

---

## 7. Cheat sheet

| File | Triggered by | Does what | Needs |
|---|---|---|---|
| `workflows/bpa.yml` | PR to `main` (model/report changes), manual | Runs BPA on every model and report | Nothing (downloads tools at runtime) |
| `workflows/deploy.yml` | Manual only | Publishes `src/` to a Fabric workspace via `fabric-cicd` | `FABRIC_CLIENT_ID`, `FABRIC_CLIENT_SECRET`, `FABRIC_TENANT_ID` |
| `copilot-instructions.md` | Auto, when Copilot runs in this repo | Steers Copilot to write TMDL/M in house style | Nothing |

---

### TL;DR

> `.github/` is where GitHub-specific configuration lives. Two workflows wire CI: `bpa.yml` is the merge gate, `deploy.yml` publishes to Fabric. `copilot-instructions.md` makes Copilot a useful TMDL collaborator. For a new project, add GitHub Environments + branch protection + CODEOWNERS + dependabot on top of what the demo ships.
