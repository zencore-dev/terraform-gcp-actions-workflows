# GitHub Actions Terraform Workflows
Reusable Terraform workflow with support for Google Cloud Platform (`terraform.yml`)

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

___


# GitHub Actions Terraform Workflows with Workload Identity Federation (WIF)

Reusable Terraform workflow with support for Google Cloud Platform (`terraform_wif.yml`). Use cases:

- Use [Workload Identity Federation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions) (Not `GOOGLE_CREDENTIALS` secret)
- Manage large tfstate files. 
- Include checkov [yaml custom policies](https://www.checkov.io/3.Custom%20Policies/YAML%20Custom%20Policies.html).

## Usage

To call this workflow, you'll first need to create a *workload-identity-pool* and a *provider*:

```
gcloud iam workload-identity-pools create "github-identity-pool" \
  --project="$PROJECT_ID" \
  --location="global" \
  --display-name="Github identity pool"
```

```
gcloud iam workload-identity-pools providers create-oidc “github-identity-provider”  \
   --location="global"  \
   --project="$PROJECT_ID" \
   --workload-identity-pool="github-identity-pool"  \
   --display-name="Github identity pool provider"  \
   --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository_owner=assertion.repository_owner" \
   --issuer-uri="https://token.actions.githubusercontent.com"  
```

Then, allow the workload identity provider to impersonate the service account:

```
gcloud iam service-accounts add-iam-policy-binding "github_actions_sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --project="$PROJECT_ID" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/github-identity-pool/attribute.repository/my-org/my-repo"
```

Now Github Actions will be able to access the GCP resources.

To use it, place the following workflow in `.github/workflows`:

```yaml
name: Your workflow

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  terraform:
    permissions:
      contents: read
      id-token: write
      pull-requests: write
    uses: zencore-dev/terraform-gcp-actions-workflows2/.github/workflows/terraform.yml@main
    with:
      working-directory: modules/my_module
      workload-identity-provider: 'projects/123456789/locations/global/workloadIdentityPools/github-pool-id/providers/github-provider-id'
      service-account: terraform@$PROJECT_ID.iam.gserviceaccount.com
```

This action will run `fmt`, `validate`, and `plan` on PR's, and will `apply` on commits to main

Remember to update the `working-directory`, `workload-identity-provider` and `service-account` inputs with the ones you want to use.

If you're using **checkov yaml custom policies**, place the policies in `.github/policies`. **Checkov** step will run all checkov scans and also include your custom policies. 


The following is an example policy that validates if GCP projects have a required label named *environment* and that the label is not empty:

```yaml
metadata:
  id: "CKV2_LABEL_9"
  name: "Ensure GCP Project has required 'environment' label and is not empty"
  category: "GENERAL_SECURITY"
scope:
  provider: "gcp"
definition:
  and:
  - cond_type: "attribute"
    resource_types: 
    - "google_project"
    attribute: "labels.environment"
    operator: "exists"
  - cond_type: "attribute"
    attribute: "labels.environment" 
    resource_types: 
    - "google_project"
    operator: is_not_empty
```