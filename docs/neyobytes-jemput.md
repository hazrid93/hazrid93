# Neyobytes Jemput — Architecture

> **Private project** — source code is not public. This document is hosted in the [`hazrid93/hazrid93`](https://github.com/hazrid93/hazrid93) profile repository so visitors can understand the architecture without needing repo access. Live site: **[jemput.neyobytes.com](https://jemput.neyobytes.com/)**

A digital wedding invitation (kad kahwin) platform for the Malaysian market. Couples sign up, build themed invitations through a drag-and-drop editor, publish them to a shareable slug (`jemput.neyobytes.com/<slug>`), and guests interact (RSVP, guestbook, AI chatbot) without an account. Stripe handles tiered subscriptions.

| | |
|---|---|
| **Frontend** | React 19 · TypeScript · Vite · React Router 7 · Mantine UI · Zustand · Framer Motion |
| **Backend** | Express 5 · TypeScript · `tsx` runtime · Helmet · rate-limit |
| **Database** | Supabase (Postgres) — service-role client, auth enforced in repository layer (not RLS) |
| **Auth** | Supabase Auth (client) + JWT issued by server (`jsonwebtoken`) · `requireAuth` / `optionalAuth` middleware |
| **Payments** | Stripe (one-time, tiered plans: Asas / Premium) · 60-day active duration |
| **AI Chatbot** | OpenAI-compatible API — Novita AI · Alibaba DashScope · Ollama · shared-pool daily quota (RPC) |
| **Editor** | `@dnd-kit` drag-and-drop · 8 theme templates · 18 section types · live phone preview · AI editor assistant |
| **Process** | PM2 (`ecosystem.config.cjs`) · Nginx (`nginx/jemput.conf`) |

---

## High-Level Architecture

```mermaid
flowchart TB
  subgraph Visitors
    GUEST["Guest<br/>visits /:slug — no login"]
    COUPLE["Couple<br/>Dashboard / Editor — auth"]
    ADMIN["Admin<br/>Admin panel — auth"]
  end

  subgraph Nginx["Nginx reverse proxy"]
    NGX["nginx/jemput.conf"]
  end

  subgraph Frontend["React 19 SPA  (Vite build)"]
    ROUTER["react-router-dom 7<br/>App.tsx routes"]
    PAGES["Pages<br/>Landing · Login · Pricing · Dashboard ·<br/>TrialEditor · Invitation/:slug · Admin"]
    STORES["Zustand stores<br/>authStore · dashboardStore ·<br/>invitationStore · paymentStore"]
    COMPS["Components<br/>invitation/* · dashboard/* · common/*"]
    APIC["src/lib/api.ts<br/>fetch + JWT bearer + 401 handling"]
  end

  subgraph Backend["Express 5 server  (server/index.ts)"]
    MW["middleware/auth.ts<br/>requireAuth · optionalAuth · JWT verify"]
    ROUTES["Routes<br/>auth · invitations · rsvps-guestbook ·<br/>plans-subscription · admin · uploads ·<br/>iconify · lottie · chat"]
    REPO["DataRepository interface<br/>server/repo/repository.ts"]
    IMPL["SupabaseRepository<br/>service-role client"]
  end

  subgraph Supabase["Supabase Cloud"]
    PG["Postgres<br/>15+ migrations"]
    SBAUTH["Supabase Auth"]
    RPC["RPC: shared-pool quota<br/>check+increment (atomic)"]
  end

  subgraph External
    STRIPE["Stripe API"]
    LLM["LLM Providers<br/>Novita · DashScope · Ollama"]
    UPLOADS["Multer uploads<br/>gallery · couple photos"]
  end

  GUEST --> NGX
  COUPLE --> NGX
  ADMIN --> NGX
  NGX --> ROUTER

  ROUTER --> PAGES
  PAGES --> STORES --> COMPS
  PAGES --> APIC

  APIC -->|/api| MW
  MW --> ROUTES
  ROUTES --> REPO
  REPO --> IMPL
  IMPL --> PG
  IMPL --> SBAUTH
  ROUTES --> RPC
  ROUTES --> STRIPE
  ROUTES --> LLM
  ROUTES --> UPLOADS
```

---

## Repository Pattern — The DB Swap Seam

Every route handler calls methods on the `DataRepository` interface — never the Supabase client directly. Today's implementation is `SupabaseRepository`; a future `PostgresRepository` can be swapped by changing one line in `server/index.ts`. **Authorization is enforced in the repository, not in RLS** — every user-owned method takes a `userId` and filters by it.

```mermaid
flowchart LR
  subgraph Interface["DataRepository  (interface)"]
    AUTH_I["signUp · signIn<br/>getUserById<br/>resetPassword · updatePassword"]
    INV_I["listInvitations<br/>getInvitationById(userId)<br/>getInvitationBySlug<br/>create/update/deleteInvitation"]
    CHILD_I["saveSections · saveCopyOverrides<br/>saveItinerary · saveContacts<br/>saveWishlist · saveGalleryImages"]
    RSVPI["listRSVPs · createRSVP<br/>listGuestbook · createGuestbookMessage"]
    PAY_I["listPlans · create/updatePlan<br/>createPayment<br/>listPayments"]
  end

  subgraph Impl["SupabaseRepository  (implements)"]
    SR["createClient(SUPABASE_URL,<br/>SERVICE_ROLE_KEY)<br/>bypasses RLS"]
  end

  ROUTES["Express routes"] -->|"method calls"| Interface
  Interface -.implemented by.-> Impl
  Impl -->|queries| PG["Postgres"]
```

---

## Frontend Route Map

`src/App.tsx` — lazy-loaded pages behind `ProtectedRoute` (redirects to `/login` if `authStore.user` is null once initialized).

```mermaid
flowchart LR
  subgraph Public["Public (no auth)"]
    R1["/  → LandingPage"]
    R2["/login → LoginPage"]
    R3["/pricing → PricingPage"]
    R4["/cuba → TrialEditorPage"]
    R5["/:slug → InvitationPage"]
    R6["/tentang-kami · /hubungi-kami<br/>/terma · /privasi · /polisi-bayaran-balik"]
  end

  subgraph Protected["Protected (requireAuth)"]
    R7["/dashboard/* → DashboardPage"]
    R8["/admin/* → AdminPage"]
  end

  AUTHINIT["useAuthStore.initialize()<br/>on mount"] --> CHECK{"user && initialized?"}
  CHECK -- no --> LOGIN["Navigate → /login"]
  CHECK -- yes --> RENDER["render children"]

  R7 --> PROTECTEDGUARD[ProtectedRoute]
  R8 --> PROTECTEDGUARD
  PROTECTEDGUARD --> CHECK
```

---

## Invitation Lifecycle

```mermaid
sequenceDiagram
  participant Couple as Couple
  participant FE as Dashboard / Editor
  participant API as /api/invitations
  participant Repo as SupabaseRepository
  participant DB as Supabase Postgres
  participant Guest as Guest

  Couple->>FE: create invitation (theme, sections)
  FE->>API: POST /invitations (JWT bearer)
  API->>Repo: createInvitation(userId, partial)
  Repo->>DB: insert invitation row
  Repo-->>API: Invitation
  API-->>FE: 201 invitation

  loop edit — section reorder, copy, images
    FE->>FE: local Zustand state + preview protocol
    FE->>API: PUT /invitations/:id (sections, copyOverrides, itinerary, contacts…)
    API->>Repo: saveSections / saveCopyOverrides / saveGalleryImages(userId)
    Repo->>DB: full-replace child collections
  end

  Couple->>FE: publish → set status + slug
  FE->>API: PATCH /invitations/:id { status: 'published', slug }
  API->>Repo: updateInvitation(id, userId, updates)
  Repo->>DB: update row

  Guest->>FE: visit /:slug
  FE->>API: GET /invitations/slug/:slug (optionalAuth)
  API->>Repo: getInvitationBySlug(slug, viewerId?)
  Repo->>DB: select invitation + rsvps + guestbook
  Repo-->>API: { invitation, rsvps, guestbook }
  API-->>FE: 200 full bundle
  FE->>FE: render InvitationPage (countdown, gallery, music…)
```

---

## RSVP + Guestbook Flow (public, no login)

```mermaid
sequenceDiagram
  participant Guest as Guest
  participant FE as RSVPForm / Guestbook
  participant Store as invitationStore (Zustand)
  participant API as /api/rsvps · /api/guestbook
  participant Repo as SupabaseRepository
  participant DB as Supabase

  Guest->>FE: fill RSVP (attending, adults, children, message)
  FE->>Store: submitRSVP(rsvp)
  alt slug === 'aiman-nadia' (demo)
    Store->>Store: use demo data from localStorage
  else
    Store->>API: POST /rsvps
    API->>Repo: createRSVP(rsvp)
    Repo->>DB: insert rsvp row
    Repo-->>API: RSVP
    API-->>FE: 201
  end
  FE->>Guest: show "submitted" confirmation

  Guest->>FE: post guestbook message
  FE->>Store: submitGuestbook(msg)
  Store->>API: POST /guestbook
  API->>Repo: createGuestbookMessage(msg)
  Repo->>DB: insert guestbook row
  Repo-->>API: GuestbookMessage
  API-->>FE: 201
  FE->>FE: refresh guestbook list
```

---

## AI Chatbot — Shared-Pool Quota

The invitation chatbot (premium) and the editor AI assistant both use a **shared-pool daily quota** enforced by an atomic Supabase RPC. The client fails **closed** — if the quota check errors, the request is blocked (never allowed through).

```mermaid
sequenceDiagram
  participant UI as ChatbotWidget /<br/>EditorChatAssistant
  participant Lib as src/lib/chatbot.ts
  participant API as /api/chat/quota + /api/chat
  participant RPC as Supabase RPC<br/>(atomic check+increment)
  participant LLM as LLM Provider

  UI->>Lib: sendChatMessage({messages, systemPrompt})
  Lib->>API: POST /chat/quota { poolKey }
  API->>RPC: check_pool_quota(pool_key, date)
  RPC-->>API: { allowed, remaining }
  alt allowed = false (or RPC error → fail closed)
    API-->>Lib: { allowed: false, remaining: 0 }
    Lib-->>UI: blocked — show limit message
  else allowed = true
    API-->>Lib: { allowed: true, remaining: N }
    Lib->>API: POST /chat { messages, systemPrompt }
    API->>LLM: chat/completions (OpenAI-compatible)
    LLM-->>API: completion
    API-->>Lib: reply text
    Lib-->>UI: render answer
  end
```

**Quota keys:** `invitation:<id>` for the public chatbot, `cuba_editor` / `editor` for the in-editor assistant. The daily limit is resolved server-side from `site_settings` — the client cannot inflate it.

---

## Payments & Subscription

```mermaid
flowchart LR
  COUPLE["Couple picks plan"] --> CHECKOUT["CheckoutPage<br/>@stripe/react-stripe-js"]
  CHECKOUT --> SCS["create Stripe Checkout session<br/>server route: plans-subscription"]
  SCS --> STRIPE["Stripe"]
  STRIPE --> REDIRECT["Redirect to Checkout"]
  REDIRECT --> RETURN["CheckoutReturnPage<br/>verify session, update local store"]
  RETURN --> PUBLISH["plan active → invitation live 60 days"]

  subgraph WebhookPath["Webhook reconciliation (server)"]
    WH["Stripe webhook"] --> ROUTE["/api/stripe webhook handler"]
    ROUTE --> PAYDB["createPayment · update subscription<br/>via SupabaseRepository"]
  end
```

**Plans:** Asas (free trial tier) / Premium (paid). Payment status tracked per invitation (`free` / `paid` / `expired`). Active duration: 60 days from payment.

---

## Editor Architecture

`src/components/dashboard/InvitationEditor.tsx` is the core builder. It uses a preview-protocol (`src/lib/preview-protocol.ts`) to bridge the editor form state with the live phone preview iframe.

```mermaid
flowchart TB
  subgraph Editor["InvitationEditor"]
    FORM["Mantine form<br/>+ helpers (invitationToFormValues,<br/>updateSectionConfig, etc.)"]
    THEME["ThemeSelector<br/>8 templates, fonts, colors, decorations"]
    SEC["SectionManager<br/>@dnd-kit drag-drop, 18 section types"]
    DECO["DecorationsPanel<br/>LottiePicker · IconPicker"]
    ASSIST["EditorChatAssistant<br/>AI copy suggestions (quota)"]
  end

  subgraph Preview["Live preview"]
    PHONE["PhonePreview iframe"]
    PROTO["preview-protocol.ts<br/>sendPreviewMessage / onPreviewMessage"]
  end

  subgraph Save["Persistence"]
    FLOAT["FloatingSaveButton<br/>SaveIndicator: saving / saved / error"]
    APIPUT["PUT /api/invitations/:id"]
  end

  FORM --> THEME
  FORM --> SEC
  FORM --> DECO
  FORM --> ASSIST
  FORM -->|"sendPreviewMessage"| PROTO
  PROTO --> PHONE
  PHONE -->|"onPreviewMessage"| PROTO
  FORM --> FLOAT
  FLOAT --> APIPUT
  SEC -->|"reorder"| FORM
```

**Trial mode:** `TrialEditorPage` (`/cuba`) stores the invitation in `localStorage` (`TRIAL_PREVIEW_STORAGE_KEY`) — no account or server round-trip needed. Demo slug `aiman-nadia` renders `demoInvitation` data, allowing the editor to be explored without signup.

---

## Data Model (key tables)

Migrations live in `supabase/` (16 numbered files). Authorization is enforced in the repository layer.

```mermaid
erDiagram
  users ||--o{ invitations : owns
  invitations ||--o{ invitation_sections : has
  invitations ||--o{ rsvps : receives
  invitations ||--o{ guestbook_messages : receives
  invitations ||--o{ gallery_images : has
  invitations ||--o{ itinerary_items : has
  invitations ||--o{ contact_persons : has
  invitations ||--o{ copy_overrides : has
  plans ||--o{ payments : "purchased via"
  users ||--o{ payments : makes
  site_settings ||--o{ invitations : "configures"

  invitations {
    uuid id PK
    uuid user_id FK
    string slug
    string status
    string template_id
    jsonb theme_config
    timestamp wedding_date
    boolean rsvp_enabled
    timestamp rsvp_deadline
  }
  rsvps {
    uuid id PK
    uuid invitation_id FK
    string guest_name
    boolean attending
    int num_adults
    int num_children
    string message
  }
  guestbook_messages {
    uuid id PK
    uuid invitation_id FK
    string guest_name
    string message
  }
  invitation_sections {
    uuid id PK
    uuid invitation_id FK
    string section_type
    jsonb config
    int sort_order
  }
  plans {
    uuid id PK
    string name
    int price_cents
    boolean active
  }
  payments {
    uuid id PK
    uuid user_id FK
    uuid plan_id FK
    string stripe_session_id
    string status
  }
```

---

## Deployment Topology

```mermaid
flowchart TB
  LB["Internet"] --> NGINX["Nginx<br/>nginx/jemput.conf<br/>TLS + static SPA + proxy /api"]
  NGINX --> VITE["PM2: web<br/>Vite build → static files"]
  NGINX --> SERVER["PM2: server<br/>tsx watch server/index.ts :3002"]
  SERVER --> SUPACLOUD["Supabase Cloud<br/>Postgres + Auth + RPC"]
  SERVER --> STRIPE["Stripe API"]
  SERVER --> LLMEXT["LLM APIs (OpenAI-compatible)"]
  VITE -.fetch /api.-> SERVER
  subgraph PM2["ecosystem.config.cjs"]
    SERVER
    VITE
  end
```

*PM2 runs the Express server (`tsx`) and serves the Vite-built SPA as static files. Nginx terminates TLS, serves the SPA, and proxies `/api/*` to the server. The server talks to Supabase (service-role), Stripe, and LLM providers.*
