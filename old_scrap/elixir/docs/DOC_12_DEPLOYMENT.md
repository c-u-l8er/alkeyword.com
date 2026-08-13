# Document 12: Deployment Guide

## Overview

This guide covers deploying Alkeyword applications to production, including Docker containers, Kubernetes clusters, database migrations, and infrastructure setup.


### Production Setup

```elixir
# config/prod.exs
config :alkeyword,
  repo: MyApp.Repo,
  enable_observable: true,
  enable_profiling: false,  # Disable in prod
  dashboard_path: "/admin/alkeyword"

config :alkeyword_observable,
  metrics_retention_days: 30,
  aggregate_interval_seconds: 60
```

### Database Migrations

```bash
# Run Alkeyword migrations
mix alkeyword.gen.migration
mix ecto.migrate

# Creates tables:
# - alkeyword_types
# - alkeyword_metrics  
# - alkeyword_pattern_matches
```

### Docker Deployment

```dockerfile
FROM elixir:1.14-alpine

RUN mix local.hex --force && \
    mix local.rebar --force

WORKDIR /app

COPY mix.exs mix.lock ./
RUN mix deps.get --only prod
RUN mix deps.compile

COPY . .
RUN mix alkeyword.install
RUN mix compile
RUN mix assets.deploy

ENV MIX_ENV=prod
CMD ["mix", "phx.server"]
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-alkeyword
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: ALKEYWORD_ENABLE_OBSERVABLE
          value: "true"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

---

