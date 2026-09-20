# homerun2-core-catcher

A Go CLI microservice that consumes messages from Redis Streams using consumer groups and logs them for processing.

[![Build & Test](https://github.com/stuttgart-things/homerun2-core-catcher/actions/workflows/build-test.yaml/badge.svg)](https://github.com/stuttgart-things/homerun2-core-catcher/actions/workflows/build-test.yaml)
[![Docs](https://img.shields.io/badge/docs-pages-blue)](https://stuttgart-things.github.io/homerun2-core-catcher/)

## How It Works

The catcher connects to a Redis Stream (default: `messages`) via a consumer group and processes incoming messages. Messages are enqueued by [homerun2-omni-pitcher](https://github.com/stuttgart-things/homerun2-omni-pitcher) (or any producer using the same stream format).

Each stream entry contains a `messageID` referencing a Redis JSON object with the full message payload. The catcher logs every received message with its metadata.

```
omni-pitcher → Redis Stream → core-catcher → slog output
```


## DEVELOPMENT

<details>
<summary><b>Project structure</b></summary>

```
main.go                    # Entrypoint, consumer setup, graceful shutdown
internal/
  banner/                  # Animated startup banner (Bubble Tea)
  config/                  # Env-based config loading, slog setup
  catcher/                 # Catcher interface (Redis consumer + Mock)
dagger/                    # CI functions (Lint, Build, Test, IntegrationTest); Scan runs locally via task build-scan-image-ko
kcl/                       # KCL deployment manifests (Kubernetes)
tests/                     # Test data (integration test messages, deploy profiles)
.ko.yaml                   # ko build configuration
Taskfile.yaml              # Task runner
```

</details>

<details>
<summary><b>Configuration reference</b></summary>

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_ADDR` | Redis server address | `localhost` |
| `REDIS_PORT` | Redis server port | `6379` |
| `REDIS_PASSWORD` | Redis password | (empty) |
| `REDIS_STREAM` | Legacy single stream env var, still honored | `messages` |
| `REDIS_STREAMS` | Comma separated list of streams, preferred over `REDIS_STREAM` | (empty) |
| `PORT` | Web mode HTTP listen port | `8080` |
| `CONSUMER_GROUP` | Consumer group name | `homerun2-core-catcher` |
| `CONSUMER_NAME` | Consumer name within the group | hostname |
| `REDIS_STARTUP_TIMEOUT` | How long startup retries Redis before exiting (Go duration) | `120s` |
| `LOG_FORMAT` | Log format: `json` or `text` | `json` |
| `LOG_LEVEL` | Log level: `debug`, `info`, `warn`, `error` | `info` |

</details>

<details>
<summary><b>Run locally (with Redis)</b></summary>

```bash
# Start Redis (via Dagger)
task run-redis-as-service

# Run the catcher
export REDIS_ADDR=localhost REDIS_PORT=6379 REDIS_STREAM=messages
go run .
```

</details>

<details>
<summary><b>Run web UI with file backend</b></summary>

```bash
# Run web dashboard with sample messages (default port 8080)
task run-web

# Custom file and port
task run-web FILE_PATH=tests/integration-test-messages.json PORT=9090
```

</details>

<details>
<summary><b>Run CLI TUI with file backend</b></summary>

```bash
# Run interactive TUI with sample messages
task run-cli

# Custom file
task run-cli FILE_PATH=tests/integration-test-messages.json
```

</details>

<details>
<summary><b>Unit tests</b></summary>

Unit tests run without Redis:

```bash
go test ./...
```

</details>

<details>
<summary><b>Integration tests (Dagger + Redis)</b></summary>

Basic integration test — builds the catcher, starts Redis, sends a test message via `redis-cli`:

```bash
task build-test-binary
```

</details>

<details>
<summary><b>End-to-end integration test (Dagger + Redis + Pitcher)</b></summary>

Full end-to-end test that spins up Redis, downloads the [omni-pitcher](https://github.com/stuttgart-things/homerun2-omni-pitcher) binary from GitHub releases, pitches test messages through it, and verifies the catcher consumes them all:

```bash
# Run with latest pitcher release
task integration-test

# Pin a specific pitcher version
task integration-test PITCHER_VERSION=v1.2.0

# Use custom test messages
task integration-test MESSAGES_FILE=path/to/messages.json
```

The test report is exported to `/tmp/integration-test-report-homerun2-core-catcher.txt`.

</details>

<details>
<summary><b>Lint</b></summary>

```bash
task lint
```

</details>

<details>
<summary><b>Build and scan container image</b></summary>

```bash
task build-scan-image-ko
```

</details>

## DEPLOYMENT

<details>
<summary><b>Deploy full stack via Flux (recommended)</b></summary>

The [homerun2 Flux app](https://github.com/stuttgart-things/flux/tree/main/apps/homerun2) deploys the complete stack (Redis Stack + omni-pitcher + core-catcher) into a shared namespace using Kustomize Components.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: homerun2-flux
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-apps
  path: ./apps/homerun2
  prune: true
  wait: true
  postBuild:
    substitute:
      HOMERUN2_NAMESPACE: homerun2-flux
      HOMERUN2_CORE_CATCHER_VERSION: v0.5.0
      HOMERUN2_CORE_CATCHER_KUSTOMIZE_VERSION: v0.5.0-web
      HOMERUN2_CORE_CATCHER_HOSTNAME: catcher
      HOMERUN2_OMNI_PITCHER_VERSION: v1.2.0
      HOMERUN2_OMNI_PITCHER_HOSTNAME: pitcher
      GATEWAY_NAME: my-gateway
      GATEWAY_NAMESPACE: default
      DOMAIN: my-cluster.example.com
      HOMERUN2_REDIS_VERSION: "17.1.4"
      HOMERUN2_REDIS_STORAGE_CLASS: nfs4-csi
    substituteFrom:
      - kind: Secret
        name: homerun2-flux-secrets
```

See the [Flux app README](https://github.com/stuttgart-things/flux/tree/main/apps/homerun2) for all variables and a complete cluster example.

</details>

<details>
<summary><b>Container image (ko / ghcr.io)</b></summary>

The container image is built with [ko](https://ko.build) on top of `cgr.dev/chainguard/static` and published to GitHub Container Registry.

```bash
# Pull the image
docker pull ghcr.io/stuttgart-things/homerun2-core-catcher:<tag>

# Run with Docker
docker run \
  -e REDIS_ADDR=redis -e REDIS_PORT=6379 \
  -e REDIS_STREAM=messages \
  ghcr.io/stuttgart-things/homerun2-core-catcher:<tag>
```

</details>

<details>
<summary><b>Deploy Redis (prerequisite)</b></summary>

```bash
helmfile apply -f \
  git::https://github.com/stuttgart-things/helm.git@database/redis-stack.yaml.gotmpl \
  --state-values-set storageClass=openebs-hostpath \
  --state-values-set password="<REPLACE>" \
  --state-values-set namespace=homerun2
```

</details>

<details>
<summary><b>Run locally against a remote Kubernetes Redis</b></summary>

Port-forward the Redis service from your cluster and run the catcher locally:

```bash
# Port-forward Redis (keep running in a separate terminal)
export KUBECONFIG=~/.kube/<your-kubeconfig>
kubectl port-forward -n redis-stack svc/redis-stack 6379:6379

# Get the Redis password from the cluster secret
kubectl get secret -n redis-stack redis-stack \
  -o jsonpath='{.data.redis-password}' | base64 -d

# Run the catcher (use a script to avoid shell escaping issues with special characters in the password)
cat > /tmp/run-catcher.sh << 'EOF'
#!/bin/bash
export REDIS_ADDR=localhost
export REDIS_PORT=6379
export REDIS_PASSWORD='<REPLACE>'
export REDIS_STREAM=messages
export LOG_FORMAT=text
go run .
EOF
bash /tmp/run-catcher.sh
```

> **Note:** Passwords containing `!` or other shell-special characters must be set inside a script with single-quoted heredoc (`<< 'EOF'`). Passing them directly via `export` or inline env vars can cause silent corruption from bash history expansion.

</details>

<details>
<summary><b>Render KCL manifests</b></summary>

The `kcl/` directory contains KCL modules that generate Kubernetes manifests (Namespace, ServiceAccount, ConfigMap, Secret, Deployment).

```bash
# Render manifests (interactive)
task render-manifests

# Render manifests (non-interactive, uses defaults)
task render-manifests-quick
```

</details>

<details>
<summary><b>Deploy to cluster via KCL</b></summary>

Deploy using the Dagger kubernetes-deployment blueprint, which pulls the kustomize OCI artifact and applies it:

```bash
# Push kustomize base as OCI artifact (requires GITHUB_USER + GITHUB_TOKEN)
task push-kustomize-base

# Deploy to cluster (uses ~/.kube/movie-scripts by default)
task deploy-kcl

# Deploy with custom parameters
task deploy-kcl KUBECONFIG=~/.kube/my-cluster NAMESPACE=my-namespace
```

</details>

<details>
<summary><b>Deploy profile</b></summary>

Edit `tests/kcl-deploy-profile.yaml` to customize the deployment:

```yaml
config.image: ghcr.io/stuttgart-things/homerun2-core-catcher:latest
config.namespace: homerun2
config.redisAddr: redis-stack.homerun2.svc.cluster.local
config.redisPort: "6379"
config.redisStream: messages
config.consumerGroup: homerun2-core-catcher
config.redisPassword: changeme
```

</details>

## Links

- [GitHub Pages](https://stuttgart-things.github.io/homerun2-core-catcher/)
- [Flux App](https://github.com/stuttgart-things/flux/tree/main/apps/homerun2)
- [Releases](https://github.com/stuttgart-things/homerun2-core-catcher/releases)
- [Container Images](https://github.com/stuttgart-things/homerun2-core-catcher/pkgs/container/homerun2-core-catcher)
- [homerun2-omni-pitcher](https://github.com/stuttgart-things/homerun2-omni-pitcher) (producer)
- [homerun-library](https://github.com/stuttgart-things/homerun-library)

## Author Information

```bash
Patrick Hermann, stuttgart-things 03/2026
```

## License

See [LICENSE](LICENSE) file.
