# Tools and Connections Reference

Detailed reference for defining tools and connecting existing services to a HUD environment.

## Defining Tools

### @env.tool() decorator

The simplest way to expose functionality to agents:

```python
from hud import Environment

env = Environment("my-env")

@env.tool()
def search(query: str, max_results: int = 10) -> str:
    """Search the knowledge base and return matching documents."""
    return db.search(query, limit=max_results)
```

Rules:
- Current HUD SDK examples and tests use return annotations. Do not carry forward older blanket "never annotate returns" guidance without re-checking the current SDK.
- The **docstring** becomes the tool description agents see -- make it clear and actionable
- **Type hints on parameters** define the input schema -- always include them
- **Default values** make parameters optional
- Both **sync and async** functions work
- **To return images or mixed content**, use `ContentResult` via a `BaseTool` subclass and `add_tool()` instead of `@env.tool()` (see below).

### Async tools

Use async for I/O-bound operations:

```python
@env.tool()
async def fetch_page(url: str) -> str:
    """Fetch a web page and return its text content."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
        return resp.text
```

### Returning images and mixed content

For tools that return images, use `ContentResult` from `hud.tools.types` with a `BaseTool` subclass:

```python
from hud.tools.base import BaseTool
from hud.tools.types import ContentResult

class RenderBoard(BaseTool):
    async def __call__(self):
        image_b64 = render_board_as_png()
        return ContentResult(
            output=f"Score: {score}, Max tile: {max_tile}",
            base64_image=image_b64,
        )

env.add_tool(RenderBoard(name="render_board", description="Render the game board as a PNG"))
```

`__call__` must declare **explicit parameters** -- `**kwargs` is not supported. FastMCP parses the signature to build the tool's input schema.

`ContentResult` converts to proper MCP content blocks -- `TextContent` for `output`/`error`, `ImageContent` for `base64_image`. Fields:

| Field | Type | Description |
|-------|------|-------------|
| `output` | `str \| None` | Text output |
| `error` | `str \| None` | Error message |
| `base64_image` | `str \| None` | Base64-encoded PNG image |
| `system` | `str \| None` | System message |
| `url` | `str \| None` | Current page URL |

### add_tool()

Add tool instances programmatically instead of using the decorator:

```python
from hud.tools.coding import ShellTool, ApplyPatchTool, EditTool
from hud.tools.filesystem import ReadTool, GrepTool, GlobTool, ListTool

env = Environment("coding-env")

env.add_tool(ShellTool(cwd="/app"))
env.add_tool(ApplyPatchTool(base_path="/app"))
```

## Pre-built Tools

HUD provides tools that match common agent interfaces. Import from `hud.tools`:

### Coding tools (`hud.tools.coding`)

| Tool | Description |
|------|-------------|
| `ShellTool(cwd=...)` | Run shell commands. Conforms to OpenAI's shell tool spec. |
| `ApplyPatchTool(base_path=...)` | Create, update, delete files using V4A diffs. Conforms to OpenAI's apply_patch spec. |
| `EditTool(...)` | View, create, and edit files using str_replace operations. |

### Filesystem tools (`hud.tools.filesystem`)

| Tool | Description |
|------|-------------|
| `ReadTool(...)` | Read file contents with pagination support. |
| `GrepTool(...)` | Search file contents using regex. |
| `GlobTool(...)` | Find files matching glob patterns. |
| `ListTool(...)` | List directory contents in tree format. |

### Example: coding agent environment

```python
from hud import Environment
from hud.tools.coding import ShellTool, ApplyPatchTool
from hud.tools.filesystem import ReadTool, GrepTool, GlobTool, ListTool

env = Environment("coding-agent")

base = "/workspace"
env.add_tool(ShellTool(cwd=base))
env.add_tool(ApplyPatchTool(base_path=base))
env.add_tool(ReadTool())
env.add_tool(GrepTool())
env.add_tool(GlobTool())
env.add_tool(ListTool())
```

## Connection Methods

Connect existing infrastructure instead of rewriting it. `prefix` is broadly supported for namespacing, but `include` / `exclude` / `transform` are only available on some connectors.

### connect_fastapi(app)

Mount a FastAPI application. All routes become tools:

```python
from fastapi import FastAPI

app = FastAPI()

@app.post("/search")
async def search(query: str) -> list[str]:
    return db.search(query)

env.connect_fastapi(app)
```

Options:
- `name` -- override the server name
- `prefix` -- namespace tool names (e.g. `prefix="api"` turns `search` into `api_search`)
- `include_hidden` -- include routes with `include_in_schema=False` (default: `True`)

### connect_hub(slug)

Connect to a deployed HUD Hub environment:

```python
env.connect_hub("browser")

# With namespace prefix
env.connect_hub("hud-evals/browser", prefix="browser")
```

Options:
- `prefix` -- namespace tool names
- `include` -- whitelist specific tools
- `exclude` -- blacklist specific tools

### connect_image(image)

Run a Docker image as an MCP server via stdio (`docker run -i --rm`):

```python
env.connect_image("mcp/fetch")

# With extra Docker args and env vars
env.connect_image(
    "my-service:v1",
    docker_args=["--network", "host"],
    env_vars={"DATABASE_URL": "postgres://..."},
)
```

Environment variables from `.env` files are auto-injected.

Options:
- `docker_args` -- additional Docker CLI arguments
- `env_vars` -- environment variables to inject
- `prefix`, `include`, `exclude` -- tool filtering

### connect_openapi(spec)

Generate tools from an OpenAPI specification:

```python
# From URL
env.connect_openapi("https://api.example.com/openapi.json")

# From file path
env.connect_openapi("./openapi.yaml")

# From dict
env.connect_openapi(spec_dict, base_url="https://api.example.com")
```

Options:
- `base_url` -- override the API base URL
- `headers` -- add auth headers to all requests
- `timeout` -- request timeout in seconds (default: 30)
- `name`, `prefix` -- naming

### connect_mcp_config(config)

Connect one or more MCP servers using a config dictionary:

```python
env.connect_mcp_config({
    "sqlite": {"command": "uvx", "args": ["mcp-server-sqlite", "db.sqlite"]},
    "github": {"command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"]},
})
```

Each key is a server name. Values follow the MCP server config format:
- `command` + `args` -- stdio transport
- `url` -- SSE/streamable HTTP transport

### connect_server(server)

Mount an in-process MCPServer or FastMCP instance:

```python
from fastmcp import FastMCP

mcp = FastMCP("my-server")

@mcp.tool()
def my_tool(x: int) -> int:
    return x * 2

env.connect_server(mcp)
```

### connect_url(url)

Connect to a remote MCP server by URL:

```python
env.connect_url("http://localhost:3000/mcp")

# With auth headers
env.connect_url(
    "https://my-server.example.com/mcp",
    headers={"Authorization": "Bearer token123"},
)
```

## Tool Filtering

Some connection methods support `include`, `exclude`, and `transform`, especially `connect_hub()`, `connect_image()`, `connect_url()`, and `connect_mcp_config()`:

```python
# Only expose specific tools
env.connect_hub("browser", include=["navigate", "click", "type"])

# Exclude tools
env.connect_hub("coding-sandbox", exclude=["rm", "shutdown"])

# Transform tool definitions (rename, modify descriptions, etc.)
def rename_tool(tool):
    tool.name = f"custom_{tool.name}"
    return tool

env.connect_hub("my-env", transform=rename_tool)
```

## Combining Local and Remote Tools

An environment can mix local tools with connections:

```python
env = Environment("hybrid-env")

# Local tools
@env.tool()
def score(text: str):
    """Score text quality."""
    return model.predict(text)

# Remote tools
env.connect_hub("browser", prefix="browser")
env.connect_fastapi(my_api, prefix="api")
```

Agents see all tools in a flat namespace. Use `prefix` to avoid name collisions.
