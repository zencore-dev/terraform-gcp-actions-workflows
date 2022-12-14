# GitHub Actions Terraform Workflows
Reusable Terraform workflow with support for Google Cloud Platform

## Usage
To call this workflow, create a `GOOGLE_CREDENTIALS` secret in your repo, containing the JSON string for a service account with the appropriate permissions, and place the following workflow in `.github/workflows`:
```yaml
name: Your workflow

on:
  push:
    branches:
      - main
    pull_request:

jobs:
  terraform:
    uses: zencore-dev/terraform-gcp-actions-workflows/.github/workflows/terraform.yml@main
    secrets: ${{secrets.GOOGLE_CREDENTIALS}}
    with:
      working-directory: modules/my_module
```

This action will run `fmt`, `validate`, and `plan` on PR's, and will `apply` on commits to main
