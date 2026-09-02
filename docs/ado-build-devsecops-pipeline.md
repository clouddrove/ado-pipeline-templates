<h2 align="center">🛡️ DevSecOps Pipeline Template</h2>

<p align="center">
<a href="../templates/ado-build-devsecops-pipeline.yaml"><strong>📄 Template reference</strong></a>
</p>

A single Azure Pipelines step template covering secret scanning, SAST, dependency/SCA scanning, unit tests with coverage, Docker build/scan/push, IaC misconfiguration scanning, and multi-environment Helm chart validation. Every concern is gated behind its own boolean parameter, so a consumer pipeline enables exactly what it needs without wiring in separate templates per check.

---

### Table of Contents

**Getting Started**
- [📋 Requirements](#-requirements)
- [`helmOverridesDir` vs `chartSource`](#helmoverridesdir-vs-chartsource)
- [🚀 Usage](#-usage)

**Reference**
- [🔧 Parameters](#-parameters)
  - [🔘 Toggles](#-toggles)
  - [🔎 Scan configuration](#-scan-configuration)
  - [🧪 Test coverage](#-test-coverage)
  - [🐳 Docker build/push](#-docker-buildpush)
  - [🚢 Helm validation](#-helm-validation)
- [📤 Outputs](#-outputs)

---

## Getting Started

### 📋 Requirements

| Requirement | Needed for | Notes |
|---|---|---|
| Azure Pipelines agent with internet egress | All scans | Trivy, Semgrep, and the Helm chart repo are fetched at runtime; no pre-baked image required |
| Docker Registry service connection | `dockerBuildPush`, `imageScan` | Passed as `containerRegistryServiceConnection` |
| `containerRegistryServiceConnection` / `imageRepository` / `registryLoginServer` passed explicitly as template parameters | `dockerBuildPush`, `imageScan` | **All three are plain parameters with no default that falls back to a same-named pipeline variable** - pass a literal value or your own `$(variable)` reference. `registryLoginServer` must be the full FQDN (e.g. `myregistry.azurecr.io`), not the bare registry name - the template validates this and fails fast with a clear error if it looks wrong |
| A `Dockerfile` at the path passed to `dockerfilePath` | `dockerBuildPush`, `imageScan`, `iacScan` | |
| A Node.js app directory with `npm run test:coverage` producing `test-results.xml` (JUnit) and `coverage/cobertura-coverage.xml` | `testCoverage` | See the sample app in [`clouddrove-sandbox/az-template`](https://dev.azure.com/clouddrove-sandbox/az-template) for a reference implementation |
| Override values files at `helmOverridesDir`: `values.yaml` + one `values-<env>.yaml` per entry in `helmEnvironments` | `helmValidate` | **Required no matter what `chartSource` is** - see [`helmOverridesDir` vs `chartSource`](#helmoverridesdir-vs-chartsource) below, this is the single most common setup mistake |

### `helmOverridesDir` vs `chartSource`

These are two independent settings, and mixing them up is the #1 cause of `helmValidate` failing with something like:

```
Error: open /path/to/your/repo/helm/k8s/values.yaml: no such file or directory
```

- **`chartSource`** (+ `chartRepoUrl`/`chartRepoAlias`/`chartName`/`chartVersion`, or `localChartPath`) says **where the chart's `Chart.yaml` and `templates/` come from** - either pulled from a Helm repository (`chartSource: repo`, the default) or a path already in your own repo (`chartSource: local`).
- **`helmOverridesDir`** says where the **override *values* files** live - `values.yaml` plus one `values-<env>.yaml` per entry in `helmEnvironments`. These are layered on top of the chart with `-f` **regardless of where the chart itself came from**. There is no `chartSource` setting that makes this optional.

In other words: even a chart pulled from a public repo needs override values from `helmOverridesDir`, and even a local chart still needs a *separate* `helmOverridesDir` - it does not automatically use the chart's own `values.yaml`.

Everything above describes `helmValuesMode: 'convention'` (the default). If your override files don't follow the `values.yaml`/`values-<env>.yaml` naming - e.g. `k8s/dev.yaml`, `k8s/prod.yaml` - set `helmValuesMode: 'explicit'` and list them in `helmValuesFiles` instead; `helmEnvironments`/`helmOverridesDir` are ignored in that mode. See the [Custom Helm values filenames](#-usage) usage example.

If `helmEnvironments: ['dev', 'stage', 'prod']`, the filenames must match **exactly** - `values-stage.yaml`, not `values-staging.yaml`.

**Using the public chart** (`chartSource: repo`, the default) - only override values are needed in your repo:

```
your-repo/
└── helm/
    └── overrides/              # helmOverridesDir points here
        ├── values.yaml
        ├── values-dev.yaml
        ├── values-stage.yaml
        └── values-prod.yaml
```

**Using your own chart** (`chartSource: local`) - the chart and its overrides are two *separate* directories, even if both live under `helm/`:

```
your-repo/
└── helm/
    ├── my-chart/               # localChartPath points here
    │   ├── Chart.yaml
    │   └── templates/
    └── overrides/              # helmOverridesDir points here - NOT the same folder as my-chart/
        ├── values.yaml
        ├── values-dev.yaml
        ├── values-stage.yaml
        └── values-prod.yaml
```

```yaml
parameters:
  chartSource: 'local'
  localChartPath: '$(Build.SourcesDirectory)/helm/my-chart'
  helmOverridesDir: '$(Build.SourcesDirectory)/helm/overrides'
```

### 🚀 Usage

**✅ Minimal** - everything enabled with defaults, registry parameters passed explicitly (they have no fallback):

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0
      endpoint: <github-service-connection-name>

steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      containerRegistryServiceConnection: 'my-acr-connection'
      imageRepository: 'my-app'
      registryLoginServer: 'myregistry.azurecr.io'
```

You can reference your own pipeline variables instead of literals if you prefer - that's your choice as the caller, not something the template requires:

```yaml
variables:
  - name: containerRegistryServiceConnection
    value: 'my-acr-connection'
  - name: imageRepository
    value: 'my-app'
  - name: registryLoginServer
    value: 'myregistry.azurecr.io'

steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      containerRegistryServiceConnection: '$(containerRegistryServiceConnection)'
      imageRepository: '$(imageRepository)'
      registryLoginServer: '$(registryLoginServer)'
```

**🎯 Selective** - only run a subset of checks, with a couple of paths overridden:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      secretsScan: true
      sastScan: true
      dependencyScan: true
      testCoverage: false        # no test suite in this repo yet
      dockerBuildPush: true
      containerRegistryServiceConnection: 'my-acr-connection'
      imageRepository: 'my-app'
      registryLoginServer: 'myregistry.azurecr.io'
      imageScan: true            # only takes effect when dockerBuildPush is true
      iacScan: true
      helmValidate: false        # this app isn't deployed via Helm
      dockerfilePath: '$(Build.SourcesDirectory)/docker/Dockerfile'
      appDir: '$(Build.SourcesDirectory)/src'
```

**📁 Local chart** - the chart lives in this repo instead of a Helm chart repository. `helmOverridesDir` is still required and is a *different* directory from `localChartPath` - see [`helmOverridesDir` vs `chartSource`](#helmoverridesdir-vs-chartsource) if this trips you up:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      dockerBuildPush: false   # this example is Helm-focused; add registry parameters if you also build/push
      chartSource: 'local'
      localChartPath: '$(Build.SourcesDirectory)/helm/my-app'      # Chart.yaml + templates/ live here
      helmOverridesDir: '$(Build.SourcesDirectory)/helm/overrides' # values.yaml + values-<env>.yaml live here - not the same folder
```

**🔎 Scans only** - a PR-validation pipeline that shouldn't build, push, or touch Helm at all:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      secretsScan: true
      sastScan: true
      dependencyScan: true
      testCoverage: true
      dockerBuildPush: false     # imageScan is skipped automatically since it depends on this
      iacScan: false             # no Dockerfile to scan yet in a PR-only flow
      helmValidate: false
```

**📂 Monorepo** - app and Dockerfile live in a subdirectory, not the repo root:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      scanPath: '$(Build.SourcesDirectory)/services/api'
      appDir: '$(Build.SourcesDirectory)/services/api'
      containerRegistryServiceConnection: 'my-acr-connection'
      imageRepository: 'my-api'
      registryLoginServer: 'myregistry.azurecr.io'
      dockerfilePath: '$(Build.SourcesDirectory)/services/api/Dockerfile'
      buildContext: '$(Build.SourcesDirectory)/services/api'
      iacScanPath: '$(Build.SourcesDirectory)/services/api/Dockerfile'
      helmOverridesDir: '$(Build.SourcesDirectory)/services/api/helm/overrides'
```

**🌎 Custom environments and stricter gate** - only two Helm environments, and CRITICAL-only fails the build:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      dockerBuildPush: false               # this example is Helm-focused
      helmEnvironments: ['dev', 'prod']    # no staging for this app
      scanSeverity: 'CRITICAL'             # HIGH findings are reported but won't fail the build
      kubeVersion: '1.28.0'                # match the actual AKS version this app targets
```

**📝 Custom Helm values filenames** - your override files don't follow the `values.yaml` / `values-<env>.yaml` convention (e.g. `k8s/dev.yaml`, `k8s/prod.yaml`):

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      dockerBuildPush: false
      helmValuesMode: 'explicit'
      helmValuesFiles:
        - name: 'dev'
          path: '$(Build.SourcesDirectory)/k8s/dev.yaml'
        - name: 'prod'
          path: '$(Build.SourcesDirectory)/k8s/prod.yaml'
```

`helmEnvironments` and `helmOverridesDir` are ignored in this mode - `helmValuesFiles` is the complete list of what gets validated, one lint/render/scan pass per entry. There's no separate base file here; each entry's `path` is the only `-f` passed for that entry.

**🚫 No artifact publish** - validate the Helm overrides, but don't stage/publish anything for a Release pipeline. Independent of `helmValidate` - you can publish without validating, or validate without publishing:

```yaml
steps:
  - template: templates/ado-build-devsecops-pipeline.yaml@templates
    parameters:
      dockerBuildPush: false
      publishHelmArtifact: false
```

---

## Reference

### 🔧 Parameters

#### 🔘 Toggles

| Name | Type | Default | Description |
|---|---|---|---|
| `secretsScan` | boolean | `true` | 🔑 Trivy secret scan over `scanPath` |
| `sastScan` | boolean | `true` | 🕵️ Semgrep SAST scan over `scanPath` |
| `dependencyScan` | boolean | `true` | 📦 Trivy dependency/SCA scan over `scanPath` |
| `testCoverage` | boolean | `true` | 🧪 Node.js unit tests + coverage in `appDir` |
| `dockerBuildPush` | boolean | `true` | 🐳 Docker build then push via `containerRegistryServiceConnection` |
| `imageScan` | boolean | `true` | 🔍 Trivy image scan; only runs when `dockerBuildPush` is also `true` |
| `iacScan` | boolean | `true` | 🏗️ Trivy config/misconfig scan over `iacScanPath` |
| `helmValidate` | boolean | `true` | 🚢 Pull, lint, render, and scan the Helm chart per entry in `helmEnvironments` |

#### 🔎 Scan configuration

| Name | Type | Default | Description |
|---|---|---|---|
| `scanPath` | string | `$(Build.SourcesDirectory)` | Path scanned by `secretsScan`, `sastScan`, `dependencyScan` |
| `scanSeverity` | string | `CRITICAL,HIGH` | Severity threshold that fails the build for Trivy-based scans |
| `scanExitCode` | string | `1` | Exit code Trivy returns when `scanSeverity` findings exist |
| `sastRuleset` | string | `p/security-audit` | Semgrep ruleset |
| `sastSeverity` | string | `ERROR` | Semgrep severity threshold that fails the build |
| `iacScanPath` | string | `$(Build.SourcesDirectory)/Dockerfile` | Path scanned by `iacScan` |

#### 🧪 Test coverage

| Name | Type | Default | Description |
|---|---|---|---|
| `appDir` | string | `$(Build.SourcesDirectory)/app` | Working directory for `npm ci` / `npm run test:coverage` |
| `nodeVersion` | string | `20.x` | Node.js version installed via `UseNode@1` |

#### 🐳 Docker build/push

| Name | Type | Default | Description |
|---|---|---|---|
| `containerRegistryServiceConnection` | string | `''` | Docker Registry service connection name - **no fallback, pass explicitly** |
| `imageRepository` | string | `''` | Repository name within the registry - **no fallback, pass explicitly** |
| `registryLoginServer` | string | `''` | Full registry FQDN, e.g. `myregistry.azurecr.io` - **no fallback, pass explicitly**. Used by `imageScan` to build the exact image reference that was just pushed |
| `dockerfilePath` | string | `$(Build.SourcesDirectory)/Dockerfile` | Dockerfile to build |
| `buildContext` | string | `$(Build.SourcesDirectory)` | Docker build context |
| `imageTag` | string | `$(Build.SourceVersion)` | Tag applied to the built image, and the same tag `imageScan` scans - no mismatch even if you override this |

`containerRegistryServiceConnection`, `imageRepository`, and `registryLoginServer` are validated in a dedicated step whenever `dockerBuildPush` is `true`, regardless of how you supply them (literal value or your own `$(variable)` reference).

#### 🚢 Helm validation

| Name | Type | Default | Description |
|---|---|---|---|
| `helmValuesMode` | string | `convention` | `convention` (default): use `helmEnvironments` + `helmOverridesDir`. `explicit`: use `helmValuesFiles` instead - for override files that don't follow the `values.yaml`/`values-<env>.yaml` naming or location |
| `helmEnvironments` | object | `['dev', 'staging', 'prod']` | Used when `helmValuesMode` is `convention`. One lint/render/scan pass runs per entry |
| `helmOverridesDir` | string | `$(Build.SourcesDirectory)/helm/overrides` | Used when `helmValuesMode` is `convention`. Directory containing `values.yaml` (optional) + `values-<env>.yaml` (required) per entry in `helmEnvironments` |
| `helmValuesFiles` | object | `[]` | Used when `helmValuesMode` is `explicit`: a list of `{ name, path }` entries, one lint/render/scan pass per entry, using only that entry's `path` (no base file) |
| `chartSource` | string | `repo` | `repo` (default): pull `chartName`@`chartVersion` from `chartRepoUrl`. `local`: skip repo add/pull entirely and lint/render `localChartPath` as-is |
| `chartRepoUrl` | string | `https://charts.clouddrove.com/` | Helm repository added at scan time; ignored when `chartSource` is `local` |
| `chartRepoAlias` | string | `clouddrove` | Local alias for the added repo; ignored when `chartSource` is `local` |
| `chartName` | string | `helmchart` | Chart name within the repo; ignored when `chartSource` is `local` |
| `chartVersion` | string | `1.4.0` | Chart version to pull; ignored when `chartSource` is `local` |
| `localChartPath` | string | `$(Build.SourcesDirectory)/helm/chart` | Path to a chart already committed in the consumer's repo; only used when `chartSource` is `local` |
| `helmLintStrict` | boolean | `true` | Adds `--strict` to `helm lint`, failing on warnings too, not just errors |
| `kubeVersion` | string | `1.29.0` | Passed to `helm template --kube-version`, to catch API-version-specific issues |
| `publishHelmArtifact` | boolean | `true` | Publish `helmArtifactSourceDir` as a pipeline artifact. **Independent of `helmValidate`** - each can be `true`/`false` regardless of the other |
| `helmArtifactSourceDir` | string | `$(Build.SourcesDirectory)/helm` | Folder staged into the artifact |
| `helmArtifactTargetSubfolder` | string | `helmchart` | Subfolder name within the staged artifact |
| `helmArtifactName` | string | `drop` | Name of the published pipeline artifact |

### 📤 Outputs

- 🧾 All scan steps publish JUnit results to `$(Common.TestResultsDirectory)`, consolidated into a single **Security Scans** test run via `PublishTestResults@2` at the end of the template.
- 📊 `testCoverage` additionally publishes a **Unit Tests** run and a Cobertura code coverage summary.
- 📦 `publishHelmArtifact` publishes `helmArtifactSourceDir` (the `helm/` folder, overrides included) as a pipeline artifact named `helmArtifactName` - a downstream Release pipeline can download it and run `helm upgrade --install` against those exact override values. Runs independently of `helmValidate` - you can publish without validating, or validate without publishing.
- 🚫 Nothing in this template deploys or pushes a Helm chart - `helmValidate` only lints/renders/scans; wiring an actual `helm upgrade --install` is left to the consumer's release process.
