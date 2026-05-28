# Customer Journey Orchestration — Phased Development Plan

> Project: 135-customer-journey-orchestration · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript 5.x (strict mode) | Full-stack type safety; largest MarTech ecosystem is TypeScript; end-to-end type sharing between API, workers, and frontend eliminates interface drift |
| Runtime | Node.js 22 LTS | Native ESM, stable LTS for production event-processing workloads, worker threads for CPU-bound tasks |
| API framework | Next.js 15 App Router + tRPC v11 | App Router provides SSR/RSC for dashboard pages; tRPC gives end-to-end type-safe RPC for the journey builder UI; REST routes co-exist for external integrations and webhook receivers |
| Database | PostgreSQL 16 | JSONB + GIN indexes power the hybrid relational/JSONB model (data-model-suggestion-3); RLS for org-level tenant isolation; partitioning for events and audit logs; pgvector for AI embedding similarity |
| ORM | Drizzle ORM | Type-safe schema-as-code with native JSONB column support; migration generation from schema diffs; lighter than Prisma for JSONB-heavy models |
| Event streaming | BullMQ 5.x on Redis 7 (MVP) / Kafka upgrade path | BullMQ handles journey step execution, message dispatch queues, event ingestion, segment recomputation; repeatable jobs for scheduled journeys; rate limiting for channel provider quotas. Kafka planned for v1.1 real-time streaming |
| Frontend | React 19 + Next.js 15 App Router | Server Components reduce client JS bundle; Suspense boundaries for progressive loading of journey canvas and analytics dashboards |
| Journey canvas | React Flow v12 + @dnd-kit | React Flow for the visual DAG-based journey builder; @dnd-kit for drag-and-drop step palette; supports zoom, pan, minimap, and edge routing |
| UI components | shadcn/ui + Tailwind CSS 4 | Accessible, composable component library; TanStack Table v8 for data tables; Recharts for analytics charts |
| Authentication | Auth.js v5 (NextAuth) | Built-in OAuth 2.0 providers (Google, Microsoft); OIDC support for enterprise SSO; JWT session strategy for API routes |
| AI / LLM | Anthropic Claude SDK + Vercel AI SDK 4 | Claude for autonomous journey design from natural language, message variant generation, anomaly analysis; Vercel AI SDK for streaming responses and structured output |
| Email provider | SendGrid API v3 (default) + provider-swappable architecture | SendGrid for email dispatch; abstracted behind a channel provider interface to support Mailgun, SES, Postmark |
| SMS provider | Twilio Programmable Messaging | Industry-standard SMS/MMS API; abstracted behind channel provider interface |
| Push notifications | Firebase Cloud Messaging (FCM) | Cross-platform push (iOS, Android, web); abstracted behind channel provider interface |
| Event schema | CloudEvents v1.0.2 + Segment Spec compatibility | CloudEvents envelope for vendor-neutral event ingestion; Segment Spec track/identify/page/group support for CDP integration compatibility |
| Consent management | Custom consent engine (GDPR/CCPA/TCF v2.3) | Consent-gated message dispatch; TCF string parsing; suppression list enforcement; data subject request processing |
| MCP server | @modelcontextprotocol/sdk | Official MCP TypeScript SDK; exposes journey builder actions, segment queries, and message dispatch to AI agents |
| Testing | Vitest + Playwright | Vitest for unit/integration (fast, ESM-native); Playwright for E2E browser tests of journey canvas and dashboard |
| Containerisation | Docker + docker-compose | Single `docker compose up` for PostgreSQL, Redis, and app; multi-stage Dockerfile for production image |
| Code quality | ESLint 9 (flat config) + Prettier | TypeScript strict mode enforced; no-explicit-any rule; consistent formatting |
| Package manager | pnpm 9 | Strict dependency resolution; workspace support for monorepo; faster installs than npm |
| API documentation | OpenAPI 3.1 via @asteasolutions/zod-to-openapi | Auto-generate OpenAPI spec from Zod validators shared with tRPC; serves Swagger UI at /api/docs |

### Project Structure

```
customer-journey-orchestration/
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── drizzle.config.ts
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── src/
│   ├── app/                              # Next.js App Router
│   │   ├── layout.tsx
│   │   ├── page.tsx                      # Landing / dashboard redirect
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── callback/route.ts
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                # Sidebar + header shell
│   │   │   ├── journeys/
│   │   │   │   ├── page.tsx              # Journey list
│   │   │   │   ├── new/page.tsx          # Create journey
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx          # Journey canvas editor
│   │   │   │       ├── analytics/page.tsx
│   │   │   │       └── settings/page.tsx
│   │   │   ├── segments/
│   │   │   │   ├── page.tsx              # Segment list
│   │   │   │   └── [id]/page.tsx         # Segment builder / members
│   │   │   ├── templates/
│   │   │   │   ├── page.tsx              # Template library
│   │   │   │   └── [id]/page.tsx         # Template editor
│   │   │   ├── customers/
│   │   │   │   ├── page.tsx              # Customer list
│   │   │   │   └── [id]/page.tsx         # Customer profile + timeline
│   │   │   ├── analytics/
│   │   │   │   ├── page.tsx              # Overview dashboard
│   │   │   │   └── funnels/page.tsx      # Funnel analytics
│   │   │   ├── settings/
│   │   │   │   ├── organisation/page.tsx
│   │   │   │   ├── channels/page.tsx
│   │   │   │   ├── integrations/page.tsx
│   │   │   │   ├── consent/page.tsx
│   │   │   │   └── team/page.tsx
│   │   │   └── ai/
│   │   │       └── journey-designer/page.tsx  # AI journey design
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts      # tRPC handler
│   │       ├── webhooks/
│   │       │   ├── events/route.ts       # CloudEvents / Segment ingest
│   │       │   ├── sendgrid/route.ts     # Email delivery webhooks
│   │       │   ├── twilio/route.ts       # SMS delivery webhooks
│   │       │   └── stripe/route.ts       # Billing webhooks
│   │       ├── mcp/route.ts              # MCP server HTTP transport
│   │       └── v1/                       # Public REST API
│   │           ├── events/route.ts
│   │           ├── customers/route.ts
│   │           ├── journeys/route.ts
│   │           ├── segments/route.ts
│   │           └── templates/route.ts
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                  # Drizzle client
│   │   │   ├── schema/
│   │   │   │   ├── organisations.ts
│   │   │   │   ├── org-members.ts
│   │   │   │   ├── customers.ts
│   │   │   │   ├── events.ts
│   │   │   │   ├── segments.ts
│   │   │   │   ├── segment-memberships.ts
│   │   │   │   ├── journeys.ts
│   │   │   │   ├── journey-steps.ts
│   │   │   │   ├── journey-edges.ts
│   │   │   │   ├── channels.ts
│   │   │   │   ├── message-templates.ts
│   │   │   │   ├── journey-enrollments.ts
│   │   │   │   ├── message-sends.ts
│   │   │   │   ├── consent-log.ts
│   │   │   │   ├── suppression-list.ts
│   │   │   │   ├── integrations.ts
│   │   │   │   ├── journey-stats.ts
│   │   │   │   ├── ab-experiments.ts
│   │   │   │   └── audit-log.ts
│   │   │   └── migrations/
│   │   ├── trpc/
│   │   │   ├── router.ts                 # Root tRPC router
│   │   │   ├── context.ts
│   │   │   └── routers/
│   │   │       ├── journeys.ts
│   │   │       ├── journey-steps.ts
│   │   │       ├── segments.ts
│   │   │       ├── customers.ts
│   │   │       ├── templates.ts
│   │   │       ├── channels.ts
│   │   │       ├── analytics.ts
│   │   │       ├── consent.ts
│   │   │       ├── integrations.ts
│   │   │       ├── ai.ts
│   │   │       └── settings.ts
│   │   ├── services/
│   │   │   ├── journey-engine/
│   │   │   │   ├── executor.ts           # Step execution orchestrator
│   │   │   │   ├── evaluator.ts          # Decision node evaluator
│   │   │   │   ├── scheduler.ts          # Delay / wait-until logic
│   │   │   │   └── enrollment.ts         # Enrollment lifecycle
│   │   │   ├── event-ingestion/
│   │   │   │   ├── ingest.ts             # CloudEvents parser + router
│   │   │   │   ├── identity-resolver.ts  # Anonymous-to-known resolution
│   │   │   │   └── segment-evaluator.ts  # Real-time segment membership
│   │   │   ├── channels/
│   │   │   │   ├── provider-interface.ts # Abstract channel provider
│   │   │   │   ├── email/
│   │   │   │   │   ├── sendgrid.ts
│   │   │   │   │   └── renderer.ts       # Liquid template rendering
│   │   │   │   ├── sms/
│   │   │   │   │   └── twilio.ts
│   │   │   │   └── push/
│   │   │   │       └── fcm.ts
│   │   │   ├── consent/
│   │   │   │   ├── consent-engine.ts     # Consent check + enforcement
│   │   │   │   ├── tcf-parser.ts         # IAB TCF string parsing
│   │   │   │   └── dsr-processor.ts      # Data subject request handler
│   │   │   ├── ai/
│   │   │   │   ├── journey-designer.ts   # NL-to-journey canvas
│   │   │   │   ├── message-generator.ts  # Generative message variants
│   │   │   │   ├── anomaly-detector.ts   # Journey anomaly detection
│   │   │   │   └── prompts.ts            # System prompts
│   │   │   ├── mcp/
│   │   │   │   ├── server.ts
│   │   │   │   ├── resources.ts
│   │   │   │   ├── tools.ts
│   │   │   │   └── prompts.ts
│   │   │   └── analytics/
│   │   │       ├── aggregator.ts         # Daily stats aggregation
│   │   │       └── funnel.ts             # Funnel computation
│   │   ├── workers/
│   │   │   ├── index.ts                  # BullMQ worker bootstrap
│   │   │   ├── journey-step.worker.ts    # Journey step execution
│   │   │   ├── message-dispatch.worker.ts
│   │   │   ├── event-ingest.worker.ts
│   │   │   ├── segment-compute.worker.ts
│   │   │   ├── analytics-aggregate.worker.ts
│   │   │   ├── consent-dsr.worker.ts
│   │   │   └── ai-generation.worker.ts
│   │   └── lib/
│   │       ├── auth.ts                   # Auth.js configuration
│   │       ├── queue.ts                  # BullMQ queue definitions
│   │       ├── redis.ts
│   │       ├── encryption.ts             # Credential encryption at rest
│   │       ├── validators.ts             # Zod schemas shared with tRPC
│   │       ├── cloudevents.ts            # CloudEvents envelope helpers
│   │       └── liquid.ts                 # Liquid template engine config
│   ├── components/
│   │   ├── ui/                           # shadcn/ui components
│   │   ├── layout/
│   │   │   ├── sidebar.tsx
│   │   │   ├── header.tsx
│   │   │   └── command-palette.tsx
│   │   ├── journey/
│   │   │   ├── canvas.tsx                # React Flow journey canvas
│   │   │   ├── step-palette.tsx          # Draggable step types
│   │   │   ├── step-nodes/
│   │   │   │   ├── action-email.tsx
│   │   │   │   ├── action-sms.tsx
│   │   │   │   ├── action-push.tsx
│   │   │   │   ├── action-webhook.tsx
│   │   │   │   ├── decision.tsx
│   │   │   │   ├── delay.tsx
│   │   │   │   ├── ab-split.tsx
│   │   │   │   └── exit.tsx
│   │   │   ├── step-config-panel.tsx      # Right-hand config sidebar
│   │   │   ├── journey-toolbar.tsx
│   │   │   └── journey-stats-overlay.tsx
│   │   ├── segments/
│   │   │   ├── segment-builder.tsx        # Visual rule builder
│   │   │   ├── condition-row.tsx
│   │   │   └── segment-preview.tsx
│   │   ├── templates/
│   │   │   ├── template-editor.tsx
│   │   │   ├── email-preview.tsx
│   │   │   └── sms-preview.tsx
│   │   ├── customers/
│   │   │   ├── customer-table.tsx
│   │   │   ├── customer-profile.tsx
│   │   │   └── customer-timeline.tsx
│   │   ├── analytics/
│   │   │   ├── overview-dashboard.tsx
│   │   │   ├── journey-funnel.tsx
│   │   │   ├── metric-card.tsx
│   │   │   └── channel-breakdown.tsx
│   │   ├── ai/
│   │   │   ├── journey-designer-chat.tsx
│   │   │   └── message-variant-generator.tsx
│   │   └── shared/
│   │       ├── status-badge.tsx
│   │       ├── empty-state.tsx
│   │       └── consent-gate.tsx
│   └── lib/
│       ├── trpc-client.ts
│       └── utils.ts
├── scripts/
│   ├── seed.ts                           # Development seed data
│   └── generate-openapi.ts              # OpenAPI spec generation
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
└── public/
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose
Establish the project skeleton, database connection, authentication, and development environment. After this phase, a developer can run the app locally with Docker, authenticate via OAuth, and hit an empty authenticated API.

### Tasks

#### 1.1 — Project Initialisation

**What**: Scaffold the Next.js 15 project with TypeScript, Tailwind CSS, ESLint, and pnpm.

**Design**:

```bash
pnpm create next-app@latest customer-journey-orchestration --typescript --tailwind --eslint --app --src-dir
cd customer-journey-orchestration
pnpm add drizzle-orm postgres @auth/core @auth/drizzle-adapter
pnpm add -D drizzle-kit @types/node vitest
```

`tsconfig.json` strict mode settings:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": false,
    "moduleResolution": "bundler",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

`.env.example`:
```env
DATABASE_URL=postgresql://cjo:cjo@localhost:5432/customer_journey_orchestration
REDIS_URL=redis://localhost:6379
AUTH_SECRET=<random-32-bytes>
AUTH_GOOGLE_ID=
AUTH_GOOGLE_SECRET=
ANTHROPIC_API_KEY=
SENDGRID_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
FIREBASE_PROJECT_ID=
ENCRYPTION_KEY=<random-32-bytes-hex>
```

**Testing**:
- `test-1.1-build`: `pnpm build` succeeds with zero errors
- `test-1.1-lint`: `pnpm lint` passes with no warnings
- `test-1.1-strict`: TypeScript strict mode catches missing null checks in a synthetic test file

#### 1.2 — Docker Compose Environment

**What**: Docker Compose configuration for PostgreSQL 16, Redis 7, and the app in development mode.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: customer_journey_orchestration
      POSTGRES_USER: cjo
      POSTGRES_PASSWORD: cjo
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://cjo:cjo@postgres:5432/customer_journey_orchestration
      REDIS_URL: redis://redis:6379
    depends_on: [postgres, redis]
    volumes: ["./src:/app/src"]

volumes:
  pgdata:
```

Multi-stage `Dockerfile`:
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable pnpm

FROM base AS development
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
CMD ["pnpm", "dev"]

FROM base AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM base AS production
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
CMD ["node", "server.js"]
```

**Testing**:
- `test-1.2-postgres`: `docker compose up -d postgres redis` starts services; `psql` connects successfully
- `test-1.2-app`: `docker compose up app` starts Next.js dev server at localhost:3000
- `test-1.2-redis`: Redis CLI `PING` returns `PONG`

#### 1.3 — Database Schema & Migrations (Core Tables)

**What**: Drizzle ORM schema definitions for organisations, org_members, and core infrastructure. Based on data-model-suggestion-3 (Hybrid Relational+JSONB).

**Design**:

```typescript
// src/server/db/schema/organisations.ts
import { pgTable, uuid, text, jsonb, timestamp } from "drizzle-orm/pg-core";

export const organisations = pgTable("organisations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name").notNull(),
  slug: text("slug").notNull().unique(),
  planTier: text("plan_tier").notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// src/server/db/schema/org-members.ts
export const orgMembers = pgTable("org_members", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull(),
  email: text("email").notNull(),
  role: text("role").notNull().default("viewer"),
  permissions: jsonb("permissions").notNull().default([]),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_org_members_org_user").on(table.orgId, table.userId),
  index("idx_org_members_org").on(table.orgId),
]);
```

Drizzle config:
```typescript
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";
export default defineConfig({
  schema: "./src/server/db/schema/*",
  out: "./src/server/db/migrations",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

**Testing**:
- `test-1.3-schema`: Drizzle schema compiles with zero type errors
- `test-1.3-migrate`: `pnpm drizzle-kit push` applies schema to local PostgreSQL; `\dt` lists organisations, org_members tables
- `test-1.3-fk`: Insert an organisation and member; verify foreign key constraint holds
- `test-1.3-unique`: Unique constraint on (org_id, user_id) rejects duplicate members

#### 1.4 — Authentication with Auth.js

**What**: Auth.js v5 configuration with Google OAuth provider, email/password fallback for development, and organisation-scoped sessions.

**Design**:

```typescript
// src/server/lib/auth.ts
import NextAuth from "next-auth";
import Google from "next-auth/providers/google";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { db } from "@/server/db";

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [
    Google({
      clientId: process.env.AUTH_GOOGLE_ID!,
      clientSecret: process.env.AUTH_GOOGLE_SECRET!,
    }),
  ],
  session: { strategy: "jwt" },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.orgId = user.orgId;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.orgId = token.orgId as string;
      session.user.role = token.role as string;
      return session;
    },
  },
});
```

Session type augmentation:
```typescript
// src/types/next-auth.d.ts
declare module "next-auth" {
  interface Session {
    user: {
      id: string;
      email: string;
      name: string;
      orgId: string;
      role: string;
    };
  }
}
```

**Testing**:
- `test-1.4-providers`: GET `/api/auth/providers` returns Google provider
- `test-1.4-unauth`: Unauthenticated request to protected tRPC route returns 401
- `test-1.4-jwt`: JWT callback enriches token with orgId and role
- `test-1.4-session`: Session callback exposes orgId on session.user

#### 1.5 — tRPC Router Scaffold

**What**: Root tRPC router with health check procedure and middleware for org-level isolation.

**Design**:

```typescript
// src/server/trpc/router.ts
import { initTRPC, TRPCError } from "@trpc/server";
import { Context } from "./context";
import superjson from "superjson";

const t = initTRPC.context<Context>().create({
  transformer: superjson,
});

export const publicProcedure = t.procedure;

export const protectedProcedure = t.procedure.use(async ({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      ...ctx,
      session: ctx.session,
      orgId: ctx.session.user.orgId,
    },
  });
});

export const appRouter = t.router({
  health: publicProcedure.query(() => ({ status: "ok", timestamp: new Date() })),
});

export type AppRouter = typeof appRouter;
```

```typescript
// src/server/trpc/context.ts
import { auth } from "@/server/lib/auth";
import { db } from "@/server/db";

export async function createContext() {
  const session = await auth();
  return { db, session };
}
export type Context = Awaited<ReturnType<typeof createContext>>;
```

**Testing**:
- `test-1.5-health`: GET `/api/trpc/health` returns `{ status: "ok" }`
- `test-1.5-protected`: Protected procedure rejects unauthenticated request with UNAUTHORIZED
- `test-1.5-orgscope`: Protected procedure attaches orgId from session to context

---

## Phase 2: Customer Data & Event Ingestion

### Purpose
Build the customer profile model and event ingestion pipeline. After this phase, the platform can receive customer events via REST API (CloudEvents/Segment-compatible), resolve customer identity, and store customer profiles with flexible attributes.

### Tasks

#### 2.1 — Customer Schema & CRUD

**What**: Drizzle schema for customers table with JSONB attributes, identities, and consent columns. Full tRPC CRUD operations.

**Design**:

```typescript
// src/server/db/schema/customers.ts
export const customers = pgTable("customers", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  externalId: text("external_id"),
  email: text("email"),
  phone: text("phone"),
  firstName: text("first_name"),
  lastName: text("last_name"),
  timezone: text("timezone"),
  locale: text("locale"),
  countryCode: text("country_code"),
  attributes: jsonb("attributes").notNull().default({}),
  identities: jsonb("identities").notNull().default([]),
  consent: jsonb("consent").notNull().default({}),
  lastSeenAt: timestamp("last_seen_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_customers_org_external").on(table.orgId, table.externalId),
  index("idx_customers_org").on(table.orgId),
  index("idx_customers_email").on(table.orgId, table.email),
  index("idx_customers_attrs").using("gin", table.attributes),
]);
```

Zod validators:
```typescript
// src/server/lib/validators.ts
import { z } from "zod";

export const customerIdentitySchema = z.object({
  type: z.enum(["email", "phone", "device_id", "anonymous_id", "user_id"]),
  value: z.string(),
  verified: z.boolean().default(false),
});

export const createCustomerSchema = z.object({
  externalId: z.string().optional(),
  email: z.string().email().optional(),
  phone: z.string().optional(),
  firstName: z.string().optional(),
  lastName: z.string().optional(),
  timezone: z.string().optional(),
  locale: z.string().optional(),
  countryCode: z.string().length(2).optional(),
  attributes: z.record(z.unknown()).optional(),
  identities: z.array(customerIdentitySchema).optional(),
});

export type CreateCustomerInput = z.infer<typeof createCustomerSchema>;
```

**Testing**:
- `test-2.1-create-customer`: Create a customer via tRPC; verify returned ID and fields
- `test-2.1-list-customers`: List customers filtered by org; verify org isolation
- `test-2.1-update-attributes`: Update customer JSONB attributes; verify merge not overwrite
- `test-2.1-external-id-unique`: Duplicate external_id within same org is rejected
- `test-2.1-cross-org-isolation`: Customer from org A is invisible to org B queries

#### 2.2 — Event Ingestion API

**What**: REST endpoint accepting CloudEvents-formatted and Segment Spec-formatted events. Events are parsed, validated, and stored in the events table.

**Design**:

```typescript
// src/server/db/schema/events.ts
export const events = pgTable("events", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  customerId: uuid("customer_id").references(() => customers.id, { onDelete: "set null" }),
  anonymousId: text("anonymous_id"),
  eventType: text("event_type").notNull(),        // identify, track, page, group, screen
  eventName: text("event_name"),                   // e.g. "Product Viewed"
  source: text("source").notNull(),                // web, ios, android, server, import
  channel: text("channel"),                        // email, sms, push, in_app, web
  data: jsonb("data").notNull().default({}),       // full CloudEvents + properties payload
  timestamp: timestamp("timestamp", { withTimezone: true }).notNull().defaultNow(),
  receivedAt: timestamp("received_at", { withTimezone: true }).notNull().defaultNow(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_events_org_customer").on(table.orgId, table.customerId, table.timestamp),
  index("idx_events_org_type").on(table.orgId, table.eventType, table.eventName),
]);
```

CloudEvents ingest handler:
```typescript
// src/server/services/event-ingestion/ingest.ts
export interface CloudEventEnvelope {
  specversion: string;
  id: string;
  source: string;
  type: string;
  time?: string;
  datacontenttype?: string;
  data: Record<string, unknown>;
}

export interface SegmentEvent {
  type: "identify" | "track" | "page" | "group" | "screen";
  userId?: string;
  anonymousId?: string;
  event?: string;          // for track events
  properties?: Record<string, unknown>;
  context?: Record<string, unknown>;
  timestamp?: string;
}

export async function ingestEvent(
  orgId: string,
  event: CloudEventEnvelope | SegmentEvent,
): Promise<{ eventId: string; customerId: string | null }> {
  // 1. Detect format (CloudEvents vs Segment)
  // 2. Resolve customer identity (anonymous_id -> customer_id)
  // 3. Store event in events table
  // 4. Enqueue journey trigger evaluation
  // 5. Return event ID and resolved customer ID
}
```

**Testing**:
- `test-2.2-cloudevent-ingest`: POST CloudEvents-formatted event to `/api/v1/events`; verify 201 and stored event
- `test-2.2-segment-track`: POST Segment track event; verify event_name and properties stored
- `test-2.2-segment-identify`: POST Segment identify event; verify customer profile upserted
- `test-2.2-invalid-event`: POST malformed event; verify 400 with validation errors
- `test-2.2-auth-required`: POST without API key returns 401

#### 2.3 — Identity Resolution Service

**What**: Service that resolves anonymous IDs to known customer profiles, merges duplicate identities, and maintains the identities JSONB array on the customer record.

**Design**:

```typescript
// src/server/services/event-ingestion/identity-resolver.ts
export interface IdentityResolutionResult {
  customerId: string;
  isNewCustomer: boolean;
  mergedFromIds: string[];
}

export async function resolveIdentity(
  db: DrizzleDB,
  orgId: string,
  identifiers: {
    userId?: string;
    anonymousId?: string;
    email?: string;
    phone?: string;
  },
): Promise<IdentityResolutionResult> {
  // 1. Look up by userId (external_id) first — strongest match
  // 2. Fall back to email or phone match
  // 3. Fall back to anonymous_id match via identities JSONB
  // 4. If no match, create new customer
  // 5. If anonymous_id matches existing customer and userId matches another, merge
  // 6. Return resolved customer ID
}
```

**Testing**:
- `test-2.3-resolve-by-userid`: Known userId resolves to existing customer
- `test-2.3-resolve-by-email`: Known email resolves to existing customer
- `test-2.3-create-new`: Unknown identifiers create a new customer record
- `test-2.3-anonymous-upgrade`: Anonymous customer upgraded to known when identify event arrives
- `test-2.3-merge-duplicate`: Two customers with same email are merged; events reassigned

#### 2.4 — Event Ingestion Worker

**What**: BullMQ worker that processes inbound events asynchronously for high-throughput scenarios. The REST endpoint enqueues events; the worker processes identity resolution and trigger evaluation.

**Design**:

```typescript
// src/server/lib/queue.ts
import { Queue } from "bullmq";
import { redis } from "./redis";

export const eventIngestQueue = new Queue("event-ingest", { connection: redis });
export const journeyTriggerQueue = new Queue("journey-trigger", { connection: redis });
export const messageDispatchQueue = new Queue("message-dispatch", { connection: redis });
export const segmentComputeQueue = new Queue("segment-compute", { connection: redis });
export const analyticsAggregateQueue = new Queue("analytics-aggregate", { connection: redis });

// src/server/workers/event-ingest.worker.ts
import { Worker, Job } from "bullmq";

interface EventIngestJob {
  orgId: string;
  event: CloudEventEnvelope | SegmentEvent;
  receivedAt: string;
}

const worker = new Worker<EventIngestJob>("event-ingest", async (job: Job<EventIngestJob>) => {
  const { orgId, event, receivedAt } = job.data;
  // 1. Resolve identity
  // 2. Store event
  // 3. Evaluate journey triggers (enqueue to journey-trigger queue)
  // 4. Evaluate segment membership changes (enqueue to segment-compute queue)
}, { connection: redis, concurrency: 10 });
```

**Testing**:
- `test-2.4-enqueue`: POST event enqueues to BullMQ; verify job appears in queue
- `test-2.4-process`: Worker picks up job, stores event, and resolves identity
- `test-2.4-retry`: Failed job retries up to 3 times with exponential backoff
- `test-2.4-concurrency`: 100 concurrent events processed without data corruption
- `test-2.4-idempotent`: Duplicate event ID (CloudEvents `id`) is deduplicated

---

## Phase 3: Segments & Audience Management

### Purpose
Build the segment definition engine with a visual rule builder and dynamic membership computation. After this phase, users can create segments with complex filter rules, and the system evaluates membership in near-real-time.

### Tasks

#### 3.1 — Segment Schema & Rule Definition

**What**: Drizzle schema for segments and segment_memberships. JSONB-based filter tree for segment rule definitions.

**Design**:

```typescript
// src/server/db/schema/segments.ts
export const segments = pgTable("segments", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  description: text("description"),
  segmentType: text("segment_type").notNull().default("dynamic"),
  definition: jsonb("definition").notNull().default({}),
  estimatedSize: bigint("estimated_size", { mode: "number" }),
  lastComputed: timestamp("last_computed", { withTimezone: true }),
  status: text("status").notNull().default("draft"),
  createdBy: uuid("created_by").references(() => orgMembers.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_segments_org").on(table.orgId, table.status),
]);

// Segment rule definition type
export interface SegmentCondition {
  field: string;                    // e.g. "attributes.plan", "events.Product Viewed"
  op: "eq" | "neq" | "gt" | "gte" | "lt" | "lte" | "contains" | "not_contains" |
      "exists" | "not_exists" | "count_gte" | "count_lte" | "occurred" | "not_occurred";
  value?: unknown;
  window?: string;                  // e.g. "30d", "7d" for event-based conditions
}

export interface SegmentRuleGroup {
  operator: "AND" | "OR";
  conditions: (SegmentCondition | SegmentRuleGroup)[];
}
```

**Testing**:
- `test-3.1-create-segment`: Create segment with filter definition via tRPC
- `test-3.1-nested-rules`: Segment with nested AND/OR groups parses correctly
- `test-3.1-validate-definition`: Invalid operator rejects with validation error
- `test-3.1-list-segments`: List segments filtered by status and org

#### 3.2 — Segment Evaluation Engine

**What**: Engine that evaluates segment definitions against customer records to compute membership. Supports attribute-based, event-based, and temporal conditions.

**Design**:

```typescript
// src/server/services/event-ingestion/segment-evaluator.ts
export async function evaluateSegment(
  db: DrizzleDB,
  orgId: string,
  segmentId: string,
): Promise<{ added: string[]; removed: string[]; totalSize: number }> {
  // 1. Load segment definition
  // 2. Translate filter tree to SQL WHERE clause (recursive)
  // 3. Query matching customers
  // 4. Diff against current memberships
  // 5. Insert new memberships, mark exited memberships
  // 6. Update estimated_size and last_computed
}

export async function evaluateCustomerSegments(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
): Promise<{ entered: string[]; exited: string[] }> {
  // Evaluate all active segments for a single customer (triggered by event ingestion)
}

function buildSegmentQuery(
  definition: SegmentRuleGroup,
  orgId: string,
): SQL {
  // Recursively build SQL from filter tree:
  // - attribute conditions -> JSONB containment or extraction
  // - event conditions -> subquery against events table with window
  // - nested groups -> AND/OR wrapping
}
```

**Testing**:
- `test-3.2-attribute-filter`: Segment with `attributes.plan = "premium"` matches correct customers
- `test-3.2-event-count-filter`: Segment with "Product Viewed count >= 3 in 30d" matches correct customers
- `test-3.2-nested-and-or`: Complex nested AND/OR conditions evaluate correctly
- `test-3.2-membership-add-remove`: Recomputation adds new members, removes no-longer-matching
- `test-3.2-per-customer-eval`: Single customer evaluation returns correct entered/exited segments

#### 3.3 — Segment Builder UI

**What**: Visual segment builder component with condition rows, operator selectors, and group nesting. Displays estimated audience size.

**Design**:

```typescript
// src/components/segments/segment-builder.tsx
interface SegmentBuilderProps {
  value: SegmentRuleGroup;
  onChange: (definition: SegmentRuleGroup) => void;
  orgId: string;
}

// src/components/segments/condition-row.tsx
interface ConditionRowProps {
  condition: SegmentCondition;
  onChange: (condition: SegmentCondition) => void;
  onRemove: () => void;
  availableFields: FieldDefinition[];
}
```

UI pattern: each condition row has three dropdowns (field, operator, value) with type-aware value inputs. Groups can be nested with AND/OR toggle. A "Preview" button triggers server-side estimation.

**Testing**:
- `test-3.3-add-condition`: Adding a condition row updates definition JSONB
- `test-3.3-nest-group`: Nesting a group within a group renders correctly
- `test-3.3-operator-options`: Field type determines available operators (string fields get eq/neq/contains; number fields get gt/gte/lt/lte)
- `test-3.3-preview-count`: Preview button displays estimated audience size
- `test-3.3-save-segment`: Saving persists definition to database via tRPC

---

## Phase 4: Journey Builder — Definition & Canvas

### Purpose
Build the visual journey builder canvas and the underlying journey definition data model. After this phase, users can create journey graphs with triggers, action steps, decision nodes, delays, and A/B splits using a drag-and-drop interface.

### Tasks

#### 4.1 — Journey Schema

**What**: Drizzle schema for journeys, journey_steps, and journey_edges tables. Represents the journey graph as an adjacency list.

**Design**:

```typescript
// src/server/db/schema/journeys.ts
export const journeys = pgTable("journeys", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  description: text("description"),
  status: text("status").notNull().default("draft"),
  version: integer("version").notNull().default(1),
  parentId: uuid("parent_id").references((): AnyPgColumn => journeys.id),
  entryConfig: jsonb("entry_config").notNull().default({}),
  goalConfig: jsonb("goal_config"),
  schedule: jsonb("schedule"),
  frequencyCap: jsonb("frequency_cap"),
  tags: text("tags").array().default([]),
  canvas: jsonb("canvas").notNull().default({}),
  createdBy: uuid("created_by").references(() => orgMembers.id),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_journeys_org").on(table.orgId, table.status),
]);

// Journey entry configuration type
export interface JourneyEntryConfig {
  type: "segment_entry" | "event" | "api_call" | "scheduled";
  segmentId?: string;
  eventName?: string;
  schedule?: string;         // cron expression
  reEntry: boolean;
  rateLimit?: { max: number; per: "hour" | "day" };
}

// src/server/db/schema/journey-steps.ts
export const journeySteps = pgTable("journey_steps", {
  id: uuid("id").primaryKey().defaultRandom(),
  journeyId: uuid("journey_id").notNull().references(() => journeys.id, { onDelete: "cascade" }),
  stepType: text("step_type").notNull(),
  name: text("name").notNull(),
  config: jsonb("config").notNull().default({}),
  positionX: integer("position_x"),
  positionY: integer("position_y"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_steps_journey").on(table.journeyId),
]);

export type StepType = "action_email" | "action_sms" | "action_push" | "action_webhook" |
                        "decision" | "delay" | "ab_split" | "exit";

// src/server/db/schema/journey-edges.ts
export const journeyEdges = pgTable("journey_edges", {
  id: uuid("id").primaryKey().defaultRandom(),
  journeyId: uuid("journey_id").notNull().references(() => journeys.id, { onDelete: "cascade" }),
  fromStepId: uuid("from_step_id").notNull().references(() => journeySteps.id, { onDelete: "cascade" }),
  toStepId: uuid("to_step_id").notNull().references(() => journeySteps.id, { onDelete: "cascade" }),
  condition: jsonb("condition"),
  label: text("label"),
  sortOrder: integer("sort_order").notNull().default(0),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_journey_edges").on(table.journeyId, table.fromStepId, table.toStepId),
  index("idx_edges_from").on(table.fromStepId),
]);
```

**Testing**:
- `test-4.1-create-journey`: Create journey with entry config via tRPC
- `test-4.1-add-steps`: Add multiple step types to a journey
- `test-4.1-add-edges`: Connect steps with edges; verify uniqueness constraint
- `test-4.1-get-graph`: Retrieve full journey graph (steps + edges) in single query
- `test-4.1-versioning`: Creating new version clones steps and edges with parent_id link

#### 4.2 — Journey Canvas UI (React Flow)

**What**: Visual journey canvas using React Flow v12. Users drag step types from a palette onto the canvas, connect them with edges, and configure each step via a side panel.

**Design**:

```typescript
// src/components/journey/canvas.tsx
import { ReactFlow, Node, Edge, Controls, MiniMap, Background } from "@xyflow/react";

interface JourneyCanvasProps {
  journeyId: string;
  initialSteps: JourneyStepWithPosition[];
  initialEdges: JourneyEdgeData[];
  onSave: (steps: JourneyStepWithPosition[], edges: JourneyEdgeData[]) => void;
  readOnly?: boolean;
}

// Custom node types registered with React Flow
const nodeTypes = {
  action_email: ActionEmailNode,
  action_sms: ActionSmsNode,
  action_push: ActionPushNode,
  action_webhook: ActionWebhookNode,
  decision: DecisionNode,
  delay: DelayNode,
  ab_split: AbSplitNode,
  exit: ExitNode,
};
```

```typescript
// src/components/journey/step-palette.tsx
interface StepDefinition {
  type: StepType;
  label: string;
  icon: React.ComponentType;
  category: "action" | "flow_control";
  description: string;
}

const STEP_DEFINITIONS: StepDefinition[] = [
  { type: "action_email", label: "Send Email", icon: MailIcon, category: "action", description: "Send an email using a template" },
  { type: "action_sms", label: "Send SMS", icon: MessageIcon, category: "action", description: "Send an SMS message" },
  { type: "action_push", label: "Push Notification", icon: BellIcon, category: "action", description: "Send a push notification" },
  { type: "action_webhook", label: "Webhook", icon: WebhookIcon, category: "action", description: "Call an external webhook" },
  { type: "decision", label: "Decision", icon: GitBranchIcon, category: "flow_control", description: "Branch based on a condition" },
  { type: "delay", label: "Wait", icon: ClockIcon, category: "flow_control", description: "Wait for a duration or until a time" },
  { type: "ab_split", label: "A/B Split", icon: SplitIcon, category: "flow_control", description: "Split traffic between paths" },
  { type: "exit", label: "Exit", icon: LogOutIcon, category: "flow_control", description: "End the journey" },
];
```

**Testing**:
- `test-4.2-render-canvas`: Journey canvas renders with initial steps and edges
- `test-4.2-drag-drop`: Dragging step from palette onto canvas creates new node
- `test-4.2-connect-nodes`: Drawing edge between compatible nodes creates connection
- `test-4.2-config-panel`: Clicking node opens configuration panel with type-specific fields
- `test-4.2-save-graph`: Save button persists graph state to database via tRPC
- `test-4.2-minimap`: Minimap displays overview of complex journey with many steps

#### 4.3 — Step Configuration Panels

**What**: Right-hand configuration panel that renders type-specific forms for each step type. Email steps select a template; decision steps define conditions; delay steps configure duration.

**Design**:

```typescript
// src/components/journey/step-config-panel.tsx
interface StepConfigPanelProps {
  step: JourneyStep;
  onUpdate: (config: Record<string, unknown>) => void;
  onClose: () => void;
}

// Email step config
interface EmailStepConfig {
  templateId: string;
  variantIds?: string[];
  sendTimeOptimization: boolean;
  suppressIfSentWithin?: string;    // e.g. "24h"
}

// Decision step config
interface DecisionStepConfig {
  condition: SegmentCondition;
  timeout: string;                    // e.g. "72h"
  timeoutPath: "yes" | "no";
}

// Delay step config
interface DelayStepConfig {
  duration?: string;                  // e.g. "2d"
  untilTime?: string;                // e.g. "09:00"
  timezoneSource: "customer" | "organisation" | "utc";
}

// A/B split config
interface AbSplitConfig {
  paths: { label: string; weight: number }[];
}
```

**Testing**:
- `test-4.3-email-config`: Email step panel shows template selector; selected template ID saved to config
- `test-4.3-decision-config`: Decision step panel renders condition builder matching segment rule format
- `test-4.3-delay-config`: Delay step supports both duration and until-time modes
- `test-4.3-ab-split-weights`: A/B split weights must sum to 100; validation prevents invalid splits
- `test-4.3-config-persistence`: Config changes auto-save on panel close

#### 4.4 — Journey Validation

**What**: Server-side validation that a journey graph is well-formed before publishing: all paths reach an exit or loop, no disconnected nodes, required step configs are complete.

**Design**:

```typescript
// src/server/services/journey-engine/validator.ts
export interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

export interface ValidationError {
  type: "disconnected_node" | "missing_exit" | "missing_config" | "cycle_without_exit" |
        "invalid_edge" | "missing_trigger" | "empty_journey";
  stepId?: string;
  message: string;
}

export async function validateJourney(
  db: DrizzleDB,
  journeyId: string,
): Promise<ValidationResult> {
  // 1. Load full graph (steps + edges)
  // 2. Check entry point exists (at least one trigger)
  // 3. DFS/BFS to find disconnected nodes
  // 4. Check all paths reach an exit node or a well-defined loop
  // 5. Check each step has required config fields for its type
  // 6. Check referenced templates and segments exist
  // 7. Return errors and warnings
}
```

**Testing**:
- `test-4.4-valid-journey`: Well-formed journey passes validation
- `test-4.4-disconnected-node`: Journey with orphan node returns disconnected_node error
- `test-4.4-missing-exit`: Path without exit node returns missing_exit error
- `test-4.4-missing-config`: Email step without template returns missing_config error
- `test-4.4-empty-journey`: Journey with no steps returns empty_journey error
- `test-4.4-missing-trigger`: Journey without entry trigger returns missing_trigger error

---

## Phase 5: Channels & Message Templates

### Purpose
Build the multi-channel messaging infrastructure and template system. After this phase, users can configure channel providers (email, SMS, push), create message templates with Liquid variable interpolation, and manage A/B message variants.

### Tasks

#### 5.1 — Channel Provider Abstraction

**What**: Abstract channel provider interface with concrete implementations for SendGrid (email), Twilio (SMS), and FCM (push). Provider credentials encrypted at rest.

**Design**:

```typescript
// src/server/services/channels/provider-interface.ts
export interface ChannelProvider {
  readonly type: ChannelType;
  readonly name: string;

  send(message: ChannelMessage): Promise<SendResult>;
  validateConfig(config: Record<string, unknown>): Promise<boolean>;
  parseWebhook(payload: unknown): Promise<DeliveryEvent>;
}

export type ChannelType = "email" | "sms" | "push" | "in_app" | "webhook";

export interface ChannelMessage {
  to: string;                       // email, phone, device token
  templateRendered: RenderedContent;
  metadata: Record<string, unknown>;
}

export interface SendResult {
  success: boolean;
  providerId: string;               // provider-side message ID
  error?: string;
}

export interface DeliveryEvent {
  providerId: string;
  status: "delivered" | "bounced" | "failed" | "opened" | "clicked" | "unsubscribed" | "complained";
  timestamp: Date;
  metadata?: Record<string, unknown>;
}
```

```typescript
// src/server/services/channels/email/sendgrid.ts
export class SendGridProvider implements ChannelProvider {
  readonly type = "email";
  readonly name = "SendGrid";

  constructor(private apiKey: string) {}

  async send(message: ChannelMessage): Promise<SendResult> { /* SendGrid v3 API call */ }
  async validateConfig(config: Record<string, unknown>): Promise<boolean> { /* verify API key */ }
  async parseWebhook(payload: unknown): Promise<DeliveryEvent> { /* parse Event Webhook */ }
}
```

**Testing**:
- `test-5.1-sendgrid-send`: SendGrid provider sends email; returns provider message ID
- `test-5.1-twilio-send`: Twilio provider sends SMS; returns provider message SID
- `test-5.1-fcm-send`: FCM provider sends push notification; returns message ID
- `test-5.1-invalid-config`: Invalid API key fails validateConfig
- `test-5.1-webhook-parse`: SendGrid webhook payload parsed into DeliveryEvent
- `test-5.1-provider-factory`: ChannelProviderFactory returns correct provider for channel type

#### 5.2 — Message Template System

**What**: Template CRUD with Liquid template rendering, per-channel content structure, and A/B variant support.

**Design**:

```typescript
// src/server/db/schema/message-templates.ts
export const messageTemplates = pgTable("message_templates", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  channelType: text("channel_type").notNull(),
  name: text("name").notNull(),
  content: jsonb("content").notNull().default({}),
  variants: jsonb("variants").notNull().default([]),
  version: integer("version").notNull().default(1),
  status: text("status").notNull().default("draft"),
  createdBy: uuid("created_by").references(() => orgMembers.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_templates_org").on(table.orgId, table.channelType, table.status),
]);

// Template content types by channel
export interface EmailContent {
  subject: string;
  bodyHtml: string;
  bodyText: string;
  previewText?: string;
  fromName: string;
  fromAddress: string;
  replyTo?: string;
}

export interface SmsContent {
  body: string;
  mediaUrl?: string;
}

export interface PushContent {
  title: string;
  body: string;
  imageUrl?: string;
  deepLink?: string;
  badgeCount?: number;
}
```

Liquid template renderer:
```typescript
// src/server/services/channels/email/renderer.ts
import { Liquid } from "liquidjs";

export interface TemplateContext {
  customer: {
    firstName?: string;
    lastName?: string;
    email?: string;
    attributes: Record<string, unknown>;
  };
  org: {
    name: string;
  };
  event?: Record<string, unknown>;
  unsubscribeUrl: string;
}

export async function renderTemplate(
  template: string,
  context: TemplateContext,
): Promise<string> {
  const engine = new Liquid();
  return engine.parseAndRender(template, context);
}
```

**Testing**:
- `test-5.2-create-email-template`: Create email template with subject, body, from fields
- `test-5.2-liquid-render`: Template with `{{ customer.firstName }}` renders correctly
- `test-5.2-variants`: Template with two variants stores weights and alternate content
- `test-5.2-sms-template`: SMS template with character count validation
- `test-5.2-preview`: Preview endpoint renders template with sample data
- `test-5.2-missing-variable`: Template with undefined variable renders empty string (not error)

#### 5.3 — Template Editor UI

**What**: Template editor component with live preview, Liquid variable autocompletion, and channel-specific field layouts.

**Design**:

```typescript
// src/components/templates/template-editor.tsx
interface TemplateEditorProps {
  template: MessageTemplate;
  onSave: (template: MessageTemplate) => void;
  channelType: ChannelType;
}

// Email-specific editor with subject, body (rich text), preview text, from fields
// SMS-specific editor with body, character counter, segment counter
// Push-specific editor with title, body, image URL, deep link
```

**Testing**:
- `test-5.3-email-editor`: Email editor renders subject, body, preview text, from fields
- `test-5.3-live-preview`: Changes in editor update live preview in real-time
- `test-5.3-variable-insert`: Variable picker inserts `{{ customer.firstName }}` at cursor
- `test-5.3-sms-char-count`: SMS editor shows remaining characters (160 limit per segment)
- `test-5.3-variant-editor`: Variant tab allows adding/removing A/B variants with weight sliders

---

## Phase 6: Journey Execution Engine

### Purpose
Build the core journey execution engine that enrolls customers, advances them through steps, evaluates decisions, dispatches messages, and tracks execution state. This is the heart of the platform.

### Tasks

#### 6.1 — Enrollment Service

**What**: Service that enrolls customers into journeys based on trigger events (segment entry, event received, API call, schedule). Enforces frequency caps and re-entry rules.

**Design**:

```typescript
// src/server/db/schema/journey-enrollments.ts
export const journeyEnrollments = pgTable("journey_enrollments", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  journeyId: uuid("journey_id").notNull().references(() => journeys.id, { onDelete: "cascade" }),
  customerId: uuid("customer_id").notNull().references(() => customers.id, { onDelete: "cascade" }),
  currentStepId: uuid("current_step_id").references(() => journeySteps.id),
  status: text("status").notNull().default("active"),
  context: jsonb("context").notNull().default({}),
  enteredAt: timestamp("entered_at", { withTimezone: true }).notNull().defaultNow(),
  exitedAt: timestamp("exited_at", { withTimezone: true }),
  exitReason: text("exit_reason"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_enrollment").on(table.orgId, table.journeyId, table.customerId),
  index("idx_enrollments_journey").on(table.journeyId, table.status),
  index("idx_enrollments_customer").on(table.customerId),
]);

// src/server/services/journey-engine/enrollment.ts
export async function enrollCustomer(
  db: DrizzleDB,
  orgId: string,
  journeyId: string,
  customerId: string,
  triggerEvent?: Record<string, unknown>,
): Promise<{ enrollmentId: string } | { rejected: string }> {
  // 1. Check journey is active
  // 2. Check customer not already enrolled (unless re-entry allowed)
  // 3. Check frequency cap not exceeded
  // 4. Check consent (customer has consent for journey's channels)
  // 5. Create enrollment record
  // 6. Set current_step_id to first step after trigger
  // 7. Enqueue first step execution
  // 8. Return enrollment ID or rejection reason
}
```

**Testing**:
- `test-6.1-enroll-success`: Customer enrolled in active journey; enrollment record created
- `test-6.1-reject-inactive`: Enrollment in draft/paused journey is rejected
- `test-6.1-reject-duplicate`: Duplicate enrollment (re-entry=false) is rejected
- `test-6.1-frequency-cap`: Enrollment rejected when frequency cap exceeded
- `test-6.1-consent-check`: Enrollment rejected for customer without required consent
- `test-6.1-trigger-context`: Trigger event data stored in enrollment context

#### 6.2 — Step Executor

**What**: Core execution engine that processes journey steps. For each step type, the executor performs the appropriate action and advances the customer to the next step.

**Design**:

```typescript
// src/server/services/journey-engine/executor.ts
export interface StepExecutionResult {
  status: "completed" | "failed" | "waiting";
  nextStepId?: string;
  result?: Record<string, unknown>;
  error?: string;
}

export async function executeStep(
  db: DrizzleDB,
  enrollmentId: string,
  stepId: string,
): Promise<StepExecutionResult> {
  // 1. Load enrollment and step
  // 2. Dispatch to type-specific executor:
  //    - action_email -> render template, check consent, enqueue message dispatch
  //    - action_sms -> render template, check consent, enqueue message dispatch
  //    - action_push -> render template, check consent, enqueue message dispatch
  //    - action_webhook -> HTTP POST to configured URL
  //    - decision -> evaluate condition, return yes/no path
  //    - delay -> schedule wake-up job, return "waiting"
  //    - ab_split -> random assignment based on weights, return selected path
  //    - exit -> mark enrollment completed
  // 3. Update enrollment current_step_id
  // 4. Enqueue next step execution (if not waiting/exit)
}
```

```typescript
// src/server/services/journey-engine/evaluator.ts
export async function evaluateDecision(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  condition: SegmentCondition,
): Promise<boolean> {
  // Evaluate a single condition against a customer's current state
  // Uses same condition evaluation logic as segment engine
}
```

**Testing**:
- `test-6.2-execute-email`: Email step renders template and enqueues message dispatch
- `test-6.2-execute-decision-yes`: Decision with true condition returns yes-path step ID
- `test-6.2-execute-decision-no`: Decision with false condition returns no-path step ID
- `test-6.2-execute-delay`: Delay step returns "waiting" and schedules wake-up
- `test-6.2-execute-ab-split`: A/B split distributes roughly according to weights over 1000 runs
- `test-6.2-execute-exit`: Exit step marks enrollment as completed
- `test-6.2-consent-block`: Action step skips message dispatch when consent not granted
- `test-6.2-webhook-step`: Webhook step sends HTTP POST with customer context

#### 6.3 — Message Dispatch Worker

**What**: BullMQ worker that picks up message dispatch jobs, selects A/B variants, renders templates, checks suppression lists, and dispatches via the appropriate channel provider.

**Design**:

```typescript
// src/server/db/schema/message-sends.ts
export const messageSends = pgTable("message_sends", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  customerId: uuid("customer_id").notNull().references(() => customers.id, { onDelete: "cascade" }),
  enrollmentId: uuid("enrollment_id").references(() => journeyEnrollments.id),
  templateId: uuid("template_id").references(() => messageTemplates.id),
  channelType: text("channel_type").notNull(),
  status: text("status").notNull().default("queued"),
  delivery: jsonb("delivery").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_sends_customer").on(table.customerId, table.createdAt),
  index("idx_sends_org_status").on(table.orgId, table.status),
]);

// src/server/workers/message-dispatch.worker.ts
interface MessageDispatchJob {
  orgId: string;
  customerId: string;
  enrollmentId?: string;
  templateId: string;
  channelType: ChannelType;
  context: TemplateContext;
}

// Worker flow:
// 1. Load template and select variant (weighted random)
// 2. Render template with Liquid engine
// 3. Check suppression list
// 4. Check consent for channel type
// 5. Get channel provider
// 6. Dispatch message
// 7. Record message_send with provider response
// 8. Return success/failure
```

**Testing**:
- `test-6.3-dispatch-email`: Worker dispatches email via SendGrid; message_send recorded
- `test-6.3-dispatch-sms`: Worker dispatches SMS via Twilio; message_send recorded
- `test-6.3-suppressed`: Message to suppressed email is skipped; status set to "suppressed"
- `test-6.3-no-consent`: Message without consent is skipped; status set to "consent_denied"
- `test-6.3-variant-selection`: Variant selection distributes according to weights
- `test-6.3-provider-error`: Provider failure sets status to "failed" with error message
- `test-6.3-retry`: Failed dispatch retries up to 3 times

#### 6.4 — Journey Trigger Evaluation

**What**: Service that evaluates whether an inbound event or segment change should trigger journey enrollments. Runs after every event ingestion.

**Design**:

```typescript
// src/server/services/journey-engine/trigger-evaluator.ts
export async function evaluateTriggers(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  event: { type: string; name?: string; properties?: Record<string, unknown> },
  segmentChanges?: { entered: string[]; exited: string[] },
): Promise<{ enrolled: string[]; skipped: Array<{ journeyId: string; reason: string }> }> {
  // 1. Find all active journeys for this org
  // 2. For each journey, check if trigger matches:
  //    - segment_entry: customer entered a segment that matches trigger
  //    - event: event type/name matches trigger filter
  //    - api_call: only via explicit API enrollment
  // 3. Attempt enrollment for matching journeys
  // 4. Return list of enrolled journey IDs and skip reasons
}
```

**Testing**:
- `test-6.4-segment-trigger`: Customer entering segment triggers journey with segment_entry trigger
- `test-6.4-event-trigger`: Track event "Purchase Completed" triggers matching journey
- `test-6.4-no-match`: Event that matches no triggers enrolls in no journeys
- `test-6.4-multiple-triggers`: Single event triggers enrollment in multiple journeys
- `test-6.4-trigger-filter`: Journey trigger with event name filter only matches specific events

---

## Phase 7: Consent & Privacy Compliance

### Purpose
Build consent management, suppression lists, and data subject request handling. After this phase, every message dispatch is consent-gated, GDPR/CCPA erasure requests are processed, and full consent audit trails are maintained.

### Tasks

#### 7.1 — Consent Engine

**What**: Consent recording, lookup, and enforcement. Consent state stored as audit trail in consent_log and denormalised on customer.consent JSONB for fast dispatch-time lookup.

**Design**:

```typescript
// src/server/db/schema/consent-log.ts
export const consentLog = pgTable("consent_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  customerId: uuid("customer_id").notNull().references(() => customers.id, { onDelete: "cascade" }),
  consentType: text("consent_type").notNull(),
  action: text("action").notNull(),     // granted, withdrawn
  details: jsonb("details").notNull().default({}),
  occurredAt: timestamp("occurred_at", { withTimezone: true }).notNull().defaultNow(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_consent_customer").on(table.customerId, table.consentType),
]);

// src/server/services/consent/consent-engine.ts
export type ConsentType = "marketing_email" | "marketing_sms" | "marketing_push" | "tcf";

export async function grantConsent(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  consentType: ConsentType,
  details: { source: string; ipAddress?: string; userAgent?: string; tcfString?: string },
): Promise<void> {
  // 1. Insert consent_log record
  // 2. Update customer.consent JSONB
}

export async function withdrawConsent(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  consentType: ConsentType,
  details: { source: string },
): Promise<void> {
  // 1. Insert consent_log record with action=withdrawn
  // 2. Update customer.consent JSONB
  // 3. Add to suppression list
  // 4. Exit all active journey enrollments for this channel
}

export async function checkConsent(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  consentType: ConsentType,
): Promise<boolean> {
  // Fast lookup via customer.consent JSONB containment query
}
```

**Testing**:
- `test-7.1-grant-consent`: Grant email consent; consent_log recorded; customer.consent updated
- `test-7.1-withdraw-consent`: Withdraw SMS consent; suppression list updated; active enrollments exited
- `test-7.1-check-consent-granted`: checkConsent returns true for granted consent
- `test-7.1-check-consent-denied`: checkConsent returns false for withdrawn/missing consent
- `test-7.1-audit-trail`: Multiple consent changes create sequential audit records

#### 7.2 — Suppression List Management

**What**: Suppression list CRUD and enforcement. Supports manual suppression, bounce-based suppression, and complaint-based suppression.

**Design**:

```typescript
// src/server/db/schema/suppression-list.ts
export const suppressionList = pgTable("suppression_list", {
  orgId: uuid("org_id").notNull(),
  channelType: text("channel_type").notNull(),
  identifier: text("identifier").notNull(),
  reason: text("reason").notNull(),    // unsubscribed, bounced, complained, manual
  suppressedAt: timestamp("suppressed_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  primaryKey({ columns: [table.orgId, table.channelType, table.identifier] }),
]);

export async function isSuppressed(
  db: DrizzleDB,
  orgId: string,
  channelType: ChannelType,
  identifier: string,
): Promise<boolean> {
  // Direct primary key lookup — O(1)
}
```

**Testing**:
- `test-7.2-add-suppression`: Add email to suppression list; subsequent lookups return true
- `test-7.2-remove-suppression`: Remove from suppression; lookups return false
- `test-7.2-bounce-auto-suppress`: Bounced email delivery webhook auto-adds to suppression
- `test-7.2-complaint-auto-suppress`: Spam complaint webhook auto-adds to suppression
- `test-7.2-bulk-import`: CSV import of suppression list entries

#### 7.3 — Data Subject Request Processing

**What**: GDPR right-to-erasure, right-to-access, and CCPA opt-out-of-sale request processing via a background worker.

**Design**:

```typescript
// src/server/services/consent/dsr-processor.ts
export type DsrType = "access" | "erasure" | "portability" | "opt_out_sale";

export async function processDsr(
  db: DrizzleDB,
  orgId: string,
  customerId: string,
  requestType: DsrType,
): Promise<{ status: "completed" | "failed"; response?: Record<string, unknown> }> {
  switch (requestType) {
    case "erasure":
      // 1. Exit all active journey enrollments
      // 2. Delete all events for this customer
      // 3. Delete all message_sends for this customer
      // 4. Delete customer record (CASCADE to consent_log, segment_memberships)
      // 5. Add identifiers to suppression lists
      break;
    case "access":
      // 1. Export customer profile
      // 2. Export all events
      // 3. Export all message sends
      // 4. Export consent history
      // 5. Return as JSON
      break;
    case "portability":
      // Same as access but formatted per GDPR Article 20
      break;
    case "opt_out_sale":
      // CCPA: mark customer data as not-for-sale; withdraw marketing consent
      break;
  }
}
```

**Testing**:
- `test-7.3-erasure`: Erasure request deletes customer and all associated data
- `test-7.3-erasure-preserves-suppression`: After erasure, identifier remains on suppression list
- `test-7.3-access-export`: Access request returns complete customer data export
- `test-7.3-portability-format`: Portability export in machine-readable JSON
- `test-7.3-opt-out-sale`: CCPA opt-out withdraws all marketing consent

---

## Phase 8: Analytics & A/B Testing

### Purpose
Build journey analytics, funnel tracking, A/B experiment management, and conversion tracking. After this phase, users can view journey performance dashboards, run message-level A/B tests, and track conversion events.

### Tasks

#### 8.1 — Analytics Aggregation

**What**: Background worker that aggregates journey performance metrics into daily stats. Pre-computes enrollment counts, message delivery rates, open/click rates, and conversion metrics.

**Design**:

```typescript
// src/server/db/schema/journey-stats.ts
export const journeyStats = pgTable("journey_stats", {
  orgId: uuid("org_id").notNull(),
  journeyId: uuid("journey_id").notNull().references(() => journeys.id, { onDelete: "cascade" }),
  statDate: date("stat_date").notNull(),
  metrics: jsonb("metrics").notNull().default({}),
}, (table) => [
  primaryKey({ columns: [table.orgId, table.journeyId, table.statDate] }),
]);

// Metrics JSONB structure
interface JourneyDailyMetrics {
  enrollments: number;
  completions: number;
  exits: number;
  goalsMet: number;
  messages: {
    sent: number;
    delivered: number;
    opened: number;
    clicked: number;
    bounced: number;
    unsubscribed: number;
  };
  conversions: {
    count: number;
    value: number;
    currency: string;
  };
  byChannel: Record<ChannelType, {
    sent: number;
    delivered: number;
    opened: number;
    clicked: number;
  }>;
}
```

**Testing**:
- `test-8.1-daily-aggregation`: Worker computes correct daily stats from message_sends and enrollments
- `test-8.1-idempotent`: Running aggregation twice for same date produces same results
- `test-8.1-by-channel`: Channel breakdown correctly splits email vs SMS vs push
- `test-8.1-conversion-value`: Conversion values summed correctly

#### 8.2 — A/B Experiment Management

**What**: A/B experiment CRUD, variant assignment, and statistical analysis (confidence intervals, winner declaration).

**Design**:

```typescript
// src/server/db/schema/ab-experiments.ts
export const abExperiments = pgTable("ab_experiments", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  journeyId: uuid("journey_id").references(() => journeys.id),
  name: text("name").notNull(),
  experimentType: text("experiment_type").notNull(),
  status: text("status").notNull().default("draft"),
  config: jsonb("config").notNull().default({}),
  winnerVariant: text("winner_variant"),
  startedAt: timestamp("started_at", { withTimezone: true }),
  endedAt: timestamp("ended_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_experiments_org").on(table.orgId, table.status),
]);

// Statistical significance calculation
export function calculateSignificance(
  controlSends: number,
  controlConversions: number,
  variantSends: number,
  variantConversions: number,
): { significant: boolean; confidence: number; lift: number } {
  // Two-proportion z-test
  // Returns significance at 95% confidence threshold
}
```

**Testing**:
- `test-8.2-create-experiment`: Create A/B experiment linked to journey
- `test-8.2-variant-assignment`: Traffic splits correctly according to weights
- `test-8.2-significance-positive`: Significant result detected with sufficient data
- `test-8.2-significance-underpowered`: Insufficient data correctly returns not-significant
- `test-8.2-declare-winner`: Winner declared when confidence threshold met
- `test-8.2-stop-experiment`: Stopped experiment freezes variant assignment

#### 8.3 — Analytics Dashboard UI

**What**: Journey analytics dashboard with overview metrics, funnel visualization, channel breakdown charts, and A/B experiment results.

**Design**:

```typescript
// src/components/analytics/overview-dashboard.tsx
interface OverviewDashboardProps {
  orgId: string;
  dateRange: { from: Date; to: Date };
}

// Components:
// - MetricCard: enrollments, completions, conversion rate, revenue
// - JourneyFunnel: step-by-step drop-off visualization (Recharts)
// - ChannelBreakdown: email/sms/push performance comparison (bar chart)
// - ExperimentResults: A/B test variant comparison with confidence intervals
```

**Testing**:
- `test-8.3-dashboard-render`: Dashboard renders with correct metric values
- `test-8.3-date-range-filter`: Changing date range updates all charts
- `test-8.3-funnel-chart`: Funnel shows step-by-step drop-off percentages
- `test-8.3-empty-state`: Dashboard shows empty state for journey with no enrollments
- `test-8.3-experiment-results`: Experiment results show variant comparison with lift percentage

#### 8.4 — Delivery Webhook Processing

**What**: Webhook endpoints that receive delivery status updates from channel providers (SendGrid, Twilio, FCM) and update message_sends delivery tracking.

**Design**:

```typescript
// src/app/api/webhooks/sendgrid/route.ts
export async function POST(request: Request): Promise<Response> {
  // 1. Verify SendGrid webhook signature
  // 2. Parse event array (delivered, opened, clicked, bounced, etc.)
  // 3. Update message_sends.delivery JSONB for each event
  // 4. Auto-suppress on bounce or complaint
  // 5. Trigger analytics re-aggregation
}
```

**Testing**:
- `test-8.4-sendgrid-delivered`: Delivery event updates message_send status to delivered
- `test-8.4-sendgrid-opened`: Open event records timestamp in delivery JSONB
- `test-8.4-sendgrid-clicked`: Click event records clicked URL in delivery JSONB
- `test-8.4-sendgrid-bounced`: Bounce event triggers auto-suppression
- `test-8.4-signature-invalid`: Invalid webhook signature returns 401
- `test-8.4-twilio-status`: Twilio delivery status callback updates SMS message_send

---

## Phase 9: Integrations & External Data

### Purpose
Build the integration framework for connecting to CRMs, CDPs, and data warehouses. After this phase, users can configure Segment as an event source, sync customer data from Salesforce, and export data to Snowflake.

### Tasks

#### 9.1 — Integration Framework

**What**: Generic integration CRUD with encrypted credential storage, connection testing, and sync scheduling.

**Design**:

```typescript
// src/server/db/schema/integrations.ts
export const integrations = pgTable("integrations", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  provider: text("provider").notNull(),
  name: text("name").notNull(),
  config: jsonb("config").notNull().default({}),
  credentials: jsonb("credentials").notNull().default({}),  // encrypted at app layer
  syncState: jsonb("sync_state").notNull().default({}),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// src/server/services/integrations/integration-interface.ts
export interface IntegrationProvider {
  readonly provider: string;
  testConnection(config: Record<string, unknown>, credentials: Record<string, unknown>): Promise<boolean>;
  sync(integration: Integration, since: Date): Promise<SyncResult>;
}

export interface SyncResult {
  recordsSynced: number;
  errors: Array<{ record: string; error: string }>;
  cursor?: string;
}
```

**Testing**:
- `test-9.1-create-integration`: Create Segment integration with encrypted credentials
- `test-9.1-test-connection`: Test connection validates API key with provider
- `test-9.1-credentials-encrypted`: Credentials stored encrypted; decrypted only at use
- `test-9.1-list-integrations`: List integrations for org shows status and last sync

#### 9.2 — Segment CDP Integration

**What**: Receive events from Twilio Segment via webhook and write key, mapping Segment Spec events directly into the event ingestion pipeline.

**Design**:

```typescript
// src/server/services/integrations/segment.ts
export class SegmentIntegration implements IntegrationProvider {
  readonly provider = "segment";

  async testConnection(config: Record<string, unknown>): Promise<boolean> {
    // Verify write key format
  }

  async handleWebhook(writeKey: string, events: SegmentEvent[]): Promise<void> {
    // 1. Validate write key against integration config
    // 2. Map each event through event ingestion pipeline
    // 3. identify -> upsert customer profile
    // 4. track -> store as event, evaluate triggers
    // 5. page/screen -> store as event
    // 6. group -> update customer group membership
  }
}
```

**Testing**:
- `test-9.2-segment-identify`: Segment identify call creates/updates customer
- `test-9.2-segment-track`: Segment track call stored as event and triggers evaluated
- `test-9.2-segment-batch`: Segment batch endpoint processes multiple events
- `test-9.2-invalid-write-key`: Invalid write key returns 401

#### 9.3 — Webhook Outbound

**What**: Configurable outbound webhooks that fire on platform events (journey completed, message delivered, customer created). HMAC-signed payloads.

**Design**:

```typescript
// src/server/services/webhook-dispatch.ts
export async function dispatchWebhook(
  orgId: string,
  eventType: string,
  payload: Record<string, unknown>,
): Promise<void> {
  // 1. Find all active webhooks for this org subscribed to this event type
  // 2. For each webhook: sign payload with HMAC, POST to URL
  // 3. Record delivery attempt (success/failure)
  // 4. Retry on failure with exponential backoff
}
```

**Testing**:
- `test-9.3-webhook-dispatch`: Platform event triggers POST to configured webhook URL
- `test-9.3-hmac-signature`: Payload signed with HMAC-SHA256; signature in header
- `test-9.3-retry-on-failure`: Failed webhook retried up to 3 times
- `test-9.3-event-filter`: Webhook only receives subscribed event types

---

## Phase 10: AI-Powered Features

### Purpose
Build AI-native features: autonomous journey design from natural language, generative message variants, and journey anomaly detection. These are the key differentiators versus incumbent platforms.

### Tasks

#### 10.1 — AI Journey Designer

**What**: Natural language interface where users describe a business goal (e.g., "reduce churn in trial users") and the LLM generates a complete journey graph with steps, conditions, and template suggestions.

**Design**:

```typescript
// src/server/services/ai/journey-designer.ts
import Anthropic from "@anthropic-ai/sdk";

export interface JourneyDesignRequest {
  goal: string;                       // "reduce churn in trial users"
  context?: {
    availableSegments: string[];
    availableChannels: ChannelType[];
    existingTemplates: string[];
  };
}

export interface JourneyDesignResult {
  journeyName: string;
  description: string;
  steps: Array<{
    type: StepType;
    name: string;
    config: Record<string, unknown>;
    position: { x: number; y: number };
  }>;
  edges: Array<{
    from: number;    // step index
    to: number;      // step index
    condition?: SegmentCondition;
    label?: string;
  }>;
  suggestedSegment?: SegmentRuleGroup;
  suggestedTemplates?: Array<{
    channelType: ChannelType;
    subject?: string;
    body: string;
  }>;
}

export async function designJourney(
  request: JourneyDesignRequest,
): Promise<JourneyDesignResult> {
  // 1. Build system prompt with journey design constraints and available resources
  // 2. Call Claude with structured output (Zod schema)
  // 3. Validate generated graph (well-formed DAG, all paths reach exit)
  // 4. Return structured journey definition
}
```

**Testing**:
- `test-10.1-design-churn-journey`: "reduce churn" goal generates journey with email + delay + decision
- `test-10.1-design-onboarding`: "onboard new users" goal generates multi-step welcome journey
- `test-10.1-valid-graph`: Generated journey passes graph validation
- `test-10.1-respects-channels`: Generated journey only uses available channels
- `test-10.1-structured-output`: Claude response matches JourneyDesignResult schema

#### 10.2 — Generative Message Variants

**What**: AI-powered generation of message content variants for A/B testing. Given a base message, generate alternative subject lines, body copy, and CTAs.

**Design**:

```typescript
// src/server/services/ai/message-generator.ts
export interface VariantGenerationRequest {
  channelType: ChannelType;
  baseContent: EmailContent | SmsContent | PushContent;
  targetAudience: string;           // description of the audience
  tone: "professional" | "casual" | "urgent" | "friendly";
  variantCount: number;             // 2-5
}

export interface GeneratedVariant {
  variantName: string;
  content: EmailContent | SmsContent | PushContent;
  rationale: string;                // why this variant might perform better
}

export async function generateVariants(
  request: VariantGenerationRequest,
): Promise<GeneratedVariant[]> {
  // 1. Build prompt with base content, audience, tone constraints
  // 2. Call Claude with structured output
  // 3. Validate output (character limits for SMS, subject line length for email)
  // 4. Return variants with rationale
}
```

**Testing**:
- `test-10.2-email-variants`: Generate 3 email subject line variants from base
- `test-10.2-sms-variants`: Generate SMS variants within 160-character limit
- `test-10.2-tone-match`: Professional tone produces formal language; casual tone produces informal
- `test-10.2-variant-count`: Requested number of variants returned
- `test-10.2-rationale`: Each variant includes non-empty rationale

#### 10.3 — Journey Anomaly Detection

**What**: Background job that monitors active journey metrics and flags anomalies (sudden drop-off spikes, conversion rate drops, delivery failures).

**Design**:

```typescript
// src/server/services/ai/anomaly-detector.ts
export interface Anomaly {
  journeyId: string;
  stepId?: string;
  type: "drop_off_spike" | "conversion_drop" | "delivery_failure_spike" | "engagement_drop";
  severity: "warning" | "critical";
  currentValue: number;
  expectedValue: number;
  deviation: number;                 // percentage deviation from baseline
  message: string;
  detectedAt: Date;
}

export async function detectAnomalies(
  db: DrizzleDB,
  orgId: string,
): Promise<Anomaly[]> {
  // 1. Load recent daily stats for all active journeys
  // 2. Compute rolling 7-day baselines for each metric
  // 3. Compare today's values against baseline
  // 4. Flag deviations > 2 standard deviations as warnings, > 3 as critical
  // 5. Return list of anomalies
}
```

**Testing**:
- `test-10.3-drop-off-detected`: Sudden 50% drop-off spike flagged as critical
- `test-10.3-normal-variation`: Normal 5% variation not flagged
- `test-10.3-delivery-failure`: Spike in email bounces detected as delivery_failure_spike
- `test-10.3-insufficient-data`: New journey with <7 days data returns no anomalies

#### 10.4 — AI Journey Designer UI

**What**: Chat-based interface where users describe goals in natural language and receive generated journey graphs that can be loaded directly into the canvas editor.

**Design**:

```typescript
// src/components/ai/journey-designer-chat.tsx
interface JourneyDesignerChatProps {
  orgId: string;
  onJourneyGenerated: (result: JourneyDesignResult) => void;
}

// UI flow:
// 1. User types goal in chat input
// 2. Streaming response shows thinking process
// 3. Generated journey displayed as mini-preview
// 4. "Load into Canvas" button imports into journey editor
// 5. User can refine via follow-up messages
```

**Testing**:
- `test-10.4-chat-submit`: Submitting goal shows streaming response
- `test-10.4-preview-render`: Generated journey displays as mini-canvas preview
- `test-10.4-load-into-canvas`: "Load into Canvas" creates journey with correct steps and edges
- `test-10.4-refine-message`: Follow-up message modifies generated journey

---

## Phase 11: Audit, Security & Multi-Tenancy Hardening

### Purpose
Harden multi-tenant isolation, implement comprehensive audit logging, API key management, and security controls. After this phase, the platform meets enterprise security requirements (SOC 2, GDPR Article 30).

### Tasks

#### 11.1 — Audit Logging

**What**: Append-only audit log capturing all administrative actions with before/after state. Partitioned by time for retention management.

**Design**:

```typescript
// src/server/db/schema/audit-log.ts
export const auditLog = pgTable("audit_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull(),
  actorId: uuid("actor_id"),
  actorType: text("actor_type").notNull().default("user"),
  action: text("action").notNull(),
  resourceType: text("resource_type").notNull(),
  resourceId: uuid("resource_id").notNull(),
  details: jsonb("details").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_audit_org").on(table.orgId, table.createdAt),
  index("idx_audit_resource").on(table.resourceType, table.resourceId),
]);

// src/server/lib/audit.ts
export async function logAuditEvent(
  db: DrizzleDB,
  event: {
    orgId: string;
    actorId?: string;
    actorType: "user" | "system" | "api";
    action: string;
    resourceType: string;
    resourceId: string;
    details?: Record<string, unknown>;
  },
): Promise<void> {
  await db.insert(auditLog).values(event);
}
```

**Testing**:
- `test-11.1-journey-publish`: Publishing journey creates audit log entry with before/after status
- `test-11.1-template-update`: Updating template records field changes in details
- `test-11.1-consent-change`: Consent grant/withdrawal creates audit entry
- `test-11.1-audit-query`: Audit log filterable by resource type, actor, and date range

#### 11.2 — API Key Management

**What**: API key CRUD for external integrations. Keys are hashed at rest (only shown once on creation). Rate limiting per key.

**Design**:

```typescript
// API key structure
export interface ApiKeyRecord {
  id: string;
  orgId: string;
  name: string;
  keyPrefix: string;        // first 8 chars for identification
  keyHash: string;           // bcrypt hash of full key
  permissions: string[];     // events:write, customers:read, etc.
  rateLimit: number;         // requests per minute
  lastUsedAt?: Date;
  expiresAt?: Date;
}
```

**Testing**:
- `test-11.2-create-key`: Create API key; full key shown once; stored as hash
- `test-11.2-authenticate`: Valid API key authenticates request; sets orgId in context
- `test-11.2-expired-key`: Expired key returns 401
- `test-11.2-rate-limit`: Exceeding rate limit returns 429
- `test-11.2-permission-check`: Key without events:write permission rejected for event ingestion

#### 11.3 — Row-Level Security

**What**: PostgreSQL RLS policies ensuring strict org-level data isolation at the database level.

**Design**:

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
ALTER TABLE journeys ENABLE ROW LEVEL SECURITY;
-- ... all tenant tables

-- Policy: users can only see data in their organisation
CREATE POLICY org_isolation ON customers
  USING (org_id = current_setting('app.current_org_id')::uuid);
```

```typescript
// src/server/db/index.ts
export async function withOrgContext<T>(
  orgId: string,
  fn: (db: DrizzleDB) => Promise<T>,
): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_org_id = ${orgId}`);
    return fn(tx);
  });
}
```

**Testing**:
- `test-11.3-rls-isolation`: Query from org A returns zero records from org B
- `test-11.3-rls-insert`: Insert with wrong org_id rejected by RLS policy
- `test-11.3-rls-bypass`: Superuser/admin role can bypass RLS for maintenance

---

## Phase 12: MCP Server & Public API

### Purpose
Expose the platform's capabilities via Model Context Protocol (MCP) for AI agent integration and a documented REST API for external developers. After this phase, AI agents can create journeys, manage segments, and dispatch messages through MCP, and developers can integrate via REST.

### Tasks

#### 12.1 — MCP Server Implementation

**What**: MCP server exposing journey orchestration primitives as tools, resources, and prompts for AI agent consumption.

**Design**:

```typescript
// src/server/services/mcp/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({
  name: "customer-journey-orchestration",
  version: "1.0.0",
});

// Resources
server.resource("journey", "journey://{journeyId}", async (uri) => {
  // Return journey definition with steps and edges
});

server.resource("segment", "segment://{segmentId}", async (uri) => {
  // Return segment definition and current size
});

server.resource("customer", "customer://{customerId}", async (uri) => {
  // Return customer profile with attributes and consent
});

// Tools
server.tool("create_journey", {
  description: "Create a new customer journey from a goal description",
  inputSchema: { /* ... */ },
}, async (input) => {
  // Call AI journey designer, then persist result
});

server.tool("enroll_customer", {
  description: "Enroll a customer into a journey",
  inputSchema: { /* ... */ },
}, async (input) => {
  // Call enrollment service
});

server.tool("send_message", {
  description: "Send a message to a customer via a specific channel",
  inputSchema: { /* ... */ },
}, async (input) => {
  // Call message dispatch
});

server.tool("evaluate_segment", {
  description: "Evaluate a segment and return matching customers",
  inputSchema: { /* ... */ },
}, async (input) => {
  // Call segment evaluator
});
```

**Testing**:
- `test-12.1-list-tools`: MCP server lists all available tools with schemas
- `test-12.1-create-journey-tool`: create_journey tool generates and persists journey
- `test-12.1-enroll-tool`: enroll_customer tool enrolls customer in journey
- `test-12.1-resource-read`: journey resource returns full journey definition
- `test-12.1-error-handling`: Invalid input returns MCP-compliant error response

#### 12.2 — Public REST API

**What**: Versioned REST API (v1) for events, customers, journeys, segments, and templates. OpenAPI 3.1 spec auto-generated from Zod schemas.

**Design**:

```typescript
// src/app/api/v1/events/route.ts
export async function POST(request: Request): Promise<Response> {
  // 1. Authenticate via API key (Bearer token)
  // 2. Validate CloudEvents or Segment payload
  // 3. Enqueue to event ingestion pipeline
  // 4. Return 202 Accepted with event ID
}

// src/app/api/v1/customers/route.ts
export async function GET(request: Request): Promise<Response> {
  // 1. Authenticate + check permissions
  // 2. Parse query params (page, limit, filters)
  // 3. Query customers with org isolation
  // 4. Return paginated response
}

export async function POST(request: Request): Promise<Response> {
  // 1. Authenticate + check permissions
  // 2. Validate customer payload
  // 3. Create customer via service
  // 4. Return 201 Created
}
```

**Testing**:
- `test-12.2-events-post`: POST event returns 202 with event ID
- `test-12.2-customers-list`: GET customers returns paginated list
- `test-12.2-customers-create`: POST customer returns 201 with customer object
- `test-12.2-openapi-spec`: GET /api/docs returns valid OpenAPI 3.1 spec
- `test-12.2-rate-limiting`: API key rate limit enforced per key
- `test-12.2-pagination`: Cursor-based pagination returns correct pages

#### 12.3 — SDK & Developer Documentation

**What**: TypeScript/JavaScript SDK for the public API. Auto-generated from OpenAPI spec. Developer documentation with quickstart guide.

**Design**:

```typescript
// SDK usage example
import { CJOClient } from "@cjo/sdk";

const client = new CJOClient({ apiKey: "cjo_..." });

// Track an event
await client.events.track({
  userId: "user-123",
  event: "Product Viewed",
  properties: { productId: "SKU-456", price: 29.99 },
});

// Create a customer
const customer = await client.customers.create({
  externalId: "user-123",
  email: "jane@example.com",
  attributes: { plan: "premium" },
});
```

**Testing**:
- `test-12.3-sdk-track`: SDK track method sends correct payload to API
- `test-12.3-sdk-customer-create`: SDK customer create method returns typed customer object
- `test-12.3-sdk-error-handling`: SDK throws typed errors for 4xx/5xx responses

---

## Phase Dependency Graph

```
Phase 1: Foundation
  └──> Phase 2: Customer Data & Events
        ├──> Phase 3: Segments
        │     └──> Phase 4: Journey Builder
        │           ├──> Phase 5: Channels & Templates
        │           │     └──> Phase 6: Journey Execution (depends on 4 + 5)
        │           │           ├──> Phase 7: Consent & Privacy (depends on 6)
        │           │           ├──> Phase 8: Analytics & A/B Testing (depends on 6)
        │           │           └──> Phase 9: Integrations (depends on 2 + 6)
        │           └──> Phase 10: AI Features (depends on 4 + 5 + 6)
        └──> Phase 11: Audit & Security (depends on 2, can start after Phase 2)
              └──> Phase 12: MCP & Public API (depends on 6 + 11)
```

Notes:
- Phases 7, 8, and 9 can be developed in parallel after Phase 6
- Phase 10 requires Phases 4, 5, and 6 to be complete
- Phase 11 can start as early as after Phase 2 (audit logging for early entities)
- Phase 12 depends on having the execution engine (Phase 6) and security hardening (Phase 11) complete

---

## Definition of Done Checklist

- [ ] All Drizzle schema migrations apply cleanly to a fresh PostgreSQL 16 database
- [ ] Every tRPC router has corresponding Zod input validators
- [ ] All API endpoints require authentication (API key or session)
- [ ] Multi-tenant isolation verified: org A cannot read/write org B data
- [ ] Row-Level Security policies active on all tenant-scoped tables
- [ ] Every message dispatch checks consent before sending
- [ ] Suppression lists enforced for all outbound channels
- [ ] CloudEvents v1.0.2 envelope validated on event ingestion
- [ ] Segment Spec track/identify/page/group events accepted and processed
- [ ] Journey graph validation prevents publishing invalid journeys
- [ ] A/B test variant assignment is deterministic per customer (no flip-flopping)
- [ ] Audit log records all create/update/delete/publish operations
- [ ] API keys hashed at rest; full key shown only once on creation
- [ ] Channel provider credentials encrypted at rest
- [ ] GDPR erasure request deletes all customer data and preserves suppression
- [ ] Journey execution handles all step types (email, SMS, push, webhook, decision, delay, A/B split, exit)
- [ ] BullMQ workers have dead-letter queues for failed jobs
- [ ] All workers are idempotent (safe to retry)
- [ ] React Flow journey canvas supports 100+ nodes without performance degradation
- [ ] OpenAPI 3.1 spec auto-generated and serves at /api/docs
- [ ] MCP server lists all tools and resources per MCP specification
- [ ] Docker Compose runs full stack with `docker compose up`
- [ ] Vitest unit test coverage > 80% for services and validators
- [ ] Playwright E2E tests cover journey creation, template editing, and segment building
- [ ] TypeScript strict mode with zero `any` types in application code
- [ ] ESLint passes with zero warnings
