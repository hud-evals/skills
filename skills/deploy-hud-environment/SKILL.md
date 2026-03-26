---
name: deploy-hud-environment
description: Deploy HUD environments to the platform, rebuild linked environments, and manage deploy-time inputs such as .env values, build args, and build secrets. Use when asked to deploy a HUD environment, push a local environment to HUD, rebuild an existing deployed env, or troubleshoot the current hud build / hud deploy flow.
license: MIT
metadata:
  author: hud-ai
  version: "1.1.0"
---

# Deploy a HUD Environment

Deploy your environment so agents can run against it in parallel on HUD, with traces, scenarios, tools, and build logs all managed by the platform.

## What Deploy Expects

The current deploy flow is directory-based. HUD expects an environment directory with a `Dockerfile.hud` or `Dockerfile`. The recommended starting point is `hud init`, but older/module-driven repos also work as long as they build correctly.

Common layouts include:

```text
my-env/
├── Dockerfile
├── pyproject.toml
├── tasks.json
├── controller/
└── environment/   # optional
```

or a simpler SDK-first repo such as:

```text
my-env/
├── env.py
├── pyproject.toml
└── Dockerfile
```

The important requirement is not the exact layout. The important requirement is that the directory builds into an MCP-capable environment image and passes HUD validation.

## Recommended Flow

Start with a local build:

```bash
hud build
```

Then deploy:

```bash
hud deploy
```

What each command does:

- `hud build` validates the environment locally, analyzes the MCP surface, and writes `hud.lock.yaml`.
- `hud deploy` packages the build context, uploads it, triggers a remote build on HUD infrastructure, streams logs, and links the local directory to the deployed environment.

## Environment Variables And Build Inputs

Current deploy behavior is more opinionated than the older minimal workflow:

- If a local `.env` file exists and you did not pass explicit env inputs, `hud deploy` may prompt whether to include those values in the deploy. The preference is saved for later deploys.
- Use `--no-env` to skip `.env` loading for a deploy.
- Use `-e KEY=VALUE` / `--env KEY=VALUE` to pass explicit runtime env vars.
- Use `--env-file PATH` to load env vars from a specific file.
- Use `--build-arg KEY=VALUE` for Docker build args.
- Use `--secret id=NAME,env=ENV_VAR` or `--secret id=NAME,src=./path` for Docker build secrets.

Examples:

```bash
hud deploy . -e API_KEY=secret
hud deploy . --no-env
hud deploy . --build-arg NODE_ENV=production
hud deploy . --secret id=GITHUB_TOKEN,env=GITHUB_TOKEN
hud deploy . --secret id=SSH_KEY,src=~/.ssh/deploy_key
```

## Rebuilds, Linking, And Names

`hud deploy` keeps local linkage and config for the environment directory. In the current flow:

- A linked environment can be rebuilt by running `hud deploy` again in the same directory.
- HUD stores local project config in `.hud/config.json` and can migrate older `.hud/deploy.json` state.
- On rebuilds, the deploy flow resolves the actual deployed environment name from the platform.
- If the requested name conflicts with an existing team environment, the CLI can offer to link to the existing environment and rebuild it instead of failing blindly.

Practical implication: deploy is no longer just "fire and forget". It remembers the relationship between the local folder and the remote environment.

## Troubleshooting Mindset

When deploy behavior is surprising, check these first:

- Does the directory contain `Dockerfile.hud` or `Dockerfile`?
- Does `hud build` pass locally and produce `hud.lock.yaml`?
- Are required runtime env vars being loaded from `.env`, explicit `--env` flags, or not at all?
- Are build-only inputs passed with `--build-arg` / `--secret` instead of runtime env vars?
- Is the directory already linked to an existing remote environment through `.hud/config.json`?

If the environment name differs between local code and the deployed environment, current deploy flows may warn and help reconcile that mismatch instead of silently renaming things.

## Result

After a successful deploy:

- the environment appears on HUD
- scenarios and tools become available on the platform
- build logs and status are visible remotely
- the local directory is linked for future rebuilds

Use the deployed environment for remote evals, task creation, and platform-managed runs.
