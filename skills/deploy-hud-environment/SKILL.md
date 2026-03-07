---
name: deploy-hud-environment
description: Deploy HUD environments to the platform for parallel evaluations and training at scale. Use when asked to "deploy my environment", "push to HUD", "run evals at scale", or "set up remote environments".
license: MIT
metadata:
  author: hud-ai
  version: "1.0.0"
---

# Deploy a HUD Environment

Deploy your environment so agents can run against it in parallel, all traced, all generating training data.

## Prerequisites

Your project needs three files:

```
my-env/
├── env.py           # Environment definition
├── pyproject.toml   # Dependencies
└── Dockerfile       # Container definition
```

### pyproject.toml

```toml
[project]
name = "my-env"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["hud-python", "openai"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY pyproject.toml uv.lock* ./
RUN pip install uv && uv sync --frozen --no-dev 2>/dev/null || uv sync --no-dev
COPY . .

CMD ["uv", "run", "python", "-m", "hud", "dev", "env:env", "--stdio"]
```

The `CMD` uses `env:env` syntax (`module:attribute`) to import `env.py` and run the `env` object as an MCP server. Adjust if your environment is in a different file or variable.

## Deploy via CLI

```bash
# Build the Docker image and lock file
hud build

# Deploy to the HUD platform
hud deploy
```

## Deploy via GitHub

1. Push your repo to GitHub
2. Go to [hud.ai](https://hud.ai) -> New -> Environment
3. Choose "From GitHub URL" and paste your repo URL
4. HUD builds and deploys automatically

## Using Deployed Environments

Once deployed, connect to your environment from anywhere:

```python
from hud import Environment

env = Environment("my-agent")
env.connect_hub("my-org/my-env")

async with env:
    result = await env.call_tool("search", query="hello")
```

Or reference it in a task:

```python
from hud.eval.task import Task

task = Task(
    env={"name": "my-org/my-env"},
    scenario="find-answer",
    args={"question": "What is 2+2?"},
)
```

## What Deployment Enables

- **Parallel evaluations** -- run hundreds of agents simultaneously, each in an isolated instance
- **Training data** -- every agent interaction generates structured training data
- **Team sharing** -- anyone with access can run evals against your environment
- **Composability** -- other environments can `connect_hub()` to yours

## Resources

- [HUD Docs: Hosted Running](https://docs.hud.ai/quick-links/deploy)
- [HUD Docs: Environments](https://docs.hud.ai/quick-links/environments)
- [HUD Platform](https://hud.ai)
