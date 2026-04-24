# `.bpa/` — Best Practice Analyzer Guide

This document explains the **`.bpa/` folder** in this repository: what it is, why it exists, what each file does, how it runs locally and in CI, and how to extend it for your own project.

---

## 1. What is BPA?

**BPA = Best Practice Analyzer.** It is the equivalent of a **linter / static-analysis tool** for Power BI content. Instead of running the published model and looking at numbers, BPA reads the source files (TMDL for models, JSON for reports) and checks them against a set of rules — for example:

- "Don’t use the `Double` (floating point) data type on columns."
- "Hide foreign-key columns."
- "Mark date tables as a date table."
- "Don’t put more than 20 visuals on a single page."
- "Don’t reference custom visuals that are not used."

When a rule is violated, BPA reports it. In CI, that report **fails the Pull Request** so bad practice never reaches `main`.

---

## 2. Why this repo needs `.bpa/`

Without an automated check, every PR reviewer would have to manually remember dozens of best practices. That doesn’t scale. The `.bpa/` folder gives the repo:

| Goal | How `.bpa/` delivers it |
|---|---|
| **Consistent quality** | Same rules applied to every commit, every author. |
| **Fast feedback** | Developers can run BPA locally before pushing. |
| **Merge gate** | CI fails the PR if rules are violated → bad models can’t reach `main`. |
| **Versioned standards** | Rules live in source control, so changes go through PR review. |
| **Self-contained** | No external server, no shared "BPA service" to maintain. |

In short, `.bpa/` is the repo’s **"linter for Power BI"**.

---

## 3. Folder contents

```
.bpa/
├── bpa.ps1                          ← Orchestrator script (PowerShell)
├── bpa-rules-semanticmodel.json     ← Rules for semantic models (Tabular Editor)
└── bpa-rules-report.json            ← Rules for reports (PBI Inspector)
```

### 3.1 `bpa.ps1`
A PowerShell script that does three things:

1. **Downloads tools on demand** into a local `_tools/` folder (which is `.gitignore`d):
   - **Tabular Editor (Portable)** — analyses semantic models.
   - **PBI Inspector** — analyses reports.
   It also downloads each tool’s **default rule set** as a fallback.
2. **Discovers Power BI items** under the path you pass in (`-src`). It looks for:
   - `*.pbism` → semantic model → run **Tabular Editor**.
   - `*.pbir` → report → run **PBI Inspector**.
3. **Runs each tool** with the appropriate rules JSON. If a rule violation is found, the underlying tool returns a non-zero exit code and the script throws — which fails the CI job.

Key behaviour points:
- If `bpa-rules-semanticmodel.json` is missing → uses Tabular Editor’s default rules.
- If `bpa-rules-report.json` is missing → uses PBI Inspector’s default rules.
- PBI Inspector is run with `-formats GitHub` so violations appear as **inline annotations on the PR diff**.

### 3.2 `bpa-rules-semanticmodel.json`
The semantic-model rule set. Each entry follows the **Microsoft "BPARules"** schema used by Tabular Editor:

```json
{
  "ID": "AVOID_FLOATING_POINT_DATA_TYPES",
  "Name": "[Performance] Do not use floating point data types",
  "Category": "Performance",
  "Description": "...",
  "Severity": 2,
  "Scope": "DataColumn, CalculatedColumn, CalculatedTableColumn",
  "Expression": "DataType = \"Double\"",
  "FixExpression": "DataType = DataType.Decimal",
  "CompatibilityLevel": 1200
}
```

Important fields:
- `Scope` — which TOM object types the rule applies to (Table, Column, Measure, Model…).
- `Expression` — a C#-flavoured boolean expression evaluated against each in-scope object. If it returns `true`, the rule fires.
- `Severity` — 1 (info), 2 (warning), 3 (error). The script treats any violation as a failure.
- `FixExpression` — optional auto-fix that Tabular Editor can apply interactively.

### 3.3 `bpa-rules-report.json`
The report-side rule set, in **PBI Inspector** format (uses JsonLogic-style operators):

```json
{
  "id": "REDUCE_VISUALS_ON_PAGE",
  "name": "Reduce the number of visible visuals on the page",
  "disabled": false,
  "part": "Pages",
  "test": [ ... ]
}
```

Important fields:
- `part` — which part of the report the test runs against (Report, Pages, Visuals, …).
- `disabled` — set to `true` to silence a rule without deleting it.
- `test` — the JsonLogic expression that decides pass/fail.

---

## 4. How `.bpa/` is wired into CI

Workflow: [.github/workflows/bpa.yml](.github/workflows/bpa.yml)

- **Triggers**: Pull Requests to `main` that touch `*.SemanticModel/**` or `*.Report/**`, plus manual `workflow_dispatch`.
- **Runner**: `windows-latest` (Tabular Editor is a .NET Windows tool).
- **Steps**:
  1. Check out the code.
  2. Run `.\.bpa\bpa.ps1 -src @("./src/*.SemanticModel")`.
  3. Run `.\.bpa\bpa.ps1 -src @("./src/*.Report")`.
- **Outcome**: Either step exits non-zero → workflow turns red → PR cannot be merged (if branch protection requires it).

---

## 5. Running BPA locally

You can run the exact same checks before you push, so you don’t waste CI cycles:

```powershell
# From the repo root
.\.bpa\bpa.ps1 -src @(".\src\*.SemanticModel")
.\.bpa\bpa.ps1 -src @(".\src\*.Report")
```

The first run downloads ~50 MB of tools into `.bpa\_tools\` (one-time). Subsequent runs are fast.

> Tip: pin this as a VS Code task so you can hit `Ctrl+Shift+B` to lint the whole repo.

---

## 6. Customising the rules

The shipped rule files are a **starting point**, not a final standard. Adapt them to your team:

### Disable a rule
- **Semantic model**: delete its entry from `bpa-rules-semanticmodel.json` (or set `Severity: 0`).
- **Report**: set `"disabled": true` on the rule in `bpa-rules-report.json`.

### Add a rule
- **Semantic model**: copy a similar entry, change `ID`, `Name`, `Scope`, and `Expression`. Microsoft maintains a community catalogue here:  
  https://github.com/microsoft/Analysis-Services/tree/master/BestPracticeRules
- **Report**: see PBI Inspector’s rule reference:  
  https://github.com/NatVanG/PBI-InspectorV2

### Ignore a rule on a specific object
TMDL supports a per-object override using an annotation:

```tmdl
column 'Order Date'
    annotation BestPracticeAnalyzer_IgnoreRules = {"RuleIDs":["HIDE_FOREIGN_KEYS"]}
```

This is already used in some columns of `Sales.tmdl`. Use it sparingly and document why.

### Reset to defaults
If you delete `bpa-rules-semanticmodel.json` and/or `bpa-rules-report.json` entirely, the script falls back to each tool’s **default** rule set — useful as a clean baseline.

---

## 7. Tools used under the hood

| Tool | Used for | Download |
|---|---|---|
| **Tabular Editor (Portable)** | Semantic-model BPA (`*.pbism`) | https://github.com/TabularEditor/TabularEditor |
| **PBI Inspector** | Report BPA (`*.pbir`) | https://github.com/NatVanG/PBI-InspectorV2 |

Both are open-source CLI tools downloaded automatically by `bpa.ps1` into `.bpa/_tools/` (gitignored).

---

## 8. Recommendations to harden `.bpa/`

If you adopt this pattern in your own repo, consider:

1. **Pin tool versions.** `bpa.ps1` currently downloads the *latest* release of each tool. A breaking release upstream can suddenly fail your pipeline. Replace `latest/download` URLs with a specific tag (e.g. `download/2.25.0/...`) and bump it via PR.
2. **Cache the `_tools/` folder in CI** with `actions/cache` — saves ~30s per run.
3. **Run BPA on every push to `main`**, not only on PRs (re-enable the commented `push` trigger in `bpa.yml`).
4. **Add a Tabular Editor 2 syntax check** before BPA so malformed TMDL fails fast.
5. **Treat severity 1 rules as warnings, not failures** by post-processing the output, if you want a softer gate.
6. **Document each custom rule** with a one-line description in this file so reviewers know why it exists.
7. **Add accessibility rules** to `bpa-rules-report.json` (alt text on visuals, sufficient colour contrast).
8. **Wire BPA into a pre-commit hook** for developers who want the fastest possible feedback loop.

---

## 9. Cheat sheet

| Question | Answer |
|---|---|
| What does `.bpa/` do? | Lints Power BI models and reports against a rule set. |
| Where do rules live? | `.bpa/bpa-rules-semanticmodel.json` and `.bpa/bpa-rules-report.json`. |
| What runs the rules? | `.bpa/bpa.ps1` — calls Tabular Editor and PBI Inspector. |
| When does it run in CI? | On every PR to `main` touching `*.SemanticModel/**` or `*.Report/**`. |
| Can I run it locally? | Yes — `.\.bpa\bpa.ps1 -src @(".\src\*.SemanticModel")`. |
| How do I silence a rule for one column? | Add `annotation BestPracticeAnalyzer_IgnoreRules = {"RuleIDs":["..."]}` in the TMDL. |
| How do I add a new rule? | Append a JSON entry following the schema documented above. |

---

### TL;DR

> `.bpa/` is the repo’s automated quality gate for Power BI. `bpa.ps1` downloads Tabular Editor and PBI Inspector, then runs them against the rule sets in `bpa-rules-*.json` over every model and report under `src/`. CI fails the PR on any violation, so substandard TMDL or report JSON never reaches `main`.
