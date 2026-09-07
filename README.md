<h1 align="center">🚀 ADO Pipeline Templates</h1>

<p align="center">
Reusable Azure DevOps YAML pipeline templates, hosted on GitHub - built to reduce duplicate pipeline code and standardize CI/CD across projects.
</p>

---

### Table of Contents

**About**
- [📖 Overview](#-overview)
- [✨ Key Features](#-key-features)
- [🤔 Why Use This Repository?](#-why-use-this-repository)

**Usage**
- [📦 Available Templates](#-available-templates)
- [✅ Requirements](#-requirements)
- [🧭 How to Use Templates from GitHub](#-how-to-use-templates-from-github)
- [🔐 Authentication / Service Connection](#-authentication--service-connection)
- [🔧 Template Parameters](#-template-parameters)
- [🔖 Template Versioning](#-template-versioning)

**Standards**
- [🧱 Design Principles](#-design-principles)
- [🔒 Security](#-security)

**Project**
- [🤝 Contributing](#-contributing)
- [📄 License](#-license)
- [👥 Maintainers](#-maintainers)

**What's Next**
- [🔮 Roadmap](#-roadmap)
- [🌟 Goal](#-goal)

---

## About

### 📖 Overview

This repository is a central place to define an Azure Pipelines template once and reference it, versioned, from any number of consumer repos - instead of copy-pasting the same YAML into every pipeline. Templates are parameterized rather than hardcoded to a specific project, so the same template serves many consumers.

### ✨ Key Features

- 🔁 **Reusable Azure DevOps YAML templates**, hosted on GitHub and consumed cross-repo
- 🧩 **Parameterized** - no project-specific values hardcoded into a template
- 🔘 **Composable by design** - each template gates its concerns behind boolean parameters, so a consumer enables only what it needs
- 🛡️ **Security-first** - secret scanning, SAST, dependency/SCA scanning, container image scanning, and IaC misconfiguration scanning are first-class, not bolted on
- 💸 **No paid extensions required** - scanners (Trivy, Semgrep) are fetched at runtime; no marketplace tasks or licensed tooling
- 🔖 **Versioned releases** - consumers pin to a tag, so template changes never silently break existing pipelines
- 📚 **Documented** - every template has a corresponding page in [`docs/`](./docs) covering requirements, parameters, and usage examples

### 🤔 Why Use This Repository?

- 🔂 Avoid rewriting the same pipeline logic for every project
- 📏 Standardize CI/CD stages, naming, and deployment patterns across teams
- 🔒 Maintain security and quality controls in one place instead of N places
- ⚡ Reduce onboarding time for new projects and new team members
- 🛠️ Fix a bug or add a check once, and every consumer benefits after bumping their pinned version

---

## Usage

### 📦 Available Templates

| Template | Type | Purpose | Docs |
|---|---|---|---|
| `templates/ado-build-devsecops-pipeline.yaml` | Step | Secret/SAST/dependency/image/IaC scanning, unit tests + coverage, Docker build/push, and multi-environment Helm chart validation - each concern independently toggleable | [ado-build-devsecops-pipeline.md](./docs/ado-build-devsecops-pipeline.md) |
| `templates/ado-terraform-pipeline.yaml` | Stage | Secret + IaC scanning → init/validate/plan → manual approval → apply, using [`smurf`](https://github.com/clouddrove/smurf) and OIDC/workload-identity auth to Azure | [ado-terraform-pipeline.md](./docs/ado-terraform-pipeline.md) |

This table lists only what exists today. Planned templates are tracked in [🔮 Roadmap](#-roadmap), not listed here until they're real.

### ✅ Requirements

- An Azure DevOps organization/project using **YAML pipelines**
- A **GitHub service connection** in that project - Azure DevOps requires one for `type: github` repository resources even though this repository is public; there's no anonymous-access option
- Per-template requirements (additional service connections, variable groups, expected file layout) are listed on each template's page in [`docs/`](./docs)

### 🧭 How to Use Templates from GitHub

Azure Pipelines' equivalent of GitHub Actions' `uses:` is a `resources.repositories` entry plus a `@alias` suffix on the `template:` reference:

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0                        # pin to a released tag, not a branch
      endpoint: <github-service-connection-name>

steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      secretsScan: true
      dockerBuildPush: true
      # see docs/ado-build-devsecops-pipeline.md for the full parameter list
```

See each template's page in [`docs/`](./docs) for a complete, ready-to-copy usage example.

### 🔐 Authentication / Service Connection

Azure DevOps needs a **GitHub service connection** to pull templates from this repository. This is required regardless of repository visibility - a `type: github` resource without an `endpoint` fails validation with `Repository templates requires an endpoint`, even against a public repo:

```
Azure DevOps
   |
   | GitHub Service Connection
   v
GitHub Template Repository (this repo)
   |
   v
Reusable YAML Templates
```

Scope the service connection to read access on this repository only, following least privilege - it doesn't need write access or access to other repositories.

### 🔧 Template Parameters

Templates are configurable through parameters rather than hardcoded, project-specific values. For example, `ado-build-devsecops-pipeline.yaml` exposes its Docker build inputs like this:

```yaml
parameters:
  - name: dockerfilePath
    type: string
    default: '$(Build.SourcesDirectory)/Dockerfile'

  - name: buildContext
    type: string
    default: '$(Build.SourcesDirectory)'

  - name: imageRepository
    type: string
    default: '$(imageRepository)'
```

A consumer overrides only what differs from the default; everything else falls back to a sensible value. See each template's `parameters:` block, or its page in [`docs/`](./docs), for the authoritative list.

### 🔖 Template Versioning

Consumers should reference a **tag**, not a branch:

```yaml
ref: refs/tags/v1.2.0     # ✅ recommended
```

```yaml
ref: refs/heads/master    # ⚠️ avoid for production - a later change here could break every consumer
```

This repository follows [Semantic Versioning](https://semver.org/) (`v1.0.0`, `v1.1.0`, `v2.0.0`, ...). A breaking parameter or behavior change bumps the major version. Changes land via pull request - never a direct push to `master` - and a tag is cut once a change is merged and ready for consumption.

---

## Standards

### 🧱 Design Principles

Templates in this repository aim to be:

- ♻️ Reusable across projects and teams
- 🛡️ Secure by default
- 🧩 Parameterized, with minimal hardcoding
- 🔍 Easy to understand and integrate
- ⏪ Backward compatible where possible
- 📚 Well documented

### 🔒 Security

- 🚫 Secrets are never stored inside a template - use Azure DevOps secret variables, variable groups, or Azure Key Vault
- 🪪 Prefer workload identity / federated authentication over long-lived credentials where the target supports it
- 🔑 Service connections follow least-privilege scoping
- 🕵️ Security scanning (secrets, SAST, dependency/SCA, container image, IaC misconfiguration) is built into templates, not left as an afterthought for consumers to add
- 🙈 Avoid exposing credentials in pipeline logs

---

## Project

### 🤝 Contributing

Contributions are welcome:

- 🐛 Report issues or request a new template
- 🔧 Improve an existing template or its documentation
- 📬 Submit a pull request

Changes are never pushed directly to `master` - open a PR against it. Please keep new templates parameterized (no hardcoded project-specific values), documented under `docs/`, and consistent with the [🧱 Design Principles](#-design-principles) above.

### 📄 License

Licensed under the [Apache License 2.0](./LICENSE), matching [`github-shared-workflows`](https://github.com/clouddrove/github-shared-workflows) and CloudDrove's other open-source repositories.

### 👥 Maintainers

Maintained by CloudDrove.

---

## What's Next

### 🔮 Roadmap

Two templates exist today: one end-to-end DevSecOps flow (step template) and one Terraform init/plan/approve/apply flow (stage template, via `smurf`). Planned additions, not yet available:

**More templates**, likely organized by category as the collection grows:

```
containers/    docker-build, docker-push, container-scan
infrastructure/ terraform-destroy, terraform-fmt/lint
deployment/    aks, kubernetes, helm-deploy, app-service, azure-functions
quality/       sonarqube, additional SAST/lint tooling
utilities/     approvals, notifications, artifact management, versioning
```

**Example end-to-end pipeline patterns**, once more templates exist to compose them from:

```
Node.js  → Build → Test → Docker Build → Security Scan → ACR → AKS
.NET     → Build → Test → SonarQube → Docker → ACR → AKS
Python   → Test → SAST → Docker Build → Container Scan → Deploy
```

**Broader platform support**: Kubernetes/AKS deployment (this repo currently only validates Helm charts, it doesn't deploy them), Azure App Service and Functions, SonarQube, additional language coverage (.NET, Python), and eventually AWS/GCP deployment templates.

### 🌟 Goal

The long-term goal isn't just a collection of YAML files - it's opinionated, production-ready CI/CD patterns that teams can adopt with minimal configuration, with security and maintainability built in rather than bolted on:

```
Code → Build → Unit Test → Code Quality → SAST → Docker Build →
Container Security Scan → Push to Registry → Deploy → Health Check
```
