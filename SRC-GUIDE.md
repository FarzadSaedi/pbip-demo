# `src/` — Power BI Content Folder Guide

This document explains the **`src/` folder** in this repository: what lives there, why each file/folder exists, and the **recommended approach when starting a new project**. It is the companion to [BPA-GUIDE.md](BPA-GUIDE.md) and [TRAINING-GUIDE.md](TRAINING-GUIDE.md).

---

## 1. Purpose

`src/` is the **single source of truth for all Power BI content** in this repository. Everything Power BI Desktop saves and everything `fabric-cicd` deploys lives here. Both the local script and the GitHub Actions point at this folder:

- `python .\src\deploy.py --src ".\src"`
- `.github/workflows/deploy.yml` → `--src "./src"`
- `.github/workflows/bpa.yml` → scans `./src/*.SemanticModel` and `./src/*.Report`

If a Power BI artefact is **not under `src/`**, it is **not deployed and not linted**.

---

## 2. Folder contents

```
src/
├── deploy.py                ← Deployment script (Python + fabric-cicd)
├── parameter.yml            ← Per-environment find/replace rules
├── Report01.pbip            ← Launcher file for Power BI Desktop
├── Report01.Report/         ← Report content (PBIR format)
├── Report02.pbip
├── Report02.Report/
├── … (Report03…05 follow the same pattern)
├── Model01.SemanticModel/   ← Semantic model (TMDL format)
├── Model02.SemanticModel/
└── Model03.SemanticModel/
```

### 2.1 `*.pbip` (the launcher)
A ~10-line JSON file that tells Power BI Desktop which `*.Report` folder to open:

```json
{
  "version": "1.0",
  "artifacts": [ { "report": { "path": "Report01.Report" } } ],
  "settings": { "enableAutoRecovery": true }
}
```

You **double-click** this file to start authoring. It contains no actual content.

### 2.2 `*.Report/` (PBIR — report definition)
The report itself, expanded into folders:
- `definition.pbir` — points the report at its semantic model (`byPath` for local, `byConnection` for cloud).
- `definition/report.json` — report-level theme & layout settings.
- `definition/pages/<pageId>/page.json` and `visuals/` — one folder per page, one folder per visual.
- `definition/bookmarks/` — saved bookmarks.
- `StaticResources/RegisteredResources/` — custom theme JSON, images uploaded inside the report.
- `StaticResources/SharedResources/` — references to built-in Power BI base themes.

> Why folders instead of one big file? Because moving a visual now produces a small JSON diff that is easy to review in a Pull Request.

### 2.3 `*.SemanticModel/` (TMDL — model definition)
The semantic model, expanded into folders:
- `definition.pbism` — manifest (compatibility level, Q&A flag).
- `diagramLayout.json` — diagram positions in the model view.
- `definition/database.tmdl` — engine compatibility level.
- `definition/model.tmdl` — model-level options + `ref` lines that include sibling files.
- `definition/expressions.tmdl` — Power Query M parameters (the prime target for `parameter.yml`).
- `definition/relationships.tmdl` — one block per relationship.
- `definition/tables/<TableName>.tmdl` — one file per table; contains measures, columns, and the M partition.
- `definition/roles/<RoleName>.tmdl` — Row-Level Security.
- `definition/perspectives/<Name>.tmdl` — subject-area views.
- `definition/cultures/<locale>.tmdl` — Q&A linguistic metadata.
- `TMDLScripts/` *(optional)* — `createOrReplace` scripts for bulk edits.
- `DAXQueries/` *(optional, e.g. `Model03`)* — saved DAX queries you run from Desktop’s DAX Query View.

### 2.4 `deploy.py`
Wraps `fabric-cicd`. Picks an auth method (interactive locally, SPN in CI), builds a `FabricWorkspace`, and calls `publish_all_items(...)`. See [TRAINING-GUIDE.md §3.2](TRAINING-GUIDE.md).

### 2.5 `parameter.yml`
Per-environment overrides applied at deploy time:
- `find_replace` — exact-text swap inside TMDL (typically against an `expression X = "..."` line).
- `key_value_replace` — JSONPath swap inside report JSON files.

This is how one repo deploys to many Fabric workspaces with different connection strings.

---

## 3. Recommended approach when starting a new project

Follow this order. It avoids the typical "I built everything and now nothing connects" trap.

### Step 1 — Decide your model topology before opening Desktop
On paper, answer:
- How many semantic models? (Start with **one shared model** unless you have a clear domain split.)
- How many reports? Will they be **thin** (model in another folder) or **thick** (model + report in the same `.pbip`)?
- Which environments? `DEV / PRD` is the minimum; many teams add `QUAL`.

> **Recommended pattern**: thin reports + shared models. Reuses one model across many reports, exactly like Reports 01–03 → Model01 in this demo.

### Step 2 — Create the model first
1. In Power BI Desktop: build/import tables, write M, define relationships, measures, RLS roles, perspectives.
2. **File → Save As → Power BI Project (.pbip)** into `src/` with a clear name, e.g. `Sales.pbip`.
3. Desktop produces three siblings — `Sales.pbip`, `Sales.Report/`, `Sales.SemanticModel/`.
4. (Optional) Rename the model folder to a domain name (`Sales.SemanticModel`) and update any path references.
5. Commit immediately so you have a baseline diff for future changes.

### Step 3 — Create thin reports
For every additional report:
1. Save a new `.pbip` into `src/`.
2. Edit `<NewReport>.Report/definition.pbir` to point `byPath` at the shared model:
   ```json
   "datasetReference": { "byPath": { "path": "../Sales.SemanticModel" } }
   ```
3. Delete the auto-generated duplicate `<NewReport>.SemanticModel/` folder.
4. Re-open in Desktop to confirm it still loads.

### Step 4 — Naming conventions
Adopt and enforce these from day one — renaming later is painful:

| Object | Convention | Example |
|---|---|---|
| `.pbip` files | `<Domain><Subject>.pbip`, PascalCase | `SalesExecutive.pbip` |
| Model folders | `<Domain>.SemanticModel` | `Sales.SemanticModel` |
| Report folders | `<Domain><Subject>.Report` | `SalesExecutive.Report` |
| Tables | Singular noun, no spaces if possible | `Sales`, `Customer`, `Calendar` |
| Measures | Title Case, descriptive | `Sales Amount`, `Margin %` |
| Hidden technical columns | Prefix with the key word | `CustomerKey` (hidden) |
| M parameters | PascalCase, environment-aware | `Server`, `Database`, `Environment` |

### Step 5 — Parameterise everything that varies per environment
- Move every server name, database, capacity ID, workspace ID into a **named M parameter** in `expressions.tmdl`.
- For each parameter, add an entry in `parameter.yml`. The `find_value` must be the **exact text** of the line in TMDL.
- Always include the current value as the DEV entry (so DEV → DEV is a no-op).

### Step 6 — Add documentation as you build, not after
- Use TMDL `///` description lines on **every** measure and on every non-obvious column.
- Use the conventions in [.github/copilot-instructions.md](.github/copilot-instructions.md) so Copilot generates descriptions in the right style.
- Comment Power Query (`M`) steps and rename them to short verbs in the past tense.

### Step 7 — Run BPA locally before pushing
```powershell
.\.bpa\bpa.ps1 -src @(".\src\*.SemanticModel")
.\.bpa\bpa.ps1 -src @(".\src\*.Report")
```
Fix violations early — they’ll block the PR otherwise. See [BPA-GUIDE.md](BPA-GUIDE.md).

### Step 8 — Smoke-test deploy to DEV
```powershell
python .\src\deploy.py --workspace "MyProject - DEV" --environment DEV --src ".\src"
```
Open the workspace in Fabric and confirm the model refreshes and reports render.

---

## 4. Folder-naming rules to live by

1. **Folder name = item name in Fabric.** `Sales.SemanticModel` becomes the dataset called *Sales* in your workspace.
2. **Don’t rename folders casually.** Renames look like deletes-and-creates to `fabric-cicd`, which can break dependencies and lose user permissions on the workspace items.
3. **Never put two semantic models with the same suffix-stripped name in the same workspace** — they collide.
4. **Avoid spaces and special characters** in folder names; some Fabric APIs and CLI tools don’t like them.

---

## 5. Things to avoid in `src/`

- ❌ Committing `*.pbix` files. The whole point of PBIP is to leave binary behind.
- ❌ Committing `.pbi/cache.abf`, `.pbi/localSettings.json`, or any other Desktop scratch file. They’re already in `.gitignore`; if you see them in `git status`, your gitignore is wrong.
- ❌ Hard-coding production server names inside partition expressions. Always use M parameters.
- ❌ Editing `lineageTag` GUIDs. They are stable identifiers; changing them breaks references.
- ❌ Mixing **thick** and **thin** copies of the same model in different `.pbip` files. Pick one.
- ❌ Storing secrets or connection strings inside TMDL. Use Fabric data source credentials (set once in the workspace) or Azure Key Vault.

---

## 6. Recommendations for new projects

1. **Start small.** One model + one report + one environment. Get the deploy working end-to-end before adding more content.
2. **One folder per domain.** Don’t put unrelated tables into one giant model just because Desktop makes it easy.
3. **Use perspectives** to expose only relevant fields per audience (Sales analysts vs Finance), instead of building separate models.
4. **Use roles** for any data that has a confidentiality dimension (region, customer segment).
5. **Add a `_docs/` or top-level `docs/` folder** for diagrams, ADRs (Architecture Decision Records), and a data dictionary. Keep `src/` for content the deploy tool consumes.
6. **Add a generated diagram of the model** to your README via Mermaid (`relationships.tmdl` → ER diagram) and refresh it via CI.
7. **Adopt branch protection on `main`** that requires the BPA workflow to pass.
8. **Use `.resources/` (or similar)** for templates and themes shared across reports — see [RESOURCES-GUIDE.md](RESOURCES-GUIDE.md).
9. **Version-tag every PRD deploy** so rollbacks are trivial (`git tag prd-2026.04.24-1`).
10. **Plan capacity before you build.** A `*.SemanticModel` will not deploy to a workspace that is not on a Fabric capacity (F-SKU, P-SKU, or trial).

---

## 7. Cheat sheet

| File / folder | Edited by | Purpose |
|---|---|---|
| `*.pbip` | Power BI Desktop | Launcher — opens the report in Desktop. |
| `*.Report/definition/` | Desktop / VS Code | Pages, visuals, theme. |
| `*.Report/StaticResources/` | Desktop | Custom theme JSON, embedded images. |
| `*.SemanticModel/definition/tables/` | Desktop / Tabular Editor / VS Code | Measures, columns, partitions. |
| `*.SemanticModel/TMDLScripts/` | Hand-written | Bulk-edit scripts (`createOrReplace`). |
| `deploy.py` | Hand-written | Calls `fabric-cicd` to publish to Fabric. |
| `parameter.yml` | Hand-written | Per-environment value swaps. |

---

### TL;DR

> `src/` is the only folder the deploy tool sees. Put one `.pbip` per report, one `.SemanticModel` folder per model. Parameterise every per-environment value into M parameters and `parameter.yml`. Run BPA locally, then deploy to DEV before opening a PR.
