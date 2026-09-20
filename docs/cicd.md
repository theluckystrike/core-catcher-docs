# CI/CD

## GitHub Actions Workflows

### Core

| Workflow | Trigger | Description |
|----------|---------|-------------|
| `build-test.yaml` | PR / push to main | Dagger lint + build + test |
| `build-scan-image.yaml` | PR / push to main | ko build; both jobs push to `ghcr.io` — PR job tags `pr-<num>-<sha>` + `pr-<num>` for preview envs, main job tags `:main`. No image scan runs yet, despite the workflow name — see [#86](https://github.com/stuttgart-things/homerun2-core-catcher/issues/86) |
| `release.yaml` | After image build / manual | Semantic release + stage image + push kustomize OCI |
| `pages.yaml` | After release / manual | Deploy MkDocs to GitHub Pages |
| `lint-repo.yaml` | PR / push to main | Repository linting |

### PR-preview env

These four together drive the per-PR ephemeral preview environment on `homerun2-dev` for PRs carrying the `preview` label.

| Workflow | Trigger | Description |
|----------|---------|-------------|
| `build-scan-image.yaml` (PR job) | PR opened/updated | ko image tagged `pr-<num>-<sha>` + `pr-<num>` consumed by the per-PR ArgoCD Application |
| `push-kustomize-pr.yaml` | PR opened/updated | Kustomize OCI tagged `pr-<num>-<sha>` (renders `kcl/main.k` against `tests/kcl-deploy-profile.yaml`) |
| `comment-preview-url.yaml` | PR opened/reopened | Sticky bot comment with the preview URL, namespace, and ArgoCD link |
| `cleanup-pr-artifacts.yaml` | PR closed | Deletes both ghcr.io packages so version histories don't fill with PR debris |

See [Preview Environments](preview-environments.md) for the full flow, AppSet anatomy, and troubleshooting.

## Dagger Functions

The `dagger/` module provides:

| Function | Description |
|----------|-------------|
| `Lint` | Go linting via golangci-lint |
| `Build` | Build Go binary |
| `BuildImage` | Build container image with ko |
| `ScanImage` | Trivy vulnerability scan |
| `BuildAndTestBinary` | Build + Redis integration test |
| `IntegrationTest` | Full e2e test with pitcher + Redis + catcher |

## Taskfile

Common tasks available via `task`:

```bash
task lint                  # Run golangci-lint
task build-test-binary     # Build + test with Redis
task integration-test      # Full e2e test with pitcher
task render-manifests      # Render KCL manifests
task build-scan-image-ko   # Build + scan with ko
task deploy-kcl            # Deploy to cluster
```

## Release Process

Releases are automated via semantic-release:

1. Push to `main` triggers build + image workflow
2. On success, release workflow runs semantic-release
3. If releasable commits exist, a new version tag is created
4. GoReleaser builds binaries for linux/darwin (amd64/arm64)
5. Container image is staged from `:main` to `:vX.Y.Z`
6. Kustomize base is pushed as OCI artifact to GHCR
7. GitHub Pages documentation is deployed
