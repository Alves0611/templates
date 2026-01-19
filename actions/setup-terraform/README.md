# Setup Terraform Action

Composite action that sets up Terraform with caching and optional tools.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `tf_version` | Terraform version to install | No | `1.7.5` |
| `enable_tflint` | Install TFLint | No | `false` |
| `enable_tfsec` | Install TFSec | No | `false` |

## Features

- Installs specified Terraform version using `hashicorp/setup-terraform@v3`
- Caches Terraform plugins based on `.terraform.lock.hcl` hash
- Optionally installs TFLint and TFSec
- Sets `terraform_wrapper: false` for compatibility

## Usage

```yaml
- name: Setup Terraform
  uses: ./actions/setup-terraform
  with:
    tf_version: '1.7.5'
    enable_tflint: true
    enable_tfsec: false
```
