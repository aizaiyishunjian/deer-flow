FROM python:3.12-slim as builder

# 安装构建工具 uv
COPY --from=ghcr.io/astral-sh/uv:0.7.20 /uv /uvx /usr/local/bin/

WORKDIR /app

# 仅复制需要构建的本地包的源码
COPY backend/packages/harness ./packages/harness
# 复制顶层 pyproject.toml 以便 uv 知道构建系统 (如 hatchling)
COPY backend/pyproject.toml ./pyproject.toml

# 1. 安装构建 deerflow-harness 所需的依赖 (比如 hatchling)
# 2. 将 deerflow-harness 构建成一个 .whl 文件，并存放到 /app/dist 目录
RUN uv pip install --system "hatchling" && \
    uv wheel --out-dir /app/dist ./packages/harness

# Backend Development Dockerfile
FROM python:3.12-slim

ARG NODE_MAJOR=22

# Install system dependencies + Node.js (provides npx for MCP servers)
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
    gnupg \
    ca-certificates \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_${NODE_MAJOR}.x nodistro main" > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Install Docker CLI (for DooD: allows starting sandbox containers via host Docker socket)
COPY --from=docker:cli /usr/local/bin/docker /usr/local/bin/docker

# Install uv from a pinned versioned image (avoids curl|sh from untrusted remote)
COPY --from=ghcr.io/astral-sh/uv:0.7.20 /uv /uvx /usr/local/bin/

# Set working directory
WORKDIR /app

COPY --from=builder /app/dist /app/dist

# Copy frontend source code
COPY backend ./backend

# Install dependencies with cache mount
RUN --mount=type=cache,target=/root/.cache/uv \
    sh -c "uv pip install /app/dist/*.whl && \
           cd backend && uv sync"

# Expose ports (gateway: 8001, langgraph: 2024)
EXPOSE 8001 2024

# Default command (can be overridden in docker-compose)
CMD ["sh", "-c", "cd backend && PYTHONPATH=. uv run uvicorn app.gateway.app:app --host 0.0.0.0 --port 8001"]
