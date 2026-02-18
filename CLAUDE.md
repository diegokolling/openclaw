# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw is a multi-channel AI gateway written in TypeScript (ESM-only). It routes AI conversations across messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, Teams, etc.) and integrates with multiple LLM providers (Anthropic, OpenAI, HuggingFace, Ollama, etc.). It runs on Node 22+ and is packaged as a CLI (`openclaw`), a gateway server, and native apps (macOS/iOS/Android).

## Build, Test, and Lint Commands

```bash
pnpm install                # Install all dependencies (pnpm 10+)
pnpm build                  # Full build (tsdown + UI + plugin SDK + hooks)
pnpm check                  # Format check + type check + lint (run before commits)
pnpm tsgo                   # TypeScript type checking only
pnpm format:check           # Check formatting (oxfmt)
pnpm format                 # Fix formatting (oxfmt --write)
pnpm lint                   # Run oxlint (type-aware)
pnpm lint:fix               # Fix lint issues + reformat

# Testing
pnpm test                   # Unit tests (vitest, parallelized)
pnpm test:fast              # Quick unit tests only (vitest.unit.config.ts)
pnpm test:coverage          # Unit tests + V8 coverage report
pnpm test:e2e               # End-to-end tests
pnpm test:live              # Tests against real LLM APIs (needs OPENCLAW_LIVE_TEST=1)
pnpm test:all               # Full suite: lint + build + test + e2e + live + docker

# Development
pnpm dev                    # Run CLI in dev mode (tsx)
pnpm openclaw <cmd>         # Run CLI commands in dev
pnpm gateway:dev            # Dev gateway (skips channels)
```

Run a single test file: `pnpm vitest run src/path/to/file.test.ts`

## Architecture

**Entry flow:** `openclaw.mjs` -> `src/entry.ts` -> `src/index.ts` -> Commander.js CLI program

**Key modules in `src/`:**
- `cli/` - CLI command wiring and program registration (Commander.js)
- `commands/` - Command handler implementations (agent, onboard, doctor, config, etc.)
- `gateway/` - WebSocket + HTTP server (Express 5 + ws); manages channels, agents, hooks, cron
- `channels/` - Shared channel infrastructure and routing
- `telegram/`, `discord/`, `slack/`, `signal/`, `imessage/`, `web/` - Built-in channel integrations
- `agents/` - Agent execution, spawning, message routing, subagent support
- `providers/` - LLM provider adapters
- `plugin-sdk/` - Plugin SDK exported as `openclaw/plugin-sdk`
- `config/` - YAML-based configuration management (`~/.openclaw/`)
- `infra/` - Infrastructure utilities (binaries, env, errors, ports)
- `media/` - Media processing pipeline
- `memory/` - Vector embedding/semantic search systems

**Extensions (`extensions/`):** Workspace packages providing additional channel plugins (Matrix, MS Teams, Mattermost, voice-call, etc.) and features (LanceDB memory, Lobster agent subsystem). Keep extension-only deps in the extension's `package.json`, not the root.

**Skills (`skills/`):** Bundled tool packages (GitHub, 1Password, Apple Reminders, etc.).

**Native apps (`apps/`):** macOS (Swift/SwiftUI), iOS (Swift), Android (Gradle). Connect via WebSocket protocol.

**Web UI (`ui/`):** Lit.js 3.x web components, built with Vite.

## Key Patterns

- **Dependency injection** via `createDefaultDeps()` factory pattern
- **Channel plugin interface:** each channel exposes adapters (Auth, Messaging, Outbound, Status, Security, Commands, Threading)
- **Plugin runtime:** extensions are npm workspace packages; runtime resolves `openclaw/plugin-sdk` via jiti alias. Use `devDependencies`/`peerDependencies` for `openclaw` (not `workspace:*` in `dependencies`)
- **Tool schemas:** avoid `Type.Union`/`anyOf`/`oneOf`/`allOf` in tool input schemas. Use `stringEnum`/`optionalStringEnum` for string lists and `Type.Optional(...)` instead of `... | null`
- **UI decorators:** the control UI uses Lit with **legacy** decorators (`@state() foo = "bar"`). Root tsconfig has `experimentalDecorators: true` and `useDefineForClassFields: false`
- **CLI progress:** use `src/cli/progress.ts` (osc-progress + @clack/prompts spinner); don't hand-roll spinners

## Coding Style

- TypeScript ESM. Strict typing; avoid `any`.
- Formatting/linting via Oxlint and Oxfmt; run `pnpm check` before commits.
- Keep files under ~500 LOC; split/refactor when it improves clarity.
- Brief comments only for tricky/non-obvious logic.
- Naming: **OpenClaw** for product/headings; `openclaw` for CLI/package/paths/config keys.

## Testing Conventions

- Framework: Vitest with V8 coverage (70% lines/branches/functions threshold).
- Tests colocated as `*.test.ts`; e2e as `*.e2e.test.ts`; live as `*.live.test.ts`.
- Test timeout: 120s. Max workers: 16 (don't increase).
- Coverage anchored to `src/` only; extensions/apps excluded from thresholds.

## Commit Conventions

- Use `scripts/committer "<msg>" <file...>` for scoped commits (avoids manual `git add`/`git commit`).
- Conventional commits: `<type>(<scope>): <description>` (e.g., `fix(telegram): include DM topic thread id in replies`).
- Group related changes; avoid bundling unrelated refactors.
- Changelog: user-facing changes only; pure test additions don't need entries.

## Multi-Channel Awareness

When refactoring shared logic (routing, allowlists, pairing, command gating, onboarding, docs), consider **all** channels:
- Core: `src/telegram`, `src/discord`, `src/slack`, `src/signal`, `src/imessage`, `src/web`, `src/channels`, `src/routing`
- Extensions: `extensions/*` (msteams, matrix, zalo, voice-call, etc.)
- When adding channels/extensions, update `.github/labeler.yml` and create matching GitHub labels.

## Docs (Mintlify)

- Internal links: root-relative, no `.md`/`.mdx` suffix (e.g., `[Config](/configuration)`).
- Anchors: `[Hooks](/configuration#hooks)`.
- Avoid em dashes and apostrophes in headings (breaks Mintlify anchors).
- Content must be generic: no personal device names/hostnames; use placeholders.

## Version Locations

When bumping versions, update: `package.json`, `apps/android/app/build.gradle.kts`, `apps/ios/Sources/Info.plist`, `apps/macos/Sources/OpenClaw/Resources/Info.plist`, `docs/install/updating.md`. Do **not** touch `appcast.xml` unless cutting a macOS Sparkle release.

## Dependencies

- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, vendored changes) requires explicit approval.
- Never update the Carbon dependency.
- Keep both pnpm and Bun install paths working (`pnpm-lock.yaml` + Bun patching in sync).
