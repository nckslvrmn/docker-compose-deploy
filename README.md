# docker-compose-deploy

A GitHub Actions composite action that deploys or updates a Docker Compose stack on a self-hosted runner.

## How it works

The action runs `docker compose up` on the self-hosted runner, which executes directly on the deployment host. The compose file is checked out from the repository, secrets are injected via the calling workflow, and Docker Compose substitutes them at runtime.

## Prerequisites

- A self-hosted GitHub Actions runner registered to your repo or org, running on the target host with Docker socket access. See [github-multi-runner](https://github.com/youruser/github-multi-runner) for a ready-made setup.
- Docker on the runner host. Docker Compose v2 is installed automatically by this action.

## Usage

```yaml
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: [self-hosted]
    steps:
      - uses: actions/checkout@v4
      - uses: youruser/docker-compose-deploy@main
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          API_KEY: ${{ secrets.API_KEY }}
```

Environment variables set on the action step are available to Docker Compose for variable substitution in the compose file:

```yaml
services:
  app:
    image: ghcr.io/youruser/app:latest
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - API_KEY=${API_KEY}
```

Secrets are never written to the compose file. They are injected at deploy time from GitHub's encrypted secret store and masked in all logs.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `file` | No | `docker-compose.yml` | Path to the compose file |
| `args` | No | — | Additional arguments passed to `docker compose up` |

## Project name

Docker Compose uses the checkout directory name as the project name by default, which matches the repository name. To override it, set the `name` key at the top of your compose file:

```yaml
name: my-custom-project

services:
  ...
```

This ensures updates are applied to the correct running stack rather than creating a duplicate.

## Patterns

### Single service repo

Each service lives in its own repository with its own `docker-compose.yml` and workflow. The compose file serves as both the deployment config and a working example for anyone using the repository.

```
my-service/
  docker-compose.yml
  .github/
    workflows/
      deploy.yml
```

### Multiple stacks in one workflow

To deploy multiple compose files as separate stacks, use the action once per file:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: youruser/docker-compose-deploy@main
    with:
      file: infra/docker-compose.yml
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  - uses: youruser/docker-compose-deploy@main
    with:
      file: services/app/docker-compose.yml
    env:
      API_KEY: ${{ secrets.API_KEY }}
```

Each step manages its own stack independently. Stacks are deployed in order.

### Central infrastructure repo

A private repository owns the base infrastructure (reverse proxy, shared networks, databases). Each service repo deploys independently and connects to the shared network declared as external:

```yaml
# base infrastructure compose (private repo)
networks:
  proxy:
    name: proxy

# per-service compose (service repo)
networks:
  proxy:
    external: true
```

The infrastructure repo and each service repo each have their own workflow. Deployments are fully independent.

## Triggering on image publish

To deploy automatically when a new image is pushed to GHCR, use the `registry_package` event:

```yaml
on:
  registry_package:
    types: [published]
```

This fires the workflow the moment a new image version is available, before any polling interval would catch it.
