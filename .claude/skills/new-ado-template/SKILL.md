---
name: new-ado-template
description: Scaffolds a new Azure DevOps pipeline template in this repo (ado-pipeline-templates) — the YAML in templates/, a matching page in docs/, and a new row in README.md's "Available Templates" table — following the exact structure of the existing ado-build-devsecops-pipeline.yaml (step template) and ado-terraform-pipeline.yaml (stage template). Use whenever the user asks to add, create, or scaffold a new pipeline template in this repo, or asks "how do I add a template here".
license: MIT
---

# new-ado-template

This repo publishes reusable Azure DevOps YAML templates. Every existing
template follows the same three-artifact pattern; a new template must match
it exactly, not invent a new layout. Read the two existing templates and
their docs before writing anything new — they are the spec, this file just
names the parts.

## The three artifacts, always together

1. **`templates/<name>.yaml`** — the template itself.
2. **`docs/<name>.md`** — a page with the *same base filename*, describing it.
3. **`README.md`** — one new row in the **📦 Available Templates** table
   (under `## Usage`), and removal of the entry from the **🔮 Roadmap**
   section if it was listed there as planned.

Never add one without the other two. A template with no doc page, or a doc
page not linked from the README table, is an incomplete PR in this repo.

## 0. Before starting — ask if not obvious

- **Step or stage template?** Step templates (`ado-build-devsecops-pipeline.yaml`)
  are consumed inside a job's `steps:` list. Stage templates
  (`ado-terraform-pipeline.yaml`) are consumed at the top-level `stages:` list
  and can define their own jobs/approval gates. This determines the doc's
  "Type" column and the top-level YAML key (`steps:` vs `stages:`).
- **What concerns does it gate?** Every concern (scan, build, test, deploy...)
  should be behind its own boolean toggle parameter, not always-on.
- **Does it reuse an existing scan pattern?** Secret scanning and IaC scanning
  in both existing templates use the same Trivy-based approach
  (`secretsScan`/`iacScan` booleans, `scanSeverity` default `'CRITICAL,HIGH'`,
  `scanExitCode` default `'1'`, JUnit output merged into one **Security
  Scans** test run per job via `$(Common.TestResultsDirectory)`). Reuse this
  pattern rather than inventing a new one if the new template also scans.

## 1. `templates/<name>.yaml`

- Filename: `ado-<purpose>-pipeline.yaml` (matches `ado-build-devsecops-pipeline.yaml`,
  `ado-terraform-pipeline.yaml`).
- `parameters:` block first, grouped under `# ---- <group> ----` comments,
  e.g. `# ---- toggles: turn each concern on/off ----`, then shared config,
  then feature-specific config groups. See
  `templates/ado-build-devsecops-pipeline.yaml` for the multi-group style and
  `templates/ado-terraform-pipeline.yaml` for the simpler flat style (stage
  templates with fewer parameters skip the grouping).
- Boolean toggles default to a sensible value (usually `true` for
  security-relevant scans, so consumers get secure-by-default behavior
  unless they opt out).
- String parameters with no real usable default (required-in-practice values
  like connection names, resource group names) still get `default: ''` so
  the template compiles without values supplied — don't make them
  `type: string` with no default, which forces every consumer to always set
  them even in doc examples.
- No project-specific values hardcoded anywhere — everything a consumer
  might need to vary is a parameter.
- If it scans anything, follow the shared Trivy pattern above and merge scan
  results into one JUnit test run.

## 2. `docs/<name>.md`

Mirror `docs/ado-terraform-pipeline.md` structure exactly:

```
<h2 align="center">{emoji} {Title} Pipeline Template</h2>

<p align="center">
<a href="../templates/<name>.yaml"><strong>📄 Template reference</strong></a>
</p>

{1-2 paragraph description: what it does, what it's built around/wraps,
 and how it relates to the OTHER existing template(s) — cross-reference
 shared patterns (e.g. "same Trivy-based pattern as ado-build-devsecops-
 pipeline.yaml") and call out step vs stage explicitly if that differs
 from the other template(s).}

---

### Table of Contents

**Getting Started**
- [📋 Requirements](#-requirements)
- [🚀 Usage](#-usage)

**Reference**
- [🔧 Parameters](#-parameters)
- [📤 Outputs](#-outputs)

---

## Getting Started

### 📋 Requirements

| Requirement | Notes |
|---|---|
| ... one row per external prerequisite (service connection, agent pool,
      storage, existing infra, network egress) ... |

### 🚀 Usage

{One fenced yaml example per realistic scenario, each with a short bold
 lead-in sentence, e.g.:}

**✅ <common case>**
**🚀 <variant, e.g. latest tool versions>**
**⚠️ <caveat/limitation worth its own example, if one exists>**
**🔒 <stricter security gate variant, if it scans anything>**
**🚫 <skip-scans variant, if it scans anything>** — mark not recommended.

---

## Reference

### 🔧 Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| ... one row per parameter, in the same order as the YAML `parameters:`
      block ... |

A short note below the table on which parameters are "required in
practice" despite having an empty-string default, if applicable.

### 📤 Outputs

- Bullet list: what each stage/job publishes, mutates, or leaves behind;
  what does NOT get re-run between stages (e.g. scans not repeated in a
  later apply stage); where a human-in-the-loop gate (Environment approval)
  needs to be configured outside the YAML.
```

Use the same emoji-heading style throughout (📋 🚀 🔧 📤 plus concern-specific
emoji like 🔑 secrets, 🏗️ IaC, 🐳 docker, ✅/⚠️/🔒/🚫 for example callouts) —
consistency here is deliberate, not decoration.

## 3. `README.md`

- Add one row to the **📦 Available Templates** table under `## Usage`:
  `| \`templates/<name>.yaml\` | Step or Stage | <one-line purpose> | [<name>.md](./docs/<name>.md) |`
- If this template was listed under **🔮 Roadmap** as a planned addition,
  remove that mention now that it's real — the roadmap only lists what
  doesn't exist yet.
- Don't touch unrelated README sections (Design Principles, Security,
  Contributing, etc.) unless the new template genuinely changes them.

## 4. Land it like every other change here

- Open a PR — this repo's convention is no direct pushes to `master`
  (see README **🤝 Contributing** / **🔖 Template Versioning**).
- A new template is additive, not a breaking change, so it doesn't by
  itself require a major version bump — a maintainer tags a new release
  (`vX.Y.0`) once merged, per semver.
