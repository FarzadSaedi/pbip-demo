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

This section is a **complete, hands-on playbook**. It is written for someone who has never set up a Power BI CI/CD pipeline before. Follow the phases in order; each phase has a clear "Definition of Done" so you know when you can move on.

> **Time budget**: First-time setup typically takes a half-day to a full day. Most of the time is spent on the Azure / Fabric admin steps, not the code.

---

### Phase 0 — Prerequisites (install once on your laptop)

Before you touch the repo, install:

| Tool | Why | Where |
|---|---|---|
| **Power BI Desktop** (latest, with PBIP enabled) | Authoring `.pbip` content | Microsoft Store or download page |
| **Git** | Version control | https://git-scm.com |
| **Visual Studio Code** | Editing TMDL/JSON, viewing diffs | https://code.visualstudio.com |
| **Python 3.10+** (3.12 recommended) | Running `fabric-cicd` locally | https://python.org |
| **PowerShell 7+** (optional) | Running `bpa.ps1` locally | `winget install Microsoft.PowerShell` |
| **VS Code extensions** | TMDL, Power Query M, GitHub Actions, YAML | Marketplace |

**Enable the PBIP file format in Power BI Desktop** (one-time):
1. Open Power BI Desktop → **File → Options and settings → Options**.
2. Go to **Preview features**.
3. Tick **Power BI Project (.pbip) save option** and **Store semantic model using TMDL format** and **Store reports using enhanced report format (PBIR)**.
4. Restart Desktop.

> Without these toggles, Desktop will save a binary `.pbix` instead of the folder format this whole pipeline relies on.

**Definition of Done**: `python --version`, `git --version`, and `pwsh --version` all return successfully, and Power BI Desktop’s **Save As** dialog now offers a `.pbip` option.

---

### Phase 1 — Get the template

You have two options. Pick one.

**Option A: Use as a GitHub template (recommended for a brand-new repo)**
1. Browse to `RuiRomano/pbip-demo` on GitHub.
2. Click the green **Use this template → Create a new repository** button.
3. Name your new repo, choose private/public, click **Create repository**.
4. Clone it locally:
   ```powershell
   git clone https://github.com/<your-org>/<your-repo>.git
   cd <your-repo>
   ```

**Option B: Copy files into an existing repo**
1. Download the demo as a ZIP and extract into a working folder.
2. Copy the following into your existing repo:
   - `.bpa/`
   - `.github/workflows/bpa.yml` and `.github/workflows/deploy.yml`
   - `.github/copilot-instructions.md` (only if you want Copilot guidance)
   - `.gitignore` (merge entries with what you already have)
   - `src/deploy.py`, `src/parameter.yml`
3. Do NOT copy the demo `Model0x.SemanticModel/` and `Report0x.*` content if you have your own.

**Definition of Done**: You have the directory layout from section 2 of this guide on your laptop, tracked in Git, on a `main` branch.

---

### Phase 2 — Strip out the demo content

You only need this phase if you used Option A above (the template ships with sample models and reports).

1. Open the repo in VS Code.
2. Inside `src/`, delete:
   - `Model01.SemanticModel/`, `Model02.SemanticModel/`, `Model03.SemanticModel/`
   - `Report01.pbip`, `Report01.Report/` … through `Report05.pbip`, `Report05.Report/`
3. Open `src/parameter.yml` and remove the demo `find_replace` and `key_value_replace` blocks. Leave the file with just the two top-level keys; you will repopulate them in Phase 5.
4. Open `src/deploy.py` and update the default workspace name to a placeholder, e.g. `--default "REPLACE_ME"`.
5. Open `README.md` and clear the demo description, badges, and the mermaid diagram (you will rewrite them in Phase 9).
6. Commit the cleanup:
   ```powershell
   git add -A
   git commit -m "chore: remove demo content"
   ```

**Definition of Done**: `src/` contains only `deploy.py` and `parameter.yml`. The repo still builds (no syntax errors) but contains no Power BI content yet.

---

### Phase 3 — Add your own Power BI content

You will produce one **semantic model** and at least one **report** in PBIP format.

#### 3a. Decide your model topology
Sketch on paper or in a diagram:
- How many semantic models will you have?
- Which reports connect to which model?
- Will reports be **thick** (model lives in the same `.pbip`) or **thin** (model lives in another `.pbip` or already in Fabric)?

> Pattern used by this demo: **thin reports + shared models**. Reports 01–03 each have their own `.pbip` but all point to `Model01.SemanticModel` via `byPath`. This is the recommended pattern at scale.

#### 3b. Create the semantic model
1. In Power BI Desktop click **File → New** (or open an existing `.pbix`).
2. Build/import your tables (Get Data, write M, define relationships, measures, RLS roles, perspectives).
3. **File → Save as** → choose **Power BI Project (.pbip)** type → save **into the `src/` folder** with a name like `SalesModel.pbip`.
4. Desktop creates two siblings:
   - `src/SalesModel.pbip` (the launcher)
   - `src/SalesModel.Report/` (the report shell — you can leave it empty or delete it later if you only want the model)
   - `src/SalesModel.SemanticModel/` (the model folder you actually care about)
5. (Optional but recommended) **Rename the model folder** to make its purpose obvious, e.g. `Sales.SemanticModel/`. After rename, edit any `.pbip` files that point at it (`artifacts.report.path` and the report’s `definition.pbir` `datasetReference.byPath.path`).

#### 3c. Create thin reports
For each additional report that should reuse the same model:
1. In Desktop click **File → New** → connect to the **same model** via *Get data → Power BI semantic models* (you may need to publish the model to a workspace first to use this) **or** start with an empty report.
2. **File → Save as** → `.pbip` into `src/` with a name like `SalesExecutive.pbip`.
3. Open `src/SalesExecutive.Report/definition.pbir` in VS Code.
4. Replace the `datasetReference` with a relative `byPath` pointer to your model:
   ```json
   {
     "version": "4.0",
     "datasetReference": {
       "byPath": { "path": "../Sales.SemanticModel" }
     }
   }
   ```
5. Delete the auto-generated `SalesExecutive.SemanticModel/` folder if Desktop created one — you do not want a duplicate model.
6. Re-open the `.pbip` in Desktop to confirm it loads against the shared model and you can author visuals.

> Alternative — connect to a **published** model in Fabric using `byConnection` (no model folder in your repo):
> ```json
> {
>   "version": "4.0",
>   "datasetReference": {
>     "byConnection": {
>       "connectionString": "Data Source=powerbi://api.powerbi.com/v1.0/myorg/<workspace>;Initial Catalog=<dataset>",
>       "pbiServiceModelId": null,
>       "pbiModelVirtualServerName": "sobe_wowvirtualserver",
>       "pbiModelDatabaseName": "<dataset-id-guid>"
>     }
>   }
> }
> ```

#### 3d. Sanity check
- Every `.pbir` resolves to a real model folder OR a real Fabric workspace.
- Every model has a `definition/` folder containing `model.tmdl`, `database.tmdl`, `expressions.tmdl`, plus a `tables/` subfolder.
- `git status` shows new TMDL/JSON files, **not** `.pbix` and **not** `.pbi/cache.abf` (if it shows the cache, your `.gitignore` is wrong).

**Definition of Done**: You can open every `.pbip` in Desktop without errors, and `git diff` after a small change (e.g. adding a measure) produces a small, readable TMDL diff.

---

### Phase 4 — Provision the Fabric / Azure side

You need an Entra **Service Principal (SPN)** that the GitHub Action can use to push to Fabric.

#### 4a. Tenant settings (admin step — may need your Fabric admin)
In the **Fabric Admin Portal → Tenant settings**, enable for your SPN’s security group:
- **Service principals can use Fabric APIs**.
- **Service principals can use Power BI APIs** (still required for some endpoints).
- **Users can create Fabric items** (or scope to your group).
- **Allow service principals to update their own credentials** (optional but useful).

> If your tenant admin won’t enable these tenant-wide, ask them to scope each setting to a specific Entra security group and put your SPN in that group.

#### 4b. Create the Service Principal
1. Go to **Entra ID → App registrations → New registration**.
2. Name it `sp-fabric-cicd-<project>` → **Register**.
3. Copy the **Application (client) ID** and **Directory (tenant) ID** somewhere safe.
4. Go to **Certificates & secrets → New client secret** → choose 12 or 24 month expiry → copy the **Value** (you will never see it again).
5. (Optional) Add the SPN to a security group that the tenant settings above are scoped to.

#### 4c. Create Fabric workspaces
Create one workspace per environment, e.g.:
- `MyProject - DEV`
- `MyProject - PRD`

For each workspace:
1. Open the workspace → **Manage access → + Add people or groups**.
2. Add your SPN (search by app name) as **Admin**.
3. Assign the workspace to a **Fabric capacity** (F-SKU, P-SKU, or trial). `fabric-cicd` cannot deploy to a non-capacity workspace.

**Definition of Done**: You can list both workspaces in `https://app.fabric.microsoft.com` and see your SPN listed as Admin on each.

---

### Phase 5 — Wire `parameter.yml` to your real parameters

This file is the bridge between *one* codebase and *many* Fabric environments.

#### 5a. Inventory your environment-dependent values
Open every `definition/expressions.tmdl` and list any value that changes per environment, for example:
- A SQL Server name: `expression Server = "sql-dev.contoso.com"`.
- A Lakehouse / Warehouse connection string.
- A REST endpoint root URL.
- A flag like `expression Environment = "DEV"`.

Also list anything inside report JSON that changes per environment (rare, but the demo uses it for a default page).

#### 5b. Write the `find_replace` blocks
For each parameter, add an entry. The `find_value` must match the **exact** text in your TMDL, including spacing and quotes:

```yaml
find_replace:
  - find_value: 'expression Server = "sql-dev.contoso.com"'
    replace_value:
      DEV: 'expression Server = "sql-dev.contoso.com"'
      PRD: 'expression Server = "sql-prd.contoso.com"'
    item_type: "SemanticModel"
    item_name: "Sales"   # folder name without the .SemanticModel suffix

  - find_value: 'expression Environment = "DEV"'
    replace_value:
      DEV: 'expression Environment = "DEV"'
      PRD: 'expression Environment = "PRD"'
    item_type: "SemanticModel"
    item_name: "Sales"
```

> Always include the **current** value as the DEV entry — fabric-cicd will replace `find_value` with `replace_value[<env>]`, so DEV needs to map back to itself.

#### 5c. (Optional) `key_value_replace` for report JSON
Use JSONPath against report files when you need to swap something inside JSON:
```yaml
key_value_replace:
  - find_key: $..activePageName
    replace_value:
      DEV: <pageId>
      PRD: <pageId>
    file_path:
      - "Sales.Report/definition/pages/pages.json"
```

#### 5d. Validate
Run a dry deploy (Phase 7) and check the published model in Fabric to confirm the right value landed.

**Definition of Done**: `parameter.yml` lists every per-environment value, with a `replace_value` entry for every environment you support.

---

### Phase 6 — Adapt `deploy.py`

Open `src/deploy.py` and adjust:

1. **Default workspace name** (`parser.add_argument("--workspace", default = "...")`) — handy for local runs.
2. **Default environment** — match what you defined in `parameter.yml`.
3. **`item_type_in_scope`** — by default it’s `["SemanticModel", "Report"]`. Add other Fabric item types if your repo holds Notebooks, Pipelines, Lakehouses, etc. (See `fabric-cicd` docs for the full list of supported types.)
4. (Optional) **Logging** — uncomment `change_log_level("DEBUG")` while you debug, then re-comment it for normal runs.

You usually do **not** need to change the auth section.

---

### Phase 7 — First local deployment (smoke test)

This step proves the whole chain works **before** you involve GitHub Actions.

1. Create and activate a Python virtual environment (recommended):
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install fabric-cicd
   ```
2. Run an interactive deploy (browser sign-in):
   ```powershell
   python .\src\deploy.py `
     --workspace "MyProject - DEV" `
     --environment DEV `
     --src ".\src"
   ```
3. A browser window pops up — sign in with a user that is Admin on the workspace.
4. Watch the console: it logs each item being uploaded. Look for ✅ on every item.
5. Open the workspace in Fabric and verify your models and reports appeared, the model refreshes (you may need to set credentials on the dataset the first time), and reports render.

#### Common errors and fixes
| Error | Likely cause | Fix |
|---|---|---|
| `401 Unauthorized` | Your user/SPN is not an Admin on the workspace | Add to workspace as Admin |
| `Workspace not found` | Wrong name / no capacity assigned | Check spelling; assign to a capacity |
| `Item type X is not supported` | Item type not in `item_type_in_scope` | Add it to the list in `deploy.py` |
| `find_value not found in any file` | `parameter.yml` text doesn’t exactly match TMDL | Copy/paste the exact line from your `.tmdl` file |
| `byPath` model can’t be found | Relative path in `definition.pbir` is wrong | Fix the `path` to point to the model folder |

**Definition of Done**: An interactive run from your laptop publishes everything to your DEV workspace, and you can open a published report and see data.

---

### Phase 8 — Wire up GitHub Actions

#### 8a. Add the three repository secrets
In GitHub: **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
|---|---|
| `FABRIC_CLIENT_ID` | The SPN’s Application (client) ID |
| `FABRIC_CLIENT_SECRET` | The client secret value you copied in Phase 4b |
| `FABRIC_TENANT_ID` | Your Entra tenant (directory) ID |

#### 8b. Adjust `deploy.yml`
Open `.github/workflows/deploy.yml` and update:
1. The `default` for the `workspace` input → your DEV workspace name.
2. The `options` list under `environment` → the environments you actually have (e.g., `DEV`, `QUAL`, `PRD`).
3. (Optional) Add a second job that auto-deploys to DEV on every push to `main`:
   ```yaml
   on:
     push:
       branches: [ main ]
     workflow_dispatch:
       inputs: { ... }
   ```

#### 8c. Adjust `bpa.yml`
Usually no changes needed. Optionally:
- Re-enable the `push` trigger (currently commented) so failed BPA blocks merges to `main` too.
- Change `branches: [ main ]` to your default branch name.

#### 8d. Test the workflows
1. Push a small change (e.g. add a measure description).
2. Open a Pull Request → confirm `bpa` workflow runs and reports any rule violations as PR annotations.
3. From **Actions → deploy → Run workflow**, choose `DEV` and run. Watch logs.
4. After it succeeds, repeat with `PRD` to deploy to production.

**Definition of Done**: A green BPA run on a PR, and a green deploy run that publishes to your chosen workspace using the SPN.

---

### Phase 9 — Tune BPA rules to your standards

The shipped rule files are starting points, not gospel.

1. Open `.bpa/bpa-rules-semanticmodel.json`. Each entry has `ID`, `Name`, `Severity`, and an `Expression` written in C#-ish syntax used by Tabular Editor.
2. **Remove rules that don’t fit your environment.** Common ones to disable: snowflake-schema warnings if you intentionally use one; partition rules if your tables are small.
3. **Add rules from the community set**: pull additions from https://github.com/microsoft/Analysis-Services/tree/master/BestPracticeRules.
4. **For reports**, edit `.bpa/bpa-rules-report.json`. Common tweaks: cap visuals per page, ban specific custom visuals, enforce naming conventions for bookmarks.
5. (Optional) Run BPA locally before pushing:
   ```powershell
   .\.bpa\bpa.ps1 -src @(".\src\*.SemanticModel")
   .\.bpa\bpa.ps1 -src @(".\src\*.Report")
   ```
6. If you delete the rules JSONs entirely, the script will use each tool’s **default** rule set — fine for a quick start.

**Definition of Done**: A clean BPA run with rules that match your team’s conventions, and developers know how to run BPA locally before pushing.

---

### Phase 10 — Adapt `copilot-instructions.md` for your team

Open `.github/copilot-instructions.md` and:
1. Replace every occurrence of `COMPANY` with your real company / project name.
2. Update the "company context" bullet list with your real business — Copilot uses this to write meaningful descriptions.
3. Add or remove style rules — for example, your own measure naming convention (`[Sum of Sales]` vs `[Sales Amount]`), preferred format strings, or DAX patterns.
4. Commit the file at `.github/copilot-instructions.md`. Copilot in VS Code automatically picks it up for any chat in this repo.

---

### Phase 11 — Update README and onboard your team

1. Rewrite `README.md` for your project: name, owners, link to the workspaces, and a fresh mermaid diagram of report→model relationships.
2. Add the GitHub Actions status badges so the README shows green/red at a glance.
3. Document branch policy (e.g. `main` is protected, requires green BPA + 1 review).
4. Schedule a 30-minute walkthrough with your team showing: open `.pbip` → make a change → commit → PR → BPA → merge → deploy.

**Definition of Done**: A new colleague can clone the repo, open a `.pbip`, and submit a PR without asking you anything.

---

### One-page checklist

```
[ ] Phase 0  — Tools installed, PBIP preview enabled
[ ] Phase 1  — Repo created from template
[ ] Phase 2  — Demo content removed
[ ] Phase 3  — Your own model + reports added in PBIP format
[ ] Phase 4  — SPN created, workspaces created, SPN is Admin
[ ] Phase 5  — parameter.yml lists every per-env value
[ ] Phase 6  — deploy.py defaults updated
[ ] Phase 7  — Local deploy to DEV succeeded
[ ] Phase 8  — GitHub secrets added, workflows tested
[ ] Phase 9  — BPA rules tuned to your standards
[ ] Phase 10 — copilot-instructions.md customised
[ ] Phase 11 — README rewritten, team onboarded
```

---

## 12. Recommendations to Improve This Repo

The demo is a great starting point but several things would make it more production-ready. Consider adopting these in **your** copy:

### 12.1 Branching & environment strategy
- Add a **branch protection rule** on `main`: require the BPA workflow to pass and at least one review.
- Introduce a **`develop` → `main` flow** where:
  - Push to `develop` → auto-deploy to DEV.
  - PR `develop → main` → run BPA + integration smoke tests.
  - Merge to `main` → manual approval → deploy to PRD.
- Use **GitHub Environments** (Settings → Environments) with required reviewers and environment-scoped secrets, instead of repo-level secrets. This lets you keep PRD credentials separate from DEV.

### 12.2 Better authentication
- Replace the SPN client secret with **federated credentials (OIDC)** using `azure/login@v2`. No secrets stored in GitHub at all — GitHub Actions exchanges its OIDC token for an Azure access token at runtime.
- Or store the secret in **Azure Key Vault** and pull it at workflow runtime, so rotation is handled centrally.

### 12.3 Stronger CI gates
- Add a **TMDL syntax check** step (Tabular Editor 3 CLI or `pbi-tools`) so malformed TMDL fails fast, before BPA.
- Add a **DAX query test step**: store sample DAX queries (like `Model03.SemanticModel/DAXQueries/`) and run them after deploy with `Invoke-ASCmd` or `pbi-tools`, asserting expected row counts. This catches breaking changes to measures.
- Add a **schema-diff comment** to PRs (e.g. via `pbi-tools diff`) so reviewers see "Measure X was renamed, Column Y was deleted" in plain English.
- Add a **Power BI Inspector report annotation summary** to the PR body so reviewers don’t need to dig through logs.

### 12.4 Repository hygiene
- Add a `CODEOWNERS` file so model folders auto-request review from data engineers and report folders from BI analysts.
- Add a `CONTRIBUTING.md` with the local dev steps from Phase 0–7 of this guide.
- Add a `.editorconfig` to lock indentation (TMDL is whitespace-sensitive in places).
- Pin tool versions in `bpa.ps1`: today it always downloads the *latest* Tabular Editor and PBI Inspector, which means a remote release can suddenly break your pipeline. Pin a specific release URL and bump it via PR.
- Cache the `_tools/` folder in CI using `actions/cache` to cut a few minutes off each run.

### 12.5 Parameterisation improvements
- Move every connection string, capacity ID, and workspace ID into named **M parameters** in `expressions.tmdl` so `parameter.yml` only has to swap one line per value, never deep inside a partition expression.
- Add a `parameter.yml` validator step in CI: a small Python script that checks every `find_value` actually exists in the codebase, so a typo fails CI instead of silently doing nothing at deploy time.
- Support a `LOCAL` environment in `parameter.yml` so devs can run against a local SQL/Fabric instance.

### 12.6 Observability & rollback
- After each deploy, write a JSON manifest (commit SHA, deployed-at timestamp, item versions) to a folder in the workspace’s OneLake. Lets you answer "what version is in PRD right now?".
- Add a **rollback workflow** that, given a Git tag, checks out that tag and re-runs `deploy.py`. Simpler than trying to undo a partial deployment.
- Tag every successful PRD deploy in Git (`git tag prd-2026.04.24-1`) automatically from the workflow.

### 12.7 Testing & data quality
- Add **DAX unit tests** with a tool like [Best Practice Tester](https://github.com/m-kovalsky/Tabular) or custom DAX queries, executed against the deployed DEV model.
- Add **row-count drift checks** comparing source vs published.
- Run **PBI Inspector accessibility rules** (alt text on visuals, color-contrast) so reports stay accessible.

### 12.8 Documentation generation
- Generate a Markdown data dictionary from the TMDL on every push (a small Python script reading `definition/tables/*.tmdl` and emitting `docs/data-dictionary.md`). Publishes the model to non-developers.
- Auto-render the model diagram (`diagramLayout.json` + `relationships.tmdl`) as a Mermaid ER diagram in the README on every commit.

### 12.9 Secret & dependency hygiene
- Add **Dependabot** or **Renovate** to keep `fabric-cicd`, `actions/checkout`, `actions/setup-python`, and the BPA tool versions current.
- Add **Gitleaks / TruffleHog** as a CI step to catch accidental commits of connection strings or tokens inside TMDL.

### 12.10 Developer experience
- Provide a `make dev` / PowerShell `Invoke-Build` script that wraps the most common commands: `setup`, `bpa`, `deploy DEV`, `deploy PRD`. Less to memorise.
- Add **VS Code workspace settings** (`.vscode/settings.json`) recommending the TMDL extension, JSON schema for `.pbir` files, and disabling word-wrap in TMDL.
- Add **GitHub issue templates** for "Bug in report", "New measure request", "New report request" — keeps requests structured.

### 12.11 Scaling beyond reports & models
- The pipeline supports more Fabric item types: **Notebooks, Data Pipelines, Lakehouses, Warehouses, Environments, Eventstreams**. As your project grows, add them to `item_type_in_scope` in `deploy.py` so the whole solution is one repo.
- Split very large repos into a **monorepo with multiple `src/<domain>/`** subfolders, deployed by separate workflows, to keep BPA runs fast.

---

## 13. Mental-Model Cheat Sheet

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

## 14. Recommended Reading / Tools

- Power BI Project (PBIP) format docs — Microsoft Learn.
- TMDL language reference — Microsoft Learn.
- `fabric-cicd` library — https://microsoft.github.io/fabric-cicd/
- Tabular Editor 2 (free) — for model editing & BPA.
- PBI Inspector v2 — report-side BPA.
- VS Code extensions: **TMDL** (syntax + IntelliSense), **Power Query / M**.

---

### TL;DR

> The repo treats Power BI content as code. **Power BI Desktop** edits the files in `src/`; **Git** versions them; **fabric-cicd + GitHub Actions** publishes them per environment using `parameter.yml`; and **BPA** keeps quality high on every PR. Copy `src/`, swap in your own models/reports, edit `parameter.yml` and the GitHub secrets, and you have a production-grade Power BI delivery pipeline.
