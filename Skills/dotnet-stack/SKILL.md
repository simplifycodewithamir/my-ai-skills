---
name: dotnet-stack
description: Aggregate router for the user's full .NET/React engineering workflow. Consult this FIRST for any substantial backend (.NET), frontend (React), testing, or CI/CD/deployment task on his projects, then follow its pointers to the specific sub-skill(s) that apply — dotnet-production-code, dotnet-testing, cicd-pipeline, react-frontend, git-workflow. Trigger for "build a feature", "add an endpoint end to end", "set up a new repo", "review this PR", or any task that spans more than one of those concerns.
---

# .NET Stack — Aggregate Skill

This skill doesn't duplicate rules — it routes to the sub-skill that owns each concern. Load only the sub-skills relevant to the current task; don't load all five for a one-line fix.

## Routing

| Task involves… | Load |
|---|---|
| ASP.NET Core endpoints, services, EF Core, resilience, logging, feature flags | `dotnet-production-code` |
| Writing or reviewing unit or integration tests | `dotnet-testing` |
| GitHub Actions, Docker, Artifactory, Argo CD, K8s manifests, Azure/AKS | `cicd-pipeline` |
| React/TypeScript UI work | `react-frontend` |
| Commit messages, PR descriptions | `git-workflow` |

## Cross-cutting rules (apply regardless of which sub-skill loads)

- **Test-first for non-trivial changes**: per `dotnet-testing`, write the failing test before the implementation — don't hand back production code from `dotnet-production-code` without it, unless the change is trivial.
- **Day-1 CI/CD**: any brand-new app repo pulls in `cicd-pipeline` even if nobody asked for "CI" yet — see Don'ts in that skill.
- **Architect/engineer relationship**: implement what he specifies as stated. If there's a real problem with the approach, say so explicitly and let him decide — never silently substitute your own architecture.
- **Report only real done**: build it, run it (tests green, app runs), then report success — never report a half-finished task as complete.
- **Flag deviations inline**: any place your output departs from a sub-skill's conventions, say why, in the response — don't bury it in a comment only.

## End-to-end feature checklist
For "build feature X end to end" style requests, touch these in order:
1. `dotnet-production-code` — endpoint group + service + data access
2. `dotnet-testing` — failing test first, then implementation satisfies it; add integration test via Testcontainers/WebApplicationFactory
3. `react-frontend` — UI consuming the new endpoint, if applicable
4. `cicd-pipeline` — only if this is a new repo, or the pipeline needs a change (new secret, new scan target, etc.)
5. `git-workflow` — commit message(s) for the change
