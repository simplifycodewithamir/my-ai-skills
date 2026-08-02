---
name: cicd-pipeline
description: Use this skill whenever setting up or reviewing CI/CD, GitHub Actions workflows, Docker builds, Artifactory/JFrog config, Argo CD/GitOps deploy repos, Kubernetes manifests, or Azure/AKS deployment for one of the user's app repos. Trigger for "set up CI/CD", "add a pipeline", "deploy this", "Dockerize this", "add a workflow", or any new repo scaffold, even before the app code itself exists.
---

# CI/CD & Deployment Conventions

## Do
- **Every app repo builds and deploys from day 1.** The first PR must already produce a working pipeline, even if the app is a stub.
- **CI (GitHub Actions, per app repo)** on every PR: restore → build → run unit + integration tests (Testcontainers) → static/security scan (CodeQL by default) → build Docker image.
- **On merge to `main`**: additionally push the image to Artifactory, tagged with the short git SHA plus `latest` for dev.
- **Artifactory**: default to self-hosted JFrog Artifactory OSS / Container Registry (free, runs as a Docker container), not JFrog SaaS. Two repos per app — `<app>-dev-docker`, `<app>-prod-docker` — plus a virtual repo in front. Prod images are **promoted** from a dev image that already passed CI — never rebuilt, never pushed straight to prod.
- **Deployment repo is separate from the app repo.** A single `k8s-deploy` GitOps repo owns all K8s manifests/Helm charts for every app: `apps/<app-name>/{base,overlays/dev,overlays/prod}` (Kustomize) or Helm values-per-env. Argo CD watches only this repo, never app repos directly.
- **Environments**: `dev` and `prod` only for now. Argo CD auto-syncs `dev` on every deploy-repo commit; `prod` sync is manual/requires approval. App-repo CI updates the deploy repo's `dev` overlay image tag via a PR (never a direct push to deploy-repo `main`). Promoting `dev` → `prod` is a human-reviewed PR bumping the `prod` overlay's tag.
- **Azure/AKS** is the deployment target unless stated otherwise.
- **Secrets** (DB passwords, connection strings, API keys) live in **Azure Key Vault only**, pulled into the cluster via the Azure Key Vault Provider for Secrets Store CSI Driver (or equivalent), referenced from the pod spec.

## GitHub Actions: .NET build & test job
Base job for the "restore → build → run tests" portion of the per-app-repo CI workflow (line above). Static/security scan and Docker image build are separate additive steps in the same workflow, not replacements for this.

```yaml
name: Build and Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore
        run: dotnet restore <path-to-solution>.slnx

      - name: Build
        run: dotnet build <path-to-solution>.slnx --configuration Release --no-restore

      - name: Test
        run: dotnet test <path-to-solution>.slnx --configuration Release --no-build --verbosity normal
```

- Pin `dotnet-version` to the exact SDK the repo targets (check `global.json` / `<TargetFramework>`), not a floating major version.
- `--no-restore` on build and `--no-build` on test avoid redundant work now that each step is explicit — keep that chain intact when adding scan/Docker steps after `Test`.
- Swap in the repo's actual `.sln`/`.slnx` path; don't hardcode a project name from another repo.

## Don't
- Don't defer CI/CD to "later" on a new repo — scaffold it in the first PR.
- Don't push directly to prod images/tags — always promote a dev image that already passed CI.
- Don't put Kubernetes manifests in the app repo — they belong only in `k8s-deploy`.
- Don't let Argo CD watch an app repo directly.
- Don't push directly to `k8s-deploy`'s `main` — always open a PR, and prod bumps are human-reviewed.
- Don't put secrets in appsettings, env files, the deploy repo, ConfigMaps, or logs — Key Vault only.
- Don't swap in a paid scanning tool (Polaris/Black Duck) instead of CodeQL unless explicitly asked.
- Don't silently pick a different cloud/target than Azure/AKS without flagging it.

## Output expectations
- New app repo PR #1 includes: Dockerfile, GitHub Actions workflow (build/test/scan/image), and either a stub `k8s-deploy` entry or explicit note that one is needed.
- Any deviation from this pipeline shape (different cloud, different registry, skipping a stage) gets flagged explicitly, not silently done.
