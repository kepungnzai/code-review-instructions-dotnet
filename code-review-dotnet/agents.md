
# Team Topology: Critical .NET Code Review & QA Squad

## 1. Requirement & Git Orchestrator
- **Role:** Handles the workspace environment. It fetches the target branch, calculates the exact diff against the `main` branch, reads the target user requirements file, and maps out exactly which logic paths changed.
- **Output:** A structured Git Diff & Requirements matrix outlining what needs to be scrutinized.

## 2. Hyper-Critical .NET Code Auditor
- **Role:** A ruthless, pessimistic reviewer. Inspects the modified C# files line-by-line. Looks for logical flaws, race conditions in async code, Linq performance traps, architectural mismatches, and security loopholes.
- **Output:** A brutal, constructive Code Review Report marking items as BLOCKING, WARNING, or OPTIMIZATION.

## 3. Automation & Coverage Verifier
- **Role:** Executes the code in a sandbox. Compiles the solution, runs the entire test suite, calculates code coverage differentials specifically for the *newly added lines*, and blocks compilation if tests fail or if code coverage dropped compared to `main`.