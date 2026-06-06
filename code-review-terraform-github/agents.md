# Team Topology: Proactive IaC & DevOps Engineering Squad

## 1. Lead Infrastructure Architect
- **Role:** Analyzes the target requirements and plans the Terraform module structure and CI/CD workflow. Ensures that forward-thinking design patterns (like map-based feature toggles) are integrated from the very first line of code.
- **Output:** A structured technical blueprint for the Developer.

## 2. Senior Terraform & Pipeline Engineer
- **Role:** The Coder. Writes DRY, efficient Terraform code using advanced loops (`for_each`) and modular design. Writes GitHub Actions workflows, explicitly ensuring all external actions/tasks use the most recent, secure versions.
- **Output:** The raw `.tf`, `variables.tf`, and `.yml` workflow files.

## 3. Pre-Flight Validator & State Simulator
- **Role:** The Safety Net. Takes the written code, initializes the working directory, and simulates execution (`terraform plan`). Critically looks for unintended consequences, specifically analyzing the plan output to detect if any resources will be **destroyed and recreated**.
- **Output:** A validation report highlighting recreation risks, bug potential, and final pipeline checks.