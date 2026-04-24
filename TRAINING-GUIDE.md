# PBIP Demo Repository — Training Guide

A complete walkthrough of this repository for someone new to **Power BI Project (PBIP)** format, **TMDL**, **fabric-cicd**, and **CI/CD for Power BI**. After reading this you should understand every file and folder, why it exists, and how to clone the pattern for your own project.

---

## 1. The Big Picture

This repo is a **template** showing how to manage Power BI assets as **source code** (text files in Git) instead of binary `.pbix` files. It demonstrates four pillars:

| Pillar | What it gives you |
|---|---|
| **PBIP format** | Power BI content stored as folders of plain-text files (TMDL + JSON) → diff-able, review-able, mergeable in Git. |
| **Multiple reports / shared models** | One semantic model can power many reports. The repo has 5 reports backed by 3 models. |
| **CI/CD with `fabric-cicd`** | A Python library + GitHub Action that publishes the whole `src/` folder to a Microsoft Fabric workspace, with environment-aware parameter substitution (DEV / PRD). |
| **Best Practice Analyzer (BPA)** | Automated quality gates on every Pull Request using **Tabular Editor** (for models) and **PBI Inspector** (for reports). |

Architecture (from the README):

```
Report01 ┐
Report02 ├─► Model01.SemanticModel
Report03 ┘
Report04 ───► Model02.SemanticModel
Report05 ───► Model03.SemanticModel
```

---

## 2. Top-Level Layout

```
pbip-demo/
├── .bpa/               ← Best Practice Analyzer scripts & rule files
├── .github/            ← Copilot instructions + GitHub Actions workflows
├── .gitignore          ← Excludes Power BI Desktop local cache
├── .resources/         ← Repo-level images / assets used in README
├── README.md           ← Short overview (what you read first)
└── src/                ← All Power BI content lives here
```

The convention `src/` is important — both `deploy.py` and the GitHub Actions point at this folder.

---

## 3. The `src/` Folder (the heart of the repo)

```
src/
├── deploy.py           ← Local + CI deployment script (uses fabric-cicd)
├── parameter.yml       ← Environment-specific find/replace rules used by fabric-cicd
├── Report01.pbip       ← "Project file" — open this in Power BI Desktop
├── Report01.Report/    ← The actual report definition (folder)
├── ... Report02..05 follow the same pattern
├── Model01.SemanticModel/   ← Semantic model #1 (Sales-style star schema)
├── Model02.SemanticModel/   ← Semantic model #2 (Northwind)
└── Model03.SemanticModel/   ← Semantic model #3
```

### 3.1 The `*.pbip` file

Tiny JSON file (≈10 lines). It is what you double-click to open in Power BI Desktop. It only points to the sibling `*.Report` folder:

```json
{
  "version": "1.0",
  "artifacts": [ { "report": { "path": "Report01.Report" } } ],
  "settings": { "enableAutoRecovery": true }
}
```

Mental model: `.pbip` = "shortcut/launcher". The real content is in the folders next to it.

### 3.2 `deploy.py`

A short Python script that:

1. Parses CLI args: `--workspace`, `--environment` (DEV/PRD), `--src`, `--spn-auth`.
2. Picks an auth method: **InteractiveBrowserCredential** (locally) or **ClientSecretCredential** using env vars `FABRIC_CLIENT_ID/SECRET/TENANT_ID` (in CI).
3. Builds a `FabricWorkspace` from the local repo folder.
4. Calls `publish_all_items(...)` to push every Semantic Model and Report into the target Fabric workspace.

Run locally:
```powershell
pip install fabric-cicd
python .\src\deploy.py --workspace "PBIP Demo" --environment DEV --src ".\src"
```

### 3.3 `parameter.yml`

This is the **environment configuration** consumed by `fabric-cicd` during publish. Two sections:

- `find_replace`: text-level swaps inside TMDL. Used here to swap a Power Query M parameter expression (`expression Environment = "DEV"` → `"PRD"`) per environment, per model.
- `key_value_replace`: JSON-path swaps inside report JSON files. Used here to set the same default page name across reports per environment.

This is how you keep ONE codebase but deploy to multiple Fabric workspaces with different connection strings, server URLs, etc.

---

## 4. Anatomy of a Semantic Model — `Model01.SemanticModel/`

```
Model01.SemanticModel/
├── definition.pbism          ← Manifest (version + qna setting)
├── diagramLayout.json        ← Visual positions in the model diagram
├── definition/               ← The model itself, in TMDL
│   ├── database.tmdl         ← compatibilityLevel
│   ├── model.tmdl            ← Model-level settings + table/role/perspective references
│   ├── expressions.tmdl      ← Power Query M parameters (HttpSource, Environment…)
│   ├── relationships.tmdl    ← One relationship per block
│   ├── cultures/             ← Per-locale linguistic metadata (Q&A synonyms)
│   │   ├── en-US.tmdl
│   │   └── pt-PT.tmdl
│   ├── perspectives/         ← Subset views of the model (e.g., Sales perspective)
│   ├── roles/                ← Row-level security (RLS)
│   │   ├── Store - Canada.tmdl
│   │   └── Store - United States.tmdl
│   └── tables/               ← One file per table (preferred granularity)
│       ├── Sales.tmdl
│       ├── Calendar.tmdl
│       └── …
└── TMDLScripts/              ← Reusable TMDL "scripts" (createOrReplace) for bulk edits
```

### Key TMDL files explained

**`database.tmdl`** — declares the engine compatibility level (`1601` = recent Fabric/PBI):
```tmdl
database
    compatibilityLevel: 1601
```

**`model.tmdl`** — top-level model object plus `ref` lines that pull in tables, roles, perspectives, cultures from sibling files. Also stores annotations like Power Query order and PBI Desktop version.

**`expressions.tmdl`** — M-language parameters used by partitions. Notice `Environment = "DEV"`: that exact string is what `parameter.yml` swaps at deploy time.

**`relationships.tmdl`** — each `relationship <guid>` block declares `fromColumn` / `toColumn`; can include `isActive: false` for inactive relationships.

**`tables/Sales.tmdl`** — the core of the model. Inside one `table` you find:
- `measure` blocks (DAX expressions, with `formatString`, sometimes a `kpi` sub-block).
- `column` blocks (data type, format, `summarizeBy`, `sourceColumn`).
- A `partition` block at the bottom containing the M expression that loads the data (`mode: import` or `directQuery`).
- `lineageTag` GUIDs — Power BI’s stable identifier so renaming doesn’t break references.
- Triple-slash `///` comments above objects = **descriptions** that show up in the field list.

**`roles/*.tmdl`** — Row-Level Security rules:
```tmdl
role 'Store - Canada'
    modelPermission: read
    tablePermission Store = 'Store'[Country] == "Canada"
```

**`perspectives/Sales.tmdl`** — A "view" exposing only certain measures/columns (used by report authors to focus on a subject area).

**`cultures/*.tmdl`** — Linguistic metadata for Q&A in each locale.

**`TMDLScripts/`** — Hand-runnable scripts of the form `createOrReplace` + object body. Use them in **TMDL View** in Power BI Desktop or Tabular Editor to apply bulk edits (for example, reset all measures of a table at once).

### Repeat for `Model02` / `Model03`
Same structure. `Model02` is a Northwind sample (an OData service in `expressions.tmdl`). `Model03` adds a `DAXQueries/` folder with `.dax` files — saved DAX queries you can run against the model from Power BI Desktop’s DAX Query View.

---

## 5. Anatomy of a Report — `Report01.Report/`

```
Report01.Report/
├── definition.pbir           ← Points the report at its semantic model
├── definition/
│   ├── report.json           ← Report-level settings (theme, layout)
│   ├── version.json
│   ├── bookmarks/            ← Bookmarks (one folder/file each)
│   └── pages/
│       ├── pages.json        ← pageOrder + activePageName
│       └── <pageId>/
│           ├── page.json     ← Page metadata
│           └── visuals/      ← One folder per visual on the page
└── StaticResources/
    ├── RegisteredResources/  ← Custom theme JSON, images uploaded to the report
    └── SharedResources/      ← Built-in base themes referenced by the report
```

### `definition.pbir` — the model link

```json
{
  "version": "4.0",
  "datasetReference": {
    "byPath": { "path": "../Model01.SemanticModel" }
  }
}
```

Two flavours:
- `byPath` → **local/thin report** that lives in the same repo as the model (used here).
- `byConnection` → connection-string to a model already published in a Fabric workspace.

This is how Reports 01–03 all share Model01.

### Report content
The report’s pages and visuals are individual JSON files. This means moving a visual or changing a slicer produces small, reviewable diffs in Pull Requests — the main reason teams adopt PBIP.

---

## 6. CI/CD: `.github/workflows/`

Two workflows are wired up:

### `deploy.yml` — manual deploy to Fabric
- Trigger: `workflow_dispatch` with inputs `workspace` and `environment` (DEV/PRD).
- Steps: checkout → set up Python 3.12 → `pip install fabric-cicd` → run `deploy.py` with `--spn-auth true`.
- Uses **GitHub Secrets**: `FABRIC_CLIENT_ID`, `FABRIC_CLIENT_SECRET`, `FABRIC_TENANT_ID` — these are the credentials of an Entra **Service Principal** added as Admin to the target Fabric workspace.

### `bpa.yml` — quality gate
- Trigger: every Pull Request to `main` that touches a `*.Report/**` or `*.SemanticModel/**` file (also `workflow_dispatch`).
- Runs `.\.bpa\bpa.ps1` twice: once for semantic models, once for reports.
- Fails the PR if rules are violated.

---

## 7. Best Practice Analyzer — `.bpa/`

```
.bpa/
├── bpa.ps1                          ← Downloads tools + runs them
├── bpa-rules-semanticmodel.json     ← Custom rules for Tabular Editor (model BPA)
└── bpa-rules-report.json            ← Custom rules for PBI Inspector (report BPA)
```

`bpa.ps1`:
1. Downloads **TabularEditor.Portable** and **PBI Inspector** into `_tools/` (gitignored).
2. For every `*.pbism` (model) found under `src/`, runs Tabular Editor with the rules JSON (`-A rules.json -G`).
3. For every `*.pbir` (report), runs PBI Inspector with `-formats GitHub` (so problems show up as PR annotations).
4. Non-zero exit code → workflow fails → PR can’t be merged.

The rules JSONs are a **starting set** of well-known performance and modelling rules — you tune these to your standards.

---

## 8. `.github/copilot-instructions.md`

A guide aimed at GitHub Copilot (and you) that codifies how TMDL files should be written: how to write `///` descriptions, where to put measures vs columns, M-code commenting style, COMPANY-specific business context for description generation, etc. Reading this file gives you the **house style** for the model.

---

## 9. `.gitignore`

Only excludes things you must never commit:
- `**/.pbi/localSettings.json` — per-machine local Desktop settings.
- `**/.pbi/cache.abf` — large Power BI Desktop cache file.
- `fabric_cicd.*.log` — deployment logs.
- `_tools/` — downloaded BPA binaries.

---

## 10. Typical Workflows

### As a report developer (daily)
1. Open the `*.pbip` file in Power BI Desktop.
2. Edit visuals / measures normally.
3. Save → Desktop writes back into the `*.Report/` and `*.SemanticModel/` folders as TMDL/JSON.
4. `git diff` shows you the actual changes; commit & push.
5. Open a PR → BPA workflow runs → fix any violations → merge.
6. Run the **deploy** workflow to publish to your Fabric workspace.

### As a CI/CD owner (one-time setup)
1. Create a Fabric workspace.
2. In Entra: create an App Registration → record `clientId`, `tenantId`, generate a `clientSecret`.
3. Add the SPN as **Admin** of the Fabric workspace.
4. In GitHub repo settings → Secrets → add `FABRIC_CLIENT_ID`, `FABRIC_CLIENT_SECRET`, `FABRIC_TENANT_ID`.
5. (Optional) enable Fabric tenant setting that allows service principals to call Fabric APIs.

---

## 11. How to Re-Use This Repo for Your Own Project

Step-by-step for a brand-new project:

1. **Fork or copy the repo** (use it as a template).
2. **Replace the content of `src/`**:
   - Delete the demo `Model0x.SemanticModel/` and `Report0x.*` folders/files you don’t need.
   - In Power BI Desktop, do **File → Save As → Power BI Project (.pbip)** into `src/`. Desktop will create the `YourReport.Report/` and (if a new model) `YourModel.SemanticModel/` folders.
   - For **thin reports** that point at an existing dataset, edit `definition.pbir` → use `byConnection` instead of `byPath`.
3. **Edit `parameter.yml`**: change `item_name`s to match your model names; replace the demo `find_value` strings with the actual M parameter expressions in **your** `expressions.tmdl` (e.g., a SQL server name or a workspace ID); add one `replace_value` entry per environment you target.
4. **Edit `deploy.py` defaults** (or always pass them on the CLI): `--workspace`, default `--environment`.
5. **Configure CI**:
   - Update `.github/workflows/deploy.yml` defaults if you renamed your environments (e.g., add `QUAL`).
   - Add the three `FABRIC_*` GitHub Secrets.
6. **Tune BPA rules**:
   - Edit `.bpa/bpa-rules-semanticmodel.json` and `.bpa/bpa-rules-report.json` — remove rules you don’t want, add your own (the script will fall back to default rule sets if you delete the files entirely).
7. **Update `copilot-instructions.md`** with your own company context, naming conventions, and description style so Copilot generates TMDL the way your team writes it.
8. **Update `README.md`** with your project name, mermaid diagram, and any extra steps.
9. **First run**: locally execute `python .\src\deploy.py --workspace "MyWS" --environment DEV --src ".\src"` to validate authentication and parameter swaps work.
10. **Open a PR** to verify the BPA workflow runs and is green; iterate on rules until clean.

---

## 12. Mental-Model Cheat Sheet

| If you see… | Think… |
|---|---|
| `*.pbip` | Just a launcher; open in PBI Desktop. |
| `*.SemanticModel/` | The model (data + DAX + M). |
| `*.Report/` | A report (visuals + pages + theme). Has `definition.pbir` pointing to a model. |
| `definition.pbism` / `definition.pbir` | Tiny manifest files; the real content is under `definition/`. |
| `*.tmdl` | Tabular Model Definition Language — text format for the model (replaces `model.bim`). |
| `///` line | A description/comment attached to the next TMDL object. |
| `lineageTag: <guid>` | Stable internal ID — leave it alone unless you really know why. |
| `expression X = ...` in `expressions.tmdl` | A Power Query M parameter — perfect target for `find_replace` per environment. |
| `byPath` vs `byConnection` in `.pbir` | Local thin report vs cloud-attached thin report. |
| `parameter.yml` | Per-environment swaps applied at deploy time. |
| `deploy.py` | The one-liner that calls fabric-cicd to publish. |
| `.bpa/bpa.ps1` | Local + CI quality gate using Tabular Editor & PBI Inspector. |

---

## 13. Recommended Reading / Tools

- Power BI Project (PBIP) format docs — Microsoft Learn.
- TMDL language reference — Microsoft Learn.
- `fabric-cicd` library — https://microsoft.github.io/fabric-cicd/
- Tabular Editor 2 (free) — for model editing & BPA.
- PBI Inspector v2 — report-side BPA.
- VS Code extensions: **TMDL** (syntax + IntelliSense), **Power Query / M**.

---

### TL;DR

> The repo treats Power BI content as code. **Power BI Desktop** edits the files in `src/`; **Git** versions them; **fabric-cicd + GitHub Actions** publishes them per environment using `parameter.yml`; and **BPA** keeps quality high on every PR. Copy `src/`, swap in your own models/reports, edit `parameter.yml` and the GitHub secrets, and you have a production-grade Power BI delivery pipeline.
