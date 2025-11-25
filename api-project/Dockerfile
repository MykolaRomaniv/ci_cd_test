FROM python:3.14-slim

WORKDIR /app

# Install UV
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
# Add UV to PATH
# ENV PATH="/root/.local/bin:$PATH"

# Copy dependency files
COPY pyproject.toml uv.lock* ./

# Install dependencies using UV
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --system --no-dev

# Copy application code
COPY main.py ./

# Expose port
EXPOSE 8000

# Run the application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
