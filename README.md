# ADO Pipeline Templates

Reusable Azure Pipelines YAML step templates, shared across repos the same way [`github-shared-workflows`](https://github.com/clouddrove/github-shared-workflows) shares GitHub Actions workflows: one repo holds the templates, consumer pipelines reference them by path + version instead of copy-pasting YAML.

## How to use

Azure Pipelines' equivalent of GitHub's `uses:` is a `resources.repositories` entry plus a `@alias` suffix on the `template:` reference. This repo is currently **private**, so consumer pipelines need a GitHub service connection with read access to it.

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0          # pin to a tag; avoid tracking a branch in production
      endpoint: <github-service-connection-name>   # Project Settings -> Service connections -> GitHub

steps:
  - template: templates/steps/secrets-scan.yaml@templates
    parameters:
      scanPath: '$(Build.SourcesDirectory)'
```

## Versioning

Pin consumers to a **tag** (`refs/tags/vX.Y.Z`), not a branch. Bump the tag on breaking parameter changes. Don't push directly to `master`/`main` — land changes via PR and tag a release once merged.

## Templates

| Template | Purpose |
|---|---|
| `templates/steps/secrets-scan.yaml` | Trivy secret scan (fs), fails on any finding |
| `templates/steps/sast-scan.yaml` | Semgrep SAST scan, JUnit output |
| `templates/steps/dependency-scan.yaml` | Trivy dependency/SCA scan (fs), JUnit output |
| `templates/steps/docker-build-push.yaml` | Docker build or push via a registry service connection |
| `templates/steps/image-scan.yaml` | Trivy container image scan, JUnit output |
| `templates/steps/iac-scan.yaml` | Trivy config/misconfig scan (Dockerfile, IaC), JUnit output |
| `templates/steps/helm-validate.yaml` | Pull a public Helm chart, lint + render with env overrides, Trivy config scan the rendered manifest |
| `templates/steps/test-coverage.yaml` | Node.js unit tests + coverage (JUnit + Cobertura) |

Each template's own `parameters:` block documents its inputs and defaults - read the file for the authoritative list.

## Origin

Extracted from [`clouddrove-sandbox/az-template`](https://dev.azure.com/clouddrove-sandbox/az-template) (Azure DevOps), which still holds its own local copy until this repo has a remote and consumer pipelines are cut over to the `@templates` reference above.
