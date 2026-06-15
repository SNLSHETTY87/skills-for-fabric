---
name: ai-readiness-fixer
description: >
  Scan a local Power BI PBIP SemanticModel folder for Copilot/AI readiness issues, apply fixes to
  local TMDL files, and generate HTML + Excel reports. Covers all 6 readiness checks: architecture,
  naming, descriptions, AI instructions, AI data schema, verified answers.
  Use when the user provides a local .SemanticModel folder path or .pbip file path.
---

> **STOP — Read this skill in full before taking any action.**
> Do not begin file operations until you have read and internalized all sections below.
> The backup step is MANDATORY before touching any file.

# AI Readiness Fixer Skill

You help users prepare Power BI semantic models for Copilot and conversational BI. You read and edit
local TMDL files, present plain-language findings, and apply changes only after explicit confirmation.

## Reference Files

Load these skills before starting:
- `skills/semantic-model-authoring/references/semantic-model-ai-readiness.md` — 6-point checklist rules
- `skills/semantic-model-authoring/references/naming-conventions.md` — naming rules
- `skills/semantic-model-authoring/references/tmdl-guidelines.md` — TMDL syntax rules
- `skills/semantic-model-authoring/references/pbip.md` — PBIP folder structure

## PBIP SemanticModel Folder Structure

```
<Name>.SemanticModel/
├── definition.pbism
├── definition/
│   ├── database.tmdl
│   ├── model.tmdl
│   ├── relationships.tmdl          (may not exist)
│   └── tables/
│       ├── <TableName>.tmdl        (one per table)
│       └── ...
└── Copilot/                        (may not exist)
    ├── AIInstructions.md           (or .txt — AI Instructions file)
    └── AIDataSchema.json           (AI Data Schema scoping file)
```

## Output Folder (always outside the PBIP folder)

```
<parent-of-SemanticModel>/
├── <Name>.SemanticModel/           ← NEVER place output here
└── _ai-readiness/                  ← ALL output goes here
    ├── backup-<YYYYMMDD-HHmm>/     ← full copy of SemanticModel
    ├── report-<YYYYMMDD-HHmm>.html
    └── report-<YYYYMMDD-HHmm>.xlsx
```

---

## Workflow

### Step 0 — Locate the SemanticModel Folder

If user provides a `.pbip` file path:
- Find the `.SemanticModel/` folder in the same directory as the `.pbip` file

If user provides the `.SemanticModel/` folder directly — use it as-is.

Confirm: *"Found Sales.SemanticModel with 5 tables. Let me ask 2 quick questions before scanning."*

### Step 1 — Gather Business Context (2 questions)

Ask both questions together before doing anything else:

> 1. What does this model measure? (e.g. retail sales, HR performance, marketing spend)
> 2. Who will query it with Copilot? (e.g. sales managers, finance team, executives)

Store answers — use them to write descriptions and AI Instructions.

### Step 2 — Backup (MANDATORY before any edits)

1. Determine timestamp: `YYYYMMDD-HHmm`
2. Determine output dir: `<parent-of-SemanticModel>/_ai-readiness/`
3. Copy entire `.SemanticModel/` folder → `_ai-readiness/backup-<timestamp>/`
4. Confirm to user: *"Backup saved to _ai-readiness/backup-20260615-1430/ — nothing changed yet."*

Do not proceed to scan until backup is confirmed complete.

### Step 3 — Scan (read-only)

Read all TMDL files from the `.SemanticModel/` folder. Run the 6-point checklist:

#### Check 1 — Architecture

Read all `definition/tables/*.tmdl` files. Flag:
- Tables with no explicit DAX measures (only columns, no `measure` blocks)
- Obvious flat/wide tables (single table with 20+ columns, no relationships)
- Columns referenced in many measures that appear unused in reports
- Duplicate or overlapping measure names (e.g. `Sales` and `Total Sales` with identical expressions)

Do NOT block on minor architecture issues — note them and continue. Only stop if there are NO explicit measures at all across the entire model.

#### Check 2 — Naming

For every visible (non-hidden) table, column, and measure, check:
- Contains underscores (`_`), dots, or is all caps → flag
- Contains abbreviations: any token under 4 chars that is not a known unit (e.g. `amt`, `qty`, `id`, `dt`, `cd`, `no`) → flag
- CamelCase without spaces (e.g. `TotalRevenue`, `CustomerID`) → flag
- Numeric columns (non-ID, non-date) with `summarizeBy: None` when they should sum → note
- ID/key/code/postal columns with `summarizeBy` set to anything other than `None` → flag
- Date columns with `summarizeBy` not set to `None` → flag
- Dimension tables with no column marked `isDefaultLabel: true` → flag

#### Check 3 — Descriptions

For every visible table, column, and measure:
- No `description` property → flag as missing
- Description is empty string → flag as missing
- Description is > 200 characters → flag (Copilot only reads first 200)
- Description restates the object name word-for-word → flag (e.g. column `Sales Amount` with description "Sales Amount")

#### Check 4 — AI Instructions

Check if `Copilot/` folder exists and contains an AI Instructions file.

**If file is missing:** flag as absent.

**If file exists — analyze quality:**
- Abbreviations used but never defined: find patterns like `USE [ABC]` or `TMS` where the expansion is not in the same file
- Ambiguous metric terms with no routing: words like "margin", "revenue", "sales", "profit", "growth" that appear without a specific measure reference `[MeasureName]`
- Contradictions with measure descriptions: if a measure description says "use for X" but AI Instructions say "use [that measure] for Y"
- Missing fiscal year / time-period definition if the model has a date table
- Missing polarity guidance for measures where lower is better (attrition, cost, errors)
- Exceeds 10,000 characters → flag with character count
- Duplicate rules: same guidance stated twice in different words

#### Check 5 — AI Data Schema

Check if `Copilot/AIDataSchema.json` (or equivalent) exists in the `Copilot/` folder.

**If missing:** flag — Copilot will see all tables including helper/bridge tables.

**If exists:** check:
- Tables in the model not present in the schema definition → note as potentially hidden from Copilot
- Helper/bridge table names (containing "Bridge", "Helper", "Mapping", "Temp", "Staging") that ARE included → flag

#### Check 6 — Verified Answers

Check if `Copilot/` folder contains any Verified Answers file.

If none, inspect all measures and suggest the top 5 question candidates based on:
- Measures with names implying totals or KPIs (Revenue, Sales, Count, Rate, %)
- Measures with time-intelligence suffixes (YTD, MTD, LY, Growth)
- Format: `"What is [measure name]?" → [MeasureName]`

### Step 4 — Scorecard

Present a plain-language scorecard. Never use technical terms (TMDL, TOM, DAX) in user-facing text.

```
AI Readiness Check — <ModelName>
====================================
✅ / ⚠️ / ❌  Architecture      <summary>
✅ / ⚠️ / ❌  Naming             <N> issues found
✅ / ⚠️ / ❌  Descriptions       <N> of <total> objects missing descriptions
✅ / ⚠️ / ❌  AI Instructions    <present/missing> [<N> issues if present]
✅ / ⚠️ / ❌  AI Data Schema     <configured/not configured>
ℹ️             Verified Answers  <N found / none — top 5 suggestions below>

Agent can fix now:  naming · descriptions · AI instructions · AI data schema
Needs Desktop:      verified answers

What would you like me to fix? (all / naming / descriptions / ai-instructions / ai-schema)
```

### Step 5 — Show Diff Before Applying

For each category the user selects, show a complete before/after table.

**Naming diff format:**
```
definition/tables/FactSales.tmdl
  tr_amt           →  Transaction Amount     (summarizeBy unchanged: Sum)
  cust_id          →  Customer ID            (summarizeBy: Sum → None)
  ord_dt           →  Order Date             (summarizeBy: Sum → None)
```

**Descriptions diff format:**
```
table Customer
  [no description]  →  "Customer records. Filter by name, region, or segment to narrow results."

measure [Total Revenue]
  [no description]  →  "Total net revenue after discounts. Primary top-line KPI."
```

**AI Instructions diff format (existing file):**
```
Line 4:  "Use TMS for campaign analysis"
       →  "Use TMS (Total Media Spend = [Total Media Spend]) for campaign analysis"

Line 12: removed — contradicts measure description; moved content to measure description instead

[File trimmed from 11,200 to 9,950 chars — last section condensed]
```

**AI Instructions diff format (new file):**
```
Creating Copilot/AIInstructions.md — draft based on your business context:
---
[preview of generated content]
---
Review this draft and tell me what to adjust before I save it.
```

Ask: *"Apply these <N> changes? (yes / no / tell me which ones to skip)"*

### Step 6 — Apply Changes

Apply only after explicit user confirmation ("yes", "go ahead", "apply", "proceed").

#### Editing TMDL files

For each flagged column/measure/table in the relevant `.tmdl` file:
- Rename: change the identifier on the `column`, `measure`, or `table` declaration line
- Description: add or replace the `description: "..."` property (on the line after the declaration, indented one level)
- summarizeBy: change the `summarizeBy:` property value
- isDefaultLabel: add `isDefaultLabel: true` on the key label column of a dimension table
- Hidden flag: add `isHidden` or change to `isHidden: true` on columns flagged as unused

**TMDL editing rules (critical):**
- Preserve ALL other content in the file exactly as-is
- Maintain exact indentation (tabs, not spaces) as found in the original file
- After editing, re-read the file to verify the change landed correctly
- Never touch `partition`, `source`, M expression blocks, or relationship definitions

#### Editing Copilot files

- If `Copilot/` folder does not exist: create it
- AI Instructions: write to `Copilot/AIInstructions.md`
- AI Data Schema: write to `Copilot/AIDataSchema.json` — generate a scoping object that excludes helper/bridge/staging tables and marks all business-facing tables as included

### Step 7 — Generate Report

After applying (or if user requests report without applying), generate output files in `_ai-readiness/`.

#### HTML Report

Write `_ai-readiness/report-<timestamp>.html` — a self-contained HTML file, no external CSS or JS dependencies.

Structure:
```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>AI Readiness — <ModelName></title>
<style>/* inline styles only */</style>
</head>
<body>
  <h1>AI Readiness Report — <ModelName></h1>
  <p>Generated: <date> | Backup: <backup-path></p>

  <!-- Scorecard -->
  <h2>Scorecard</h2>
  <table>... one row per check with status icon ...</table>

  <!-- Changes Applied -->
  <h2>Changes Applied</h2>
  <table>... before/after for every change ...</table>

  <!-- Still To Do -->
  <h2>What To Do Next (Power BI Desktop)</h2>
  <ul>
    <li>Verified Answers — top 5 candidates: ...</li>
    <li>Review AI Instructions draft in Copilot/AIInstructions.md</li>
  </ul>

  <!-- Restore Instructions -->
  <h2>Restore From Backup</h2>
  <p>Replace <Name>.SemanticModel/ with the backup folder at: <backup-path></p>
</body>
</html>
```

#### Excel Report

Write `_ai-readiness/report-<timestamp>.xlsx` using this Python snippet (run via Bash):

```python
import openpyxl
from openpyxl.styles import PatternFill, Font
from datetime import datetime

wb = openpyxl.Workbook()

# Sheet 1: Summary
ws = wb.active
ws.title = "Summary"
ws.append(["Check", "Status", "Issues Found", "Fixed by Agent", "Needs Desktop"])
# ... populate rows from scan results ...

# Sheet 2: Naming Changes
ws2 = wb.create_sheet("Naming Changes")
ws2.append(["File", "Original Name", "New Name", "Property Changed", "Old Value", "New Value"])
# ... populate from changes list ...

# Sheet 3: Descriptions
ws3 = wb.create_sheet("Descriptions Added")
ws3.append(["Object Type", "Object Name", "Description"])
# ... populate ...

# Sheet 4: AI Instructions
ws4 = wb.create_sheet("AI Instructions")
ws4.append(["Line", "Issue", "Before", "After"])
# ... populate ...

# Sheet 5: Verified Answer Suggestions
ws5 = wb.create_sheet("Verified Answer Suggestions")
ws5.append(["Priority", "Question", "Recommended Measure", "How to Configure"])
# ... populate top 5 ...

# Sheet 6: Restore Guide
ws6 = wb.create_sheet("Restore Guide")
ws6.append(["Step", "Action"])
ws6.append(["1", "Close Power BI Desktop if the model is open"])
ws6.append(["2", f"Delete or rename: <SemanticModel path>"])
ws6.append(["3", f"Copy backup folder to original location"])
ws6.append(["4", "Rename copied folder to original name"])
ws6.append(["5", "Open .pbip file in Power BI Desktop to verify"])

wb.save("_ai-readiness/report-<timestamp>.xlsx")
```

Install openpyxl if needed: `pip install openpyxl` (tell the user if it's not installed).

### Step 8 — Final Confirmation

```
✅ <N> changes applied to <ModelName>

  Naming:          <N> objects renamed
  Descriptions:    <N> descriptions added/updated
  AI Instructions: <created/updated> — Copilot/AIInstructions.md
  AI Data Schema:  <created/updated> — Copilot/AIDataSchema.json

Output saved to: <parent>/_ai-readiness/
  📄 report-<timestamp>.html    ← open in browser
  📊 report-<timestamp>.xlsx    ← open in Excel
  💾 backup-<timestamp>/        ← restore point

Still to do in Power BI Desktop:
  → Verified Answers (top 5 candidates in the report)
  → Review AI Instructions draft before publishing

Next: open the .pbip file in Power BI Desktop to validate, then publish to Fabric.
```

---

## Must / Prefer / Avoid

### MUST
- Backup before any edits — no exceptions
- Show diff and get "yes" before applying any change
- Save all output outside the `.SemanticModel/` folder
- Re-read each TMDL file after editing to confirm changes are correct
- Tag every finding: agent-fixable vs Desktop-only
- If no explicit measures exist in the model → stop, report architecture failure, do not proceed with other checks

### PREFER
- Plain business language — never expose TMDL, TOM, DAX terminology to the user
- Gathering business context before writing any descriptions or AI Instructions
- Generating both HTML and Excel report by default

### AVOID
- Editing partition blocks, M expressions, or data source connection strings in TMDL files
- Auto-authoring Verified Answers (suggest candidates only)
- Placing any file inside the `.SemanticModel/` folder
- Generating AI descriptions without user business context
