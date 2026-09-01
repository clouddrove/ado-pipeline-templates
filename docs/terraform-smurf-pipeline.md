<h2 align="center">🏗️ Terraform (Smurf) Pipeline Template</h2>

<p align="center">
<a href="../templates/terraform-smurf-pipeline.yaml"><strong>📄 Template reference</strong></a>
</p>

A three-stage Azure Pipelines **stage template** - init/validate/plan, a manual approval gate, then apply - built around [`clouddrove/smurf`](https://github.com/clouddrove/smurf), CloudDrove's Terraform CLI wrapper, with OIDC/workload-identity federation to Azure (no stored client secret).

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
| Internet egress on the agent | Installs Terraform (`TerraformInstaller@1`), Go (`GoTool@0`), and `smurf` (`go install`) at runtime |

### 🚀 Usage

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: clouddrove/ado-pipeline-templates
      ref: refs/tags/v1.0.0
      endpoint: <github-service-connection-name>

stages:
  - template: templates/terraform-smurf-pipeline.yaml@templates
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

This produces three stages: `terraform_init_plan` runs `smurf stf init` / `validate` / `plan`; `terraform_approval` is a `deployment` job that waits on the named Environment's approval check; `terraform_apply` then runs `smurf stf init` again followed by `smurf stf apply --auto-approve`.

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

None of these have real defaults - every parameter is required in practice; the empty-string defaults exist so the template compiles without values supplied, not because any of them are optional.

### 📤 Outputs

- 🏗️ `terraform_init_plan` leaves a Terraform plan computed by `smurf stf plan` in the working directory on that stage's agent - it is **not** published as a pipeline artifact or passed to `terraform_apply`, which re-runs `smurf stf init` from a fresh checkout and re-plans implicitly as part of `apply`.
- ✅ `terraform_approval` produces no infrastructure change itself - it's purely a gate. Configure the approval check on the `environment` Environment in Azure DevOps (Environments → Approvals and checks), not in this YAML.
- 🚀 `terraform_apply` is the only stage that mutates infrastructure, via `smurf stf apply --auto-approve`.
