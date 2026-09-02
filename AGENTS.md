# AGENTS.md

This file provides guidance to AI agents working in this repository.

## Overview

`pi-azure-foundry` is a [pi](https://pi.dev) extension that connects to Azure AI
Foundry, auto-discovers chat-capable deployments, and registers them as models in
pi. Supports API key and Azure identity (DefaultAzureCredential) authentication.

Single source file: `src/index.ts` (mirrored to
`.pi/extensions/azure-foundry/index.ts` for local development).

## Build Commands

```bash
npm run build        # compile src/ → dist/
npm run type-check   # tsc --noEmit — run this after every code change
npm run dev          # tsc --watch
```

There is no test runner. **Always run `npm run type-check` before committing.**

## Commit Messages

**REQUIRED**: Before `git commit`, spawn the
`pi-claude-marketplace-dlr-dev-commit-message-generator` agent to generate the
commit message. Do not write it yourself.

- Conventional Commits format: `<type>(<scope>): <subject>`
- Subject ≤50 chars, body wrapped at 72 chars, imperative mood
- No AI attribution

## Git Remotes & Branches

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `git@github.com:pureport/pi-azure-foundry.git` | Fork — push branches and open PRs here |
| `upstream` | `https://github.com/nquandt/pi-azure-foundry.git` | Original repo — pull updates from here |

| Branch | Tracks | Purpose |
|--------|--------|---------|
| `main` | `origin/main` | Primary working branch |
| `upstream-tracking` | `upstream/main` | Upstream mirror — do not commit work here |

### Create a feature branch

```bash
git fetch upstream
git checkout -b <branch-name> upstream/main
git push -u origin <branch-name>
```

### Sync upstream-tracking

```bash
git fetch upstream
git checkout upstream-tracking && git rebase upstream/main
git push origin upstream-tracking && git checkout main
```

PRs go from `origin/<branch>` → `nquandt/pi-azure-foundry:main`.

## OpenSpec

Behavioral specs live in `openspec/specs/` and are the source of truth.

| Domain | Spec file |
| -------- | ----------- |
| `config` | `openspec/specs/config/spec.md` |
| `auth` | `openspec/specs/auth/spec.md` |
| `deployment-discovery` | `openspec/specs/deployment-discovery/spec.md` |
| `streaming` | `openspec/specs/streaming/spec.md` |
| `extension` | `openspec/specs/extension/spec.md` |

Changes live in `openspec/changes/` (gitignored except `archive/`).

## Key Architecture Notes

- **Routing**: Anthropic deployments → `/anthropic/v1/messages`; all others →
  `/openai/deployments/{id}/chat/completions?api-version=2024-10-21`
- **Auth headers**: OpenAI route — `api-key: <key>` for api-key auth,
  `Authorization: Bearer` for azure-identity; Anthropic route — always
  `Authorization: Bearer` regardless of auth type
- **Token limit param**: GPT-5 / o-series → `max_completion_tokens`; others →
  `max_tokens` (inferred from model name or `MODEL_DEFAULTS`)
- **Model `id`**: always the deployment name, never `modelName`
- **Config search order**: `<cwd>/azure-foundry.config.json` →
  `~/.pi/azure-foundry.config.json`
