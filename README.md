<h1 align="center">ADO Pipeline Templates</h1>

<p align="center">
A collection of reusable Azure Pipelines YAML templates for DevSecOps CI/CD - build, scan, test, and validate workflows shared across repositories instead of duplicated in each one.
</p>

---

This repository plays the same role for Azure Pipelines that [`clouddrove/github-shared-workflows`](https://github.com/clouddrove/github-shared-workflows) plays for GitHub Actions: a central place to define a pipeline template once and reference it, versioned, from any number of consumer repos.

### Key Features

- **Composable by design** - each template gates its concerns behind boolean parameters, so a consumer enables only what it needs
- **Security-first** - secret scanning, SAST, dependency/SCA scanning, container image scanning, and IaC misconfiguration scanning are first-class, not bolted on
- **No paid extensions required** - scanners (Trivy, Semgrep) are fetched at runtime; no marketplace tasks or licensed tooling
- **Versioned releases** - consumers pin to a tag, so template changes never silently break existing pipelines
- **Documented** - every template has a corresponding page in [`docs/`](./docs) covering requirements, parameters, and usage examples

## Requirements

- An Azure DevOps organization/project with **YAML pipelines**
- A **GitHub service connection** in that project with read access to this repository (it's private)
- Per-template requirements (service connections, variable groups, expected file layout) are listed on each template's page in [`docs/`](./docs)

## How to Use

Azure Pipelines' equivalent of GitHub Actions' `uses:` is a `resources.repositories` entry plus a `@alias` suffix on the `template:` reference:

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0                        # pin to a released tag
      endpoint: <github-service-connection-name>

steps:
  - template: templates/<template-file>.yaml@templates
    parameters:
      # see the template's docs/ page for the full parameter list
```

See each template's page in [`docs/`](./docs) for a complete usage example and parameter reference.

## Templates

| Template | Description | Docs |
|---|---|---|
| `templates/ado-build-devsecops-pipeline.yaml` | Secret/SAST/dependency/image/IaC scanning, unit tests + coverage, Docker build/push, and multi-environment Helm chart validation - each concern independently toggleable | [ado-build-devsecops-pipeline.md](./docs/ado-build-devsecops-pipeline.md) |

Additional templates will be added here as they're extracted and generalized from consumer pipelines.

## Versioning

Consumers should pin to a **tag** (`refs/tags/vX.Y.Z`), never a branch, so that changes here are opt-in. Changes land via pull request and are never pushed directly to `master`; a tag is cut once a change is merged and ready for consumption.
