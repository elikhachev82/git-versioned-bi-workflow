# Project 2: Git-Versioned BI Development Workflow

## The problem with the current state
A `.pbix` file is a binary ZIP blob. You cannot diff it, you cannot review a
change in a pull request, and if someone breaks a measure, "what changed" is
a guess. With 100+ dashboards and 3 analysts contributing, that's a real risk.

## What was actually built
1. **PBIP project structure** — Power BI's file-based project format (`.pbip`),
   which serializes the report and semantic model as JSON/TMDL text files
   instead of one binary blob. See `sample-project/` for the exact folder
   layout you'd get from Power BI Desktop's "Save as .pbip".
2. **Branching strategy** (below) adapted from standard trunk-based
   development for a 3-analyst team.
3. **Pull request template** (`PULL_REQUEST_TEMPLATE.md`) with a checklist
   specific to BI changes (measure changes, RLS impact, refresh impact).
4. **CI pipeline** (`pipelines/azure-pipelines.yml`) that runs the Tabular
   Editor 2 Best Practice Analyzer (BPA) against every semantic model change
   in a PR, so DAX/naming issues are caught before merge, not after
   executive distribution.

## How PBIP changes the workflow
| Step | Before | After |
|---|---|---|
| Save work | Overwrite the shared .pbix on SharePoint | Commit to a feature branch |
| Review a DAX change | Open the file, hope you remember what changed | `git diff` shows the exact TMDL line that changed |
| Roll back a bad measure | Restore from OneDrive version history (if available) | `git revert` |
| Onboard analyst #4 | Walk them through the file by screen-share | They read the repo + README |

## Branching strategy
```
main                 ── always reflects what's in the Power BI Service "Production" workspace
 └─ dev               ── integration branch, deploys to a "UAT" workspace
     └─ feature/*      ── one branch per dashboard change (e.g. feature/add-rep-scorecard)
```
- Analysts branch off `dev`, open a PR back into `dev` for peer review
  (using the checklist in `PULL_REQUEST_TEMPLATE.md`).
- You (as lead) merge `dev` → `main` on a release cadence (e.g. weekly),
  which triggers the deployment pipeline to Production.
- Hotfixes branch off `main` directly and merge back to both `main` and `dev`.

## CI pipeline (`pipelines/azure-pipelines.yml`)
Runs on every PR into `dev` or `main`:
1. Checks out the repo.
2. Installs the Tabular Editor 2 command-line tool.
3. Runs the Best Practice Analyzer ruleset against every `.SemanticModel`
   folder that changed in the PR (naming conventions, unused columns,
   inefficient DAX patterns, missing descriptions).
4. Fails the build (blocking merge) if any BPA rule is violated at
   "error" severity.
5. On merge to `main`, a second stage uses the Power BI REST API
   (`deploy-to-service.yml` step) to publish the updated `.pbix` (compiled
   from the PBIP project) to the Production workspace via a service
   principal — no manual "Publish" button click.

