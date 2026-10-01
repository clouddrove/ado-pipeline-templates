<h2 align="center">🏗️ Terraform (Smurf) Pipeline Template</h2>

<p align="center">
<a href="../templates/ado-terraform-pipeline.yaml"><strong>📄 Template reference</strong></a>
</p>

A three-stage Azure Pipelines **stage template** - security scans, then init/validate/plan, a manual approval gate, then apply - built around [`clouddrove/smurf`](https://github.com/clouddrove/smurf), CloudDrove's Terraform CLI wrapper, with OIDC/workload-identity federation to Azure (no stored client secret). Secret and IaC misconfiguration scanning use the same Trivy-based pattern as [`ado-build-devsecops-pipeline.yaml`](./ado-build-devsecops-pipeline.md).

Unlike [`ado-build-devsecops-pipeline.yaml`](./ado-build-devsecops-pipeline.md), which is a **step** template consumed inside a job's `steps:`, this is a **stage** template consumed at the top-level `stages:` list of a pipeline.

---

### Table of Contents

**Getting Started**
- [📋 Requirements](#-requirements)
- [🚀 Usage](#-usage)

**Reference**
- [🔧 Parameters](#-parameters)
- [🔐 Authentication (OIDC)](#-authentication-oidc)
- [📤 Outputs](#-outputs)

---

## Getting Started

### 📋 Requirements

| Requirement | Notes |
|---|---|
| An Azure DevOps **agent pool** (named, not the hosted `vmImage` shorthand) | Passed as `agentPool`; can be a self-hosted pool or a named pool backed by Microsoft-hosted agents |
| An **Azure Resource Manager service connection** using **workload identity federation (OIDC)** | Passed as `ServiceArm`. The `AzureCLI@2` steps run with `addSpnToEnvironment: true`, and the template turns the connection's federated token and IDs into the `ARM_*` variables Terraform needs (`ARM_USE_OIDC`, `ARM_OIDC_TOKEN`, client/tenant/subscription IDs). See [🔐 Authentication (OIDC)](#-authentication-oidc). A classic secret-based service connection doesn't provide an `idToken`, so it won't work |
| An existing **Azure Storage Account + container** for the Terraform remote state backend | `ResourceGroupName`, `StorageAccountName`, `ContainerName`, `Key` map directly to `smurf stf init --backend-config=...` |
| An **Azure DevOps Environment** | Passed as `environment`; add an approval check on it to gate the `terraform_apply` stage. Without one, `terraform_apply` runs immediately after `terraform_init_plan` succeeds |
| A Terraform working directory in the consumer's repo | Passed as `terraformWorkingDirectory`; the template `cd`s into it before running any `smurf stf` command |
| Internet egress on the agent | Installs Terraform (`TerraformInstaller@1`), Go (`GoTool@0`), `smurf` (`go install`), and Trivy (if `secretsScan` or `iacScan` is enabled) at runtime |

### 🚀 Usage

**✅ Single environment** - the common case:

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0
      endpoint: <github-service-connection-name>

stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra'
      terraformVersion: '1.9.5'
      goVersion: '1.22'
      smurfVersion: 'latest'
      agentPool: 'my-self-hosted-pool'
      ServiceArm: 'my-azure-arm-oidc-connection'
      ResourceGroupName: 'tfstate-rg'
      StorageAccountName: 'tfstateacct001'
      ContainerName: 'tfstate'
      Key: 'myproject/terraform.tfstate'
      environment: 'Prod-Terraform-Approval'
```

This produces three stages: `terraform_init_plan` validates required parameters, runs the secret + IaC scans (if enabled) *before* touching cloud state, then `smurf stf init` / `validate` / `plan --out=<plan-file>` and publishes the saved plan as a pipeline artifact; `terraform_approval` is a `deployment` job that waits on the named Environment's approval check; `terraform_apply` downloads that same plan artifact, runs `smurf stf init` again, then applies the approved saved plan with `smurf stf apply <plan-file>` (scans are not repeated here - they already gated `terraform_init_plan`).

**🚀 Latest tool versions** - let `TerraformInstaller@1` and `GoTool@0` resolve the newest release instead of pinning:

```yaml
stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra'
      terraformVersion: 'latest'
      goVersion: '1.x'
      smurfVersion: 'latest'
      agentPool: 'my-self-hosted-pool'
      ServiceArm: 'my-azure-arm-oidc-connection'
      ResourceGroupName: 'tfstate-rg'
      StorageAccountName: 'tfstateacct001'
      ContainerName: 'tfstate'
      Key: 'myproject/terraform.tfstate'
      environment: 'Dev-Terraform-Approval'
```

**⚠️ Multiple environments (dev/staging/prod)**: the three stages this template defines - `terraform_init_plan`, `terraform_approval`, `terraform_apply` - have **fixed names**, not parameterized per environment. Calling the template more than once in the *same* pipeline file fails at compile time with a duplicate-stage-name error. Until the template supports a stage-name prefix/suffix parameter, use **one pipeline YAML file per environment** instead, each with its own trigger/path filter and its own call to this template:

```yaml
# azure-pipelines-dev.yml
stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra/dev'
      ResourceGroupName: 'tfstate-rg'
      StorageAccountName: 'tfstateacct001'
      ContainerName: 'tfstate'
      Key: 'myproject/dev/terraform.tfstate'
      environment: 'Dev-Terraform-Approval'
      # ...terraformVersion, goVersion, smurfVersion, agentPool, ServiceArm as above
```

```yaml
# azure-pipelines-prod.yml
stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra/prod'
      ResourceGroupName: 'tfstate-rg'
      StorageAccountName: 'tfstateacct001'
      ContainerName: 'tfstate'
      Key: 'myproject/prod/terraform.tfstate'
      environment: 'Prod-Terraform-Approval'
      # ...terraformVersion, goVersion, smurfVersion, agentPool, ServiceArm as above
```

**🔒 Stricter security gate** - only CRITICAL fails the build, HIGH is reported but non-blocking:

```yaml
stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra'
      scanSeverity: 'CRITICAL'
      # ...remaining required parameters as above
```

**🚫 Skip security scans** - not recommended, but available if scanning is handled elsewhere in the pipeline:

```yaml
stages:
  - template: templates/ado-terraform-pipeline.yaml@templates
    parameters:
      terraformWorkingDirectory: '$(Build.SourcesDirectory)/infra'
      secretsScan: false
      iacScan: false
      # ...remaining required parameters as above
```

---

## Reference

### 🔧 Parameters

| Name | Type | Default | Description |
|---|---|---|---|
| `terraformWorkingDirectory` | string | `''` | Directory the template `cd`s into before running any `smurf stf` command |
| `terraformVersion` | string | `''` | Version passed to `TerraformInstaller@1` |
| `goVersion` | string | `''` | Version passed to `GoTool@0` (needed to `go install` smurf) |
| `smurfVersion` | string | `''` | Version/tag passed to `go install github.com/clouddrove/smurf@<version>` |
| `agentPool` | string | `''` | Named agent pool for `terraform_init_plan` and `terraform_apply` (used as `pool.name`, not `vmImage`) |
| `ServiceArm` | string | `''` | Azure Resource Manager service connection name (OIDC/workload identity) |
| `ResourceGroupName` | string | `''` | Resource group of the Terraform state storage account |
| `StorageAccountName` | string | `''` | Storage account holding the Terraform state container |
| `ContainerName` | string | `''` | Blob container within that storage account |
| `Key` | string | `''` | State file path/key within the container |
| `environment` | string | `''` | Azure DevOps Environment name the `terraform_approval` stage waits on |
| `secretsScan` | boolean | `true` | 🔑 Trivy secret scan over `terraformWorkingDirectory`, before any `smurf stf` command runs |
| `iacScan` | boolean | `true` | 🏗️ Trivy config/misconfig scan over `terraformWorkingDirectory` (native Terraform HCL support - AWS/Azure/GCP resource misconfiguration detection) |
| `scanSeverity` | string | `CRITICAL,HIGH` | Severity threshold that fails `terraform_init_plan` for either scan |
| `scanExitCode` | string | `1` | Exit code Trivy returns when `scanSeverity` findings exist |
| `scanContinueOnError` | boolean | `false` | Allows Trivy scan steps to continue the job after findings while still surfacing the failed scan step |
| `trivyVersion` | string | `''` | Optional Trivy version passed to the install script. Empty keeps the install script's default/latest behavior |
| `terraformPlanArtifactName` | string | `terraform-plan` | Pipeline artifact name used to pass the saved plan from `terraform_init_plan` to `terraform_apply` |
| `terraformPlanFileName` | string | `tfplan` | Saved plan filename generated by `smurf stf plan --out` and later applied by `smurf stf apply <plan-file>` |

Everything above `secretsScan` has no real default - every one of those parameters is required in practice; the empty-string defaults exist so the template compiles without values supplied, not because any of them are optional. The template fails early with a clear message if a required value is missing or if `terraformWorkingDirectory` does not exist. `secretsScan` / `iacScan` / `scanSeverity` / `scanExitCode` mirror [`ado-build-devsecops-pipeline.yaml`](./ado-build-devsecops-pipeline.md)'s scan parameters and genuinely default to something usable.

### 🔐 Authentication (OIDC)

Terraform (through `smurf stf`) authenticates to Azure with the service connection's
**federated token**, with no client secret stored anywhere. Both the `terraform_init_plan` and
`terraform_apply` steps run inside `AzureCLI@2` with `addSpnToEnvironment: true` and export:

| Variable | Value | Source |
|---|---|---|
| `ARM_USE_OIDC` | `true` | Tells the AzureRM provider and backend to use OIDC |
| `ARM_OIDC_TOKEN` | `$idToken` | The federated ID token from the service connection (`addSpnToEnvironment: true`) |
| `ARM_CLIENT_ID` | `$servicePrincipalId` | App registration / managed identity behind the service connection |
| `ARM_TENANT_ID` | `$tenantId` | Tenant of the service connection |
| `ARM_SUBSCRIPTION_ID` | `az account show --query id` | Subscription the service connection is scoped to |
| `ARM_OIDC_AZURE_SERVICE_CONNECTION_ID` | `$AZURESUBSCRIPTION_SERVICE_CONNECTION_ID` | ID of the service connection, set by `AzureCLI@2` |

> **Troubleshooting:** if `smurf stf init` fails with
> `Error building ARM Config: Authenticating using the Azure CLI is only supported as a User (not a Service Principal)`,
> the provider fell back to Azure CLI auth because it didn't get a federated token. Make sure
> the pipeline consumes a version of this template that includes `ARM_OIDC_TOKEN` (added in
> [#3](https://github.com/clouddrove/ado-pipeline-templates/pull/3)), so pin `ref` to `master`
> or a later release tag, not an older branch. Also check that `ServiceArm` is a **workload identity
> federation** connection.

### 📤 Outputs

- 🔑🏗️ `secretsScan` and `iacScan` publish JUnit results to `$(Common.TestResultsDirectory)`, consolidated into a single **Security Scans** test run in the `terraform_init_plan` job - same pattern as the devsecops template.
- 🏗️ `terraform_init_plan` publishes the saved plan file as the `terraformPlanArtifactName` pipeline artifact. Terraform saved plans can contain sensitive values, so keep Azure DevOps artifact access limited to the same users allowed to review/apply infrastructure changes.
- ✅ `terraform_approval` produces no infrastructure change itself - it's purely a gate. Configure the approval check on the `environment` Environment in Azure DevOps (Environments → Approvals and checks), not in this YAML.
- 🚀 `terraform_apply` is the only stage that mutates infrastructure, via `smurf stf apply <downloaded-plan-file>`. It does not re-run the security scans or generate a new plan.
