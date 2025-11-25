# CI/CD Test Project

This project consists of two services: a Python API backend and a React Router UI frontend, both containerized with Docker.

## Prerequisites

- [Docker](https://www.docker.com/get-started) (version 20.10 or later)

## Project Structure

- `api-project/` - Python FastAPI backend service
- `ui-project/` - React Router frontend application
- `docker-compose.yml` - Docker Compose configuration for orchestrating both services

## Quick Start

### Starting the Project

To start both services using Docker Compose:

```bash
docker-compose up
```

This will:
1. Build the Docker images for both the API and UI services
2. Start the API service on port `8000`
3. Start the UI service on port `3000`
4. Create a shared network for service communication

### Running in Detached Mode

To run the services in the background:

```bash
docker-compose up -d
```

### Viewing Logs

To view logs from all services:

```bash
docker-compose logs -f
```

To view logs from a specific service:

```bash
docker-compose logs -f api
docker-compose logs -f ui
```

### Stopping the Project

To stop the running services:

```bash
docker-compose down
```

To stop and remove volumes (if any):

```bash
docker-compose down -v
```

## Services

### API Service

- **Container Name**: `api-project`
- **Port**: `8000`
- **URL**: http://localhost:8000
- **Technology**: Python 3.14 with FastAPI/uvicorn
- **Dockerfile**: `api-project/Dockerfile`

The API service uses UV for dependency management and runs a uvicorn server.

### UI Service

- **Container Name**: `ui-project`
- **Port**: `3000`
- **URL**: http://localhost:3000
- **Technology**: Node.js 20 with React Router
- **Dockerfile**: `ui-project/Dockerfile`
- **Dependencies**: Depends on the API service

The UI service is built using a multi-stage Docker build process for optimal production images.

## Rebuilding Images

If you make changes to the code or dependencies, rebuild the images:

```bash
docker-compose build
```

To rebuild without using cache:

```bash
docker-compose build --no-cache
```

Then start the services:

```bash
docker-compose up
```

## Troubleshooting

### Port Already in Use

If ports 8000 or 3000 are already in use, you can modify the port mappings in `docker-compose.yml`:

```yaml
ports:
  - "8001:8000"  # Change host port from 8000 to 8001
```

### View Container Status

To check the status of running containers:

```bash
docker-compose ps
```

### Access Container Shell

To access a running container's shell:

```bash
# API container
docker exec -it api-project sh

# UI container
docker exec -it ui-project sh
```
