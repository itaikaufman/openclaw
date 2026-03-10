# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Full project conventions (commit rules, release process, channel list, agent-specific rules) are in `AGENTS.md`. Read it before any non-trivial change.

## Commands

```bash
pnpm install                   # install deps (bun install also works)
pnpm build                     # type-check + compile → dist/
pnpm tsgo                      # TypeScript checks only
pnpm check                     # lint + format (oxlint + oxfmt)
pnpm format:fix                # auto-fix formatting
pnpm test                      # run all tests (vitest)
pnpm test:coverage             # tests + coverage report

# Run a single test file
pnpm test src/agents/skills.ts

# Low-memory run (WSL2 / small VMs)
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test
```

## Local Docker Images

This fork builds two images:

**`openclaw:local`** — standard image, built from the multi-stage `Dockerfile`:
```bash
docker build -t openclaw:local .
# Slim variant (smaller, bookworm-slim base):
docker build --build-arg OPENCLAW_VARIANT=slim -t openclaw:local-slim .
```

**`openclaw:readers`** — extends `openclaw:local` with Python document libraries (docx, pdf, xlsx, pptx, bs4):
```bash
docker build -t openclaw:local .
docker build -f Dockerfile.readers --build-arg OPENCLAW_IMAGE=openclaw:local -t openclaw:readers .
```

`docker-compose.yml` defaults to `openclaw:readers` (`OPENCLAW_IMAGE` overrides it). Additional env vars (all optional with empty defaults):
- `GOG_KEYRING_PASSWORD` — gogcli keyring password
- `GOGCLI_CONFIG_DIR` — host path to gogcli config dir (defaults to `~/.config/gogcli`)

### Dockerfile multi-stage structure

| Stage | Purpose |
|---|---|
| `ext-deps` | Extracts `package.json` from opted-in extensions; isolated to avoid cache busting on source changes |
| `build` | Installs Bun + pnpm, compiles TypeScript + UI; heavyweight, never ships |
| `runtime-assets` | Prunes dev deps and strips `.d.ts`/`.map` files from `build` |
| `base-default` / `base-slim` | Selectable runtime base (full bookworm or bookworm-slim) |
| **runtime** (final) | Fresh base — copies only pruned assets; installs system tools, Docker CLI (optional), and gogcli |

`gogcli` v0.11.0 is installed in the **runtime stage** at `/usr/local/bin/gog` (lines 205–223), after `curl` is available and before `USER node`. Version and sha256 hashes are parameterised as `ARG` values — update them together when bumping.

## Code Architecture

### Message flow

```
Channel (WhatsApp / Telegram / Discord / ...)
  → Gateway HTTP server       src/gateway/
    → ACP server              src/acp/server.ts
      → AcpSessionManager     src/acp/control-plane/manager.core.ts  (singleton)
        → agent spawn         src/agents/acp-spawn.ts
          → pi-embedded-runner  src/agents/pi-embedded-runner/
            → model provider  src/providers/
```

### ACP (Agent Control Plane) — `src/acp/`

- `server.ts` — WebSocket/HTTP transport, dispatches turns to `AcpSessionManager`
- `control-plane/manager.core.ts` — session create / run / close, identity reconciliation
- `persistent-bindings.ts` — maps channel contacts to specific agent configs
- `translator.ts` — converts channel messages → ACP turn format
- `policy.ts` — per-session rate limits and access policy

### Skills loading — `src/agents/skills/`

`workspace.ts::loadWorkspaceSkillEntries` merges four sources in ascending precedence:

1. Bundled skills (`skills/` dir in the package, resolved by `bundled-dir.ts`)
2. `~/.openclaw/skills` (managed/local — shared across all agents)
3. `<workspace>/skills` (per-agent — highest precedence)
4. Plugin skill dirs (`plugin-skills.ts`) + `skills.load.extraDirs` from config

Gating (`requires.bins`, `requires.env`, `requires.config`) is evaluated in `config.ts::shouldIncludeSkill()`. The eligible set is snapshotted at session start; changes hot-reload via the skills watcher on `SKILL.md` saves.

### Built-in channels vs extensions

Built-in channels (`src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`, `src/imessage/`, `src/web/`, `src/line/`) are always present. Extension channels (`extensions/*/`) are optional npm workspace packages loaded at runtime via `src/plugins/`. Both implement the same interface from `src/channels/`. When refactoring routing, pairing, allowlists, or onboarding, both sets must be updated.

### Routing and pairing

`src/routing/` decides which agent handles an incoming message based on persistent ACP bindings, pairing state, and per-channel rules. `src/pairing/` handles the initial device-to-channel linking flow. `src/auto-reply/` handles rule-based replies that bypass the agent entirely.
