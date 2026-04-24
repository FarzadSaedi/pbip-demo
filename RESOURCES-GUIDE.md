# `.resources/` — Shared Resources Folder Guide

This document explains the **`.resources/` folder** in this repository: what lives there, why it’s separate from `src/`, and the **recommended approach when starting a new project**.

---

## 1. Purpose

`.resources/` holds **reusable assets that are shared across many reports or models** but are **not themselves Power BI items that get deployed**. Think of it as a "scaffolding / template" folder used by humans (and Copilot), not by `fabric-cicd`.

Crucially:
- The CI deploy script points at `src/`, **not** `.resources/`. Anything in `.resources/` is **never** pushed to Fabric.
- The leading dot keeps it out of the way of the main authoring folder.

---

## 2. Folder contents

```
.resources/
├── templateReport/          ← Empty/seed report you copy when creating new reports
│   ├── definition.pbir
│   ├── definition/
│   ├── StaticResources/
│   └── .platform
├── themes/                  ← Brand themes + the brand logo
│   ├── fabric_48_color.svg
│   ├── theme-black.json
│   ├── theme-blue.json
│   └── theme-purple.json
└── tmdl/                    ← Reusable TMDL snippets (Calendar, Time Intelligence, …)
    ├── Calendar.tmdl
    └── Time Intelligence.tmdl
```

### 2.1 `templateReport/`
A pre-built **empty PBIR report** with the team’s defaults already applied (theme, page size, default visuals header settings, etc.). When a developer needs a new report, they copy this folder into `src/` and rename it instead of starting from a blank Power BI Desktop file.

This guarantees every new report starts with the same look and feel.

### 2.2 `themes/`
JSON theme files (and the brand logo SVG) that reports reference under `StaticResources/RegisteredResources/`. Keeping a single canonical copy here means:
- One place to update brand colours.
- Consistency across all reports.
- The same theme can be embedded into the template and into existing reports.

### 2.3 `tmdl/`
Reusable TMDL building blocks. The two shipped examples are:
- `Calendar.tmdl` — a fully-formed Calendar table you can paste into a model that doesn’t yet have one.
- `Time Intelligence.tmdl` — a calculation group with the standard YTD / MTD / PY / YoY measures.

These are not consumed by `fabric-cicd`. A developer manually copies them into a model’s `definition/tables/` folder (or into a TMDL script) when needed.

---

## 3. Why a separate, dotted folder?

| Concern | Why `.resources/` solves it |
|---|---|
| **Don’t deploy templates** | `fabric-cicd` only sees `src/`; `.resources/` is invisible to it. |
| **Don’t lint templates** | The BPA workflow targets `src/`, so half-built snippets in `.resources/` won’t fail CI. |
| **Discoverability** | One canonical location for "things every project copies in". |
| **Versioning** | Standards live in Git; updates go through PR review. |
| **Avoid noise** | The dot prefix collapses it in many file explorers and keeps it out of the main authoring path. |

---

## 4. Recommended approach when starting a new project

### Step 1 — Decide what is truly reusable
A good candidate for `.resources/` is anything that:
- Is copied into more than one report or model.
- Defines house style (colours, fonts, page size, default visual settings).
- Is a known-good pattern (calendar table, time-intelligence calc group, RLS role template).

Don’t put project-specific content here — that belongs in `src/`.

### Step 2 — Build a `templateReport`
1. In Power BI Desktop, create an empty report.
2. Apply your brand theme, set page size, header/footer text boxes, default tooltip page, etc.
3. Save as `.pbip` into `.resources/templateReport/`.
4. Strip out anything experimental — this folder is the **bare minimum** every report must start from.

### Step 3 — Curate `themes/`
- Export your final theme(s) from Power BI Desktop (View → Themes → Save current theme).
- Save the JSON into `.resources/themes/`.
- Place the company logo (SVG preferred) next to them.
- Document one theme as the **default** in your project README.

### Step 4 — Curate reusable TMDL snippets
- A standard **Calendar** table.
- A **Time Intelligence** calculation group.
- A **RLS role template** with placeholders.
- Optional: a "starter measures" file with the 10 measures every model in your domain needs (Sales Amount, # Customers, Margin %, etc.).

Each snippet should be **self-contained** (no `lineageTag`, no model-specific references) so it pastes cleanly into any model.

### Step 5 — Document the "how to use" in this folder
Add a short `README.md` inside `.resources/` describing:
- How to start a new report (copy `templateReport/` → `src/<MyReport>.Report/`, rename, point `definition.pbir` at the right model).
- How to apply a theme to an existing report.
- How to import a TMDL snippet (paste into `definition/tables/` or run via TMDL Scripts).

### Step 6 — Keep it lean
Anything unused for 3 months should be reviewed for deletion. `.resources/` should not become a graveyard.

---

## 5. How developers consume `.resources/` in practice

### Creating a new report from the template
```powershell
# From the repo root
Copy-Item -Recurse .\.resources\templateReport .\src\SalesExecutive.Report
# Rename the .pbir reference if needed and update its datasetReference.byPath path.
```

### Updating a report’s theme
1. Open the report folder in VS Code.
2. Replace `StaticResources/RegisteredResources/<theme>.json` with the file from `.resources/themes/`.
3. Update `report.json` if the theme name changed.

### Adding a Calendar table to a new model
1. Copy `.resources/tmdl/Calendar.tmdl` into `src/<Model>.SemanticModel/definition/tables/`.
2. Add `ref table Calendar` to `definition/model.tmdl`.
3. Open the model in Desktop, mark the table as a date table, build relationships.

---

## 6. Recommendations for new projects

1. **Start with empty `.resources/` subfolders** — only add an asset once you’ve copied it twice. Premature templating wastes effort.
2. **Pin a single default theme.** Multiple themes are nice but lead to inconsistent dashboards.
3. **Put a `.resources/README.md`** in the folder explaining each asset and how to use it. Future-you will thank you.
4. **Keep TMDL snippets generic.** No `lineageTag`, no hard-coded server names, no measures referencing other measures that may not exist.
5. **Treat `.resources/` as code.** Changes to it should go through a PR, not be silently overwritten.
6. **Don’t store secrets here.** Connection strings, API keys, passwords have no business in any file — use Fabric data source credentials or Key Vault.
7. **Add a `CHANGELOG.md`** in `.resources/` so consumers know when a theme or template changed and need to update.
8. **Provide a small PowerShell/Python helper** (`new-report.ps1 -Name SalesExecutive -Model Sales`) that automates the copy+rename+`byPath` rewrite. Less manual error.
9. **Apply BPA / linting** to your TMDL snippets too. You can do this by symlinking them temporarily under a throwaway model in CI, or simply by running them through Tabular Editor manually before checking in.
10. **Decide between `.resources/`, `docs/`, and `tools/`** intentionally. Rule of thumb:
    - `.resources/` = reusable Power BI artefacts (themes, templates, TMDL snippets).
    - `docs/` = human-readable documentation (Markdown, diagrams).
    - `tools/` (or `scripts/`) = automation scripts that touch the repo or CI.

---

## 7. Cheat sheet

| File / folder | Used for | Consumed by |
|---|---|---|
| `templateReport/` | Starter for any new report | Developer copy → `src/` |
| `themes/*.json` | Brand themes | Pasted into a report’s `StaticResources` |
| `themes/*.svg` | Brand logo | Embedded in reports / docs |
| `tmdl/Calendar.tmdl` | Standard date table | Pasted into a model’s `tables/` |
| `tmdl/Time Intelligence.tmdl` | Calc group with standard time measures | Pasted into a model |

---

### TL;DR

> `.resources/` is the team’s **toolbox of reusable Power BI assets** — templates, themes, TMDL snippets — that humans copy from when bootstrapping new reports and models. It is intentionally outside `src/` so the deploy and BPA pipelines never see it.
