# HUD Skills

A collection of skills for AI coding agents that work with the [HUD](https://hud.ai) platform. Skills teach agents how to build, test, and deploy HUD environments.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### write-hud-environment

Create HUD environments that expose tools and evaluation scenarios for AI agents. Covers the full development workflow from scaffolding to local testing.

**Use when:**
- Creating a new HUD environment
- Adding tools for agents to call
- Defining evaluation scenarios
- Connecting existing APIs, FastAPI apps, or Docker images
- Testing environments locally with `hud dev`

**Topics covered:**
- Project scaffolding (`hud init`)
- Tool definition (`@env.tool()`)
- Scenario definition (`@env.scenario()` two-yield pattern)
- Connecting existing services (`connect_fastapi`, `connect_hub`, `connect_image`, `connect_openapi`)
- Local iteration (`hud dev`, `hud.eval()`)
- Project structure (`env.py`, `pyproject.toml`, `Dockerfile`)

### deploy-hud-environment

Deploy HUD environments to the platform for parallel evaluation and training at scale.

**Use when:**
- "Deploy my environment"
- "Push this to HUD"
- "Run evals at scale"
- Setting up remote environments for team use

**Topics covered:**
- Dockerfile and build configuration
- `hud build` and `hud deploy` workflow
- GitHub-based deployment
- Connecting to deployed environments via `connect_hub`

## Installation

```bash
npx skills add hud-evals/hud-skills
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**
```
Create a HUD environment for my search API
```
```
Add an evaluation scenario that tests retrieval accuracy
```
```
Deploy my environment to HUD
```

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for the agent
- `references/` - Supporting documentation (optional)

## Resources

- [HUD Python SDK](https://github.com/hud-evals/hud-python)
- [HUD Documentation](https://docs.hud.ai)
- [HUD Platform](https://hud.ai)

## License

MIT
