## DevSecOps Pipeline Template
#### [Template reference](../templates/ado-build-devsecops-pipeline.yaml)

A single Azure Pipelines step template covering secret scanning, SAST, dependency/SCA scanning, unit tests with coverage, Docker build/scan/push, IaC misconfiguration scanning, and multi-environment Helm chart validation. Every concern is gated behind its own boolean parameter, so a consumer pipeline enables exactly what it needs without wiring in separate templates per check.

### Requirements

| Requirement | Needed for | Notes |
|---|---|---|
| Azure Pipelines agent with internet egress | All scans | Trivy, Semgrep, and the Helm chart repo are fetched at runtime; no pre-baked image required |
| Docker Registry service connection | `dockerBuildPush`, `imageScan` | Passed as `containerRegistryServiceConnection` |
| A `containerRegistryServiceConnection` / `registryLoginServer` / `imageRepository` variable group | `dockerBuildPush`, `imageScan` | `registryLoginServer` must be the full FQDN (e.g. `myregistry.azurecr.io`), not the bare registry name - the template validates this and fails fast with a clear error if it looks wrong |
| A `Dockerfile` at the path passed to `dockerfilePath` | `dockerBuildPush`, `imageScan`, `iacScan` | |
| A Node.js app directory with `npm run test:coverage` producing `test-results.xml` (JUnit) and `coverage/cobertura-coverage.xml` | `testCoverage` | See the sample app in [`clouddrove-sandbox/az-template`](https://dev.azure.com/clouddrove-sandbox/az-template) for a reference implementation |
| Helm override values (`values.yaml` + `values-<env>.yaml` per environment) | `helmValidate` | The template pulls the chart itself from `chartRepoUrl`; it doesn't ship one |

### Usage

Minimal - everything enabled with defaults:

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0
      endpoint: <github-service-connection-name>

variables:
  - group: acr-common   # containerRegistryServiceConnection, registryLoginServer, imageRepository

steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
```

Selective - only run a subset of checks, with a couple of paths overridden:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      secretsScan: true
      sastScan: true
      dependencyScan: true
      testCoverage: false        # no test suite in this repo yet
      dockerBuildPush: true
      imageScan: true            # only takes effect when dockerBuildPush is true
      iacScan: true
      helmValidate: false        # this app isn't deployed via Helm
      dockerfilePath: '$(Build.SourcesDirectory)/docker/Dockerfile'
      appDir: '$(Build.SourcesDirectory)/src'
```

### Parameters

#### Toggles

| Name | Type | Default | Description |
|---|---|---|---|
| `secretsScan` | boolean | `true` | Trivy secret scan over `scanPath` |
| `sastScan` | boolean | `true` | Semgrep SAST scan over `scanPath` |
| `dependencyScan` | boolean | `true` | Trivy dependency/SCA scan over `scanPath` |
| `testCoverage` | boolean | `true` | Node.js unit tests + coverage in `appDir` |
| `dockerBuildPush` | boolean | `true` | Docker build then push via `containerRegistryServiceConnection` |
| `imageScan` | boolean | `true` | Trivy image scan; only runs when `dockerBuildPush` is also `true` |
| `iacScan` | boolean | `true` | Trivy config/misconfig scan over `iacScanPath` |
| `helmValidate` | boolean | `true` | Pull, lint, render, and scan the Helm chart per entry in `helmEnvironments` |

#### Scan configuration

| Name | Type | Default | Description |
|---|---|---|---|
| `scanPath` | string | `$(Build.SourcesDirectory)` | Path scanned by `secretsScan`, `sastScan`, `dependencyScan` |
| `scanSeverity` | string | `CRITICAL,HIGH` | Severity threshold that fails the build for Trivy-based scans |
| `scanExitCode` | string | `1` | Exit code Trivy returns when `scanSeverity` findings exist |
| `sastRuleset` | string | `p/security-audit` | Semgrep ruleset |
| `sastSeverity` | string | `ERROR` | Semgrep severity threshold that fails the build |
| `iacScanPath` | string | `$(Build.SourcesDirectory)/Dockerfile` | Path scanned by `iacScan` |

#### Test coverage

| Name | Type | Default | Description |
|---|---|---|---|
| `appDir` | string | `$(Build.SourcesDirectory)/app` | Working directory for `npm ci` / `npm run test:coverage` |
| `nodeVersion` | string | `20.x` | Node.js version installed via `UseNode@1` |

#### Docker build/push

| Name | Type | Default | Description |
|---|---|---|---|
| `containerRegistryServiceConnection` | string | `$(containerRegistryServiceConnection)` | Docker Registry service connection name |
| `imageRepository` | string | `$(imageRepository)` | Repository name within the registry |
| `dockerfilePath` | string | `$(Build.SourcesDirectory)/Dockerfile` | Dockerfile to build |
| `buildContext` | string | `$(Build.SourcesDirectory)` | Docker build context |
| `imageTag` | string | `$(Build.SourceVersion)` | Tag applied to the built image |

#### Helm validation

| Name | Type | Default | Description |
|---|---|---|---|
| `helmEnvironments` | object | `['dev', 'staging', 'prod']` | One lint/render/scan pass runs per entry |
| `helmOverridesDir` | string | `$(Build.SourcesDirectory)/helm/overrides` | Directory containing `values.yaml` + `values-<env>.yaml` |
| `chartRepoUrl` | string | `https://charts.clouddrove.com/` | Helm repository added at scan time |
| `chartRepoAlias` | string | `clouddrove` | Local alias for the added repo |
| `chartName` | string | `helmchart` | Chart name within the repo |
| `chartVersion` | string | `1.4.0` | Chart version to pull |

### Outputs

- All scan steps publish JUnit results to `$(Common.TestResultsDirectory)`, consolidated into a single **Security Scans** test run via `PublishTestResults@2` at the end of the template.
- `testCoverage` additionally publishes a **Unit Tests** run and a Cobertura code coverage summary.
- Nothing in this template deploys or pushes a Helm chart - `helmValidate` only lints/renders/scans; wiring an actual `helm upgrade --install` is left to the consumer's release process.
