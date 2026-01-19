# Terraform Pipeline

Centralized Terraform pipeline repository for GitHub Actions. This repository provides reusable workflows and composite actions to automate validation, planning, and application of infrastructure as code with Terraform on AWS.

## 🎯 Objective

Project repositories **MUST NOT copy Terraform steps**. Instead, they should only call the central workflow:

```yaml
uses: ORG/terraform-pipeline/.github/workflows/core-terraform.yml@v1
```

The `core-terraform.yml` automatically orchestrates the correct flow based on the event (PR, push to main, or workflow_dispatch).

## 📋 Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── core-terraform.yml        # Main orchestrator (called by consumers)
│       ├── terraform-plan.yml        # Executes scans, validations and plan
│       └── terraform-apply.yml       # Executes apply
├── actions/
│   ├── setup-terraform/
│   │   ├── action.yml                # Composite action: Setup Terraform
│   │   └── README.md
│   └── render-summary/
│       ├── action.yml                # Composite action: Step Summary
│       └── README.md
└── README.md
```

## 🚀 How to Use

### Step-by-Step Setup

#### 1. Create the Workflow File

In your project repository, create `.github/workflows/iac.yml` (or any name you prefer):

```yaml
name: Infrastructure as Code

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  terraform:
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    with:
      aws_region: us-east-1
      working_directory: terraform
      tf_version: "1.7.5"
      enable_infracost: true
      enable_trivy: true
      enable_checkov: true
      enable_tflint: true
      enable_tfsec: false
      plan_on_push_main: true
      apply_on_push_main: true
      comment_plan_on_pr: true
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}
      INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

**Important:** Replace `Alves0611/templates` with your organization/repository name where the pipeline is hosted. Use `@main` for latest changes or `@v1` for a specific version tag.

#### 2. Configure GitHub Secrets

Go to your repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

- **`AWS_ASSUME_ROLE`**: The ARN of your IAM role (e.g., `arn:aws:iam::123456789012:role/github-actions-terraform`)
- **`INFRACOST_API_KEY`** (optional): Your Infracost API key for cost estimation

#### 3. Configure AWS IAM Role

Ensure your IAM role's trust policy allows your repository to assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

#### 4. Configure Approval Environment (Optional but Recommended)

For apply operations, set up an approval gate:

1. Go to **Settings** → **Environments**
2. Click **New environment** → Name it `apply`
3. Add **Required reviewers** (users or teams)
4. Optionally restrict to specific branches

#### 5. Test the Pipeline

- **Pull Request**: Create a PR to trigger plan and security scans
- **Push to main**: Will trigger plan and apply (with approval if configured)
- **Manual**: Use workflow_dispatch in the Actions tab

### Basic Example

Here's a minimal working example:

```yaml
name: Infrastructure as Code

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  terraform:
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    with:
      aws_region: us-east-1
      working_directory: terraform
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}
```

### Multiple Terraform Directories

If your repository has multiple Terraform directories (e.g., `terraform/00-vpc`, `terraform/01-eks-cluster`), you have two options:

#### Option 1: Multiple Jobs (Recommended)

Create separate jobs for each directory:

```yaml
name: Infrastructure as Code

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  terraform-vpc:
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    with:
      aws_region: us-east-1
      working_directory: terraform/00-vpc
      plan_on_push_main: true
      apply_on_push_main: true
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}

  terraform-eks:
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    with:
      aws_region: us-east-1
      working_directory: terraform/01-eks-cluster
      plan_on_push_main: true
      apply_on_push_main: true
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}
```

**Note:** Jobs run in parallel by default. If you need sequential execution (e.g., EKS depends on VPC), use `needs`:

```yaml
  terraform-eks:
    needs: terraform-vpc
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    # ... rest of config
```

#### Option 2: Matrix Strategy

Use a matrix to run the same workflow for multiple directories:

```yaml
name: Infrastructure as Code

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  terraform:
    strategy:
      matrix:
        directory:
          - terraform/00-vpc
          - terraform/01-eks-cluster
          - terraform/02-rds
    uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
    with:
      aws_region: us-east-1
      working_directory: ${{ matrix.directory }}
      plan_on_push_main: true
      apply_on_push_main: true
    secrets:
      AWS_ASSUME_ROLE: ${{ secrets.AWS_ASSUME_ROLE }}
```

**Note:** With matrix strategy, all directories run in parallel. Dependencies between directories are not automatically handled.

### Required Inputs

| Input | Type | Description |
|-------|------|-------------|
| `aws_region` | string | AWS region to use |

### Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ASSUME_ROLE` | ARN of the OIDC role for all operations (plan and apply) |

### Optional Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `tf_version` | string | `1.7.5` | Terraform version |
| `working_directory` | string | `.` | Terraform working directory |
| `backend_config_file` | string | `""` | Optional backend config file path |
| `enable_infracost` | boolean | `true` | Enable Infracost cost estimation |
| `enable_trivy` | boolean | `true` | Enable Trivy security scanning |
| `enable_checkov` | boolean | `true` | Enable Checkov security scanning |
| `enable_tflint` | boolean | `true` | Enable TFLint |
| `enable_tfsec` | boolean | `false` | Enable TFSec security scanning |
| `plan_on_push_main` | boolean | `true` | Run plan on push to main branch |
| `apply_on_push_main` | boolean | `true` | Run apply on push to main branch |
| `comment_plan_on_pr` | boolean | `true` | Comment plan summary on PR |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AWS_ASSUME_ROLE` | Yes | ARN of the OIDC role for all operations (plan and apply) |
| `INFRACOST_API_KEY` | No | Infracost API key for cost estimation |

## 🔐 AWS Authentication via OIDC

This pipeline uses **OIDC (OpenID Connect)** for AWS authentication, eliminating the need to store access keys as secrets.

### IAM Role Configuration

1. Create an IAM role with the necessary permissions for Terraform.

2. Configure the role's trust policy to allow GitHub Actions to assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/terraform-pipeline:*"
        }
      }
    }
  ]
}
```

**Note:** Adjust the `sub` to match your repository or organization. Examples:
- `repo:ORG/*:*` - All repositories in the organization
- `repo:ORG/my-repo:*` - Specific repository
- `repo:ORG/my-repo:ref:refs/heads/main` - Main branch only

3. For consumer projects, use:
```json
"token.actions.githubusercontent.com:sub": "repo:ORG/my-project:*"
```

### Required Variables

Configure the following secret in your GitHub repository (Settings → Secrets and variables → Actions):

- `AWS_ASSUME_ROLE`: ARN of the OIDC role for all operations (e.g., `arn:aws:iam::123456789012:role/github-actions-terraform`)

**Note:** The ARN is stored as a secret to avoid exposing the account ID in workflows. The same role is used for both plan and apply operations.

## 🛡️ Environment Apply (Approval Gate)

The pipeline implements a **mandatory gate** for apply operations through a GitHub Environment called `apply`.

### How to Configure

1. In the repository, go to **Settings** → **Environments**
2. Create a new environment named `apply`
3. Configure **Required reviewers** (add users or teams that must approve)
4. Optionally, configure **Deployment branches** to restrict to specific branches

### Behavior

- **Pull Requests**: Only executes plan and scans (no apply)
- **Push to main**: If `apply_on_push_main=true`, requires approval in the `apply` environment before executing apply
- **workflow_dispatch**: If `mode=apply` and `run_apply=true`, also requires approval in the `apply` environment

The gate ensures that no apply is executed without explicit manual approval.

## 📦 Versioning

The pipeline is versioned using **Git tags**. Use semantic tags:

- `v1` - Stable version
- `v1.0.1` - Patch release
- `v1.1.0` - Minor release

**Usage example:**
```yaml
uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
# Or use a version tag:
uses: Alves0611/templates/.github/workflows/core-terraform.yml@v1
```

We recommend pinning the major version (`@v1`) to receive patches automatically, or pinning a specific version (`@v1.0.1`) for maximum stability.

## 🔍 Tools and Scans

### Terraform

- **fmt**: Checks formatting (`terraform fmt -check -recursive`)
- **validate**: Validates configuration (`terraform validate -no-color`)
- **init**: Initializes backend (supports `backend_config_file`)
- **plan**: Generates plan (`terraform plan -out=tfplan`)
- **apply**: Applies changes using plan artifact

### TFLint

Linter for Terraform. Enabled by default (`enable_tflint: true`).

### Security Scans

#### Trivy
- Terraform configuration scan
- Fails on CRITICAL or HIGH vulnerabilities
- Enabled by default

#### Checkov
- Static infrastructure analysis
- Framework: Terraform
- Enabled by default

#### TFSec
- Security scanner specific to Terraform
- Disabled by default (`enable_tfsec: false`)
- Fails on HIGH or CRITICAL vulnerabilities

### Infracost

Infrastructure cost estimation:
- Generates JSON report
- Automatically comments on PR (if enabled)
- Requires `INFRACOST_API_KEY` secret

## 📊 Step Summary

All workflows generate a beautiful **Step Summary** at the end of execution, including:
- Status of each step (fmt, validate, lint, scans, plan)
- Link to plan artifact
- Infracost information (if enabled)
- Overall pipeline status

The summary is visible in the "Summary" tab of the workflow run.

## 🔄 Execution Flow

### Pull Request

1. Input validation
2. **Terraform Plan** (`terraform-plan.yml`):
   - fmt check
   - validate
   - init
   - Security Scans (Trivy, Checkov, TFSec - if enabled)
   - tflint (if enabled)
   - plan
   - upload artifact
   - Infracost (if enabled and API key present)
   - Comment on PR (if enabled)
3. Step Summary

### Push to Main

1. Input validation
2. **Approval Gate** (environment `apply`)
3. **Terraform Plan** (`terraform-plan.yml`) - if `plan_on_push_main=true`
4. **Terraform Apply** (`terraform-apply.yml`) - if `apply_on_push_main=true`, depends on plan
5. Step Summary

### Workflow Dispatch

- **mode=pr**: Executes PR pipeline (no apply)
- **mode=apply**: 
  - Validates that it's on the `main` branch
  - Validates that `run_apply=true`
  - Passes through approval gate
  - Executes plan + apply

## 🔧 Backend Configuration

If you need to pass custom configurations to the Terraform backend, use the `backend_config_file` input:

```yaml
with:
  backend_config_file: terraform/backend.hcl
```

The file should contain backend variables, for example:
```hcl
bucket = "my-terraform-state"
key    = "prod/terraform.tfstate"
region = "us-east-1"
```

The pipeline will execute `terraform init -backend-config=backend_config_file`.

## ❓ FAQ

### Why does apply need approval?

Apply modifies production infrastructure. The approval gate ensures that critical changes are manually reviewed before being applied, reducing the risk of incidents.

### How to enable/disable scanners?

Use the `enable_*` inputs:

```yaml
with:
  enable_trivy: true
  enable_checkov: true
  enable_tfsec: false
  enable_tflint: true
```

### How to configure backend_config_file?

Create a configuration file (e.g., `backend.hcl`) and pass the path:

```yaml
with:
  backend_config_file: terraform/backend.hcl
```

### How to handle multiple Terraform directories?

If you have multiple Terraform directories (e.g., `terraform/00-vpc`, `terraform/01-eks-cluster`), you can:

1. **Use multiple jobs** (recommended for dependencies):
   ```yaml
   jobs:
     terraform-vpc:
       uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
       with:
         working_directory: terraform/00-vpc
     terraform-eks:
       needs: terraform-vpc  # Run after VPC
       uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
       with:
         working_directory: terraform/01-eks-cluster
   ```

2. **Use matrix strategy** (for parallel execution):
   ```yaml
   jobs:
     terraform:
       strategy:
         matrix:
           directory: [terraform/00-vpc, terraform/01-eks-cluster]
       uses: Alves0611/templates/.github/workflows/core-terraform.yml@main
       with:
         working_directory: ${{ matrix.directory }}
   ```

### Can I use separate roles for plan and apply?

Yes, but by default the pipeline uses the same role (`AWS_ASSUME_ROLE`) for all operations. If you need separate roles, you can:

1. Create two roles with different permissions:
   - Role for plan: Read permissions (Get, List, Describe)
   - Role for apply: Full permissions (Create, Update, Delete)

2. Modify the pipeline to use different roles (requires changes to pipeline code)

**Recommendation:** For most cases, a single role with appropriate permissions is sufficient and simpler to manage.

### What happens if plan fails?

Apply will not be executed. The workflow fails at the plan step.

### How to disable automatic apply on push?

```yaml
with:
  plan_on_push_main: true
  apply_on_push_main: false
```

This will only execute plan on push to main.

### Can I use this pipeline with other clouds?

Currently, the pipeline is optimized for AWS with OIDC. For other clouds, it would be necessary to adapt the authentication and workflows.

### How does Terraform cache work?

The pipeline uses cache based on the hash of `.terraform.lock.hcl`. This speeds up `terraform init` in subsequent runs.

### Workflow not found error

If you see `workflow was not found`, check:

1. **Repository visibility**: If the pipeline repository is private, ensure:
   - The consuming repository has access (same organization or explicit access)
   - Or make the pipeline repository public

2. **Branch/Tag exists**: Verify the branch or tag exists:
   ```bash
   # Check if branch exists
   git ls-remote --heads origin main
   
   # Check if tag exists
   git ls-remote --tags origin v1
   ```

3. **Workflow file exists**: Ensure `.github/workflows/core-terraform.yml` exists in the specified branch/tag

4. **Use a tag instead of branch**: For stability, create and use a tag:
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```
   Then use: `uses: Alves0611/templates/.github/workflows/core-terraform.yml@v1.0.0`

5. **Check repository name**: Verify the repository path is correct:
   - Format: `OWNER/REPO/.github/workflows/FILE.yml@REF`
   - Example: `Alves0611/templates/.github/workflows/core-terraform.yml@main`

## 📝 License

This repository is provided as-is. Adjust as needed for your requirements.

## 🤝 Contributing

For improvements or fixes, open an issue or pull request in the repository.

---

**Note:** Replace `Alves0611/templates` with your organization/repository name where the pipeline is hosted. Use `@main` for latest changes or `@v1` (after creating a tag) for version pinning.
