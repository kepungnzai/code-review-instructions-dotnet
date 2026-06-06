# Agent Skills Portfolio: Modern .NET Critical Code Review

## Skill: Git Branch Checkout & Diff Isolation
To isolate what needs to be reviewed without noise, the agent must execute:
1. Fetch latest changes: `git fetch origin main`
2. Identify altered files: `git diff --name-only origin/main...HEAD`
3. Extract exact line changes for review: `git diff origin/main...HEAD -- [file_path]`

## Skill: Testing Differential & Coverage Validation
The agent must verify that new code does not lower test coverage and that all new public logic has matching unit tests:
1. Run the test suite with coverage enabled (using coverlet):
   `dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura`
2. Generate a detailed report using `reportgenerator` to isolate coverage on modified files:
   `reportgenerator -reports:**/coverage.cobertura.xml -targetdir:CoverageReport -reporttypes:TextSummary`
3. **The Delta Rule:** Parse the output. If the line coverage of the *newly introduced methods* is below 90%, flag a BLOCKING review item.

## Skill: Hyper-Critical .NET Logic & Performance Audits
When analyzing C# changes, the Auditor agent must explicitly scan for these common .NET anti-patterns:
- **Async/Await Traps:** Ensure no `.Result` or `.Wait()` calls are introduced (which cause thread-pool starvation). Ensure `ConfigureAwait(false)` is used if reviewing class library code.
- **LINQ EF Core Inefficiencies:** Look out for hidden `IEnumerable` casting causing client-side evaluation, or missing `.AsNoTracking()` on read-only queries.
- **Null Safety:** Verify that `Nullable Reference Types (NRT)` are respected. Flag any potential `NullReferenceException` where an object isn't checked or annotated with `?`.
- **Resource Leaks:** Ensure any class implementing `IDisposable` or `IAsyncDisposable` is instantiated within a `using` statement or block.
- **Performance** - Ensure code used are of high performance, if not suggest improvements and what's need to be added to what file and what line number. This can be used by the code reviewer.