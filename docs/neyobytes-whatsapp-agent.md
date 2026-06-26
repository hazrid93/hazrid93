# Neyobytes WhatsApp Agent — Architecture

> **Private project** — source code is not public. This document is hosted in the [`hazrid93/hazrid93`](https://github.com/hazrid93/hazrid93) profile repository so visitors can understand the architecture without needing repo access.

A multi-tenant WhatsApp Bot SaaS platform. Users sign up, create AI-powered WhatsApp bots (via QR-code pairing or WhatsApp Business API), configure behaviour through a dashboard, and manage subscriptions/billing through Stripe.

| | |
|---|---|
| **Backend** | Node.js 18+ · Express 5 · Socket.IO · ES Modules |
| **WhatsApp** | `whatsapp-web.js` (QR pairing) · WhatsApp Business Cloud API |
| **Frontend** | Next.js 14 (App Router) · TypeScript · Supabase Auth |
| **Database** | Supabase (Postgres + RLS) · service-role + anon clients |
| **AI / LLM** | OpenRouter · OpenAI · Ollama · Groq · ZAI · plus vision delegation |
| **Payments** | Stripe (subscriptions, wallet credits, webhook reconciliation) |
| **Process** | PM2 (`ecosystem.config.cjs`) · Nginx reverse proxy |

---

## High-Level Architecture

```mermaid
flowchart TB
  subgraph Clients
    WA[WhatsApp Users]
    WEB["Dashboard / Web Chat<br/>Next.js 14"]
    ADMIN[Admin Panel]
  end

  subgraph Nginx["Nginx reverse proxy"]
    NX["nginx.conf / nginx-unified.conf"]
  end

  subgraph App["Node.js + Express 5  (src/app.js)"]
    API["REST API  (src/api/routes)<br/>users · bots · stripe · wallet · admin · chat"]
    MW["Middleware<br/>auth · admin · web-chat-auth · error"]
    IO["Socket.IO server<br/>/ws — real-time QR, status, messages"]
    BOTMGR["BotManager<br/>src/bot/manager.js"]
  end

  subgraph BotLayer["Bot clients  (src/bot/clients)"]
    WEBJS["WebJSClient<br/>whatsapp-web.js · QR + LocalAuth"]
    BIZ["BusinessAPIClient<br/>Meta Cloud API"]
  end

  subgraph BotServices["Bot services  (src/bot/services)"]
    MSGPROC["MessageProcessor<br/>FAQ · auto-reply · menu · CRM"]
    LLMSVC["LLMService<br/>multi-provider routing + fallback"]
    CMD["CommandHandler<br/>!ai !image !clear !info !provider"]
  end

  subgraph Data["Data layer"]
    SUPA["Supabase Admin<br/>service-role, bypasses RLS"]
    PG["Direct PG pool<br/>web-chat history (src/database.js)"]
  end

  subgraph External["External services"]
    LLM["LLM Providers<br/>OpenRouter · OpenAI · Ollama · Groq · ZAI"]
    VISION["Vision model<br/>image understanding"]
    STRIPE["Stripe<br/>payments + webhooks"]
    META["Meta Webhook<br/>WhatsApp Business events"]
  end

  WA --> NX
  WEB --> NX
  ADMIN --> NX
  NX --> API
  NX --> IO

  API --> MW --> SUPA
  API --> BOTMGR
  IO --> BOTMGR

  BOTMGR --> WEBJS
  BOTMGR --> BIZ
  BOTMGR -.event.-> IO

  WEBJS --> WA
  BIZ --> META
  META --> API

  WEBJS --> MSGPROC
  BIZ --> MSGPROC
  MSGPROC --> CMD
  MSGPROC --> LLMSVC
  LLMSVC --> LLM
  LLMSVC --> VISION

  MSGPROC --> SUPA
  CMD --> PG
  API --> STRIPE
```

---

## Startup & Bot Lifecycle

```mermaid
sequenceDiagram
  participant App as src/app.js
  participant Cfg as validateConfig()
  participant DB as initDatabase()
  participant Server as startServer()
  participant BM as BotManager
  participant IO as Socket.IO

  App->>Cfg: check SUPABASE_URL, keys,<br/>ADMIN_*, JWT_SECRET
  alt missing/placeholder
    Cfg-->>App: process.exit(1)
  end
  App->>DB: initDatabase(DATABASE_URL)<br/>PG pool for web-chat history
  App->>Server: startServer() — Express + helmet + cors
  App->>BM: setSocketIO(io)
  App->>BM: startAllBots()<br/>resume previously-connected bots
  loop every interval
    BM->>BM: startHeartbeatMonitoring()
  end
  Note over App: SIGTERM/SIGINT → stopAllBots → server.close → exit
```

---

## Incoming WhatsApp Message — Full Flow

```mermaid
sequenceDiagram
  participant WA as WhatsApp User
  participant Client as WebJSClient /<br/>BusinessAPIClient
  participant MP as MessageProcessor
  participant DB as Supabase
  participant LLM as LLMService
  participant Provider as LLM Provider
  participant IO as Socket.IO → Dashboard

  WA->>Client: inbound message
  Client->>MP: processMessage(chatId, content, sender, type)

  MP->>DB: upsert contact (CRM)
  MP->>DB: isFirstMessage(chatId)?
  alt first message + welcome_enabled
    MP-->>WA: send welcome_message
  end

  MP->>MP: business hours check
  alt outside hours
    MP-->>WA: outside_hours_message
    MP-->>Client: { handled, stopAI: true }
  end

  MP->>MP: human handoff keyword?
  alt matches handoff keyword
    MP->>DB: triggerHumanHandoff
    MP-->>WA: human_handoff_message
  end

  MP->>MP: menu trigger?
  MP->>MP: auto-reply rule match?
  MP->>MP: FAQ match?
  MP->>MP: quick-link match?

  alt any rule handled & stopAI
    MP-->>Client: { handled, response, stopAI }
    Client->>WA: send response
  else no rule / AI continues
    MP-->>Client: { handled: false }
    Client->>LLM: generate(messages, provider, model)
    LLM->>LLM: image present? → _delegateToVision()
    LLM->>Provider: chat/completions
    alt primary fails & fallback set
      LLM->>Provider: retry on fallback provider/model
    end
    Provider-->>LLM: completion
    LLM-->>Client: formatted reply
    Client->>DB: save chat message + context
    Client->>IO: broadcast (web chat)
    Client->>WA: send AI reply
  end
```

---

## Message Processor — Decision Pipeline

`src/bot/services/message-processor.js` runs every inbound message through an ordered pipeline. The first handler that sets `stopAI: true` short-circuits the AI call.

```mermaid
flowchart TD
  IN["Incoming message<br/>chatId · content · sender · type"] --> CRM["1. upsert contact in CRM"]
  CRM --> FIRST{"2. first message?"}
  FIRST --|yes and welcome_enabled|--> WELCOME["send welcome_message<br/>(AI may still respond)"]
  FIRST -- no --> HOURS
  WELCOME --> HOURS{"3. business hours enabled?"}
  HOURS -- closed --> OUT["outside_hours_message<br/>stopAI = true → return"]
  HOURS -- open --> HANDOFF{"4. human handoff keyword?"}
  HANDOFF -- match --> HO["trigger handoff<br/>stopAI = true → return"]
  HANDOFF -- no --> MENU{"5. menu trigger?"}
  MENU -- match --> MENURES["send menu body<br/>stopAI = true → return"]
  MENU -- no --> AUTO{"6. auto-reply rule?"}
  AUTO -- match --> AUTORES["send response<br/>stopAI = per-rule"]
  AUTO --|no and not stopAI|--> FAQ{"7. FAQ match?"}
  FAQ -- match --> FAQRES["send answer<br/>stopAI = true → return"]
  FAQ -- no --> LINKS{"8. quick-link?"}
  LINKS -- match --> LINKRES["send link<br/>AI may add context"]
  LINKS -- no --> AI["9. fall through to LLM<br/>via CommandHandler / client"]
  AI --> SAVE["save message + context to DB"]
  SAVE --> BROADCAST["Socket.IO broadcast to web chat"]
  BROADCAST --> REPLY["send AI reply to WhatsApp"]
```

---

## Multi-Provider LLM Routing

`src/bot/services/llm-service.js` — each bot is configured with a primary provider+model and an optional fallback. Vision-capable models are detected by regex; images are delegated to a dedicated vision endpoint.

```mermaid
flowchart LR
  REQ["chat request<br/>(messages, hasImage?)"] --> CHECK{"_hasImageContent<br/>and model vision-capable?"}
  CHECK -- yes --> VISION["_delegateToVision<br/>visionProvider endpoint"]
  CHECK -- no --> PRIMARY
  VISION --> PRIMARY

  PRIMARY["getApiKey(provider)<br/>getEndpoint(provider)"] --> CALL["POST {endpoint}/chat/completions"]
  CALL --> OK{"200 OK?"}
  OK -- yes --> OUT["return completion"]
  OK --|error and fallback set|--> FB["switch to<br/>fallbackProvider + fallbackModel"]
  FB --> CALL2["POST {fallback endpoint}/chat/completions"]
  CALL2 --> OUT

  subgraph Providers["PROVIDER_ENDPOINTS"]
    P1[openrouter]
    P2[openai]
    P3["ollama — local"]
    P4["zai / bigmodel"]
    P5[synthetic]
    P6["firepass / fireworks"]
    P7[crofai]
    P8[waferai]
    P9[neuralwatt]
    P10[opencodego]
    P11[routingrun]
    P12[chatgpt gateway]
  end
  PRIMARY -.select.-> Providers
```

---

## 🧠 Context Management & VCC Compaction

WhatsApp group chats can run for thousands of messages. Sending all of them to the LLM on every turn would blow past context windows and cost a fortune. The platform solves this with a **View-oriented Conversation Compiler (VCC)** — a two-tier compaction system that gives the LLM access to long group conversation history **without bloating the context window**.

### How it works

The system maintains **two views** of every conversation:

| View | Purpose | Injected into LLM? |
|---|---|---|
| **Full View** | Lossless, line-numbered transcript — the canonical source for recall | No (stored in DB, retrieved on demand) |
| **Brief View** | Compact view with `(file:line-range)` pointers back to Full View | Yes — replaces thousands of messages with a few hundred lines |

The LLM context window receives only: **brief view (older messages) + recent N messages (verbatim)**. The full conversation is never lost — the `conversation_recall` tool lets the LLM fetch specific line ranges or search the Full View on demand.

```mermaid
flowchart TB
  subgraph Raw["Raw conversation  (Postgres)"]
    HISTORY["whatsapp_chat_history<br/>every message stored"]
    CONTEXT["whatsapp_chat_context<br/>all group messages (not just !ai)"]
  end

  subgraph VCC["VCC Compiler  (src/vcc-compiler.js)"]
    JSONL["Export to JSONL"]
    COMPILE["Python VCC compiler<br/>vendor/VCC/skills/conversation-compiler"]
    FULL["Full View (.txt)<br/>lossless line-numbered transcript"]
    BRIEF["Brief View (.min.txt)<br/>compact with (file:line) pointers"]
  end

  subgraph DB["whatsapp_chat_views  (Postgres)"]
    VIEWS["full_view + brief_view<br/>messages_compiled, compiled_at"]
  end

  subgraph LLM["LLM context window"]
    BRIEF_INJ["Brief view as system message<br/>(older messages, compacted)"]
    RECENT["Recent N messages verbatim<br/>(config.keepRecentMessages)"]
    RECALL["conversation_recall tool<br/>fetches from full_view on demand"]
  end

  HISTORY --> JSONL
  CONTEXT --> JSONL
  JSONL --> COMPILE
  COMPILE --> FULL
  COMPILE --> BRIEF
  FULL --> VIEWS
  BRIEF --> VIEWS
  BRIEF --> BRIEF_INJ
  RECENT --> RECENT
  VIEWS -.recall.-> RECALL
  RECALL --> LLM
```

### Compaction flow

```mermaid
sequenceDiagram
  participant WA as WhatsApp Group
  participant Client as Bot Client
  participant DB as Postgres
  participant VCC as VCC Compiler
  participant LLM as LLM Service

  WA->>Client: new message in group
  Client->>DB: saveChatMessage + saveChatContext
  Client->>LLM: generate response with context
  LLM->>DB: getChatViews(chatId)

  alt brief_view exists in DB
    LLM->>DB: getChatHistory(chatId, keepRecentMessages)
    Note over LLM: Load brief_view (system msg)<br/>+ recent N messages only<br/>Skip loading thousands of older msgs
  else no brief_view yet
    LLM->>DB: getChatHistory(chatId, all)
    Note over LLM: Load all messages (first compaction pending)
  end

  LLM->>LLM: send to provider: brief + recent + new msg
  LLM-->>Client: AI reply
  Client->>WA: send reply

  Note over Client: After reply sent — background compaction

  Client->>DB: getChatMessageCount(chatId)
  Client->>DB: getChatViews(chatId)

  alt new messages since last compile >= threshold
    Client->>DB: getChatHistory(chatId, rawCap)
    Client->>VCC: compileConversation(olderMessages)
    VCC->>VCC: export to JSONL
    VCC->>VCC: run Python VCC compiler
    VCC-->>Client: brief_view + full_view
    Client->>DB: saveChatViews(full_view, brief_view)
    Client->>DB: pruneOldMessages(keep only rawCap)
    Note over Client: Next request loads brief_view + recent only
  else below threshold
    Note over Client: Skip — not enough new messages
  end
```

### Why this is awesome

**The problem:** A 2000-message group chat would consume the entire context window of most LLMs — leaving no room for the actual response, and costing 100k+ tokens per turn. Naive summarization loses detail permanently.

**The VCC solution:**

1. **Lossless recall** — nothing is ever deleted from the Full View. The LLM can retrieve exact words from message #47 using `conversation_recall` with line references. Older messages are compacted in the Brief View, but the Full View preserves them byte-for-byte.

2. **Brief View with line pointers** — the compact view replaces verbose messages with truncated summaries, but each entry carries a `(file:line-range)` pointer back to the Full View. The LLM sees "user asked about pricing (truncated from #abc.txt:218-226)" and can fetch lines 218-226 if it needs the full text.

3. **DB-driven compaction** — VCC always compiles from **raw DB messages**, never from a previous brief view. This prevents compaction drift: re-compiling always starts from the original source, so the Full View stays canonical.

4. **Background, non-blocking** — compaction runs via `setImmediate` after the reply is sent. The user never waits for compaction; it happens asynchronously.

5. **Triggered by DB freshness, not memory** — the auto-trigger checks how many new raw DB messages exist since the last compile (`rawMessageCount - compiledCount >= threshold`), not the in-memory context length. This is correct because the in-memory history may already be compacted.

6. **In-memory compile cache** — `compileConversation` caches results by SHA-256 hash of the message content + truncation params. If the same messages are compiled again (e.g. rapid messages), the cache returns instantly.

7. **Group context beyond !ai** — `whatsapp_chat_context` stores **all** group messages (not just `!ai` interactions), so the LLM has awareness of the full group conversation, not just the turns where it was invoked.

### Database tables involved

| Table | Purpose |
|---|---|
| `whatsapp_chat_history` | Every `!ai` interaction (user + assistant), with images and mentions |
| `whatsapp_chat_context` | All group messages (not just AI) — for full group awareness |
| `whatsapp_chat_views` | VCC compiled `full_view` + `brief_view`, `messages_compiled` count |
| `whatsapp_chat_summaries` | Legacy summaries table (replaced by `chat_views`) |
| `whatsapp_group_members` | Group participant names for sender attribution |
| `whatsapp_unlocked_groups` | Groups where the bot is active (AI responds to all messages) |

### Configuration

| Setting | Default | Effect |
|---|---|---|
| `summarizationEnabled` | true | Enable VCC compaction |
| `summarizeAfterMessages` | configurable | Auto-compact threshold (new messages since last compile) |
| `keepRecentMessages` | configurable | N recent messages kept verbatim in context |
| `alwayslogMaxMessagesPerChat` | 1000 | Raw DB row cap per chat (hard cap 50000) |
| `alwayslogContextLimit` | 200 | Group context messages fetched for VCC input |

Manual compaction is also available via the `!compact` command (owner-only), which forces a VCC compilation regardless of the auto-trigger threshold.

---

## API Surface

`src/api/routes/index.js` mounts everything under `/api`:

| Route | Controller | Purpose |
|---|---|---|
| `/api/users` | `users.js` | signup, auth, profile |
| `/api/bots` | `bots.js` + `bot-features.js` | create/list bots, FAQ, auto-replies, menus, contacts, quick-links |
| `/api/stripe` | `stripe.js` | checkout, wallet top-up, **webhook** (raw body) |
| `/api/admin` | `admin.js` | user/plan/pricing/settings management (admin middleware) |
| `/api/messages` | `messages.js` | chat history, context |
| `/api/wallet` | `wallet.js` | credits, transactions |
| `/api/chat` | `web-chat.js` | public web chat interface |
| `/api/whatsapp/webhook/meta` | `meta-webhook.js` | Meta Business webhook (no auth) |

**Web chat** runs over Socket.IO on path `/ws`, authenticated via JWT from the handshake. File uploads go through `multer` (images only, 10 MB) into `uploads/<chatId>/`.

---

## Data Layer

- **`src/database/supabase.js`** — three clients:
  - `supabaseAdmin` — service-role key, bypasses RLS (server-side writes)
  - `supabaseAnon` — anon key, respects RLS (client-side reads)
  - `createUserClient(token)` — per-user token for RLS-scoped queries
- **`src/database.js`** — direct `pg` pool for web-chat history (separate from Supabase-managed tables)
- Schema files: `schema.sql`, `schema-wallet.sql`, `schema-enhanced-config.sql`, `schema-business-api.sql`

---

## Multi-Tenancy & Security

```mermaid
flowchart LR
  subgraph Tenant["Per-tenant isolation"]
    USER["User signup"]
    USER --> BOTCREATE["create bot<br/>connection_type: webjs | business_api"]
    BOTCREATE --> CONFIG["bot config<br/>trigger_prefix · LLM provider/model<br/>welcome · business hours · handoff"]
    CONFIG --> QUAOTA["monthly message quota<br/>+ wallet credits for overage"]
  end

  subgraph Auth["Auth layer"]
    JWT["JWT (jsonwebtoken)<br/>SESSION_SECRET · JWT_SECRET"]
    ADMINMW["admin middleware<br/>ADMIN_USERNAME / ADMIN_PASSWORD"]
    WEBAUTH["web-chat-auth middleware"]
  end

  subgraph StripeFlow["Billing"]
    SBT["Stripe checkout"]
    SBT --> WH["/api/stripe/webhook<br/>raw body → signature verify"]
    WH --> DBUPDATE["update subscription / wallet<br/>in Supabase"]
  end

  JWT -.protects.-> BOTCREATE
  ADMINMW -.protects.-> SBT
  WEBAUTH -.protects.-> WEBCHAT["/api/chat"]
```

---

## Deployment Topology

```mermaid
flowchart TB
  LB["Internet / WhatsApp Cloud"] --> NGINX["Nginx<br/>nginx-unified.conf<br/>TLS + static + proxy"]
  NGINX --> EXPRESS["PM2: api<br/>node src/app.js :3000"]
  NGINX --> NEXT["PM2: frontend<br/>next start :3001"]
  NGINX --> STATIC["/uploads static"]
  EXPRESS --> SUPACLOUD["Supabase Cloud<br/>Postgres + Auth"]
  EXPRESS --> LLMEXT["External LLM APIs"]
  EXPRESS --> STRIPEAPI["Stripe API"]
  NEXT --> EXPRESS
  subgraph PM2["ecosystem.config.cjs"]
    EXPRESS
    NEXT
  end
```

*PM2 manages two processes — the Express API and the Next.js frontend. Nginx fronts both, serves uploaded files, and terminates TLS.*

---

## Key Features & Highlights


`neyobytes-whatsapp-agent` is a multi-tenant WhatsApp Bot SaaS platform — users create AI-powered bots via QR pairing or WhatsApp Business API, configure behavior through a dashboard, and manage subscriptions via Stripe. What makes it impressive is the depth of engineering across context management, tool integration, reliability, and multi-tenancy. Below are notable features read directly from the source code.

- **VCC (View-oriented Conversation Compiler) two-tier compaction** — Group chats run thousands of messages; sending all to the LLM each turn would blow context windows. The VCC system (`src/vcc-compiler.js`) exports conversation to JSONL, runs a Python compiler (`vendor/VCC/skills/conversation-compiler`), and produces two views: a **Full View** (lossless, line-numbered transcript stored in DB) and a **Brief View** (compact with `(file:line-range)` pointers). The LLM context receives only `brief_view (system msg) + recent N messages verbatim`. Raw DB is always the compile source — never a previous brief view — preventing compaction drift. An in-memory SHA-256 cache (`MAX_CACHE_SIZE = 20`, LRU eviction) skips redundant compilations. Background compaction runs via `setImmediate` after the reply is sent, so users never wait.

- **`conversation_recall` tool with batch line-range reads** — The LLM can fetch exact message text from the Full View on demand. `conversation_recall` (`src/llm-service.js`) accepts `line_start`/`line_end`, raw VCC refs (parsed via `/:L?(\d+)(?:-L?(\d+))?/i`), batch `line_refs` arrays, batch `ranges` objects, or a search `query`. The tool fetches `getChatViews(chatId)` from Postgres, reads the requested lines from the Full View, and returns them. The system prompt explicitly instructs the LLM: *"Do NOT tell the user anything is gone or unavailable without first using conversation_recall to check."* — so the LLM always attempts recall before claiming something was lost.

- **Per-chat mutex with timeout-bounded lock acquisition** — To prevent race conditions on `conversationHistory` and duplicate LLM calls from rapid messages, `src/index.js` implements `acquireChatLock(chatId)`. It chains a `Promise` per chat with a `CHAT_LOCK_TIMEOUT_MS = 90_000` race — if the previous lock doesn't release within 90s, the wait is abandoned with a warning and the new message proceeds. `clearTimeout` cancels the timer if the previous lock resolves first, avoiding spurious log noise.

- **Three-phase context trimmer that fits any context window** — `_trimContextToFit()` (`src/llm-service.js`) runs a multi-stage budget: (1) truncate large `tool` results through cutoffs `[50000, 40000, 30000, 20000, 10000]` chars; (2) remove old non-system messages from the top while `> minKeep` (10); (3) crop the brief_view system message from the top, removing lines while `lines.length > 2000`. Token estimation uses `chars / 3.0 * 1.15` (conservative for multilingual). This guarantees the payload always fits the model's `LLM_CONTEXT_LENGTH` — gracefully degrading rather than crashing on oversized context.

- **Multi-provider LLM routing with cross-provider fallback** — `LLMService` (`src/llm-service.js`) supports 16+ providers: OpenRouter, OpenAI, Ollama, ZAI, Synthetic, Alibaba, Firepass, Crofai, Waferai, Neuralwatt, OpenCodeGo, RoutingRun, Deepseek, Makora, LiteLLM, and a ChatGPT gateway. Each bot gets a primary provider+model and an optional cross-provider fallback. Vision-capable models are detected by regex; images are delegated to a dedicated vision endpoint via `_delegateToVision()`. `_requestWithRetry` implements exponential backoff with jitter: `initialDelay * 2^attempt * (0.5 + random*0.5)`, capped at 30s, retrying only on 429/5xx/network errors with `maxRetries ≈ 3`.

- **MCP (Model Context Protocol) multi-server tool registry** — `src/mcp/mcp-manager.js` manages multiple MCP servers (e.g. Brave Search, Alpha Vantage) as **stdio child processes**. Each server's tools are discovered via `client.listTools()`, filtered via an optional `allowedTools` allowlist, and converted to OpenAI-compatible function definitions via `mcpToolsToOpenAI()`. A `toolIndex` map routes tool calls to the correct server. If a transport closes unexpectedly, `_autoRestart` re-launches it (with `MAX_RETRIES = 3`). Tool name collisions are detected and warned. This gives the WhatsApp bot extensible tool-calling without hardcoding each tool.

- **Ephemeral AI-generated website builder** — `src/site-manager.js` lets the LLM create complete React/HTML websites deployed as static files served by Nginx at `/sites/{id}/`. Each site gets an 8-char hex ID (`randomBytes(4).toString('hex')`). Features include: full-replace file writes (`updateSite`), exact-string find-and-replace edits with occurrence counting (`editSite` — fails if ambiguous unless `replaceAll=true`), `.bak` backups before every write, per-chat tracking, path traversal protection (`_validateFilePath` rejects `..` escapes), single-level undo, Puppeteer screenshots/validation, and a site manifest with automatic cleanup at `DEFAULT_MAX_SITES = 50`. The LLM tools (`create_website`, `read_website`, `edit_website`, `list_websites`) expose this to the chat.

- **Multi-provider web search + fetch pipeline** — The LLM can search the web and fetch pages. `_webSearch` dispatches to Brave, Exa, SearXNG, or Ollama search providers. `_webFetch` delegates to Firecrawl, Jina, or Ollama fetch. The search result includes a `_next_step` reminder: *"IMPORTANT: These are only snippets. You MUST now call web_fetch_all with these URLs to get the full content"* — deliberately prepended so it survives `_truncateToolResult()` truncation. `web_fetch_all` fetches multiple URLs in parallel via `Promise.allSettled` and returns `{ pages: [...] }`. Firecrawl content is capped at 50k chars with a truncation marker.

- **Dual WhatsApp connection types with unified interface** — `WebJSClient` (`src/bot/clients/webjs-client.js`) uses Puppeteer with `whatsapp-web.js` + `LocalAuth` for QR pairing; `BusinessAPIClient` (`src/bot/clients/business-client.js`) uses the Meta Cloud API with `accessToken` + `phoneNumberId`. Both extend `EventEmitter` and emit the same events (`qr`, `ready`, `disconnected`, `error`, `message`, `session`). Both have `checkHealth()` with `Promise.race([... , timeout(10s)])`, `updateActivity()`, and `getTimeSinceLastActivity()`. `BotManager` (`src/bot/manager.js`) routes to the correct client based on `bot.connection_type`.

- **Heartbeat monitoring with auto-restart** — `BotManager.performHealthCheck()` (`src/bot/manager.js`) runs on a configurable interval and checks every active bot via `client.checkHealth()` + `getTimeSinceLastActivity()`. A consecutive failure counter (`botFailureCounts`) tracks unhealthy bots; at `maxFailures` it schedules a restart via `restartBot()` (stop → 2s delay → start). Real-time warnings emit via Socket.IO to the user's room (`botHealthWarning`). On recovery, the failure count resets. Puppeteer crash detection (`isRecoverableBrowserError`, `attachBrowserLifecycleHooks`) monitors `disconnected` and `exit` events with exponential backoff recovery (`RECOVERY_BASE_DELAY_MS = 5000`, `RECOVERY_MAX_DELAY_MS = 120000`).

- **Anthropic OAuth token auto-refresh** — `src/utils/anthropic-token-manager.js` reads credentials from `~/.claude/.credentials.json`, checks expiry 5 minutes before expiration (`EXPIRY_BUFFER_MS = 5 * 60 * 1000`), and automatically refreshes via the OAuth token endpoint with `grant_type=refresh_token`. Updated tokens are written back to the credentials file. A `forceRefresh()` method handles 401 errors on demand. This enables the ChatGPT gateway provider without manual token rotation.

- **WhatsApp-native Markdown formatter** — `src/utils/whatsapp-formatter.js` converts standard Markdown to WhatsApp's limited formatting subset: `**bold**` → `*bold*`, `__bold__` → `*bold*`, headers → `*bold*`, links → `text (url)`, tables → label-value lists, code blocks preserved, inline code preserved, image links stripped, HTML tags stripped, horizontal rules removed. Code blocks and inline code are extracted into placeholders before processing to protect them, then restored after — a careful tokenization approach that prevents double-processing.

- **Always-log group context for full conversation awareness** — When a group is "always-log" enabled, ALL messages (not just `!ai` interactions) are saved to `whatsapp_chat_context` via `saveChatContextBatch`. This gives the LLM awareness of the full group conversation. The feature persists across PM2 restarts via `data/llm-settings.json` with `alwaysLogEnabled` and `alwaysLogGroups` set. On load, the `_loadedSettings` restore the state at boot, and `saveAlwaysLogSettings()` writes it atomically.

- **Per-group image caching with chat isolation** — `src/utils/image-cache.js` downloads remote images and stores them locally in `/uploads/<sanitized-chatId>/` for per-group isolation. ChatId sanitization (`120363399760651860@g.us` → `120363399760651860_g_us`) prevents directory traversal. Content-type detection determines extension (`png`, `webp`, `jpg`). `cacheImagesLocally` processes URLs in parallel via `Promise.all`. If download fails, the original remote URL is returned as fallback — graceful degradation.

- **Message processor pipeline with ordered handler chain** — `MessageProcessor` (`src/bot/services/message-processor.js`) runs every inbound message through a fixed pipeline: (1) CRM contact upsert; (2) welcome message on first message; (3) business hours check (timezone-aware via `Intl.DateTimeFormat`); (4) human handoff keyword detection; (5) menu trigger; (6) auto-reply rules; (7) FAQ matching; (8) quick-link matching. The first handler that sets `stopAI: true` short-circuits the AI call. FAQ and auto-reply caches have a 60-second TTL (`cacheTTL = 60000`) to avoid DB queries on every message.

- **Dual connection type Stripe billing** — `getPricingForConnectionType()` (`src/database/pricing.js`) returns different pricing tiers for `webjs` vs `business_api` connections, reflecting the different cost structures of Puppeteer-based vs API-based WhatsApp. The Stripe webhook (`src/api/routes/stripe.js`) verifies raw-body signatures and reconciles subscriptions and wallet credits back to Supabase.
