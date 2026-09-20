# Configuration

## Environment Variables

All configuration is via environment variables, loaded once at startup.

| Variable         | Default                    | Description                          |
|------------------|----------------------------|--------------------------------------|
| `REDIS_ADDR`     | `localhost`                | Redis server address                 |
| `REDIS_PORT`     | `6379`                     | Redis server port                    |
| `REDIS_PASSWORD` | (empty)                    | Redis password                       |
| `REDIS_STREAM`   | `messages`                 | Redis stream to consume from         |
| `CONSUMER_GROUP`| `homerun2-core-catcher`     | Consumer group name                  |
| `CONSUMER_NAME` | hostname                    | Consumer name within the group       |
| `REDIS_STARTUP_TIMEOUT` | `120s`              | How long startup retries Redis before exiting (Go duration) |
| `LOG_FORMAT`     | `json`                     | Log format: `json` or `text`         |
| `LOG_LEVEL`      | `info`                     | Log level: `debug`, `info`, `warn`, `error` |

## Consumer Behavior

The catcher uses Redis consumer groups (`XREADGROUP`) to ensure reliable message delivery:

- Messages are acknowledged after processing
- Multiple catcher instances can share the same consumer group for load balancing
- Each instance should have a unique `CONSUMER_NAME`

## Running Locally Against Remote Redis

Port-forward Redis from a Kubernetes cluster:

```bash
kubectl port-forward -n redis-stack svc/redis-stack 6379:6379

# Use a script to avoid shell escaping issues with special characters
cat > /tmp/run-catcher.sh << 'EOF'
#!/bin/bash
export REDIS_ADDR=localhost
export REDIS_PORT=6379
export REDIS_PASSWORD='<your-password>'
export REDIS_STREAM=messages
export LOG_FORMAT=text
go run .
EOF
bash /tmp/run-catcher.sh
```

!!! note
    Passwords containing `!` or other shell-special characters must be set inside a script with single-quoted heredoc to avoid bash history expansion.
