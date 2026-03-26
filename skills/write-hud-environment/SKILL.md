---
name: write-hud-environment
description: Create HUD environments with tools, scenarios, tasks, and connectors for AI agents. Use when asked to scaffold or write a HUD environment, define @env.tool() or @env.scenario() logic, wrap existing APIs or services as agent tools, or set up local HUD development and testing with the hud-python SDK.
license: MIT
metadata:
  author: hud-ai
  version: "1.1.0"
---

# Write a HUD Environment

An environment is the harness an agent operates in. It packages tools (what the agent can do) and scenarios (how the agent is evaluated) into one deployable unit. Each deployed environment spins up fresh for every evaluation, so runs stay isolated and reproducible.

## Start With The Right Workflow

HUD currently has two common authoring paths. Use the template-first path for new projects, and the SDK-first path when working in an existing repo or a simple single-module environment.

### Template-first path for new projects

```bash
uv tool install hud-python --python 3.12
hud set HUD_API_KEY=your-key-here
hud init my-env
cd my-env
hud dev --inspector
```

`hud init` now downloads a preset environment template. The current CLI/docs flow is centered on an environment directory with a Dockerfile, `tasks.json`, and generated source folders such as `controller/` and optional `environment/`.

Typical generated structure:

```text
my-env/
├── Dockerfile
├── pyproject.toml
├── tasks.json
├── controller/
└── environment/   # optional, depending on template
```

### SDK-first path for simple or existing repos

If you are building a smaller module-driven environment or editing an older/example repo, author an `Environment` object directly:

```python
from hud import Environment

env = Environment("my-env")
```

Then run it explicitly with:

```bash
hud dev env:env
```

The `env:env` syntax is `module:attribute`, meaning "import module `env` and serve attribute `env`". This path is still supported, but it is no longer the only environment workflow exposed by the CLI/docs.

## Core SDK Model

Even when you start from a generated template, the core HUD model is still the same:

```python
from hud import Environment
from hud.tools.types import EvaluationResult

env = Environment("my-env")


@env.tool()
def search(query: str) -> str:
    """Search the knowledge base."""
    return db.search(query)


@env.scenario("find-answer")
async def find_answer(question: str):
    answer = yield f"Find the answer to: {question}"
    yield EvaluationResult(
        reward=1.0 if "correct" in answer.lower() else 0.0,
        content=f"Agent answer: {answer}",
    )
```

## Tools

Every tool is a function an agent can call while it is solving a task. Decorate it with `@env.tool()` and HUD turns the function signature into an MCP tool schema:

```python
@env.tool()
def search(query: str) -> str:
    """Search the knowledge base."""
    return db.search(query)


@env.tool()
async def fetch_page(url: str) -> str:
    """Fetch a web page and return its text content."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return resp.text
```

Rules of thumb:

- The docstring becomes the tool description the agent sees.
- Parameter type hints define the input schema.
- Both sync and async functions work.
- Current SDK examples and tests use return annotations. Do not carry forward older blanket "never annotate returns" advice without re-verifying against the current SDK.

For rich content such as images, return a `ContentResult` from a `BaseTool` subclass and register it with `add_tool()`:

```python
from hud.tools.base import BaseTool
from hud.tools.types import ContentResult


class ScreenshotTool(BaseTool):
    async def __call__(self, url: str):
        image_b64 = capture_screenshot(url)
        return ContentResult(output="Screenshot taken", base64_image=image_b64)


env.add_tool(ScreenshotTool(name="screenshot", description="Take a screenshot of a URL"))
```

Keep `__call__` explicit: declare concrete parameters instead of `**kwargs`.

For detailed connector/tool patterns, see [references/tools-and-connections.md](references/tools-and-connections.md). Treat those examples as supporting material and verify edge cases against the current SDK when behavior matters.

## Scenarios

A scenario defines the evaluation. In the current SDK, it is an async generator with two yields:

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

- First yield: the prompt sent to the agent. The value sent back in becomes the agent's final answer.
- Second yield: the evaluation result. `EvaluationResult` is the canonical type, though floats and bools still coerce in many cases.

```python
from hud.tools.types import EvaluationResult, SubScore

yield EvaluationResult(
    reward=0.85,
    done=True,
    content="Found 17 of 20 items",
    subscores=[
        SubScore(name="detection", weight=0.7, value=0.85),
        SubScore(name="accuracy", weight=0.3, value=1.0),
    ],
)

yield 1.0
yield correct_answer in response
```

Instantiate tasks from scenarios with keyword arguments:

```python
task = env("checkout", product_name="Widget Pro")
```

Useful current capabilities to remember:

- Scenarios can require env vars.
- Scenarios can filter which tools the agent sees with `exclude_tools`, `exclude_sources`, and `allowed_tools`.
- The SDK supports richer answer and evaluation handling than a bare string-in/float-out loop.

For detailed patterns including parameterized scenarios and scoring examples, see [references/scenarios.md](references/scenarios.md).

## Connecting Existing Services

Do not rewrite your stack if you can wrap it:

```python
from my_app import app
env.connect_fastapi(app)

env.connect_openapi("https://api.example.com/openapi.json")

env.connect_image("my-service:v1")

env.connect_mcp_config({
    "my-server": {"command": "uvx", "args": ["some-mcp-server"]}
})

env.connect_hub("my-org/my-env", prefix="remote")

from my_server import mcp
env.connect_server(mcp)
```

The common connector set is still:

- `connect_fastapi`
- `connect_openapi`
- `connect_image`
- `connect_mcp_config`
- `connect_hub`
- `connect_server`

## Local Development And Testing

### Local dev via generated environment directory

For current template-based environments, start from the environment directory and use `hud dev`:

```bash
hud dev
hud dev -w controller -w environment --inspector
```

Use `--watch` to opt into hot reload. Without `--watch`, `hud dev` runs once without file watching.

### Local dev via explicit module mode

For `Environment`-based repos with an `env.py` file or similar:

```bash
hud dev env:env
hud dev env:env -w env.py
```

In HTTP mode, Cursor or another MCP client can connect to the local server:

```json
{
  "my-dev-env": { "url": "http://localhost:8765/mcp" }
}
```

### Evaluate locally from tasks or code

For task files generated or maintained in the environment directory:

```bash
hud eval tasks.json claude
```

For direct code-driven loops, use `hud.eval(task)`:

```python
import hud
from hud.agents import create_agent

task = env("checkout", product_name="Widget Pro")
agent = create_agent("claude-sonnet-4-5")

async with hud.eval(task) as ctx:
    result = await agent.run(ctx, max_steps=10)

print(f"Reward: {result.reward}")
```

You can also build a custom loop by converting the environment and tools into the provider's tool format and calling `ctx.call_tool()` / `ctx.submit()` yourself:

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

### Validate the deployable environment

Before deployment, build locally:

```bash
hud build
```

This validates the environment, analyzes the MCP surface, and writes `hud.lock.yaml`.

## What's Next

- [references/tools-and-connections.md](references/tools-and-connections.md) for connector and tool patterns
- [references/scenarios.md](references/scenarios.md) for scenario patterns and evaluation examples
- [Deploy to HUD](../deploy-hud-environment/SKILL.md) for the current build/deploy flow
- [HUD Docs](https://docs.hud.ai/platform/environments) for the live product/docs view
