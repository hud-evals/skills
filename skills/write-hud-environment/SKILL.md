---
name: write-hud-environment
description: Create HUD environments with tools and evaluation scenarios for AI agents. Use when asked to "create an environment", "write a HUD env", "add tools for agents", "define evaluation scenarios", "wrap my API for agents", or when building agent-callable tools with the hud-python SDK.
license: MIT
metadata:
  author: hud-ai
  version: "1.0.0"
---

# Write a HUD Environment

An environment is everything an agent can interact with -- your APIs, services, databases, wrapped as tools. It also defines how agents are evaluated through scenarios. Each environment spins up fresh for every evaluation: isolated, deterministic, reproducible.

## Quick Start

Scaffold a new environment:

```bash
uv tool install hud-python --python 3.12
hud set HUD_API_KEY=your-key-here
hud init
```

This creates `env.py`, `pyproject.toml`, and `Dockerfile`. Or create manually:

```python
from hud import Environment

env = Environment("my-env")
```

**Naming rule:** The environment name must start with a letter. It becomes a URI scheme internally (e.g. `my-env:scenario-name`), and URI schemes must begin with a letter per RFC 3986.

## Project Structure

A minimal environment needs three files:

```
my-env/
├── env.py           # Environment definition (tools + scenarios)
├── pyproject.toml   # Dependencies
└── Dockerfile       # For deployment (optional for local dev)
```

**pyproject.toml:**

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

## Tools

Every tool is a function. Decorate it with `@env.tool()` and agents can call it:

```python
@env.tool()
def search(query: str):
    """Search the knowledge base."""
    return db.search(query)

@env.tool()
async def fetch_page(url: str):
    """Fetch a web page and return its text content."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return resp.text
```

The docstring becomes the tool description. Type hints on **parameters** define the input schema. Both sync and async functions work.

**Do not add return type annotations** to `@env.tool()` functions or `BaseTool.__call__` methods. A return type annotation generates an MCP `outputSchema` that the current SDK does not handle correctly over MCP, causing `-32600` errors.

**To return images**, use `ContentResult` with a `BaseTool` subclass via `add_tool()`:

```python
from hud.tools.types import ContentResult
from hud.tools.base import BaseTool

class ScreenshotTool(BaseTool):
    async def __call__(self, url: str):
        image_b64 = capture_screenshot(url)
        return ContentResult(output="Screenshot taken", base64_image=image_b64)

env.add_tool(ScreenshotTool(name="screenshot", description="Take a screenshot of a URL"))
```

`__call__` must declare **explicit parameters** -- `**kwargs` is not supported.

For detailed patterns including pre-built tools, all connection types, and `ContentResult` fields, see [references/tools-and-connections.md](references/tools-and-connections.md).

## Scenarios

Scenarios define what to tell the agent and how to score what it did. Two `yield` statements:

```python
from hud.tools.types import EvaluationResult, SubScore

@env.scenario("checkout")
async def checkout_flow(product_name: str):
    answer = yield f"Add '{product_name}' to cart and complete checkout"

    order_exists = await check_order_status(product_name)
    yield EvaluationResult(
        reward=1.0 if order_exists else 0.0,
        content=f"Order exists: {order_exists}",
    )
```

- First yield: sends the prompt, returns the agent's final answer
- Second yield: an `EvaluationResult` (from `hud.tools.types`). Floats and bools also work as shorthand and get coerced to `EvaluationResult` internally, but `EvaluationResult` is the canonical type:

```python
from hud.tools.types import EvaluationResult, SubScore

# Full structured result with subscores
yield EvaluationResult(
    reward=0.85,
    done=True,
    content="Found 17 of 20 items",
    subscores=[
        SubScore(name="detection", weight=0.7, value=0.85),
        SubScore(name="accuracy", weight=0.3, value=1.0),
    ],
)

# Shorthand (coerced to EvaluationResult internally)
yield 1.0
yield correct_answer in response
```

Create tasks from scenarios with keyword arguments:

```python
task = env("checkout", product_name="Widget Pro")
```

For detailed patterns including `EvaluationResult`, structured scoring, and parameterized scenarios, see [references/scenarios.md](references/scenarios.md).

## Connecting Existing Services

Don't rewrite your stack. Wrap what you already have:

```python
# FastAPI app -- all routes become tools
from my_app import app
env.connect_fastapi(app)

# OpenAPI spec -- auto-generates tools from endpoints
env.connect_openapi("https://api.example.com/openapi.json")

# Docker image -- runs as MCP server via stdio
env.connect_image("my-service:v1")

# MCP server config -- stdio or SSE
env.connect_mcp_config({
    "my-server": {"command": "uvx", "args": ["some-mcp-server"]}
})

# Another HUD environment (deployed on hub)
env.connect_hub("my-org/my-env", prefix="remote")

# FastMCP / MCPServer -- in-process
from my_server import mcp
env.connect_server(mcp)
```

## Testing Locally

### MCP Server Mode

Spawn your environment as an MCP server for Cursor, Claude Code, or any MCP client:

```bash
hud dev env:env
```

The `env:env` syntax is `module:attribute` -- import `env.py` and serve the `env` object. Enable hot-reload with `-w`:

```bash
hud dev env:env -w env.py -w tools/
```

In Cursor's MCP settings:

```json
{
  "my-dev-env": { "url": "http://localhost:8765/mcp" }
}
```

### Agent Loop

Run a full agent loop locally:

```python
import hud
from hud.agents import create_agent

task = env("checkout", product_name="Widget Pro")
agent = create_agent("claude-sonnet-4-5")

async with hud.eval(task) as ctx:
    result = await agent.run(ctx, max_steps=10)

print(f"Reward: {result.reward}")
```

### Custom Agent Loop

Build your own loop using format converters:

```python
async with hud.eval(task) as ctx:
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": ctx.prompt}],
        tools=ctx.as_openai_chat_tools()
    )

    message = response.choices[0].message
    if message.tool_calls:
        result = await ctx.call_tool(message.tool_calls[0])
        answer = str(result["content"])
    else:
        answer = message.content

    await ctx.submit(answer or "")
```

## Complete Example

```python
"""my-env - HUD Environment"""

import asyncio
import hud
from hud.settings import settings
from hud.tools.types import EvaluationResult
from openai import AsyncOpenAI, Omit
from hud.environment import Environment

env = Environment("my-env")


@env.tool()
def count_letter(text: str, letter: str):
    """Count occurrences of a letter in text."""
    return text.lower().count(letter.lower())


@env.scenario("count")
async def count_scenario(sentence: str, letter: str, fmt: str = "integer"):
    """Agent must count a letter in a sentence."""
    answer = yield f"How many times does '{letter}' appear in: '{sentence}'? Format: {fmt}."

    correct = str(sentence.lower().count(letter.lower()))
    yield EvaluationResult(
        reward=1.0 if correct in answer else 0.0,
        content=f"Expected: {correct}, Got: {answer}",
    )


async def test():
    client = AsyncOpenAI(
        base_url=settings.hud_gateway_url,
        api_key=settings.api_key,
    )

    task = env("count", sentence="Strawberry world", letter="r")

    async with hud.eval(task) as ctx:
        response = await client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": ctx.prompt}],
            tools=ctx.as_openai_chat_tools(),
        )

        message = response.choices[0].message
        if message.tool_calls:
            result = await ctx.call_tool(message.tool_calls[0])
            answer = str(result["content"])
        else:
            answer = message.content

        await ctx.submit(answer or "")


if __name__ == "__main__":
    asyncio.run(test())
```

## What's Next

- [references/tools-and-connections.md](references/tools-and-connections.md) -- detailed tool patterns and connection types
- [references/scenarios.md](references/scenarios.md) -- scenario patterns and evaluation strategies
- [Deploy to HUD](../deploy-hud-environment/SKILL.md) -- deploy for parallel evals and training
- [HUD Docs](https://docs.hud.ai/quick-links/environments) -- full environment documentation
