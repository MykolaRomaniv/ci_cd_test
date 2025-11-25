# CI/CD Pipelines

This directory contains GitHub Actions workflows for continuous integration and continuous deployment.

## Workflows

### CI Workflows

#### `ci-api.yml`
Runs on changes to the `api-project` directory:
- **Test Job**: Installs dependencies using UV, verifies the FastAPI application can start and respond to requests
- **Docker Build Job**: Builds the Docker image to ensure it compiles correctly

#### `ci-ui.yml`
Runs on changes to the `ui-project` directory:
- **Test Job**: Installs dependencies using pnpm, runs TypeScript type checking, and builds the React Router application
- **Docker Build Job**: Builds the Docker image to ensure it compiles correctly

#### `ci-docker-compose.yml`
Runs on changes to either project or `docker-compose.yml`:
- Builds and tests the full stack using Docker Compose
- Verifies both API and UI services start correctly and are accessible

### CD Workflow

#### `cd-deploy.yml`
Runs on pushes to `main` branch or manual trigger:
- Builds and pushes Docker images to GitHub Container Registry (GHCR)
- Tags images with:
  - Branch name
  - Commit SHA
  - Semantic version (if tags are used)
  - `latest` tag for main branch

## Workflow Triggers

All CI workflows trigger on:
- Pushes to `main` or `develop` branches
- Pull requests targeting `main` or `develop` branches
- Only when relevant files change (path-based triggers)

CD workflow triggers on:
- Pushes to `main` branch
- Manual workflow dispatch

## Image Registry

Docker images are pushed to GitHub Container Registry:
- API: `ghcr.io/<owner>/<repo>/api-project`
- UI: `ghcr.io/<owner>/<repo>/ui-project`

To pull images:
```bash
docker pull ghcr.io/<owner>/<repo>/api-project:latest
docker pull ghcr.io/<owner>/<repo>/ui-project:latest
```

## Required Permissions

The CD workflow requires:
- `contents: read` - to read repository contents
- `packages: write` - to push images to GHCR

These are automatically granted via `GITHUB_TOKEN`.

## Caching

All workflows use GitHub Actions cache for:
- Docker layer caching (via `cache-from` and `cache-to`)
- Node.js dependencies (via `actions/setup-node` cache)
- UV dependencies (via UV's built-in caching)

## Customization

To customize deployment targets or add additional steps:
1. Modify the relevant workflow file
2. Add secrets to GitHub repository settings if needed
3. Update environment variables in the workflow files
