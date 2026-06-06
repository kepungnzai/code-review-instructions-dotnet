# Agent Skills & Knowledge Base

You possess expert-level proficiency in the following domains:

## 1. Terraform & HCL
- Advanced state management (remote backends, state locking, Terraform Cloud/Enterprise).
- Idempotent resource creation and safe `terraform import` workflows. We need to support this.
- Variable management (`.tfvars`, `variable.tf`, strict typing).
- Terraform CLI advanced flags (`-detailed-exitcode`, `-out=plan.tfplan`, `-var-file`).

## 2. GitHub Actions CI/CD
- Workflow syntax (`on`, `jobs`, `steps`, `needs`, `if` conditionals).
- **GitHub Environments:** Configuring `environment`, `required_reviewers`, and protection rules for manual approvals.
- **Exit Code Handling:** Using `actions/github-script` or bash to evaluate `terraform plan -detailed-exitcode` (0 = success/no-op, 2 = success/changes, >2 = error).
- Secure secret management (`secrets.TF_API_TOKEN`, `secrets.TF_VAR_*`).
- Workflow triggers (`workflow_dispatch`, `push`, `pull_request`, `repository_dispatch`).

## 3. Recommended GitHub Actions
You are authorized to use and configure the following proven actions:
- `actions/checkout@v4`
- `hashicorp/setup-terraform@v3` (with Terraform Cloud credential support)
- `actions/github-script@v7` (for parsing plan output and evaluating exit codes)
- `peter-evans/create-pull-request@v6` (optional, for posting plan output to PRs)

## 4. DevOps Best Practices
- Principle of Least Privilege (PoLP) for CI/CD service accounts.
- Separation of Concerns (Plan, Apply, and Destroy are separate pipelines).
- Drift detection and prevention.
- Immutable infrastructure patterns.