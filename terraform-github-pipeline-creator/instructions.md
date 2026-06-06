# Execution Instructions & Strict Rules

When generating Terraform and GitHub Actions code, you MUST strictly adhere to the following rules. Deviating from these rules is considered a failure.

## 1. Directory Structure Enforcement
- All GitHub Actions workflow templates MUST be placed in a `template/` directory (e.g., `template/plan.yml`, `template/apply.yml`).
- All Terraform code (`.tf` files, modules, variables) MUST be placed in an `infrastructure/` directory.
- Terraform resources is restricted to AzureRM (azure) and use other provider sch as azapi, azuread when required

## 2. Environment & Variable Management
- You must support four distinct environments: `dev`, `test`, `sit`, and `prod`.
- Variables MUST be passed exclusively via `.tfvars` files (`dev.tfvars`, `test.tfvars`, `sit.tfvars`, `prod.tfvars`).
- **STRICT RULE:** Do NOT use inline `-var` or `-var="key=value"` arguments in any Terraform CLI commands. Use ONLY `-var-file=environments/${{ env.ENVIRONMENT }}.tfvars`.

## 3. Workflow Separation (The 3-Workflow Rule)
You must generate three distinct GitHub Actions workflow templates:
1. `template/terraform-plan.yml`: Handles planning and change detection.
2. `template/terraform-apply.yml`: Handles execution. MUST include a GitHub Actions Environment with `required_reviewers` for manual approval.
3. `template/terraform-destroy-plan.yml`: A separate workflow specifically for planning destruction (to review what will be deleted before any action is taken).

## 4. Change Detection (No-Op Optimization)
- In the plan workflow, you MUST use `terraform plan -detailed-exitcode`.
- Implement logic (e.g., using `actions/github-script` or bash conditionals) to check the exit code:
  - Exit code `0`: No changes. Skip further steps / exit gracefully.
  - Exit code `2`: Changes detected. Proceed to output the plan and trigger the apply workflow (via `workflow_dispatch` or repository dispatch).
  - Exit code `>2`: Error. Fail the pipeline.

## 5. Terraform Import Validation (Crucial Safety Step)
- If the workflow involves `terraform import`, it MUST NOT blindly update the state.
- The sequence MUST be:
  1. `terraform import <address> <id>`
  2. `terraform plan -detailed-exitcode` (to validate that the imported resource perfectly matches the code and does not trigger unintended deletions or modifications).
  3. Only if the plan exit code is `0` (or `2` with expected, reviewed changes), proceed. Otherwise, fail and alert.

## 6. Manual Approval (Absolute Requirement)
- The `terraform-apply.yml` workflow MUST use GitHub Environments (e.g., `environment: prod`).
- The workflow must pause and require manual approval (`required_reviewers`) before the `terraform apply` step executes.

<!-- ## 7. Terraform Cloud Support
- The templates must be designed to support Terraform Cloud (TFC) or Terraform Enterprise.
- Include comments or configuration blocks showing how to use `hashicorp/setup-terraform` with `cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}`.
- Ensure the `backend "remote"` or `cloud` block is accounted for in the `infrastructure/` setup. -->
## Use modules whenever possible. Please provide more instructions which module to use.

## Please update README.md with instructions how we can run terraform init and validate locally. When running local, plase avoid using azure rm backend configurations. 

## Use terraform map whenever possible to prevent code duplication where we can use for_each to traverse through the variable for resource creations.

## Output Format
- Provide the complete file paths for every code block (e.g., ````yaml:template/terraform-plan.yml`).
- Include brief, professional comments explaining *why* a specific safety measure (like `-detailed-exitcode`) is used.