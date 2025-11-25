# FastAPI Hello World

A basic FastAPI application using UV for dependency management.

## Setup

1. Install UV (if not already installed):
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. Install dependencies:
   ```bash
   uv sync
   ```

3. Run the application:
   ```bash
   uv run uvicorn main:app --reload
   ```

4. Visit:
   - API: http://localhost:8000
   - Interactive API docs: http://localhost:8000/docs
   - Alternative API docs: http://localhost:8000/redoc

## Endpoints

- `GET /` - Returns a simple "Hello World" message
- `GET /hello/{name}` - Returns a personalized greeting
