# ADO Pipeline Templates

Reusable Azure Pipelines YAML template, shared across repos the same way [`github-shared-workflows`](https://github.com/clouddrove/github-shared-workflows) shares GitHub Actions workflows: one repo holds the template, consumer pipelines reference it by path + version instead of copy-pasting YAML.

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
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      # flip any of these to false to skip that concern
      secretsScan: true
      sastScan: true
      dependencyScan: true
      testCoverage: true
      dockerBuildPush: true
      imageScan: true      # only takes effect when dockerBuildPush is also true
      iacScan: true
      helmValidate: true
      containerRegistryServiceConnection: '$(containerRegistryServiceConnection)'
      imageRepository: '$(imageRepository)'
```

## Versioning

Pin consumers to a **tag** (`refs/tags/vX.Y.Z`), not a branch. Bump the tag on breaking parameter changes. Don't push directly to `master`/`main` — land changes via PR and tag a release once merged.

## Template

`templates/ado-build-devsecops-pipeline.yaml` is one template covering the full DevSecOps flow, with each concern gated behind its own boolean parameter:

| Toggle | What it runs when `true` |
|---|---|
| `secretsScan` | Trivy secret scan (fs), fails on any finding |
| `sastScan` | Semgrep SAST scan |
| `dependencyScan` | Trivy dependency/SCA scan (fs) |
| `testCoverage` | Node.js unit tests + coverage (JUnit + Cobertura) |
| `dockerBuildPush` | Docker build, then push, via a registry service connection |
| `imageScan` | Trivy container image scan — only runs if `dockerBuildPush` is also `true` |
| `iacScan` | Trivy config/misconfig scan (Dockerfile) |
| `helmValidate` | For each env in `helmEnvironments` (default `['dev','staging','prod']`): pull a public Helm chart, lint + render with that env's overrides, Trivy config scan the rendered manifest |

All scan steps publish JUnit results to `$(Common.TestResultsDirectory)`, picked up by a single `PublishTestResults@2` at the end of the template — the consumer doesn't need to wire that up separately.

See the template's own `parameters:` block for every configurable input (paths, severities, chart repo/version, node version, etc.) and their defaults — that's the authoritative list.

## Origin

Extracted from [`clouddrove-sandbox/az-template`](https://dev.azure.com/clouddrove-sandbox/az-template) (Azure DevOps), which still holds its own local copy until this repo has a remote and its consumer pipeline is cut over to the `@templates` reference above. Started as 8 separate step templates, consolidated into this one file so callers control each concern with a single boolean instead of picking which templates to wire in.
