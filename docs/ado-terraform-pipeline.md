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
- [📤 Outputs](#-outputs)

---

## Getting Started

### 📋 Requirements

| Requirement | Notes |
|---|---|
| An Azure DevOps **agent pool** (named, not the hosted `vmImage` shorthand) | Passed as `agentPool`; can be a self-hosted pool or a named pool backed by Microsoft-hosted agents |
| An **Azure Resource Manager service connection** using **workload identity federation (OIDC)** | Passed as `ServiceArm`. The template sets `ARM_USE_OIDC=true` and reads `$servicePrincipalId` / `$tenantId` from the connection via `addSpnToEnvironment: true` - a classic secret-based service connection won't populate those variables the same way |
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

This produces three stages: `terraform_init_plan` runs the secret + IaC scans (if enabled) *before* touching cloud state, then `smurf stf init` / `validate` / `plan`; `terraform_approval` is a `deployment` job that waits on the named Environment's approval check; `terraform_apply` then runs `smurf stf init` again followed by `smurf stf apply --auto-approve` (scans are not repeated here - they already gated `terraform_init_plan`).

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

Everything above `secretsScan` has no real default - every one of those parameters is required in practice; the empty-string defaults exist so the template compiles without values supplied, not because any of them are optional. `secretsScan` / `iacScan` / `scanSeverity` / `scanExitCode` mirror [`ado-build-devsecops-pipeline.yaml`](./ado-build-devsecops-pipeline.md)'s scan parameters and genuinely default to something usable.

### 📤 Outputs

- 🔑🏗️ `secretsScan` and `iacScan` publish JUnit results to `$(Common.TestResultsDirectory)`, consolidated into a single **Security Scans** test run in the `terraform_init_plan` job - same pattern as the devsecops template.
- 🏗️ `terraform_init_plan` leaves a Terraform plan computed by `smurf stf plan` in the working directory on that stage's agent - it is **not** published as a pipeline artifact or passed to `terraform_apply`, which re-runs `smurf stf init` from a fresh checkout and re-plans implicitly as part of `apply`.
- ✅ `terraform_approval` produces no infrastructure change itself - it's purely a gate. Configure the approval check on the `environment` Environment in Azure DevOps (Environments → Approvals and checks), not in this YAML.
- 🚀 `terraform_apply` is the only stage that mutates infrastructure, via `smurf stf apply --auto-approve`. It does not re-run the security scans.
