# Bitbucket Pipelines Self-Hosted Runner

> Custom Docker image for running Bitbucket Pipelines with Docker + Compose support.

## Overview

This runner extends the official `bitbucket-pipelines-runner` image with Docker CLI and Docker Compose pre-installed. It allows pipeline steps to run `docker` and `docker compose` commands on the host's Docker daemon via the mounted socket.

## Files

```
bitbucket-runner/
├── Dockerfile            # Custom runner image with Docker CLI
├── docker-compose.yml    # Runner deployment
├── .env                  # Environment variables (account, repo, OAuth)
└── workspace/            # Mounted working directory (/tmp)
```

## Prerequisites

- Docker Engine on the host
- Bitbucket account with Pipelines enabled
- A self-hosted runner registered in Bitbucket

## Setup

### 1. Configure `.env`

```env
ACCOUNT_UUID={your-account-uuid}
REPOSITORY_UUID={your-repo-uuid}
RUNNER_UUID={your-runner-uuid}
OAUTH_CLIENT_ID=your-oauth-client-id
OAUTH_CLIENT_SECRET=your-oauth-client-secret
WORKING_DIRECTORY=/tmp
```

### 2. Start the runner

```bash
docker compose up -d
```

### 3. Verify

```bash
docker compose logs -f
```

You should see the runner connect to Bitbucket and wait for jobs.

## How It Works

```
Host (ipdrtesting)
│
├── Bitbucket Runner Container
│   ├── Atlassian runner agent
│   ├── Docker CLI (docker)
│   ├── Docker Compose (docker compose)
│   └── Mounts:
│       ├── /var/run/docker.sock  → Host Docker daemon
│       ├── ./workspace:/tmp       → Build workspace
│       └── /var/lib/docker/containers → Container logs (read-only)
│
└── Pipeline Step Container (docker:24.0)
    ├── Docker CLI
    └── Docker socket forwarded by runner
        → Commands execute on host's Docker daemon
```

When a pipeline runs with `image: docker:24.0`, Bitbucket spins up a separate container with Docker CLI. The runner forwards the Docker socket into this container, so all `docker compose` commands run on the host daemon.

## Bitbucket Pipeline Config

In your repository's `bitbucket-pipelines.yml`:

```yaml
pipelines:
  branches:
    develop:
      - step:
          name: Deploy
          image: docker:24.0
          runs-on:
            - self.hosted
            - syslog.test
            - linux
          script:
            - docker compose build
            - docker compose up -d
```

## Building the Image

```bash
docker compose build
```

The Dockerfile installs:
- `docker-ce-cli` — Docker command line
- `docker-compose-plugin` — Docker Compose v2

## Troubleshooting

### Runner won't connect

Check the logs and verify environment variables in `.env`:

```bash
docker compose logs
```

### Docker commands fail inside pipeline

Ensure the runner container has the Docker socket mounted:

```bash
docker exec runner-bitbucket docker ps
```

If this fails, restart the runner with the socket mount.

### Pipeline uses wrong image

Verify `image:` is set in `bitbucket-pipelines.yml`. Without it, Bitbucket defaults to `atlassian/default-image` which lacks Docker CLI.

## Environment Variables

| Variable | Description |
|---|---|
| `ACCOUNT_UUID` | Bitbucket account UUID |
| `REPOSITORY_UUID` | Repository UUID |
| `RUNNER_UUID` | Runner UUID (registered in Bitbucket) |
| `OAUTH_CLIENT_ID` | OAuth consumer key |
| `OAUTH_CLIENT_SECRET` | OAuth consumer secret |
| `WORKING_DIRECTORY` | Pipeline workspace path (`/tmp`) |
| `RUNTIME_PREREQUISITES_ENABLED` | Enables Docker socket forwarding |
