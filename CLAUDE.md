# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Feishu/Lark (飞书) plugin for [OpenClaw](https://github.com/openclaw/openclaw).

It provides:
- Feishu channel integration (receive events, route messages, send replies/media/cards)
- Feishu tool integrations (`feishu_doc`, `feishu_app_scopes`, `feishu_wiki`, `feishu_drive`, `feishu_perm`, `feishu_bitable`, `feishu_task_*`)

## Development

This is a TypeScript ESM project. No build step is required - the plugin is loaded directly as `.ts` files by OpenClaw.

```bash
# Install dependencies
npm install

# Type check
npx tsc --noEmit
```

## Architecture

### Entry Point
- `index.ts` - Plugin registration entry
  - Registers Feishu channel plugin (`api.registerChannel`)
  - Registers Feishu tools (doc/wiki/drive/perm/bitable)
  - Exports public helpers (`monitorFeishuProvider`, send/media/reaction/mention utilities)

### Core Modules (src/)

**Channel Runtime (Connection + Events + Replies):**
- `channel.ts` - Main `ChannelPlugin` implementation (capabilities, config/account lifecycle, onboarding, directory, outbound, status)
- `client.ts` - Feishu SDK client factory (REST + WebSocket client + event dispatcher)
- `monitor.ts` - Event monitor bootstrap
  - Supports both `websocket` and `webhook` modes
  - Supports multi-account startup (all enabled accounts)
- `bot.ts` - Incoming event handler
  - Message deduplication
  - DM/group policy checks
  - Mention parsing and forward-mention logic
  - Inbound media/resource resolution
  - Optional dynamic agent creation for DMs
- `reply-dispatcher.ts` - Agent reply dispatch (render mode `auto/raw/card`, chunking, typing indicator integration)
- `outbound.ts` - `ChannelOutboundAdapter` implementation for text/media delivery

- `send.ts` - Text messages, interactive cards, message editing
- `media.ts` - Upload/download images and files, inbound media resource fetch

**Configuration, Accounts, Policy:**
- `config-schema.ts` - Zod schema definitions for Feishu config
  - Includes single-account + multi-account config
  - Includes tools toggles and dynamic agent creation config
- `accounts.ts` - Account resolution and merged config logic (top-level defaults + account overrides)
- `policy.ts` - DM/group allowlist and mention policy resolution
- `tools-config.ts` - Default tool switches (`doc/wiki/drive/scopes` on, `perm` off)
- `types.ts` - TypeScript types inferred from config/schema

**Feishu Tool Modules:**
- `docx.ts` / `doc-schema.ts` - Feishu document helpers and tool registration (`feishu_doc`, `feishu_app_scopes`)
- `wiki.ts` / `wiki-schema.ts` - Wiki space/node operations (`feishu_wiki`)
- `drive.ts` / `drive-schema.ts` - Drive file/folder operations (`feishu_drive`)
- `perm.ts` / `perm-schema.ts` - Drive permission member operations (`feishu_perm`)
- `bitable.ts` - Bitable tools entry export
- `bitable-tools/` - Bitable modular implementation:
  - `register.ts` tool registration + shared wrapper (`feishu_bitable_*`)
  - `schemas.ts` tool parameter schemas
  - `actions.ts` Feishu Bitable API operations
  - `meta.ts` URL parsing + app/table metadata resolution
  - `common.ts` shared types/formatting/error helpers
- `task-tools/` - Task v2 API implementation:
  - `register.ts` - Tool registration (`feishu_task_create`, `feishu_task_get`, `feishu_task_update`, `feishu_task_delete`)
  - `schemas.ts` - Task parameter schemas
  - `actions.ts` - Task API operations
  - `common.ts` - Shared types/error helpers

**Supporting Utilities:**
- `targets.ts` - Normalize `user:xxx`/`chat:xxx` target formats
- `directory.ts` - User/group lookup
- `reactions.ts` - Emoji reactions API
- `typing.ts` - Typing indicator (emoji-based)
- `probe.ts` - Bot health check
- `mention.ts` - Mention extraction/formatting and mention-forward helpers
- `dynamic-agent.ts` - Auto-create dedicated DM agents (workspace + binding updates)
- `onboarding.ts` - Channel onboarding adapter
- `runtime.ts` - Plugin runtime holder/getter
- `dedup.ts` - Message deduplication (TTL 30min, max 1000 entries)

**Tool Execution Infrastructure:**
- `tools-common/tool-context.ts` - AsyncLocalStorage-based context propagation for account/session tracking
- `tools-common/tool-exec.ts` - `withFeishuToolClient` wrapper for account resolution and tool toggle enforcement
- `tools-common/feishu-api.ts` - Shared API helpers

### Message Flow

1. `monitor.ts` resolves enabled account(s) and starts event listener in `websocket` or `webhook` mode.
2. Feishu event dispatcher routes `im.message.receive_v1` to `bot.ts`.
3. `bot.ts` validates policies, parses mentions/content, optionally resolves media resources.
4. `bot.ts` dispatches to OpenClaw runtime using `reply-dispatcher.ts`.
5. `reply-dispatcher.ts` chooses render path (`raw` text vs markdown card) and sends via `send.ts`.
6. For outbound tool/API calls, `outbound.ts` sends text/media through `send.ts` and `media.ts`.

### Critical Dispatch Pattern

When dispatching inbound messages in `bot.ts`, **always use `core.channel.reply.dispatchInboundMessage`** instead of `dispatchReplyFromConfig`:

```typescript
// Correct - waits for async delivery (including streaming) to complete
await core.channel.reply.dispatchInboundMessage({
  ctx: ctxPayload,
  cfg,
  dispatcher,
  replyOptions,
});

// Incorrect - returns immediately, causing premature cleanup
await core.channel.reply.dispatchReplyFromConfig({...});
```

`dispatchInboundMessage` wraps the call in `withReplyDispatcher` which:
1. Calls `dispatcher.markComplete()` to release the reservation
2. Awaits `dispatcher.waitForIdle()` for all async delivery to complete
3. Only then returns, allowing `markDispatchIdle()` to safely clean up

Using `dispatchReplyFromConfig` directly causes `markDispatchIdle()` to be called too early, triggering `onIdle` and closing streaming before content is delivered.

### Key Configuration Options

| Option | Description |
|--------|-------------|
| `connectionMode` | `websocket` (default) or `webhook` |
| `webhookPath` / `webhookPort` | Webhook callback path/port when `connectionMode=webhook` |
| `accounts` | Multi-account config map; account config overrides top-level defaults |
| `dmPolicy` | `pairing` / `open` / `allowlist` |
| `allowFrom` | DM allowlist (required to include `"*"` when `dmPolicy=open`) |
| `groupPolicy` | `open` / `allowlist` / `disabled` |
| `groupAllowFrom` | Group sender allowlist |
| `requireMention` | Require @bot in groups (default: true) |
| `topicSessionMode` | Group topic-thread isolation (`disabled` / `enabled`) |
| `renderMode` | Reply render mode: `auto` / `raw` / `card` |
| `streaming` | Enable streaming card updates (default: false) |
| `dynamicAgentCreation` | Auto-create isolated DM agents/workspaces |
| `tools` | Tool category switches (`doc`, `wiki`, `drive`, `perm`, `scopes`, `task`) |
| `mediaMaxMb` | Max inbound/outbound media size limit |

### Defaults and Behavior Notes

- `connectionMode` defaults to `websocket`.
- `dmPolicy` defaults to `pairing`.
- `groupPolicy` defaults to `allowlist`.
- `requireMention` defaults to `true`.
- `renderMode` behaves as `auto` when unset at runtime.
- `streaming` defaults to `false` (rate limit concerns).
- Tool defaults:
  - `doc: true`
  - `wiki: true`
  - `drive: true`
  - `perm: false` (sensitive)
  - `scopes: true`
  - `task: true`

### Feishu SDK Usage

Uses `@larksuiteoapi/node-sdk`. Key APIs:
- `client.im.message.create/reply` - Send messages
- `client.im.message.get/patch` - Read and edit messages
- `client.im.messageResource.get` - Download media from messages
- `client.im.image.create` - Upload images
- `client.im.file.create` - Upload files
- `client.docx.*` - Document read/write and markdown conversion
- `client.wiki.*` - Wiki space/node operations
- `client.drive.*` - Drive file and permission operations
- `client.bitable.*` - Bitable metadata/record operations
- `client.task.*` - Task v2 API operations
- `WSClient` + `Lark.adaptDefault(...)` - WebSocket and webhook event delivery

## Tool Development Pattern

When adding new Feishu tools:

1. **Define schema** in a `*-schema.ts` file using `@sinclair/typebox`
2. **Implement actions** in a `*.ts` file that use the Feishu SDK client
3. **Register tool** using `withFeishuToolClient` wrapper which:
   - Resolves the correct account from AsyncLocalStorage context (message-driven) or default
   - Enforces per-account tool toggles
   - Creates the Feishu SDK client
4. **Use `runWithFeishuToolContext`** in `bot.ts` when dispatching to propagate account context

```typescript
// Pattern from tools-common/tool-exec.ts
export async function withFeishuToolClient<T>(params: {
  api: OpenClawPluginApi;
  toolName: string;
  requiredTool?: FeishuToolFlag;
  run: (args: { client: Lark.Client; account: ResolvedFeishuAccount }) => Promise<T>;
}): Promise<T>
```

## Streaming Card Implementation

Streaming cards use Feishu's CardKit API (`/cardkit/v1/cards`):
- `streaming-card.ts` - `FeishuStreamingSession` class manages card lifecycle
- Requires `streaming: true` in account config
- Only works with `renderMode: "card"` or `renderMode: "auto"`
- Throttles updates to 100ms to avoid rate limits
