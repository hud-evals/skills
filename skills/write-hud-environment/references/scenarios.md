# Scenarios Reference

Detailed reference for defining evaluation scenarios in HUD environments.

## The Two-Yield Pattern

Every scenario is an async generator that yields twice:

1. **First yield** -- the prompt sent to the agent. Returns the agent's submitted answer.
2. **Second yield** -- the evaluation result (reward).

```python
@env.scenario("find-answer")
async def find_answer(question: str, expected: str):
    answer = yield f"Find the answer to: {question}"

    yield 1.0 if expected.lower() in answer.lower() else 0.0
```

Between the two yields, the agent runs -- calling tools, reasoning, and eventually submitting an answer via `ctx.submit()`.

## Scenario Parameters

Parameters become the scenario's input schema. They must be JSON-serializable types:

```python
@env.scenario("search-eval")
async def search_eval(
    query: str,                    # Required
    max_results: int = 10,         # Optional with default
    strict: bool = False,          # Optional with default
):
    answer = yield f"Search for '{query}' and return the top {max_results} results."
    yield 1.0 if len(answer) > 0 else 0.0
```

Create tasks from scenarios by passing keyword arguments:

```python
task = env("search-eval", query="python async", max_results=5, strict=True)
```

Or use the typed `.task()` method for IDE autocomplete:

```python
@env.scenario("fix-bug")
async def fix_bug(difficulty: int = 1, hint: str | None = None):
    answer = yield f"Fix the bug. Difficulty: {difficulty}."
    yield 1.0

task = fix_bug.task(difficulty=3, hint="check line 42")
```

## What You Can Yield as a Prompt

The first yield accepts several types:

| Type | Behavior |
|------|----------|
| `str` | Single prompt string (most common) |
| `list[str]` | Multiple prompt strings, joined with newlines |
| `TextContent` | MCP text content block |
| `list[ContentBlock]` | Multiple MCP content blocks |

```python
@env.scenario("multi-step")
async def multi_step(task: str):
    answer = yield [
        "You are an expert assistant.",
        f"Complete the following task: {task}",
        "Show your work step by step.",
    ]
    yield 1.0
```

## Evaluation Return Types

The second yield should be an `EvaluationResult` (from `hud.tools.types`). Floats and bools also work as shorthand and get coerced internally, but `EvaluationResult` is the canonical type.

### EvaluationResult (canonical)


```python
from hud.tools.types import EvaluationResult, SubScore

@env.scenario("code-review")
async def code_review(code: str):
    answer = yield f"Review this code:\n\n{code}"

    correctness = check_correctness(answer)
    style = check_style(answer)
    completeness = check_completeness(answer)

    yield EvaluationResult(
        reward=0.5 * correctness + 0.3 * style + 0.2 * completeness,
        done=True,
        content=f"Correctness: {correctness:.0%}, Style: {style:.0%}, Completeness: {completeness:.0%}",
        subscores=[
            SubScore(name="correctness", weight=0.5, value=correctness),
            SubScore(name="style", weight=0.3, value=style),
            SubScore(name="completeness", weight=0.2, value=completeness),
        ],
    )
```

### Shorthand (coerced to EvaluationResult)

```python
yield 1.0 if success else 0.0           # float
yield "correct" in answer.lower()       # bool → 1.0 / 0.0
```

**EvaluationResult fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `reward` | `float` | `0.0` | Final score, typically 0.0 to 1.0 |
| `done` | `bool` | `True` | Whether the task is complete |
| `content` | `str \| None` | `None` | Human-readable explanation |
| `info` | `dict` | `{}` | Additional metadata |
| `isError` | `bool` | `False` | Whether the evaluation itself failed |
| `subscores` | `list[SubScore] \| None` | `None` | Score breakdown |

**SubScore fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | `str` | required | Name of this component |
| `weight` | `float` | `1.0` | Proportional weight (positive weights should sum to ~1.0) |
| `value` | `float` | required | Score from 0.0 to 1.0 |

## Calling Tools During Evaluation

Scenarios can call the environment's own tools during both setup and evaluation phases:

```python
@env.scenario("browser-task")
async def browser_task(url: str, target: str):
    await env.call_tool("navigate", url=url)
    answer = yield f"Find '{target}' on the page and click it."

    screenshot = await env.call_tool("screenshot")
    found = target.lower() in screenshot.lower()
    yield 1.0 if found else 0.0
```

## Scenario Options

The `@env.scenario()` decorator accepts several options:

```python
@env.scenario(
    name="my-task",                           # Override name (default: function name)
    description="Tests the agent's ability...", # Human-readable description
    required_env_vars=["OPENAI_API_KEY"],     # Required env vars (platform checks these)
    exclude_tools=["browser_*", "screenshot"], # Hide tools from agent (fnmatch patterns)
    exclude_sources=["browser"],               # Hide all tools from a connection
    allowed_tools=["browser_navigate"],        # Rescue specific tools after exclusion
)
async def my_task(query: str):
    answer = yield f"Complete: {query}"
    yield 1.0
```

### Tool exclusion

Use `exclude_tools` and `exclude_sources` to control which tools the agent sees during a scenario. The environment can still call excluded tools in its own code -- they're only hidden from the agent's tool list.

Use `allowed_tools` to rescue specific tools back after broad exclusions.

## Running Scenarios

### With create_agent

```python
import hud
from hud.agents import create_agent

task = env("find-answer", question="What is 2+2?", expected="4")
agent = create_agent("claude-sonnet-4-5")

async with hud.eval(task) as ctx:
    result = await agent.run(ctx, max_steps=10)

print(f"Reward: {result.reward}")
```

### With a custom loop

```python
async with hud.eval(task) as ctx:
    # ctx.prompt contains the first yield's output
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": ctx.prompt}],
        tools=ctx.as_openai_chat_tools(),
    )

    # Handle tool calls, get final answer
    answer = process_response(response)
    await ctx.submit(answer)

# ctx.reward contains the second yield's output
print(f"Reward: {ctx.reward}")
```

### A/B testing with variants

```python
task = env("find-answer", question="What is 2+2?", expected="4")

async with hud.eval(task, variants={"model": ["gpt-4o", "claude-sonnet-4-5"]}) as ctx:
    response = await client.chat.completions.create(
        model=ctx.variants["model"],
        messages=[{"role": "user", "content": ctx.prompt}],
    )
    await ctx.submit(response.choices[0].message.content or "")
```

## Patterns

### Environment state checking

Score based on what actually happened, not just the agent's answer:

```python
@env.scenario("file-edit")
async def file_edit(filename: str, expected_content: str):
    answer = yield f"Edit {filename} to contain the correct implementation."

    actual = open(filename).read()
    yield 1.0 if expected_content in actual else 0.0
```

### Partial credit

Give partial credit for partially correct answers:

```python
@env.scenario("multi-item")
async def multi_item(items: list[str]):
    answer = yield f"Find all of these items: {', '.join(items)}"

    found = sum(1 for item in items if item.lower() in answer.lower())
    yield found / len(items)
```

### No second yield

If you omit the second yield, the scenario defaults to a reward of 1.0 (success):

```python
@env.scenario("demo")
async def demo(task: str):
    yield f"Complete: {task}"
    # No second yield -- defaults to reward=1.0, done=True
```
