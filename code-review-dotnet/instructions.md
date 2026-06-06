# Project Instructions: Critical Code Review of .NET Todo Application Feature

## 1. Task Objective
Perform a comprehensive, hyper-critical automated code review on a newly submitted feature branch for based on details stated in 'code-review.md'. You must verify if the code accurately meets the user requirements, validate that comprehensive unit tests (find out if there's existing test already in-place) were added, ensure test coverage hasn't regressed, and identify potential systemic risks if this code is merged into `main`.

---

## 2. Step-by-Step Execution Plan for Agents

### Phase 1: Environment Isolation & Requirement Check
- Checkout the feature branch and generate a clean git diff against `origin/main`.
- Map out the files changed.

### Phase 2: Logic & Quality Review (The "Super Critical" Lens)
Analyze the C# code changes line-by-line and explicitly check for:
- **Implementation** by the developer ensuring it meets the feature required, if it is a fix ensure that the code fixes the bug.
- **Coding standdard ** 
- **Reproduce** the bug - if unable to re-produce the bug, highlight to the developer and ask for more information-
- **Fixes** the bug - ensure bug are fixed and backed by unit tests

### Phase 3: Testing & Coverage Delta Audit
- Locate the test project. Ensure new unit tests were created specifically for this task on hand.
- Execute `dotnet test` with code coverage tracking.
- **Strict Constraint:** Compare the list of modified lines against the coverage report. If the new feature code contains paths (like catch blocks or validation checks) that are not exercised by tests, document them explicitly.
- All test must pass - if not output this and appended the error to REVIEW_GATE_REPORT.md
---

## 4. Final Deliverable: The Code Review Gate Report
The agent squad must output a unified markdown report in the workspace root named `REVIEW_GATE_REPORT.md` containing:

1.  **Git Summary:** Branch name, list of files reviewed, and total line delta.
2.  **Requirement Compliance Score:** (PASS/FAIL) Dictating whether all 4 core business rules of the feature were met.
3.  **Critical Code Flaws:** A prioritized list of logical, performance, or security issues discovered in the C# code.
4.  **Test Metrics:** A summary of overall test execution results and the exact coverage percentage achieved on the *new* code.
5.  **Merge Recommendation:** A final verdict: `[APPROVED]`, `[APPROVED WITH WARNINGS]`, or `[REJECTED - CHANGES REQUIRED]`.
