# Multi-Tenant Onda 1 — Fronteira Dura + Equipas — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a fronteira de isolamento entre organizações (tenants) e o conceito de equipas dentro de cada organização, deixando os dados existentes num tenant `sanjila`, sem ainda construir as funcionalidades *por* tenant (fan-out do Inngest, email por tenant, subdomínios, branding, RLS) — essas ficam para a Onda 2.

**Architecture:** Coluna discriminadora `tenantId` em todas as tabelas de negócio, com o tenant activo resolvido uma única vez em `requireAuthenticated()` a partir da tabela `Membership` e transportado por `AsyncLocalStorage`. Uma Prisma Client Extension injecta o predicado de tenant em todas as operações dos modelos não-globais, de modo a que as ~481 chamadas `prismadb.` existentes fiquem correctas por construção. As equipas são **derivadas**, não uma coluna nos registos: pertencer a uma equipa alarga o conjunto de `userId` que o utilizador pode ver, reutilizando os `OR` de posse que os scopes já têm.

**Tech Stack:** Next.js 16.2.6 (App Router), Prisma 7.6 + `@prisma/adapter-pg` sobre PostgreSQL, better-auth 1.6.22, next-intl 4.11, Jest + ts-jest, pnpm.

## Global Constraints

- **Não quebrar o better-auth.** `Users.role` (`AppRole`) é escrito pelo `adminPlugin` em [lib/auth.ts](../../../lib/auth.ts). Mantém-se e passa a significar *papel de plataforma*. O papel dentro do tenant vive em `Membership.role`.
- **`Users.email` continua `@unique` global.** A identidade é uma só; o multi-tenant vive no `Membership`.
- **`requireRole` passa a hierárquico.** Hoje é uma allowlist plana (`requireRole(["manager"])` rejeitaria um `admin`). Com `owner` acima de `admin`, plano quebra.
- **A extensão Prisma nunca faz fallback silencioso.** Sem tenant no contexto, atira `TenantContextMissingError`. Falhar alto.
- **Modelos globais nunca recebem `tenantId`:** `Currency`, `ExchangeRate`, `systemServices`, `Users`, `Tenant`, `Membership`, `Team`, `TeamMembership`, `TenantInvite`, e as tabelas do better-auth (`Session`, `Account`, `Verification`).
- **Convenção de nomes:** campo Prisma `tenantId`, tipo `String @db.Uuid`, sempre com `@@index([tenantId])` e, onde já exista `deletedAt`, `@@index([tenantId, deletedAt])`.
- **Migrações:** `prisma migrate dev --create-only`, editar o SQL à mão para o backfill, depois `pnpm db:migrate`. Nunca `migrate dev` directo em tabelas com dados.
- **Testes:** Jest, `testMatch: **/__tests__/**/*.test.ts`. Testes unitários mockam `@/lib/prisma`; o harness de isolamento (Task 15) usa a base de dados Docker real.
- **Idioma:** identificadores, comentários de código e mensagens de commit em inglês; documentação em português.

---

## File Structure

**Novos ficheiros:**

| Ficheiro | Responsabilidade |
|---|---|
| `lib/tenant/context.ts` | `AsyncLocalStorage` com `{ tenantId, userId }`; `runWithTenant`, `getTenantId`, `getTenantIdOrNull`, `TenantContextMissingError` |
| `lib/tenant/models.ts` | `GLOBAL_MODELS` — a allowlist de modelos sem `tenantId` |
| `lib/tenant/resolve.ts` | `resolveActiveTenant(userId)` — lê memberships + cookie `active_tenant` |
| `lib/db/tenant-client.ts` | A Prisma Client Extension que injecta o predicado |
| `lib/authz/scopes/team.ts` | `teamMemberIds(userId, tenantId)` e o fragmento `OR` de equipa |
| `actions/tenant/*.ts` | Server actions: criar equipa, gerir membros, convidar, trocar tenant |
| `app/[locale]/(routes)/admin/teams/**` | UI de gestão de equipas |
| `lib/authz/__tests__/tenant-*.test.ts` | Testes unitários da nova camada |
| `__tests__/tenant-isolation/**` | Harness de integração contra Postgres real |
| `eslint-rules/no-direct-prismadb.js` | Guarda de CI |

**Ficheiros alterados (principais):** `prisma/schema.prisma`, `lib/authz/session.ts`, `lib/authz/scopes/crm.ts`, `lib/authz/scopes/report-scope.ts`, `lib/authz/index.ts`, `lib/auth.ts`, `app/[locale]/(routes)/layout.tsx`, `actions/crm/get-crm-data.ts`, `actions/get-users.ts`, `actions/dashboard/*.ts`, `app/api/crm/leads/create-lead-from-web/route.ts`, `app/api/upload/presigned-url/route.ts`, os 8 ficheiros com `$queryRaw`, `prisma/seeds/seed.ts`, `jest.config.ts`.

---

## Task 1: Modelos de tenancy e equipas

**Files:**
- Modify: `prisma/schema.prisma` (acrescentar no fim, antes dos modelos better-auth)
- Create: `prisma/migrations/<timestamp>_add_tenancy_models/migration.sql`
- Test: `lib/authz/__tests__/tenant-models.test.ts`

**Interfaces:**
- Consumes: nada
- Produces: modelos `Tenant`, `Membership`, `Team`, `TeamMembership`, `TenantInvite`; enums `TenantStatus`, `TenantRole`. `TenantRole` = `owner | admin | manager | user`.

- [ ] **Step 1: Acrescentar os modelos ao schema**

Em `prisma/schema.prisma`:

```prisma
enum TenantStatus {
  ACTIVE
  SUSPENDED
  ARCHIVED
}

enum TenantRole {
  owner
  admin
  manager
  user
}

model Tenant {
  id            String       @id @default(uuid()) @db.Uuid
  slug          String       @unique
  name          String
  status        TenantStatus @default(ACTIVE)
  logoUrl       String?
  primaryColor  String?
  defaultLocale Language     @default(pt)
  timezone      String       @default("Africa/Luanda")
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt

  memberships   Membership[]
  teams         Team[]
  invites       TenantInvite[]

  @@index([status])
}

model Membership {
  id        String       @id @default(uuid()) @db.Uuid
  userId    String       @db.Uuid
  tenantId  String       @db.Uuid
  role      TenantRole   @default(user)
  status    ActiveStatus @default(ACTIVE)
  createdAt DateTime     @default(now())
  updatedAt DateTime     @updatedAt

  user   Users  @relation(fields: [userId], references: [id], onDelete: Cascade)
  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@unique([userId, tenantId])
  @@index([tenantId, role])
  @@index([userId])
}

model Team {
  id        String   @id @default(uuid()) @db.Uuid
  tenantId  String   @db.Uuid
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  tenant  Tenant           @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  members TeamMembership[]

  @@unique([tenantId, name])
  @@index([tenantId])
}

model TeamMembership {
  teamId    String   @db.Uuid
  userId    String   @db.Uuid
  createdAt DateTime @default(now())

  team Team  @relation(fields: [teamId], references: [id], onDelete: Cascade)
  user Users @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@id([teamId, userId])
  @@index([userId])
}

model TenantInvite {
  id        String     @id @default(uuid()) @db.Uuid
  tenantId  String     @db.Uuid
  email     String
  role      TenantRole @default(user)
  token     String     @unique
  invitedBy String     @db.Uuid
  expiresAt DateTime
  acceptedAt DateTime?
  createdAt DateTime   @default(now())

  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@unique([tenantId, email])
  @@index([tenantId])
}
```

E no modelo `Users`, acrescentar às relações existentes:

```prisma
  memberships      Membership[]
  teamMemberships  TeamMembership[]
```

- [ ] **Step 2: Gerar a migração e verificar o SQL**

```bash
pnpm exec prisma migrate dev --create-only --name add_tenancy_models
```

Abrir o `migration.sql` gerado e confirmar que só contém `CREATE TYPE` / `CREATE TABLE` / `CREATE INDEX` / `ADD CONSTRAINT` — nenhum `DROP` e nenhum `ALTER TABLE ... ALTER COLUMN` em tabelas existentes.

- [ ] **Step 3: Aplicar e gerar o client**

```bash
pnpm db:migrate && pnpm exec prisma generate
```

- [ ] **Step 4: Escrever o teste que confirma os tipos gerados**

Criar `lib/authz/__tests__/tenant-models.test.ts`:

```ts
import type { TenantRole, TenantStatus } from "@prisma/client";

describe("tenancy prisma types", () => {
  it("exposes the TenantRole enum values", () => {
    const roles: TenantRole[] = ["owner", "admin", "manager", "user"];
    expect(roles).toHaveLength(4);
  });

  it("exposes the TenantStatus enum values", () => {
    const statuses: TenantStatus[] = ["ACTIVE", "SUSPENDED", "ARCHIVED"];
    expect(statuses).toHaveLength(3);
  });
});
```

- [ ] **Step 5: Correr o teste**

Run: `pnpm test -- tenant-models`
Expected: PASS (falha a compilar se os enums não existirem no client gerado)

- [ ] **Step 6: Commit**

```bash
git add prisma/schema.prisma prisma/migrations lib/authz/__tests__/tenant-models.test.ts
git commit -m "feat(tenancy): add Tenant, Membership, Team, TeamMembership, TenantInvite models"
```

---

## Task 2: `tenantId` nas entidades CRM core + backfill do tenant Sanjila

**Files:**
- Modify: `prisma/schema.prisma` (7 modelos)
- Create: `prisma/migrations/<timestamp>_add_tenant_id_crm_core/migration.sql`

**Interfaces:**
- Consumes: `Tenant` da Task 1
- Produces: coluna `tenantId` NOT NULL em `crm_Accounts`, `crm_Leads`, `crm_Contacts`, `crm_Opportunities`, `crm_Contracts`, `crm_Targets`, `crm_TargetLists`; o tenant seed `sanjila` com id fixo `00000000-0000-0000-0000-000000000001`.

- [ ] **Step 1: Acrescentar o campo aos 7 modelos**

Em cada um de `crm_Accounts`, `crm_Leads`, `crm_Contacts`, `crm_Opportunities`, `crm_Contracts`, `crm_Targets`, `crm_TargetLists`, acrescentar como primeiro campo depois do `id`:

```prisma
  tenantId String @db.Uuid
```

E no bloco de índices de cada um:

```prisma
  @@index([tenantId])
  @@index([tenantId, deletedAt])
```

- [ ] **Step 2: Gerar a migração sem aplicar**

```bash
pnpm exec prisma migrate dev --create-only --name add_tenant_id_crm_core
```

- [ ] **Step 3: Editar o SQL para fazer o backfill**

O Prisma gera `ADD COLUMN "tenantId" UUID NOT NULL`, que falha em tabelas com dados. Substituir o conteúdo de `migration.sql` por:

```sql
-- 1. Seed tenant for all pre-existing data
INSERT INTO "Tenant" ("id", "slug", "name", "status", "defaultLocale", "timezone", "createdAt", "updatedAt")
VALUES ('00000000-0000-0000-0000-000000000001', 'sanjila', 'Sanjila', 'ACTIVE', 'pt', 'Africa/Luanda', now(), now())
ON CONFLICT ("id") DO NOTHING;

-- 2. Add nullable, backfill, then enforce
DO $$
DECLARE t text;
BEGIN
  FOREACH t IN ARRAY ARRAY[
    'crm_Accounts','crm_Leads','crm_Contacts','crm_Opportunities',
    'crm_Contracts','crm_Targets','crm_TargetLists'
  ] LOOP
    EXECUTE format('ALTER TABLE %I ADD COLUMN "tenantId" UUID', t);
    EXECUTE format('UPDATE %I SET "tenantId" = %L', t, '00000000-0000-0000-0000-000000000001');
    EXECUTE format('ALTER TABLE %I ALTER COLUMN "tenantId" SET NOT NULL', t);
    EXECUTE format('CREATE INDEX %I ON %I ("tenantId")', t || '_tenantId_idx', t);
    EXECUTE format('CREATE INDEX %I ON %I ("tenantId", "deletedAt")', t || '_tenantId_deletedAt_idx', t);
  END LOOP;
END $$;
```

- [ ] **Step 4: Aplicar e confirmar que nenhuma linha ficou órfã**

```bash
pnpm db:migrate && pnpm exec prisma generate
```

Depois, confirmar contagem zero:

```bash
docker compose -f docker-compose.dev.yml exec -T postgres psql -U nextcrm -d nextcrm -c \
  'SELECT count(*) FROM "crm_Accounts" WHERE "tenantId" IS NULL;'
```

Expected: `0`

- [ ] **Step 5: Confirmar que o schema e a BD estão em sincronia**

Run: `pnpm exec prisma migrate status`
Expected: `Database schema is up to date!`

- [ ] **Step 6: Commit**

```bash
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(tenancy): add tenantId to core CRM entities with sanjila backfill"
```

---

## Task 3: `tenantId` em documentos, projectos, actividades, facturas e produtos

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/<timestamp>_add_tenant_id_rest/migration.sql`

**Interfaces:**
- Consumes: o tenant seed da Task 2
- Produces: `tenantId` NOT NULL nos restantes modelos raiz; constraints únicos convertidos em compostos.

- [ ] **Step 1: Acrescentar `tenantId String @db.Uuid` + `@@index([tenantId])` a estes modelos**

`Documents`, `Boards`, `Sections`, `Tasks`, `crm_Accounts_Tasks`, `tasksComments`, `crm_Activities`, `crm_ActivityLinks`, `Invoices`, `Invoice_LineItems`, `Invoice_Payments`, `Invoice_Attachments`, `Invoice_Activity`, `Invoice_Series`, `Invoice_Settings`, `Invoice_TaxRates`, `crm_Products`, `crm_ProductCategories`, `crm_AccountProducts`, `crm_OpportunityLineItems`, `crm_ContractLineItems`, `crm_AuditLog`, `crm_Report_Config`, `crm_Report_Schedule`, `crm_campaigns`, `crm_campaign_templates`, `crm_campaign_steps`, `crm_campaign_sends`, `crm_Target_Contact`, `crm_FunnelSettings`, `crm_SystemSettings`, `Employees`, `EmailAccount`, `Email`, `CalendarConnection`, `crm_CalendarEvents`, `ApiToken`, `ApiKeys`, `crm_Contact_Enrichment`, `crm_Target_Enrichment`, e os lookups `crm_Lead_Sources`, `crm_Lead_Statuses`, `crm_Lead_Types`, `crm_Contact_Types`, `crm_Industry_Type`, `crm_Opportunities_Sales_Stages`, `crm_Opportunities_Type`, `Documents_Types`.

Onde o modelo tenha `deletedAt`, acrescentar também `@@index([tenantId, deletedAt])`.

- [ ] **Step 2: Converter os constraints únicos que colidem entre tenants**

| Modelo | Antes | Depois |
|---|---|---|
| `crm_Products` | `sku String @unique` | `sku String` + `@@unique([tenantId, sku])` |
| `crm_Contact_Types` | `name String @unique` | `name String` + `@@unique([tenantId, name])` |
| `crm_Lead_Sources` | `name String @unique` | `name String` + `@@unique([tenantId, name])` |
| `crm_Lead_Statuses` | `name String @unique` | `name String` + `@@unique([tenantId, name])` |
| `crm_Lead_Types` | `name String @unique` | `name String` + `@@unique([tenantId, name])` |
| `crm_SystemSettings` | `key String @id` | `key String` + `@@id([tenantId, key])` |
| `crm_CalendarEvents` | `@@unique([source, externalId])` | `@@unique([tenantId, source, externalId])` |
| `CalendarConnection` | `@@unique([userId, provider, accountEmail])` | `@@unique([tenantId, userId, provider, accountEmail])` |

Ficam inalterados: `crm_campaign_sends.unsubscribe_token` e `ApiToken.tokenHash` (tokens aleatórios, únicos globais por desenho), e `Invoices @@unique([seriesId, number])` (fica seguro porque `Invoice_Series` passa a ser por tenant).

- [ ] **Step 3: Gerar e editar a migração**

```bash
pnpm exec prisma migrate dev --create-only --name add_tenant_id_rest
```

Substituir o `migration.sql` gerado pelo mesmo padrão da Task 2, com o array alargado. **`crm_ActivityLinks` e `crm_AuditLog` são polimórficos** (`entityType` texto + `entityId` uuid, sem FK) — nesta onda o backfill é uniforme para o tenant Sanjila, o que é correcto porque só existe um tenant. A seguir ao bloco `DO $$`, os constraints:

```sql
-- Simple unique → composite unique
ALTER TABLE "crm_Products" DROP CONSTRAINT IF EXISTS "crm_Products_sku_key";
CREATE UNIQUE INDEX "crm_Products_tenantId_sku_key" ON "crm_Products" ("tenantId", "sku");

ALTER TABLE "crm_Contact_Types" DROP CONSTRAINT IF EXISTS "crm_Contact_Types_name_key";
CREATE UNIQUE INDEX "crm_Contact_Types_tenantId_name_key" ON "crm_Contact_Types" ("tenantId", "name");

ALTER TABLE "crm_Lead_Sources" DROP CONSTRAINT IF EXISTS "crm_Lead_Sources_name_key";
CREATE UNIQUE INDEX "crm_Lead_Sources_tenantId_name_key" ON "crm_Lead_Sources" ("tenantId", "name");

ALTER TABLE "crm_Lead_Statuses" DROP CONSTRAINT IF EXISTS "crm_Lead_Statuses_name_key";
CREATE UNIQUE INDEX "crm_Lead_Statuses_tenantId_name_key" ON "crm_Lead_Statuses" ("tenantId", "name");

ALTER TABLE "crm_Lead_Types" DROP CONSTRAINT IF EXISTS "crm_Lead_Types_name_key";
CREATE UNIQUE INDEX "crm_Lead_Types_tenantId_name_key" ON "crm_Lead_Types" ("tenantId", "name");

ALTER TABLE "crm_CalendarEvents" DROP CONSTRAINT IF EXISTS "crm_CalendarEvents_source_externalId_key";
CREATE UNIQUE INDEX "crm_CalendarEvents_tenantId_source_externalId_key"
  ON "crm_CalendarEvents" ("tenantId", "source", "externalId");

ALTER TABLE "CalendarConnection" DROP CONSTRAINT IF EXISTS "CalendarConnection_userId_provider_accountEmail_key";
CREATE UNIQUE INDEX "CalendarConnection_tenantId_userId_provider_accountEmail_key"
  ON "CalendarConnection" ("tenantId", "userId", "provider", "accountEmail");

-- Primary key change: crm_SystemSettings goes from (key) to (tenantId, key).
-- Not a drop/create of a unique index — the PK constraint itself moves.
ALTER TABLE "crm_SystemSettings" DROP CONSTRAINT "crm_SystemSettings_pkey";
ALTER TABLE "crm_SystemSettings" ADD CONSTRAINT "crm_SystemSettings_pkey"
  PRIMARY KEY ("tenantId", "key");
```

Confirmar antes de aplicar que os nomes de constraint existem — o Postgres nomeia-os `<tabela>_<colunas>_key` por omissão, mas uma migração anterior pode ter usado outro nome:

```bash
docker compose -f docker-compose.dev.yml exec -T postgres psql -U nextcrm -d nextcrm -c \
  "\d \"crm_Products\"" | grep -i unique
```

- [ ] **Step 4: Aplicar e verificar**

```bash
pnpm db:migrate && pnpm exec prisma generate && pnpm exec prisma migrate status
```

Expected: `Database schema is up to date!`

- [ ] **Step 5: Confirmar que a build de tipos passa**

Run: `pnpm exec tsc --noEmit`
Expected: erros **apenas** onde código existente usa `findUnique({ where: { sku } })` ou `{ where: { key } }` — anotar esses ficheiros, são corrigidos na Task 11.

- [ ] **Step 6: Commit**

```bash
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(tenancy): add tenantId to remaining root models and scope unique constraints"
```

---

## Task 4: `tenantId` nas junções e nas tabelas de embedding

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/<timestamp>_add_tenant_id_junctions/migration.sql`

**Interfaces:**
- Consumes: Tasks 2 e 3
- Produces: `tenantId` nas 13 junções e nas 7 tabelas de embedding — necessário porque as tabelas de embedding são consultadas por SQL cru (Task 12) e um JOIN por tenant degradaria a pesquisa vectorial.

- [ ] **Step 1: Acrescentar `tenantId String @db.Uuid` + `@@index([tenantId])`**

Junções: `DocumentsToOpportunities`, `DocumentsToContacts`, `DocumentsToTasks`, `DocumentsToCrmAccountsTasks`, `DocumentsToLeads`, `DocumentsToAccounts`, `AccountWatchers`, `BoardWatchers`, `ContactsToOpportunities`, `TargetsToTargetLists`, `CampaignToTargetLists`, `EmailsToContacts`, `EmailsToAccounts`.

Embeddings: `crm_Embeddings_Accounts`, `crm_Embeddings_Contacts`, `crm_Embeddings_Leads`, `crm_Embeddings_Opportunities`, `crm_Embeddings_Documents`, `crm_Document_Chunks`, `EmailEmbedding`.

- [ ] **Step 2: Gerar e editar a migração com o mesmo padrão `DO $$`**

```bash
pnpm exec prisma migrate dev --create-only --name add_tenant_id_junctions
```

Nota: estas tabelas não têm `deletedAt` — omitir a linha do índice composto no loop.

- [ ] **Step 3: Aplicar**

```bash
pnpm db:migrate && pnpm exec prisma generate
```

- [ ] **Step 4: Verificar que não sobrou nenhuma tabela de negócio sem `tenantId`**

```bash
docker compose -f docker-compose.dev.yml exec -T postgres psql -U nextcrm -d nextcrm -t -c "
SELECT t.table_name FROM information_schema.tables t
WHERE t.table_schema='public' AND t.table_type='BASE TABLE'
AND NOT EXISTS (
  SELECT 1 FROM information_schema.columns c
  WHERE c.table_schema='public' AND c.table_name=t.table_name AND c.column_name='tenantId'
) ORDER BY 1;"
```

Expected — **exactamente** esta lista e nada mais:
`Currency`, `ExchangeRate`, `systemServices`, `Users`, `Tenant`, `Membership`, `Team`, `TeamMembership`, `TenantInvite`, `session`, `account`, `verification`, `_prisma_migrations`, `ImageUpload`, `TodoList`.

(`ImageUpload` e `TodoList` são modelos mortos — só têm `id` / não têm FK. Deixar como estão.)

- [ ] **Step 5: Commit**

```bash
git add prisma/schema.prisma prisma/migrations
git commit -m "feat(tenancy): add tenantId to junction and embedding tables"
```

---

## Task 5: Contexto de tenant com AsyncLocalStorage

**Files:**
- Create: `lib/tenant/context.ts`
- Create: `lib/tenant/models.ts`
- Test: `lib/tenant/__tests__/context.test.ts`

**Interfaces:**
- Consumes: nada
- Produces: `runWithTenant<T>(ctx, fn): Promise<T>`, `enterTenantScope(ctx): void`, `getTenantId(): string` (atira `TenantContextMissingError`), `getTenantIdOrNull(): string | null`, `getTenantContextOrNull(): TenantContext | null`, `TenantContextMissingError`, `type TenantContext = { tenantId: string; userId: string }`, `GLOBAL_MODELS: ReadonlySet<string>`, `isGlobalModel(model)`.

Duas formas de entrar no scope, porque há dois modos de execução:
- `runWithTenant(ctx, fn)` — a forma estrita, com callback. Para jobs Inngest e scripts, onde o limite é explícito.
- `enterTenantScope(ctx)` — sem callback, via `AsyncLocalStorage.enterWith()`. É esta que `requireAuthenticated()` usa (Task 8): estabelece o tenant para o resto do contexto assíncrono do pedido sem obrigar a reescrever os ~152 sítios que fazem `const user = await requireAuthenticated()`.

- [ ] **Step 1: Escrever o teste que falha**

Criar `lib/tenant/__tests__/context.test.ts`:

```ts
import {
  runWithTenant,
  getTenantId,
  getTenantIdOrNull,
  TenantContextMissingError,
} from "../context";

describe("tenant context", () => {
  it("throws outside of a tenant scope", () => {
    expect(() => getTenantId()).toThrow(TenantContextMissingError);
  });

  it("returns null from the soft accessor outside a scope", () => {
    expect(getTenantIdOrNull()).toBeNull();
  });

  it("exposes the tenant inside the scope", async () => {
    const seen = await runWithTenant({ tenantId: "t1", userId: "u1" }, async () =>
      getTenantId(),
    );
    expect(seen).toBe("t1");
  });

  it("does not leak across sibling async scopes", async () => {
    const [a, b] = await Promise.all([
      runWithTenant({ tenantId: "t1", userId: "u1" }, async () => {
        await new Promise((r) => setTimeout(r, 10));
        return getTenantId();
      }),
      runWithTenant({ tenantId: "t2", userId: "u2" }, async () => getTenantId()),
    ]);
    expect(a).toBe("t1");
    expect(b).toBe("t2");
  });

  it("restores the outer scope after a nested one", async () => {
    const result = await runWithTenant({ tenantId: "t1", userId: "u1" }, async () => {
      await runWithTenant({ tenantId: "t2", userId: "u2" }, async () => getTenantId());
      return getTenantId();
    });
    expect(result).toBe("t1");
  });
});

describe("enterTenantScope", () => {
  it("sets the tenant for the rest of the current async context", async () => {
    await runWithTenant({ tenantId: "outer", userId: "u0" }, async () => {
      enterTenantScope({ tenantId: "t9", userId: "u9" });
      expect(getTenantId()).toBe("t9");
      await new Promise((r) => setTimeout(r, 1));
      expect(getTenantId()).toBe("t9");
    });
  });

  it("does not leak into a sibling context entered separately", async () => {
    const [a, b] = await Promise.all([
      runWithTenant({ tenantId: "seed", userId: "u0" }, async () => {
        enterTenantScope({ tenantId: "t1", userId: "u1" });
        await new Promise((r) => setTimeout(r, 10));
        return getTenantId();
      }),
      runWithTenant({ tenantId: "seed", userId: "u0" }, async () => {
        enterTenantScope({ tenantId: "t2", userId: "u2" });
        return getTenantId();
      }),
    ]);
    expect(a).toBe("t1");
    expect(b).toBe("t2");
  });
});
```

Acrescentar `enterTenantScope` ao import no topo do ficheiro de teste.

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- tenant/__tests__/context`
Expected: FAIL — `Cannot find module '../context'`

- [ ] **Step 3: Implementar `lib/tenant/context.ts`**

```ts
import { AsyncLocalStorage } from "node:async_hooks";

export type TenantContext = {
  tenantId: string;
  userId: string;
};

export class TenantContextMissingError extends Error {
  readonly code = "TENANT_CONTEXT_MISSING";
  constructor() {
    super(
      "No tenant in scope. Wrap this call in runWithTenant(), or use the global " +
        "prisma client explicitly if the model is tenant-independent.",
    );
    this.name = "TenantContextMissingError";
  }
}

const storage = new AsyncLocalStorage<TenantContext>();

export function runWithTenant<T>(
  ctx: TenantContext,
  fn: () => Promise<T>,
): Promise<T> {
  return storage.run(ctx, fn);
}

export function getTenantIdOrNull(): string | null {
  return storage.getStore()?.tenantId ?? null;
}

export function getTenantId(): string {
  const tenantId = getTenantIdOrNull();
  if (!tenantId) throw new TenantContextMissingError();
  return tenantId;
}

/**
 * Enter a tenant scope without a callback, for the remainder of the current
 * async context.
 *
 * This is what lets requireAuthenticated() establish the tenant without the
 * ~152 `const user = await requireAuthenticated()` call sites having to be
 * restructured into callbacks. Next.js gives each request its own async
 * context, so one request's scope cannot be observed by another.
 *
 * Prefer runWithTenant() anywhere the boundary is explicit — Inngest jobs,
 * scripts, tests — because it also *leaves* the scope on the way out.
 */
export function enterTenantScope(ctx: TenantContext): void {
  storage.enterWith(ctx);
}

export function getTenantContextOrNull(): TenantContext | null {
  return storage.getStore() ?? null;
}
```

- [ ] **Step 4: Implementar `lib/tenant/models.ts`**

```ts
// Models that have no tenantId column and must never be filtered by tenant.
// Keep in sync with the verification query in the tenancy migration plan (Task 4).
export const GLOBAL_MODELS: ReadonlySet<string> = new Set([
  // Reference data
  "Currency",
  "ExchangeRate",
  "systemServices",
  // Identity — scoped through Membership, not through a tenantId column
  "Users",
  // Tenancy control plane
  "Tenant",
  "Membership",
  "Team",
  "TeamMembership",
  "TenantInvite",
  // better-auth
  "Session",
  "Account",
  "Verification",
  // Dead models kept for schema parity
  "ImageUpload",
  "TodoList",
]);

export function isGlobalModel(model: string | undefined): boolean {
  return model === undefined || GLOBAL_MODELS.has(model);
}
```

- [ ] **Step 5: Correr os testes**

Run: `pnpm test -- tenant/__tests__/context`
Expected: PASS, 7 testes

- [ ] **Step 6: Commit**

```bash
git add lib/tenant/context.ts lib/tenant/models.ts lib/tenant/__tests__/context.test.ts
git commit -m "feat(tenancy): add AsyncLocalStorage tenant context and global model allowlist"
```

---

## Task 6: Prisma Client Extension que injecta o tenant

**Files:**
- Create: `lib/db/tenant-client.ts`
- Test: `lib/db/__tests__/tenant-client.test.ts`

**Interfaces:**
- Consumes: `getTenantId`, `TenantContextMissingError` (Task 5); `isGlobalModel` (Task 5)
- Produces: `tenantScopedClient` — o cliente Prisma estendido, e `buildTenantArgs(operation, model, args, tenantId)` exportado para teste.

Regra crítica: `findUnique`/`findUniqueOrThrow` **não aceitam** campos não-únicos no `where`. A extensão converte-os em `findFirst`/`findFirstOrThrow`.

**Estatuto nesta onda:** o cliente fica disponível e funcional (o scope é estabelecido por `enterTenantScope` na Task 8), mas a migração dos ~481 sítios que importam `prismadb` é da Onda 2 — a regra ESLint da Task 15 entra em `warn` precisamente por isso. Nesta onda o isolamento vem dos scope helpers (Task 10); a extensão é a rede que torna a Onda 2 mecânica em vez de arriscada. Usar `tenantScopedClient` em código novo desde já.

- [ ] **Step 1: Escrever o teste que falha**

Criar `lib/db/__tests__/tenant-client.test.ts`:

```ts
import { buildTenantArgs } from "../tenant-client";

const T = "00000000-0000-0000-0000-000000000001";

describe("buildTenantArgs", () => {
  it("adds tenantId to a findMany where", () => {
    const { args } = buildTenantArgs("findMany", "crm_Accounts", { where: { deletedAt: null } }, T);
    expect(args.where).toEqual({ deletedAt: null, tenantId: T });
  });

  it("adds a where to a findMany that has none", () => {
    const { args } = buildTenantArgs("findMany", "crm_Accounts", {}, T);
    expect(args.where).toEqual({ tenantId: T });
  });

  it("rewrites findUnique to findFirst so a non-unique field is allowed", () => {
    const { operation, args } = buildTenantArgs("findUnique", "crm_Accounts", { where: { id: "a1" } }, T);
    expect(operation).toBe("findFirst");
    expect(args.where).toEqual({ id: "a1", tenantId: T });
  });

  it("rewrites findUniqueOrThrow to findFirstOrThrow", () => {
    const { operation } = buildTenantArgs("findUniqueOrThrow", "crm_Accounts", { where: { id: "a1" } }, T);
    expect(operation).toBe("findFirstOrThrow");
  });

  it("stamps tenantId onto create data", () => {
    const { args } = buildTenantArgs("create", "crm_Accounts", { data: { name: "Acme" } }, T);
    expect(args.data).toEqual({ name: "Acme", tenantId: T });
  });

  it("stamps tenantId onto every row of createMany", () => {
    const { args } = buildTenantArgs("createMany", "crm_Accounts", { data: [{ name: "A" }, { name: "B" }] }, T);
    expect(args.data).toEqual([
      { name: "A", tenantId: T },
      { name: "B", tenantId: T },
    ]);
  });

  it("stamps both branches of an upsert", () => {
    const { args } = buildTenantArgs(
      "upsert",
      "crm_Accounts",
      { where: { id: "a1" }, create: { name: "A" }, update: { name: "B" } },
      T,
    );
    expect(args.where).toEqual({ id: "a1", tenantId: T });
    expect(args.create).toEqual({ name: "A", tenantId: T });
    expect(args.update).toEqual({ name: "B" });
  });

  it("refuses to let a caller override the tenant", () => {
    const { args } = buildTenantArgs("findMany", "crm_Accounts", { where: { tenantId: "other" } }, T);
    expect(args.where.tenantId).toBe(T);
  });

  it("leaves global models untouched", () => {
    const { args, operation } = buildTenantArgs("findUnique", "Users", { where: { id: "u1" } }, T);
    expect(operation).toBe("findUnique");
    expect(args.where).toEqual({ id: "u1" });
  });
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- db/__tests__/tenant-client`
Expected: FAIL — `Cannot find module '../tenant-client'`

- [ ] **Step 3: Implementar `lib/db/tenant-client.ts`**

```ts
import { prismadb } from "@/lib/prisma";
import { getTenantId } from "@/lib/tenant/context";
import { isGlobalModel } from "@/lib/tenant/models";

const WHERE_OPS = new Set([
  "findFirst",
  "findFirstOrThrow",
  "findMany",
  "count",
  "aggregate",
  "groupBy",
  "update",
  "updateMany",
  "delete",
  "deleteMany",
]);

// findUnique cannot carry a non-unique field in `where`; downgrade to findFirst.
const UNIQUE_REWRITE: Record<string, string> = {
  findUnique: "findFirst",
  findUniqueOrThrow: "findFirstOrThrow",
};

export function buildTenantArgs(
  operation: string,
  model: string | undefined,
  rawArgs: any,
  tenantId: string,
): { operation: string; args: any } {
  if (isGlobalModel(model)) {
    return { operation, args: rawArgs };
  }

  const args = { ...(rawArgs ?? {}) };
  const op = UNIQUE_REWRITE[operation] ?? operation;

  if (WHERE_OPS.has(op)) {
    // tenantId last: a caller-supplied tenantId can never win.
    args.where = { ...(args.where ?? {}), tenantId };
  }

  if (op === "create" || op === "upsert") {
    const key = op === "create" ? "data" : "create";
    args[key] = { ...(args[key] ?? {}), tenantId };
  }

  if (op === "createMany" || op === "createManyAndReturn") {
    const rows = Array.isArray(args.data) ? args.data : [args.data];
    args.data = rows.map((row: any) => ({ ...row, tenantId }));
  }

  if (op === "upsert") {
    args.where = { ...(args.where ?? {}), tenantId };
  }

  return { operation: op, args };
}

export const tenantScopedClient = prismadb.$extends({
  name: "tenant-scope",
  query: {
    $allModels: {
      async $allOperations({ model, operation, args, query }) {
        if (isGlobalModel(model)) return query(args);

        // Throws TenantContextMissingError when there is no tenant in scope.
        const tenantId = getTenantId();
        const built = buildTenantArgs(operation, model, args, tenantId);

        if (built.operation !== operation) {
          // Prisma's `query` is bound to the original operation, so a rewritten
          // operation has to be dispatched through the base client instead.
          const delegate = (prismadb as any)[modelToDelegate(model!)];
          return delegate[built.operation](built.args);
        }

        return query(built.args);
      },
    },
  },
});

// Prisma delegates are the model name with a lowercased first letter.
function modelToDelegate(model: string): string {
  return model.charAt(0).toLowerCase() + model.slice(1);
}
```

- [ ] **Step 4: Correr os testes**

Run: `pnpm test -- db/__tests__/tenant-client`
Expected: PASS, 9 testes

- [ ] **Step 5: Commit**

```bash
git add lib/db/tenant-client.ts lib/db/__tests__/tenant-client.test.ts
git commit -m "feat(tenancy): add prisma extension that injects tenantId into every query"
```

---

## Task 7: Resolução do tenant activo

**Files:**
- Create: `lib/tenant/resolve.ts`
- Test: `lib/tenant/__tests__/resolve.test.ts`

**Interfaces:**
- Consumes: `Membership` (Task 1)
- Produces: `resolveActiveTenant(userId: string): Promise<ResolvedTenant | null>` onde `ResolvedTenant = { tenantId: string; role: TenantRole }`; `ACTIVE_TENANT_COOKIE = "active_tenant"`.

Nesta onda **não há subdomínios** — o tenant vem do `Membership`. Com uma só membership, é essa; com várias, é a do cookie `active_tenant` desde que o utilizador seja membro; se o cookie apontar para um tenant sem membership, é ignorado e usa-se a primeira por ordem de criação.

- [ ] **Step 1: Escrever o teste que falha**

Criar `lib/tenant/__tests__/resolve.test.ts`:

```ts
jest.mock("@/lib/prisma", () => ({
  prismadb: { membership: { findMany: jest.fn() } },
}));
jest.mock("next/headers", () => ({ cookies: jest.fn() }));

import { prismadb } from "@/lib/prisma";
import { cookies } from "next/headers";
import { resolveActiveTenant, ACTIVE_TENANT_COOKIE } from "../resolve";

const findMany = prismadb.membership.findMany as jest.Mock;
const mockedCookies = cookies as jest.Mock;

function withCookie(value: string | undefined) {
  mockedCookies.mockResolvedValue({
    get: (name: string) =>
      name === ACTIVE_TENANT_COOKIE && value ? { value } : undefined,
  });
}

beforeEach(() => {
  jest.clearAllMocks();
  withCookie(undefined);
});

describe("resolveActiveTenant", () => {
  it("returns null when the user has no membership", async () => {
    findMany.mockResolvedValue([]);
    await expect(resolveActiveTenant("u1")).resolves.toBeNull();
  });

  it("returns the only membership when there is exactly one", async () => {
    findMany.mockResolvedValue([{ tenantId: "t1", role: "manager" }]);
    await expect(resolveActiveTenant("u1")).resolves.toEqual({
      tenantId: "t1",
      role: "manager",
    });
  });

  it("honours the active_tenant cookie when the user is a member", async () => {
    withCookie("t2");
    findMany.mockResolvedValue([
      { tenantId: "t1", role: "user" },
      { tenantId: "t2", role: "admin" },
    ]);
    await expect(resolveActiveTenant("u1")).resolves.toEqual({
      tenantId: "t2",
      role: "admin",
    });
  });

  it("ignores a cookie pointing at a tenant the user does not belong to", async () => {
    withCookie("t9");
    findMany.mockResolvedValue([
      { tenantId: "t1", role: "user" },
      { tenantId: "t2", role: "admin" },
    ]);
    await expect(resolveActiveTenant("u1")).resolves.toEqual({
      tenantId: "t1",
      role: "user",
    });
  });

  it("only considers ACTIVE memberships of ACTIVE tenants", async () => {
    findMany.mockResolvedValue([]);
    await resolveActiveTenant("u1");
    expect(findMany).toHaveBeenCalledWith(
      expect.objectContaining({
        where: {
          userId: "u1",
          status: "ACTIVE",
          tenant: { status: "ACTIVE" },
        },
      }),
    );
  });
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- tenant/__tests__/resolve`
Expected: FAIL — `Cannot find module '../resolve'`

- [ ] **Step 3: Implementar `lib/tenant/resolve.ts`**

```ts
import { cookies } from "next/headers";
import { prismadb } from "@/lib/prisma";
import type { TenantRole } from "@prisma/client";

export const ACTIVE_TENANT_COOKIE = "active_tenant";

export type ResolvedTenant = {
  tenantId: string;
  role: TenantRole;
};

export async function resolveActiveTenant(
  userId: string,
): Promise<ResolvedTenant | null> {
  const memberships = await prismadb.membership.findMany({
    where: {
      userId,
      status: "ACTIVE",
      tenant: { status: "ACTIVE" },
    },
    select: { tenantId: true, role: true },
    orderBy: { createdAt: "asc" },
  });

  if (memberships.length === 0) return null;
  if (memberships.length === 1) return memberships[0];

  const requested = (await cookies()).get(ACTIVE_TENANT_COOKIE)?.value;
  const matched = requested
    ? memberships.find((m) => m.tenantId === requested)
    : undefined;

  // An unknown or non-member cookie value is ignored, never honoured.
  return matched ?? memberships[0];
}
```

- [ ] **Step 4: Correr os testes**

Run: `pnpm test -- tenant/__tests__/resolve`
Expected: PASS, 5 testes

- [ ] **Step 5: Commit**

```bash
git add lib/tenant/resolve.ts lib/tenant/__tests__/resolve.test.ts
git commit -m "feat(tenancy): resolve the active tenant from memberships and cookie"
```

---

## Task 8: `AuthzUser` com tenant e equipas, e `requireRole` hierárquico

**Files:**
- Modify: `lib/authz/session.ts`
- Modify: `lib/authz/roles.ts`
- Create: `lib/authz/scopes/team.ts`
- Modify: `lib/authz/index.ts`
- Test: `lib/authz/__tests__/session.test.ts` (alterar), `lib/authz/__tests__/tenant-session.test.ts` (novo)

**Interfaces:**
- Consumes: `resolveActiveTenant` (Task 7), `TenantRole` (Task 1)
- Produces: `AuthzUser = { id: string; role: TenantRole; tenantId: string; teamMemberIds: string[]; isPlatformAdmin: boolean }`; `requireAuthenticated()` inalterado na assinatura; `requireRole(roles: ReadonlyArray<TenantRole>)` agora hierárquico; `ROLE_RANK: Record<TenantRole, number>`; `satisfiesRole(actual, allowed): boolean`; `resolveTeamMemberIds(userId, tenantId): Promise<string[]>`.

Nota de nomes: a **função** chama-se `resolveTeamMemberIds` e o **campo** de `AuthzUser` chama-se `teamMemberIds`. Nomes diferentes de propósito — `teamMemberIds: await teamMemberIds(...)` compilaria mas lê-se mal.

`resolveTeamMemberIds` devolve sempre o próprio `userId` mais os colegas de todas as equipas do utilizador nesse tenant. Sem equipas, devolve `[userId]` — o comportamento actual, portanto nada regride.

- [ ] **Step 1: Escrever os testes que falham**

Criar `lib/authz/__tests__/tenant-session.test.ts`:

```ts
import { AuthenticationError, AuthorizationError } from "../errors";

jest.mock("@/lib/auth-server", () => ({ getSession: jest.fn() }));
jest.mock("@/lib/tenant/resolve", () => ({ resolveActiveTenant: jest.fn() }));
jest.mock("@/lib/prisma", () => ({
  prismadb: {
    users: { findUnique: jest.fn() },
    teamMembership: { findMany: jest.fn() },
  },
}));

import { getSession } from "@/lib/auth-server";
import { resolveActiveTenant } from "@/lib/tenant/resolve";
import { prismadb } from "@/lib/prisma";
import { requireAuthenticated, requireRole } from "../session";

const mockedGetSession = getSession as jest.Mock;
const mockedResolve = resolveActiveTenant as jest.Mock;
const mockedUser = prismadb.users.findUnique as jest.Mock;
const mockedTeams = prismadb.teamMembership.findMany as jest.Mock;

beforeEach(() => {
  jest.clearAllMocks();
  mockedGetSession.mockResolvedValue({ user: { id: "u1" } });
  mockedUser.mockResolvedValue({ id: "u1", role: "user" });
  mockedResolve.mockResolvedValue({ tenantId: "t1", role: "manager" });
  mockedTeams.mockResolvedValue([]);
});

describe("requireAuthenticated with tenancy", () => {
  it("throws AuthenticationError when the user has no active membership", async () => {
    mockedResolve.mockResolvedValue(null);
    await expect(requireAuthenticated()).rejects.toBeInstanceOf(AuthenticationError);
  });

  it("takes the role from the membership, not from Users.role", async () => {
    mockedUser.mockResolvedValue({ id: "u1", role: "user" });
    mockedResolve.mockResolvedValue({ tenantId: "t1", role: "admin" });
    const user = await requireAuthenticated();
    expect(user.role).toBe("admin");
    expect(user.tenantId).toBe("t1");
  });

  it("marks a platform admin from Users.role", async () => {
    mockedUser.mockResolvedValue({ id: "u1", role: "admin" });
    const user = await requireAuthenticated();
    expect(user.isPlatformAdmin).toBe(true);
  });

  it("always includes the user's own id in teamMemberIds", async () => {
    const user = await requireAuthenticated();
    expect(user.teamMemberIds).toEqual(["u1"]);
  });

  it("includes teammates and de-duplicates", async () => {
    mockedTeams.mockResolvedValue([
      { team: { members: [{ userId: "u1" }, { userId: "u2" }] } },
      { team: { members: [{ userId: "u2" }, { userId: "u3" }] } },
    ]);
    const user = await requireAuthenticated();
    expect(user.teamMemberIds.sort()).toEqual(["u1", "u2", "u3"]);
  });
});

describe("requireRole hierarchy", () => {
  it("owner satisfies a requirement of admin", async () => {
    mockedResolve.mockResolvedValue({ tenantId: "t1", role: "owner" });
    await expect(requireRole(["admin"])).resolves.toMatchObject({ role: "owner" });
  });

  it("admin satisfies a requirement of manager", async () => {
    mockedResolve.mockResolvedValue({ tenantId: "t1", role: "admin" });
    await expect(requireRole(["manager"])).resolves.toMatchObject({ role: "admin" });
  });

  it("user does not satisfy a requirement of manager", async () => {
    mockedResolve.mockResolvedValue({ tenantId: "t1", role: "user" });
    await expect(requireRole(["manager"])).rejects.toBeInstanceOf(AuthorizationError);
  });
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- tenant-session`
Expected: FAIL — `resolveActiveTenant` não é usado por `session.ts`

- [ ] **Step 3: Acrescentar a hierarquia a `lib/authz/roles.ts`**

Acrescentar ao fim do ficheiro, mantendo tudo o que já lá está:

```ts
import type { TenantRole } from "@prisma/client";

export const TENANT_ROLES: readonly TenantRole[] = [
  "user",
  "manager",
  "admin",
  "owner",
] as const;

// Higher rank satisfies any lower-ranked requirement.
export const ROLE_RANK: Record<TenantRole, number> = {
  user: 0,
  manager: 1,
  admin: 2,
  owner: 3,
};

export function satisfiesRole(
  actual: TenantRole,
  allowed: ReadonlyArray<TenantRole>,
): boolean {
  const needed = Math.min(...allowed.map((r) => ROLE_RANK[r]));
  return ROLE_RANK[actual] >= needed;
}
```

- [ ] **Step 4: Implementar `lib/authz/scopes/team.ts`**

```ts
import { prismadb } from "@/lib/prisma";

/**
 * The set of user ids whose records this user may see through team membership.
 * Always contains the user's own id, so a user with no team behaves exactly as
 * before this feature existed.
 */
export async function resolveTeamMemberIds(
  userId: string,
  tenantId: string,
): Promise<string[]> {
  const rows = await prismadb.teamMembership.findMany({
    where: { userId, team: { tenantId } },
    select: { team: { select: { members: { select: { userId: true } } } } },
  });

  const ids = new Set<string>([userId]);
  for (const row of rows) {
    for (const member of row.team.members) ids.add(member.userId);
  }
  return [...ids];
}
```

- [ ] **Step 5: Reescrever `lib/authz/session.ts`**

```ts
import { getSession } from "@/lib/auth-server";
import { prismadb } from "@/lib/prisma";
import { resolveActiveTenant } from "@/lib/tenant/resolve";
import { enterTenantScope } from "@/lib/tenant/context";
import { resolveTeamMemberIds } from "./scopes/team";
import { satisfiesRole } from "./roles";
import { AuthenticationError, AuthorizationError } from "./errors";
import type { TenantRole } from "@prisma/client";

export interface AuthzUser {
  id: string;
  /** Role inside the active tenant, from Membership — not from Users.role. */
  role: TenantRole;
  tenantId: string;
  /** Own id plus every teammate's id in this tenant. Never empty. */
  teamMemberIds: string[];
  /** Users.role === "admin" — platform operator, orthogonal to tenant role. */
  isPlatformAdmin: boolean;
}

export async function requireAuthenticated(): Promise<AuthzUser> {
  const session = await getSession();
  const userId = session?.user?.id;
  if (!userId) throw new AuthenticationError();

  const dbUser = await prismadb.users.findUnique({
    where: { id: userId },
    select: { id: true, role: true },
  });
  if (!dbUser) throw new AuthenticationError();

  const active = await resolveActiveTenant(dbUser.id);
  // A user with no active membership cannot act, platform admin included:
  // they must pick a tenant first.
  if (!active) throw new AuthenticationError();

  // Establishes the tenant for the rest of this request's async context, so
  // that tenantScopedClient works without every caller threading it through.
  enterTenantScope({ tenantId: active.tenantId, userId: dbUser.id });

  return {
    id: dbUser.id,
    role: active.role,
    tenantId: active.tenantId,
    teamMemberIds: await resolveTeamMemberIds(dbUser.id, active.tenantId),
    isPlatformAdmin: dbUser.role === "admin",
  };
}

export async function requireRole(
  allowedRoles: ReadonlyArray<TenantRole>,
): Promise<AuthzUser> {
  const user = await requireAuthenticated();
  if (!satisfiesRole(user.role, allowedRoles)) {
    throw new AuthorizationError();
  }
  return user;
}

export function isAdmin(user: AuthzUser): boolean {
  return user.role === "admin" || user.role === "owner";
}

export function isManagerOrAdmin(user: AuthzUser): boolean {
  return satisfiesRole(user.role, ["manager"]);
}
```

- [ ] **Step 6: Actualizar o barrel `lib/authz/index.ts`**

Substituir a linha `export { APP_ROLES, parseRole, mapLegacyRole } from "./roles";` por:

```ts
export { APP_ROLES, parseRole, mapLegacyRole, TENANT_ROLES, ROLE_RANK, satisfiesRole } from "./roles";
export { resolveTeamMemberIds } from "./scopes/team";
```

- [ ] **Step 7: Reparar o segundo produtor de `AuthzUser` — `lib/mcp/auth.ts`**

`requireAuthenticated` não é o único sítio que constrói um `AuthzUser`. [lib/mcp/auth.ts](../../../lib/mcp/auth.ts) constrói um a partir de um bearer token `nxtc__` para o servidor MCP, e deixa de compilar com os campos novos.

Acrescentar ao `prisma/schema.prisma`, no modelo `ApiToken` (que já ganhou `tenantId` na Task 3), nada de novo — o `tenantId` do token é a fonte de verdade. Na função que resolve o token, substituir a construção do `AuthzUser` por:

```ts
  const token = await prismadb.apiToken.findFirst({
    where: { tokenHash: hash, revokedAt: null },
    select: { id: true, userId: true, tenantId: true },
  });
  if (!token) return null;

  // The token's tenant wins — an MCP client never picks its own tenant.
  const membership = await prismadb.membership.findFirst({
    where: {
      userId: token.userId,
      tenantId: token.tenantId,
      status: "ACTIVE",
      tenant: { status: "ACTIVE" },
    },
    select: { role: true },
  });
  if (!membership) return null;

  enterTenantScope({ tenantId: token.tenantId, userId: token.userId });

  const dbUser = await prismadb.users.findUnique({
    where: { id: token.userId },
    select: { role: true },
  });

  return {
    id: token.userId,
    role: membership.role,
    tenantId: token.tenantId,
    teamMemberIds: await resolveTeamMemberIds(token.userId, token.tenantId),
    isPlatformAdmin: dbUser?.role === "admin",
  };
```

Isto revoga implicitamente qualquer token cujo dono tenha perdido a membership — que é o comportamento correcto.

- [ ] **Step 8: Actualizar `lib/authz/__tests__/session.test.ts`**

Os quatro testes existentes de `requireAuthenticated` assumem `{ id, role }` como resultado. Acrescentar os mocks de `resolveActiveTenant` e `teamMembership.findMany` (como no ficheiro do Step 1) e trocar as asserções `resolves.toEqual({ id, role })` por `resolves.toMatchObject({ id, role })`. Apagar o teste `"falls back to user when DB role is unrecognized"` — o papel deixou de vir de `Users.role`; a sua substituição é o teste `"takes the role from the membership"` da Task 8.

- [ ] **Step 9: Correr toda a suite authz e o typecheck**

```bash
pnpm test -- lib/authz
pnpm exec tsc --noEmit
```
Expected: testes PASS (os `scopes-crm-*` ainda passam porque `AuthzUser` só ganhou campos); o typecheck sinaliza os sítios que constroem `AuthzUser` à mão — devem ser apenas ficheiros de teste, que a Task 10 Step 6 corrige.

- [ ] **Step 10: Commit**

```bash
git add lib/authz lib/mcp/auth.ts
git commit -m "feat(tenancy): widen AuthzUser with tenantId and teams, make requireRole hierarchical"
```

---

## Task 9: Backfill das memberships e criação de tenant no registo

**Files:**
- Create: `prisma/migrations/<timestamp>_backfill_memberships/migration.sql`
- Modify: `lib/auth.ts` (callback `onUserCreated`)
- Test: `lib/__tests__/auth-on-user-created.test.ts`

**Interfaces:**
- Consumes: `Membership`, `Tenant` (Task 1); o tenant `sanjila` (Task 2)
- Produces: uma `Membership` por utilizador existente; `ensureMembershipForNewUser(userId, isFirstUser)`.

- [ ] **Step 1: Escrever a migração de backfill**

```bash
pnpm exec prisma migrate dev --create-only --name backfill_memberships
```

Substituir o `migration.sql` (vem vazio) por:

```sql
-- Every pre-existing user becomes a member of the sanjila tenant.
-- Users.role maps 1:1 onto TenantRole for the three legacy values.
INSERT INTO "Membership" ("id", "userId", "tenantId", "role", "status", "createdAt", "updatedAt")
SELECT
  gen_random_uuid(),
  u."id",
  '00000000-0000-0000-0000-000000000001',
  (CASE u."role"::text
     WHEN 'admin'   THEN 'admin'
     WHEN 'manager' THEN 'manager'
     ELSE 'user'
   END)::"TenantRole",
  (CASE u."userStatus"::text WHEN 'ACTIVE' THEN 'ACTIVE' ELSE 'PENDING' END)::"ActiveStatus",
  now(),
  now()
FROM "Users" u
ON CONFLICT ("userId", "tenantId") DO NOTHING;

-- The oldest admin becomes the tenant owner.
UPDATE "Membership" SET "role" = 'owner'
WHERE "id" = (
  SELECT m."id" FROM "Membership" m
  JOIN "Users" u ON u."id" = m."userId"
  WHERE m."tenantId" = '00000000-0000-0000-0000-000000000001'
    AND u."role"::text = 'admin'
  ORDER BY u."created_on" ASC
  LIMIT 1
);
```

- [ ] **Step 2: Aplicar e verificar que nenhum utilizador ficou sem membership**

```bash
pnpm db:migrate
docker compose -f docker-compose.dev.yml exec -T postgres psql -U nextcrm -d nextcrm -c \
  'SELECT count(*) FROM "Users" u LEFT JOIN "Membership" m ON m."userId"=u."id" WHERE m."id" IS NULL;'
```

Expected: `0`

- [ ] **Step 3: Escrever o teste que falha**

Criar `lib/__tests__/auth-on-user-created.test.ts`:

```ts
jest.mock("@/lib/prisma", () => ({
  prismadb: {
    tenant: { findFirst: jest.fn(), create: jest.fn() },
    membership: { count: jest.fn(), create: jest.fn() },
  },
}));

import { prismadb } from "@/lib/prisma";
import { ensureMembershipForNewUser } from "../auth-tenancy";

const tenantFindFirst = prismadb.tenant.findFirst as jest.Mock;
const membershipCount = prismadb.membership.count as jest.Mock;
const membershipCreate = prismadb.membership.create as jest.Mock;

beforeEach(() => jest.clearAllMocks());

describe("ensureMembershipForNewUser", () => {
  it("makes the very first user of an empty tenant its owner", async () => {
    tenantFindFirst.mockResolvedValue({ id: "t1" });
    membershipCount.mockResolvedValue(0);
    await ensureMembershipForNewUser("u1");
    expect(membershipCreate).toHaveBeenCalledWith({
      data: { userId: "u1", tenantId: "t1", role: "owner", status: "ACTIVE" },
    });
  });

  it("makes a subsequent user a plain member, pending activation", async () => {
    tenantFindFirst.mockResolvedValue({ id: "t1" });
    membershipCount.mockResolvedValue(3);
    await ensureMembershipForNewUser("u2");
    expect(membershipCreate).toHaveBeenCalledWith({
      data: { userId: "u2", tenantId: "t1", role: "user", status: "PENDING" },
    });
  });

  it("does nothing when there is no tenant to join", async () => {
    tenantFindFirst.mockResolvedValue(null);
    await ensureMembershipForNewUser("u3");
    expect(membershipCreate).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 4: Correr para confirmar que falha**

Run: `pnpm test -- auth-on-user-created`
Expected: FAIL — `Cannot find module '../auth-tenancy'`

- [ ] **Step 5: Implementar `lib/auth-tenancy.ts`**

```ts
import { prismadb } from "@/lib/prisma";

/**
 * Attach a newly registered user to the default tenant.
 *
 * Onda 1 has a single tenant, so "default" is the oldest ACTIVE one. Onda 2
 * replaces this with invite-driven membership (TenantInvite).
 */
export async function ensureMembershipForNewUser(userId: string): Promise<void> {
  const tenant = await prismadb.tenant.findFirst({
    where: { status: "ACTIVE" },
    orderBy: { createdAt: "asc" },
    select: { id: true },
  });
  if (!tenant) return;

  const existing = await prismadb.membership.count({
    where: { tenantId: tenant.id },
  });

  await prismadb.membership.create({
    data: {
      userId,
      tenantId: tenant.id,
      role: existing === 0 ? "owner" : "user",
      status: existing === 0 ? "ACTIVE" : "PENDING",
    },
  });
}
```

- [ ] **Step 6: Ligar ao callback existente em `lib/auth.ts`**

Dentro de `callbacks.onUserCreated`, imediatamente antes da lógica de `newUserNotify` que já lá está, acrescentar:

```ts
      await ensureMembershipForNewUser(user.id);
```

E o import no topo do ficheiro:

```ts
import { ensureMembershipForNewUser } from "@/lib/auth-tenancy";
```

- [ ] **Step 7: Correr os testes**

Run: `pnpm test -- auth-on-user-created`
Expected: PASS, 3 testes

- [ ] **Step 8: Commit**

```bash
git add prisma/migrations lib/auth-tenancy.ts lib/auth.ts lib/__tests__/auth-on-user-created.test.ts
git commit -m "feat(tenancy): backfill memberships and attach new users to the default tenant"
```

---

## Task 10: Predicado de tenant e de equipa nos scope helpers

**Files:**
- Modify: `lib/authz/scopes/crm.ts` (todos os `*ScopeWhere` e `*ScopedWhere`)
- Modify: `lib/authz/scopes/report-scope.ts`
- Test: `lib/authz/__tests__/scopes-crm-account-read.test.ts` (alterar), `lib/authz/__tests__/tenant-scopes.test.ts` (novo)

**Interfaces:**
- Consumes: `AuthzUser` com `tenantId` e `teamMemberIds` (Task 8)
- Produces: todos os helpers de scope passam a devolver `{ tenantId, ... }`. Novo helper exportado `tenantBase(user: AuthzUser): { tenantId: string }`.

Esta é a task mais importante do plano. A regra: **o `tenantId` entra sempre, incluindo nos early-returns de `admin`/`manager`, que hoje devolvem `{ deletedAt: null }` sem qualquer restrição.**

- [ ] **Step 1: Escrever o teste que falha**

Criar `lib/authz/__tests__/tenant-scopes.test.ts`:

```ts
jest.mock("@/lib/prisma", () => ({
  prismadb: {
    crm_Accounts: { findFirst: jest.fn() },
    crm_Leads: { findFirst: jest.fn() },
  },
}));

import {
  accountReadScopeWhere,
  leadReadScopeWhere,
  contactReadScopeWhere,
  opportunityReadScopeWhere,
  contractReadScopeWhere,
  campaignReadScopeWhere,
  documentReadScopeWhere,
  boardReadScopeWhere,
} from "../scopes/crm";
import type { AuthzUser } from "../session";

const owner: AuthzUser = {
  id: "u1", role: "owner", tenantId: "t1",
  teamMemberIds: ["u1"], isPlatformAdmin: true,
};
const manager: AuthzUser = { ...owner, role: "manager", isPlatformAdmin: false };
const plain: AuthzUser = { ...manager, role: "user" };
const teamed: AuthzUser = { ...plain, teamMemberIds: ["u1", "u2"] };

const READ_SCOPES = [
  ["account", accountReadScopeWhere],
  ["lead", leadReadScopeWhere],
  ["contact", contactReadScopeWhere],
  ["opportunity", opportunityReadScopeWhere],
  ["contract", contractReadScopeWhere],
  ["campaign", campaignReadScopeWhere],
  ["document", documentReadScopeWhere],
  ["board", boardReadScopeWhere],
] as const;

describe("every read scope carries the tenant, for every role", () => {
  it.each(READ_SCOPES)("%s scope pins tenantId for owner", (_name, fn) => {
    expect(fn(owner)).toMatchObject({ tenantId: "t1" });
  });
  it.each(READ_SCOPES)("%s scope pins tenantId for manager", (_name, fn) => {
    expect(fn(manager)).toMatchObject({ tenantId: "t1" });
  });
  it.each(READ_SCOPES)("%s scope pins tenantId for user", (_name, fn) => {
    expect(fn(plain)).toMatchObject({ tenantId: "t1" });
  });
});

describe("team widening", () => {
  it("a user with no team sees only their own records", () => {
    expect(accountReadScopeWhere(plain)).toMatchObject({
      OR: expect.arrayContaining([{ assigned_to: { in: ["u1"] } }]),
    });
  });

  it("a user in a team sees their teammates' records too", () => {
    expect(accountReadScopeWhere(teamed)).toMatchObject({
      OR: expect.arrayContaining([{ assigned_to: { in: ["u1", "u2"] } }]),
    });
  });

  it("manager is not narrowed by team", () => {
    expect(accountReadScopeWhere(manager)).toEqual({
      tenantId: "t1",
      deletedAt: null,
    });
  });
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- tenant-scopes`
Expected: FAIL — os scopes devolvem `{ deletedAt: null }` sem `tenantId`

- [ ] **Step 3: Acrescentar os dois helpers de base ao topo de `lib/authz/scopes/crm.ts`**

Logo a seguir aos imports:

```ts
/** Every scoped where starts here. The tenant is not negotiable. */
export function tenantBase(user: AuthzUser): { tenantId: string } {
  return { tenantId: user.tenantId };
}
```

E reescrever `accountUserScopeOR` para usar o conjunto da equipa em vez do id isolado:

```ts
export function accountUserScopeOR(user: AuthzUser) {
  const ids = user.teamMemberIds;
  return [
    { assigned_to: { in: ids } },
    { createdBy: { in: ids } },
    { watchers: { some: { user_id: { in: ids } } } },
  ];
}
```

- [ ] **Step 4: Aplicar o padrão a todos os `*ScopeWhere`**

O padrão, aplicado sem excepção a `accountReadScopeWhere`, `leadReadScopeWhere`, `contactReadScopeWhere`, `opportunityReadScopeWhere`, `contractReadScopeWhere`, `targetReadScopeWhere`, `targetListReadScopeWhere`, `documentReadScopeWhere`, `campaignReadScopeWhere`, `campaignTemplateReadScopeWhere`, `boardReadScopeWhere`, `boardWriteScopeWhere`, `contactScopedWhere`, `targetScopedWhere`:

```ts
export function accountReadScopeWhere(user: AuthzUser) {
  if (user.role === "admin" || user.role === "owner" || user.role === "manager") {
    return { ...tenantBase(user), deletedAt: null };
  }
  return {
    ...tenantBase(user),
    deletedAt: null,
    OR: accountUserScopeOR(user),
  };
}
```

Notas ao aplicar:
- **`owner` tem de entrar em cada condição** que hoje lista `admin || manager` — caso contrário um owner cai no ramo restrito.
- As chamadas internas a `accountUserScopeOR(user.id)` passam a `accountUserScopeOR(user)`.
- Nos `assertCanWrite*` que constroem o `where` inline (ex.: `assertCanWriteAccount`), acrescentar `...tenantBase(user)` a **ambos** os ramos do ternário.
- Em `assertCanReadActivityForEntity` e `assertCanWriteActivity`, que fazem dispatch por `entityType`, o `tenantBase` entra no `where` final, não em cada ramo.

- [ ] **Step 5: Aplicar o mesmo a `lib/authz/scopes/report-scope.ts`**

Cada fragmento devolvido por `getReportScope(user)` recebe `tenantId: user.tenantId`, incluindo o ramo de `admin`/`manager`.

- [ ] **Step 6: Corrigir os testes existentes**

Os testes em `scopes-crm-account-read.test.ts`, `scopes-crm-read.test.ts`, `scopes-crm-write.test.ts`, `scopes-crm-entity-read.test.ts`, `scopes-crm-target-read.test.ts`, `scopes-crm-campaign-read.test.ts`, `scopes-document-read.test.ts`, `scopes-projects.test.ts` e `report-scope.test.ts` constroem `AuthzUser` como `{ id, role }`. Acrescentar `tenantId: "t1", teamMemberIds: [id], isPlatformAdmin: false` a cada literal, e acrescentar `tenantId: "t1"` a cada `toEqual` de resultado esperado.

Nota: as asserções sobre `accountUserScopeOR` mudam de `{ assigned_to: "u1" }` para `{ assigned_to: { in: ["u1"] } }`.

- [ ] **Step 7: Correr toda a suite authz**

Run: `pnpm test -- lib/authz`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add lib/authz
git commit -m "feat(tenancy): pin tenantId in every scope helper and widen ownership by team"
```

---

## Task 11: Fechar os buracos sem scope

**Files:**
- Modify: `actions/crm/get-crm-data.ts`
- Modify: `actions/get-users.ts`
- Modify: `actions/dashboard/*.ts`
- Modify: `app/api/crm/leads/create-lead-from-web/route.ts`
- Modify: `prisma/schema.prisma` (novo modelo `TenantWebhookToken`)
- Test: `actions/crm/__tests__/get-crm-data-scope.test.ts`, `actions/__tests__/get-users-scope.test.ts`, `app/api/crm/leads/__tests__/create-lead-from-web.test.ts`

**Interfaces:**
- Consumes: `requireAuthenticated`, os `*ScopeWhere` da Task 10
- Produces: `getAllCrmData()` autenticada e scoped; `getUsers()` limitada aos membros do tenant; webhook autenticado por token por tenant.

- [ ] **Step 1: Escrever o teste que falha para `getAllCrmData`**

Criar `actions/crm/__tests__/get-crm-data-scope.test.ts`:

```ts
import { AuthenticationError } from "@/lib/authz/errors";

jest.mock("@/lib/authz/session", () => ({ requireAuthenticated: jest.fn() }));
jest.mock("@/lib/prisma", () => {
  const model = () => ({ findMany: jest.fn().mockResolvedValue([]) });
  return {
    prismadb: {
      crm_Accounts: model(), crm_Opportunities: model(), crm_Leads: model(),
      crm_Contacts: model(), crm_Contracts: model(), crm_campaigns: model(),
      crm_Industry_Type: model(), crm_Opportunities_Sales_Stages: model(),
      crm_Opportunities_Type: model(), crm_Lead_Sources: model(),
      crm_Lead_Statuses: model(), crm_Lead_Types: model(),
      crm_Contact_Types: model(), users: model(),
      crm_campaign_templates: model(), crm_TargetLists: model(),
    },
  };
});

import { requireAuthenticated } from "@/lib/authz/session";
import { prismadb } from "@/lib/prisma";
import { getAllCrmData } from "../get-crm-data";

const mockedRequire = requireAuthenticated as jest.Mock;
beforeEach(() => jest.clearAllMocks());

it("refuses to run unauthenticated", async () => {
  mockedRequire.mockRejectedValue(new AuthenticationError());
  await expect(getAllCrmData()).rejects.toBeInstanceOf(AuthenticationError);
});

it("filters accounts by tenant", async () => {
  mockedRequire.mockResolvedValue({
    id: "u1", role: "manager", tenantId: "t1",
    teamMemberIds: ["u1"], isPlatformAdmin: false,
  });
  await getAllCrmData();
  expect(prismadb.crm_Accounts.findMany).toHaveBeenCalledWith(
    expect.objectContaining({ where: expect.objectContaining({ tenantId: "t1" }) }),
  );
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- get-crm-data-scope`
Expected: FAIL — `getAllCrmData` não chama `requireAuthenticated`

- [ ] **Step 3: Corrigir `actions/crm/get-crm-data.ts`**

No topo da função `getAllCrmData`, antes dos 16 `findMany`:

```ts
  const user = await requireAuthenticated();
```

E cada `findMany` recebe o scope correspondente em vez de `{ deletedAt: null }`:

```ts
    prismadb.crm_Accounts.findMany({ where: accountReadScopeWhere(user) }),
    prismadb.crm_Opportunities.findMany({ where: opportunityReadScopeWhere(user) }),
    prismadb.crm_Leads.findMany({ where: leadReadScopeWhere(user) }),
    prismadb.crm_Contacts.findMany({ where: contactReadScopeWhere(user) }),
    prismadb.crm_Contracts.findMany({ where: contractReadScopeWhere(user) }),
    prismadb.crm_campaigns.findMany({ where: campaignReadScopeWhere(user) }),
```

Os lookups (`crm_Industry_Type`, `crm_Lead_Sources`, …) não têm ownership, só tenant:

```ts
    prismadb.crm_Industry_Type.findMany({ where: { tenantId: user.tenantId } }),
```

E a lista de utilizadores passa a vir das memberships:

```ts
    prismadb.users.findMany({
      where: { memberships: { some: { tenantId: user.tenantId, status: "ACTIVE" } } },
      select: { id: true, name: true, email: true, avatar: true },
    }),
```

- [ ] **Step 4: Corrigir `actions/get-users.ts` do mesmo modo**

Cada uma de `getUsers`, `getActiveUsers`, `getUsersByMonth` começa com `const user = await requireAuthenticated();` e filtra por `memberships: { some: { tenantId: user.tenantId } }`. Para `getActiveUsers`, acrescentar `status: "ACTIVE"` dentro do `some`.

- [ ] **Step 5: Corrigir as contagens de dashboard**

Em cada ficheiro de `actions/dashboard/`, o padrão passa de:

```ts
  return prismadb.crm_Accounts.count({ where: { deletedAt: null } });
```

para:

```ts
  const user = await requireAuthenticated();
  return prismadb.crm_Accounts.count({ where: accountReadScopeWhere(user) });
```

- [ ] **Step 6: Substituir o token partilhado do webhook por um token por tenant**

Acrescentar ao `prisma/schema.prisma`:

```prisma
model TenantWebhookToken {
  id         String    @id @default(uuid()) @db.Uuid
  tenantId   String    @db.Uuid
  name       String
  tokenHash  String    @unique
  tokenPrefix String
  createdBy  String    @db.Uuid
  lastUsedAt DateTime?
  revokedAt  DateTime?
  createdAt  DateTime  @default(now())

  tenant Tenant @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@index([tenantId])
}
```

Acrescentar `webhookTokens TenantWebhookToken[]` a `Tenant`, gerar e aplicar a migração, e reescrever a verificação em `app/api/crm/leads/create-lead-from-web/route.ts`:

```ts
import { createHash, timingSafeEqual } from "node:crypto";

const raw = req.headers.get("authorization")?.trim();
if (!raw) {
  return NextResponse.json({ message: "Unauthorized" }, { status: 401 });
}

const hash = createHash("sha256").update(raw).digest("hex");
const record = await prismadb.tenantWebhookToken.findFirst({
  where: { tokenHash: hash, revokedAt: null },
  select: { id: true, tenantId: true },
});
if (!record) {
  return NextResponse.json({ message: "Unauthorized" }, { status: 401 });
}

// … depois, na criação:
await prismadb.crm_Leads.create({
  data: { v: 1, tenantId: record.tenantId, firstName, lastName, company: account, jobTitle: job, email, phone },
});
```

Nota: `findFirst` por `tokenHash` já é uma comparação em índice, constante do lado do Postgres — não há necessidade de `timingSafeEqual` sobre o hash. Remover o import se não for usado. O `NEXTCRM_TOKEN` sai do `.env.example` e do README.

- [ ] **Step 7: Correr os testes e o typecheck**

```bash
pnpm test -- get-crm-data-scope get-users-scope create-lead-from-web
pnpm exec tsc --noEmit
```
Expected: PASS e zero erros de tipos

- [ ] **Step 8: Commit**

```bash
git add actions app/api/crm/leads prisma lib
git commit -m "fix(tenancy): scope get-crm-data, get-users, dashboard counts and the lead webhook"
```

---

## Task 12: Filtro de tenant no SQL cru

**Files:**
- Modify: `actions/crm/similarity/get-similar-{accounts,contacts,leads,opportunities}.ts`
- Modify: `actions/fulltext/unified-search.ts`
- Modify: `actions/documents/search-documents.ts`
- Modify: `inngest/functions/embed-{account,contact,lead,opportunity}.ts`, `inngest/functions/emails/embed-email.ts`, `inngest/functions/documents/enrich-document.ts`
- Test: alterar os `__tests__/*-scope.test.ts` já existentes ao lado de cada um

**Interfaces:**
- Consumes: `AuthzUser.tenantId`; a coluna `tenantId` nas tabelas de embedding (Task 4)
- Produces: nenhuma nova API — todas as queries `$queryRaw` passam a ter `e."tenantId" = ${user.tenantId}::uuid`.

A extensão Prisma **não intercepta `$queryRaw`**. Estes 11 ficheiros são o único sítio onde o isolamento é manual nesta onda.

- [ ] **Step 1: Alterar o teste de scope de `get-similar-accounts`**

Em `actions/crm/similarity/__tests__/get-similar-accounts-scope.test.ts`, acrescentar:

```ts
it("pins the tenant in the raw vector query", async () => {
  mockedRequire.mockResolvedValue({
    id: "u1", role: "manager", tenantId: "t1",
    teamMemberIds: ["u1"], isPlatformAdmin: false,
  });
  queryRaw.mockResolvedValue([]);
  await getSimilarAccounts("a1");
  const params = queryRaw.mock.calls[0].slice(1);
  expect(params).toContain("t1");
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- get-similar-accounts-scope`
Expected: FAIL — `"t1"` não aparece nos parâmetros

- [ ] **Step 3: Acrescentar o predicado à query**

Em cada um dos 6 ficheiros de leitura, o `WHERE` da template tag ganha uma linha. Exemplo em `get-similar-accounts.ts`:

```ts
  const rows = await prismadb.$queryRaw<SimilarRow[]>`
    SELECT a."id", a."name",
           1 - (e."embedding" <=> ${vector}::vector) AS score
    FROM "crm_Embeddings_Accounts" e
    JOIN "crm_Accounts" a ON a."id" = e."accountId"
    WHERE e."tenantId" = ${user.tenantId}::uuid
      AND a."tenantId" = ${user.tenantId}::uuid
      AND a."deletedAt" IS NULL
      AND a."id" <> ${accountId}::uuid
    ORDER BY e."embedding" <=> ${vector}::vector
    LIMIT ${limit}
  `;
```

O filtro entra em **ambas** as tabelas: na de embedding para que o índice vectorial seja percorrido já restrito, e na tabela de negócio como redundância.

- [ ] **Step 4: Fazer o mesmo em `unified-search.ts` e `search-documents.ts`**

Nestes, o `WHERE` usa `search_vector @@ ...`; acrescentar `AND "tenantId" = ${user.tenantId}::uuid` a cada uma das sub-queries `UNION`.

- [ ] **Step 5: Escrever o `tenantId` nas 5 funções Inngest de escrita**

Estas funções fazem `INSERT ... ON CONFLICT` nas tabelas de embedding. Como não têm sessão, o `tenantId` tem de vir do registo pai. Em `inngest/functions/embed-account.ts`:

```ts
  const account = await prismadb.crm_Accounts.findUnique({
    where: { id: accountId },
    select: { id: true, tenantId: true, name: true, description: true },
  });
  if (!account) return { skipped: "account-not-found" };

  await prismadb.$executeRaw`
    INSERT INTO "crm_Embeddings_Accounts" ("id", "tenantId", "accountId", "embedding", "content_hash", "embedded_at")
    VALUES (gen_random_uuid(), ${account.tenantId}::uuid, ${account.id}::uuid, ${vector}::vector, ${hash}, now())
    ON CONFLICT ("accountId") DO UPDATE
      SET "embedding" = EXCLUDED."embedding",
          "content_hash" = EXCLUDED."content_hash",
          "embedded_at" = now()
  `;
```

Nota: estas funções usam `prismadb` directamente, não o cliente estendido — correcto, porque correm sem contexto de tenant. É por isso que o `tenantId` tem de vir explicitamente do registo pai.

- [ ] **Step 6: Correr os testes de similaridade e pesquisa**

Run: `pnpm test -- similarity fulltext search-documents`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add actions/crm/similarity actions/fulltext actions/documents inngest/functions
git commit -m "fix(tenancy): filter raw pgvector and tsvector queries by tenant"
```

---

## Task 13: Prefixo de tenant no armazenamento e séries de factura por tenant

**Files:**
- Modify: `app/api/upload/presigned-url/route.ts`
- Test: `app/api/upload/__tests__/presigned-url.test.ts`
- Modify: `lib/invoices/` (a função que incrementa o contador da série)

**Interfaces:**
- Consumes: `requireAuthenticated`
- Produces: chaves S3 no formato `tenants/{tenantId}/{folder}/{uuid}.{ext}`; incremento de contador de série sob bloqueio pessimista.

Estas duas coisas são baratas agora e caras depois: mudar o formato das chaves com objectos já criados obriga a migrar ficheiros, e um contador de facturas partilhado é um problema de conformidade.

- [ ] **Step 1: Escrever o teste que falha**

Criar `app/api/upload/__tests__/presigned-url.test.ts`:

```ts
jest.mock("@/lib/authz/session", () => ({ requireAuthenticated: jest.fn() }));
jest.mock("@/lib/minio", () => ({
  s3Client: { send: jest.fn() },
  MINIO_BUCKET: "bucket",
  MINIO_PUBLIC_URL: "http://minio",
}));
jest.mock("@aws-sdk/s3-request-presigner", () => ({
  getSignedUrl: jest.fn().mockResolvedValue("http://signed"),
}));

import { requireAuthenticated } from "@/lib/authz/session";
import { POST } from "../presigned-url/route";

beforeEach(() => {
  (requireAuthenticated as jest.Mock).mockResolvedValue({
    id: "u1", role: "user", tenantId: "t1",
    teamMemberIds: ["u1"], isPlatformAdmin: false,
  });
});

it("prefixes the object key with the tenant", async () => {
  const req = new Request("http://x/api/upload/presigned-url", {
    method: "POST",
    body: JSON.stringify({ folder: "documents", filename: "a.pdf" }),
    headers: { "content-type": "application/json" },
  });
  const res = await POST(req as any);
  const body = await res.json();
  expect(body.key).toMatch(/^tenants\/t1\/documents\/[0-9a-f-]+\.pdf$/);
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- presigned-url`
Expected: FAIL — a chave sai como `documents/<uuid>.pdf`

- [ ] **Step 3: Alterar a geração da chave**

Em `app/api/upload/presigned-url/route.ts`, substituir a linha 41:

```ts
const key = `tenants/${user.tenantId}/${folder}/${randomUUID()}.${ext}`;
```

garantindo que `const user = await requireAuthenticated();` corre antes (se ainda não correr).

- [ ] **Step 4: Tornar o incremento da série pessimista**

Localizar a função que faz `update` no `counter` de `Invoice_Series` em `lib/invoices/`. Envolver a leitura-e-incremento numa transacção interactiva com `FOR UPDATE`:

```ts
export async function nextInvoiceNumber(
  tenantId: string,
  seriesId: string,
): Promise<string> {
  return prismadb.$transaction(async (tx) => {
    const [series] = await tx.$queryRaw<
      { id: string; prefixTemplate: string; counter: number; currentYear: number | null; resetPolicy: string }[]
    >`
      SELECT "id", "prefixTemplate", "counter", "currentYear", "resetPolicy"
      FROM "Invoice_Series"
      WHERE "id" = ${seriesId}::uuid AND "tenantId" = ${tenantId}::uuid
      FOR UPDATE
    `;
    if (!series) throw new Error("Invoice series not found in this tenant");

    const year = new Date().getFullYear();
    const reset = series.resetPolicy === "YEARLY" && series.currentYear !== year;
    const next = reset ? 1 : series.counter + 1;

    await tx.invoice_Series.update({
      where: { id: series.id },
      data: { counter: next, currentYear: year },
    });

    return series.prefixTemplate
      .replace("{YYYY}", String(year))
      .replace("{NNNN}", String(next).padStart(4, "0"));
  });
}
```

- [ ] **Step 5: Correr os testes**

Run: `pnpm test -- presigned-url invoices`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/api/upload lib/invoices
git commit -m "feat(tenancy): prefix storage keys by tenant and lock invoice series counters"
```

---

## Task 14: UI de equipas e troca de tenant

**Files:**
- Create: `actions/tenant/create-team.ts`, `actions/tenant/add-team-member.ts`, `actions/tenant/remove-team-member.ts`, `actions/tenant/switch-tenant.ts`
- Create: `app/[locale]/(routes)/admin/teams/page.tsx`, `app/[locale]/(routes)/admin/teams/components/TeamList.tsx`, `.../TeamMemberPicker.tsx`
- Modify: `app/[locale]/(routes)/components/app-sidebar.tsx` (selector de tenant, só visível com >1 membership)
- Modify: `locales/pt.json`, `locales/en.json` (chaves novas)
- Test: `actions/tenant/__tests__/create-team.test.ts`, `actions/tenant/__tests__/switch-tenant.test.ts`

**Interfaces:**
- Consumes: `requireRole` hierárquico (Task 8), `ACTIVE_TENANT_COOKIE` (Task 7)
- Produces: `createTeam(name: string)`, `addTeamMember(teamId: string, userId: string)`, `removeTeamMember(teamId: string, userId: string)`, `switchTenant(tenantId: string)` — todas server actions que devolvem `{ ok: true } | { error: string }`.

- [ ] **Step 1: Escrever os testes que falham**

Criar `actions/tenant/__tests__/create-team.test.ts`:

```ts
import { AuthorizationError } from "@/lib/authz/errors";

jest.mock("@/lib/authz/session", () => ({ requireRole: jest.fn() }));
jest.mock("@/lib/prisma", () => ({
  prismadb: { team: { create: jest.fn() } },
}));
jest.mock("next/cache", () => ({ revalidatePath: jest.fn() }));

import { requireRole } from "@/lib/authz/session";
import { prismadb } from "@/lib/prisma";
import { createTeam } from "../create-team";

const mockedRole = requireRole as jest.Mock;
beforeEach(() => jest.clearAllMocks());

it("requires at least the admin role", async () => {
  mockedRole.mockRejectedValue(new AuthorizationError());
  await expect(createTeam("Comercial")).resolves.toEqual({ error: "FORBIDDEN" });
  expect(mockedRole).toHaveBeenCalledWith(["admin"]);
});

it("creates the team inside the caller's tenant", async () => {
  mockedRole.mockResolvedValue({
    id: "u1", role: "admin", tenantId: "t1",
    teamMemberIds: ["u1"], isPlatformAdmin: false,
  });
  await createTeam("Comercial");
  expect(prismadb.team.create).toHaveBeenCalledWith({
    data: { tenantId: "t1", name: "Comercial" },
  });
});

it("rejects an empty name without touching the database", async () => {
  mockedRole.mockResolvedValue({
    id: "u1", role: "admin", tenantId: "t1",
    teamMemberIds: ["u1"], isPlatformAdmin: false,
  });
  await expect(createTeam("   ")).resolves.toEqual({ error: "NAME_REQUIRED" });
  expect(prismadb.team.create).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Correr para confirmar que falha**

Run: `pnpm test -- actions/tenant`
Expected: FAIL — `Cannot find module '../create-team'`

- [ ] **Step 3: Implementar `actions/tenant/create-team.ts`**

```ts
"use server";

import { revalidatePath } from "next/cache";
import { prismadb } from "@/lib/prisma";
import { requireRole } from "@/lib/authz/session";
import { AuthenticationError, AuthorizationError } from "@/lib/authz/errors";

export async function createTeam(
  name: string,
): Promise<{ ok: true } | { error: string }> {
  let user;
  try {
    user = await requireRole(["admin"]);
  } catch (e) {
    if (e instanceof AuthenticationError) return { error: "UNAUTHENTICATED" };
    if (e instanceof AuthorizationError) return { error: "FORBIDDEN" };
    throw e;
  }

  const trimmed = name.trim();
  if (!trimmed) return { error: "NAME_REQUIRED" };

  await prismadb.team.create({
    data: { tenantId: user.tenantId, name: trimmed },
  });

  revalidatePath("/admin/teams");
  return { ok: true };
}
```

- [ ] **Step 4: Implementar `addTeamMember` e `removeTeamMember`**

Mesmo esqueleto de erros. O ponto crítico de segurança em ambas: confirmar que a equipa **e** o utilizador pertencem ao tenant do chamador antes de escrever.

```ts
"use server";

import { revalidatePath } from "next/cache";
import { prismadb } from "@/lib/prisma";
import { requireRole } from "@/lib/authz/session";
import { AuthenticationError, AuthorizationError } from "@/lib/authz/errors";

export async function addTeamMember(
  teamId: string,
  userId: string,
): Promise<{ ok: true } | { error: string }> {
  let user;
  try {
    user = await requireRole(["admin"]);
  } catch (e) {
    if (e instanceof AuthenticationError) return { error: "UNAUTHENTICATED" };
    if (e instanceof AuthorizationError) return { error: "FORBIDDEN" };
    throw e;
  }

  // Both sides must live in the caller's tenant — otherwise an admin could
  // graft a foreign user onto their team and widen their own read scope.
  const team = await prismadb.team.findFirst({
    where: { id: teamId, tenantId: user.tenantId },
    select: { id: true },
  });
  if (!team) return { error: "TEAM_NOT_FOUND" };

  const membership = await prismadb.membership.findFirst({
    where: { userId, tenantId: user.tenantId, status: "ACTIVE" },
    select: { id: true },
  });
  if (!membership) return { error: "USER_NOT_IN_TENANT" };

  await prismadb.teamMembership.upsert({
    where: { teamId_userId: { teamId, userId } },
    create: { teamId, userId },
    update: {},
  });

  revalidatePath("/admin/teams");
  return { ok: true };
}
```

`removeTeamMember` é o mesmo até à verificação da equipa, seguido de `prismadb.teamMembership.deleteMany({ where: { teamId, userId } })`.

- [ ] **Step 5: Implementar `actions/tenant/switch-tenant.ts`**

```ts
"use server";

import { cookies } from "next/headers";
import { prismadb } from "@/lib/prisma";
import { getSession } from "@/lib/auth-server";
import { ACTIVE_TENANT_COOKIE } from "@/lib/tenant/resolve";

export async function switchTenant(
  tenantId: string,
): Promise<{ ok: true } | { error: string }> {
  const session = await getSession();
  const userId = session?.user?.id;
  if (!userId) return { error: "UNAUTHENTICATED" };

  // Never trust the submitted tenantId — it must match a live membership.
  const membership = await prismadb.membership.findFirst({
    where: { userId, tenantId, status: "ACTIVE", tenant: { status: "ACTIVE" } },
    select: { id: true },
  });
  if (!membership) return { error: "FORBIDDEN" };

  (await cookies()).set(ACTIVE_TENANT_COOKIE, tenantId, {
    httpOnly: true,
    sameSite: "lax",
    secure: process.env.NODE_ENV === "production",
    path: "/",
    maxAge: 60 * 60 * 24 * 30,
  });

  return { ok: true };
}
```

- [ ] **Step 6: Construir a página de equipas**

`app/[locale]/(routes)/admin/teams/page.tsx` — server component sob o `AdminLayout` que já faz `requireRole(["admin"])`:

```tsx
import { prismadb } from "@/lib/prisma";
import { requireRole } from "@/lib/authz/session";
import TeamList from "./components/TeamList";

export default async function TeamsPage() {
  const user = await requireRole(["admin"]);

  const [teams, members] = await Promise.all([
    prismadb.team.findMany({
      where: { tenantId: user.tenantId },
      select: {
        id: true,
        name: true,
        members: { select: { user: { select: { id: true, name: true, email: true } } } },
      },
      orderBy: { name: "asc" },
    }),
    prismadb.users.findMany({
      where: { memberships: { some: { tenantId: user.tenantId, status: "ACTIVE" } } },
      select: { id: true, name: true, email: true },
      orderBy: { name: "asc" },
    }),
  ]);

  return <TeamList teams={teams} tenantMembers={members} />;
}
```

`app/[locale]/(routes)/admin/teams/components/TeamList.tsx`:

```tsx
"use client";

import { useState, useTransition } from "react";
import { useRouter } from "@/i18n/navigation";
import { useTranslations } from "next-intl";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import {
  Table, TableBody, TableCell, TableHead, TableHeader, TableRow,
} from "@/components/ui/table";
import {
  Select, SelectContent, SelectItem, SelectTrigger, SelectValue,
} from "@/components/ui/select";
import { createTeam } from "@/actions/tenant/create-team";
import { addTeamMember } from "@/actions/tenant/add-team-member";
import { removeTeamMember } from "@/actions/tenant/remove-team-member";

type Member = { id: string; name: string | null; email: string };
type TeamRow = { id: string; name: string; members: { user: Member }[] };

export default function TeamList({
  teams,
  tenantMembers,
}: {
  teams: TeamRow[];
  tenantMembers: Member[];
}) {
  const t = useTranslations("Teams");
  const router = useRouter();
  const [pending, startTransition] = useTransition();
  const [newName, setNewName] = useState("");

  function run(action: () => Promise<{ ok: true } | { error: string }>) {
    startTransition(async () => {
      const result = await action();
      if ("error" in result) {
        toast.error(result.error);
        return;
      }
      router.refresh();
    });
  }

  return (
    <div className="space-y-6 p-4">
      <div className="flex items-end gap-2">
        <div className="flex-1">
          <label className="text-sm font-medium" htmlFor="team-name">
            {t("teamName")}
          </label>
          <Input
            id="team-name"
            value={newName}
            onChange={(e) => setNewName(e.target.value)}
            disabled={pending}
          />
        </div>
        <Button
          disabled={pending || !newName.trim()}
          onClick={() =>
            run(async () => {
              const r = await createTeam(newName);
              if (!("error" in r)) setNewName("");
              return r;
            })
          }
        >
          {t("newTeam")}
        </Button>
      </div>

      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>{t("title")}</TableHead>
            <TableHead>{t("members")}</TableHead>
            <TableHead className="w-64">{t("addMember")}</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {teams.map((team) => {
            const memberIds = new Set(team.members.map((m) => m.user.id));
            const candidates = tenantMembers.filter((u) => !memberIds.has(u.id));
            return (
              <TableRow key={team.id}>
                <TableCell className="font-medium">{team.name}</TableCell>
                <TableCell>
                  <ul className="space-y-1">
                    {team.members.map(({ user }) => (
                      <li key={user.id} className="flex items-center gap-2">
                        <span>{user.name ?? user.email}</span>
                        <Button
                          variant="ghost"
                          size="sm"
                          disabled={pending}
                          onClick={() => run(() => removeTeamMember(team.id, user.id))}
                        >
                          {t("removeMember")}
                        </Button>
                      </li>
                    ))}
                  </ul>
                </TableCell>
                <TableCell>
                  <Select
                    disabled={pending || candidates.length === 0}
                    onValueChange={(userId) => run(() => addTeamMember(team.id, userId))}
                  >
                    <SelectTrigger>
                      <SelectValue placeholder={t("addMember")} />
                    </SelectTrigger>
                    <SelectContent>
                      {candidates.map((u) => (
                        <SelectItem key={u.id} value={u.id}>
                          {u.name ?? u.email}
                        </SelectItem>
                      ))}
                    </SelectContent>
                  </Select>
                </TableCell>
              </TableRow>
            );
          })}
        </TableBody>
      </Table>
    </div>
  );
}
```

Acrescentar a entrada `/admin/teams` ao array de navegação em [app/[locale]/(routes)/admin/_components/AdminSidebarNav.tsx](../../../app/[locale]/(routes)/admin/_components/AdminSidebarNav.tsx) — seguir exactamente a forma das entradas já lá listadas (`users`, `currencies`, `audit-log`, …).

- [ ] **Step 7: Acrescentar o selector de tenant à sidebar**

Em `app/[locale]/(routes)/components/app-sidebar.tsx`, acima da navegação, renderizar um `Select` com as memberships do utilizador — **só quando houver mais do que uma**. `onValueChange` chama `switchTenant` e depois `router.refresh()`.

Chaves novas em `locales/pt.json` e `locales/en.json`:

```json
{
  "Teams": {
    "title": "Equipas",
    "newTeam": "Nova equipa",
    "teamName": "Nome da equipa",
    "members": "Membros",
    "addMember": "Adicionar membro",
    "removeMember": "Remover",
    "switchTenant": "Organização"
  }
}
```

Traduzir para as restantes locales (`en`, `cz`, `de`, `uk`) — o `en` é obrigatório porque é o fallback.

- [ ] **Step 8: Correr os testes e o lint**

```bash
pnpm test -- actions/tenant
pnpm lint
```
Expected: PASS e zero warnings

- [ ] **Step 9: Commit**

```bash
git add actions/tenant "app/[locale]/(routes)/admin/teams" "app/[locale]/(routes)/components/app-sidebar.tsx" locales
git commit -m "feat(tenancy): add team management screens and tenant switcher"
```

---

## Task 15: Harness de isolamento e guarda de CI

**Files:**
- Create: `__tests__/tenant-isolation/setup.ts`, `__tests__/tenant-isolation/crm-isolation.integration.test.ts`
- Create: `eslint-rules/no-direct-prismadb.js`
- Modify: `jest.config.ts`, `eslint.config.mjs`, `package.json`

**Interfaces:**
- Consumes: tudo o que veio antes
- Produces: `pnpm test:isolation`; a regra ESLint `local/no-direct-prismadb`.

- [ ] **Step 1: Configurar um segundo projecto Jest para integração**

Substituir `jest.config.ts` por:

```ts
import type { Config } from "jest";

const shared = {
  preset: "ts-jest",
  testEnvironment: "node" as const,
  setupFiles: ["<rootDir>/jest.env.setup.ts"],
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/$1",
    "^e2b$": "<rootDir>/__mocks__/e2b.ts",
  },
  transform: {
    "^.+\\.tsx?$": ["ts-jest", { tsconfig: { jsx: "react", target: "es2017" } }],
  },
};

const config: Config = {
  projects: [
    {
      ...shared,
      displayName: "unit",
      testMatch: ["**/__tests__/**/*.test.ts", "**/__tests__/**/*.test.tsx"],
      testPathIgnorePatterns: ["/node_modules/", "\\.integration\\.test\\.ts$"],
    },
    {
      ...shared,
      displayName: "isolation",
      testMatch: ["**/__tests__/tenant-isolation/**/*.integration.test.ts"],
    },
  ],
};

export default config;
```

E em `package.json`:

```json
    "test": "jest --selectProjects unit",
    "test:isolation": "pnpm db:guard && jest --selectProjects isolation --runInBand",
```

- [ ] **Step 2: Escrever o setup com dois tenants**

Criar `__tests__/tenant-isolation/setup.ts`:

```ts
import { PrismaClient } from "@prisma/client";
import { Pool } from "pg";
import { PrismaPg } from "@prisma/adapter-pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = new PrismaClient({ adapter: new PrismaPg(pool) });

export const TENANT_A = "aaaaaaaa-0000-0000-0000-000000000001";
export const TENANT_B = "bbbbbbbb-0000-0000-0000-000000000002";
export const USER_A = "aaaaaaaa-1111-0000-0000-000000000001";
export const USER_B = "bbbbbbbb-1111-0000-0000-000000000002";

/** Two tenants, one manager each, one account each. Symmetric on purpose. */
export async function seedTwoTenants() {
  await teardownTwoTenants();

  for (const [tenantId, slug, userId, email] of [
    [TENANT_A, "tenant-a", USER_A, "a@test.local"],
    [TENANT_B, "tenant-b", USER_B, "b@test.local"],
  ] as const) {
    await db.tenant.create({
      data: { id: tenantId, slug, name: slug, status: "ACTIVE" },
    });
    await db.users.create({
      data: { id: userId, email, name: slug, role: "user", userStatus: "ACTIVE" },
    });
    await db.membership.create({
      data: { userId, tenantId, role: "manager", status: "ACTIVE" },
    });
    await db.crm_Accounts.create({
      data: { tenantId, name: `Account of ${slug}`, assigned_to: userId, createdBy: userId },
    });
  }
}

export async function teardownTwoTenants() {
  const tenants = [TENANT_A, TENANT_B];
  await db.crm_Accounts.deleteMany({ where: { tenantId: { in: tenants } } });
  await db.teamMembership.deleteMany({ where: { userId: { in: [USER_A, USER_B] } } });
  await db.membership.deleteMany({ where: { tenantId: { in: tenants } } });
  await db.users.deleteMany({ where: { id: { in: [USER_A, USER_B] } } });
  await db.tenant.deleteMany({ where: { id: { in: tenants } } });
}
```

- [ ] **Step 3: Escrever o teste de isolamento**

Criar `__tests__/tenant-isolation/crm-isolation.integration.test.ts`:

```ts
import {
  db, seedTwoTenants, teardownTwoTenants,
  TENANT_A, TENANT_B, USER_A,
} from "./setup";
import {
  accountReadScopeWhere,
  leadReadScopeWhere,
  contactReadScopeWhere,
  opportunityReadScopeWhere,
} from "@/lib/authz/scopes/crm";
import type { AuthzUser } from "@/lib/authz/session";

const managerA: AuthzUser = {
  id: USER_A, role: "manager", tenantId: TENANT_A,
  teamMemberIds: [USER_A], isPlatformAdmin: false,
};

beforeAll(seedTwoTenants);
afterAll(async () => {
  await teardownTwoTenants();
  await db.$disconnect();
});

describe("a manager of tenant A never sees tenant B", () => {
  it("reads only its own accounts", async () => {
    const rows = await db.crm_Accounts.findMany({
      where: accountReadScopeWhere(managerA),
      select: { tenantId: true },
    });
    expect(rows.length).toBeGreaterThan(0);
    expect(rows.every((r) => r.tenantId === TENANT_A)).toBe(true);
  });

  it.each([
    ["leads", () => db.crm_Leads.findMany({ where: leadReadScopeWhere(managerA), select: { tenantId: true } })],
    ["contacts", () => db.crm_Contacts.findMany({ where: contactReadScopeWhere(managerA), select: { tenantId: true } })],
    ["opportunities", () => db.crm_Opportunities.findMany({ where: opportunityReadScopeWhere(managerA), select: { tenantId: true } })],
  ])("returns zero rows of tenant B for %s", async (_name, run) => {
    const rows = await run();
    expect(rows.filter((r) => r.tenantId === TENANT_B)).toHaveLength(0);
  });

  it("cannot update tenant B's account through a scoped where", async () => {
    const result = await db.crm_Accounts.updateMany({
      where: { ...accountReadScopeWhere(managerA), name: "Account of tenant-b" },
      data: { name: "hijacked" },
    });
    expect(result.count).toBe(0);
  });
});
```

- [ ] **Step 4: Correr contra a base de dados real**

```bash
pnpm db:up && pnpm db:wait && pnpm test:isolation
```
Expected: PASS, todos os testes

- [ ] **Step 5: Escrever a regra ESLint**

Criar `eslint-rules/no-direct-prismadb.js`:

```js
/** Business code must go through the tenant-scoped client, not the raw one. */
module.exports = {
  meta: {
    type: "problem",
    docs: { description: "disallow importing prismadb outside allowlisted paths" },
    schema: [],
    messages: {
      direct:
        "Import { tenantScopedClient } from '@/lib/db/tenant-client' instead. " +
        "If this file genuinely runs without a tenant (seeds, inngest, auth, " +
        "the tenancy layer itself), add it to ALLOWED in eslint-rules/no-direct-prismadb.js.",
    },
  },
  create(context) {
    const ALLOWED = [
      /[\\/]lib[\\/]prisma\.ts$/,
      /[\\/]lib[\\/]db[\\/]/,
      /[\\/]lib[\\/]tenant[\\/]/,
      /[\\/]lib[\\/]authz[\\/]/,
      /[\\/]lib[\\/]auth(-server|-tenancy)?\.ts$/,
      /[\\/]inngest[\\/]/,
      /[\\/]prisma[\\/]seeds[\\/]/,
      /[\\/]scripts[\\/]/,
      /__tests__[\\/]/,
      /__mocks__[\\/]/,
    ];
    const filename = context.filename ?? context.getFilename();
    if (ALLOWED.some((re) => re.test(filename))) return {};

    return {
      ImportDeclaration(node) {
        if (!/@\/lib\/prisma$/.test(node.source.value)) return;
        const hit = node.specifiers.find(
          (s) => s.imported && s.imported.name === "prismadb",
        );
        if (hit) context.report({ node: hit, messageId: "direct" });
      },
    };
  },
};
```

- [ ] **Step 6: Registar a regra em `eslint.config.mjs`**

```js
import noDirectPrismadb from "./eslint-rules/no-direct-prismadb.js";

// dentro do array de config, num novo objecto:
{
  plugins: { local: { rules: { "no-direct-prismadb": noDirectPrismadb } } },
  rules: { "local/no-direct-prismadb": "warn" },
},
```

Começa em `"warn"` porque os ~481 sítios existentes vão disparar. A migração progressiva desses ficheiros é a Onda 2; subir para `"error"` quando a lista chegar a zero.

- [ ] **Step 7: Correr lint e a suite completa**

```bash
pnpm lint
pnpm test
pnpm test:isolation
```
Expected: lint sem *errors* (warnings da regra nova são esperados), ambas as suites verdes

- [ ] **Step 8: Commit**

```bash
git add jest.config.ts package.json __tests__/tenant-isolation eslint-rules eslint.config.mjs
git commit -m "test(tenancy): add cross-tenant isolation harness and prismadb import guard"
```

---

## Task 16: Seed de dois tenants e verificação ponta a ponta

**Files:**
- Modify: `prisma/seeds/seed.ts`
- Create: `prisma/seeds/tenants.ts`
- Modify: `docs/specs/2026-08-08-multi-tenant-analysis.md` (marcar Onda 1 concluída)

**Interfaces:**
- Consumes: tudo
- Produces: `seedTenants(prisma)` — cria `sanjila` e, sob `SEED_DEMO_DATA=1`, também `china-express` com lookups próprios.

Nota crítica: o `upsertByName` actual em `prisma/seeds/seed.ts` faz `where: { name: item.name }`, que deixa de compilar depois da Task 3 (o único passou a `[tenantId, name]`). Esta task corrige-o.

- [ ] **Step 1: Criar `prisma/seeds/tenants.ts`**

```ts
import type { PrismaClient } from "@prisma/client";

export const SANJILA_TENANT_ID = "00000000-0000-0000-0000-000000000001";
export const CHINA_EXPRESS_TENANT_ID = "00000000-0000-0000-0000-000000000002";

export async function seedTenants(prisma: PrismaClient): Promise<string[]> {
  await prisma.tenant.upsert({
    where: { id: SANJILA_TENANT_ID },
    update: {},
    create: {
      id: SANJILA_TENANT_ID,
      slug: "sanjila",
      name: "Sanjila",
      status: "ACTIVE",
      defaultLocale: "pt",
      timezone: "Africa/Luanda",
    },
  });

  const ids = [SANJILA_TENANT_ID];

  if (process.env.SEED_DEMO_DATA === "1") {
    await prisma.tenant.upsert({
      where: { id: CHINA_EXPRESS_TENANT_ID },
      update: {},
      create: {
        id: CHINA_EXPRESS_TENANT_ID,
        slug: "china-express",
        name: "China Express",
        status: "ACTIVE",
        defaultLocale: "pt",
        timezone: "Africa/Luanda",
      },
    });
    ids.push(CHINA_EXPRESS_TENANT_ID);
  }

  return ids;
}
```

- [ ] **Step 2: Corrigir `upsertByName` para o unique composto**

Em `prisma/seeds/seed.ts`, substituir a função:

```ts
async function upsertByName(
  model: any,
  tenantId: string,
  items: { name: string; [key: string]: any }[],
) {
  for (const item of items) {
    await model.upsert({
      where: { tenantId_name: { tenantId, name: item.name } },
      update: { ...item, tenantId },
      create: { ...item, tenantId },
    });
  }
}
```

- [ ] **Step 3: Semear os lookups por tenant**

Em `main()`, envolver a secção de lookups num ciclo:

```ts
  const tenantIds = await seedTenants(prisma);

  for (const tenantId of tenantIds) {
    await upsertByName(prisma.crm_Contact_Types, tenantId, contactTypesData);
    await upsertByName(prisma.crm_Lead_Sources, tenantId, leadSourcesData);
    await upsertByName(prisma.crm_Lead_Statuses, tenantId, leadStatusesData);
    await upsertByName(prisma.crm_Lead_Types, tenantId, leadTypesData);

    // These four have no unique on name — findFirst scoped by tenant, then create/update.
    for (const [model, data] of [
      [prisma.crm_Opportunities_Type, crmOpportunityTypeData],
      [prisma.crm_Opportunities_Sales_Stages, crmOpportunitySaleStagesData],
      [prisma.crm_Industry_Type, crmIndustryTypeData],
    ] as const) {
      for (const item of data as any[]) {
        const existing = await (model as any).findFirst({
          where: { tenantId, name: item.name },
        });
        if (existing) {
          await (model as any).update({ where: { id: existing.id }, data: { ...item, tenantId } });
        } else {
          await (model as any).create({ data: { ...item, tenantId } });
        }
      }
    }
  }
```

`seedCurrencies` fica fora do ciclo — `Currency` é global. `seedInvoices` corre uma vez por tenant, recebendo o `tenantId`.

- [ ] **Step 4: Reset completo e seed**

```bash
pnpm db:reset
```
Expected: termina sem erro e imprime as linhas de seed já existentes

- [ ] **Step 5: Confirmar que os dois tenants têm lookups independentes**

```bash
docker compose -f docker-compose.dev.yml exec -T postgres psql -U nextcrm -d nextcrm -c \
  'SELECT t."slug", count(l.*) FROM "Tenant" t LEFT JOIN "crm_Lead_Sources" l ON l."tenantId"=t."id" GROUP BY 1 ORDER BY 1;'
```
Expected: duas linhas, ambas com a mesma contagem não-nula

- [ ] **Step 6: Verificação manual na aplicação**

```bash
pnpm dev
```

Percorrer, e confirmar cada ponto:
1. Login como o utilizador da Sanjila → o dashboard mostra contagens só da Sanjila
2. `/admin/teams` → criar a equipa "Comercial", adicionar dois utilizadores
3. Login como um `user` (não manager) dessa equipa → vê as contas atribuídas ao colega de equipa, e nenhuma das restantes
4. Retirar o utilizador da equipa → deixa de ver as contas do colega
5. Dar ao utilizador uma segunda membership em `china-express` por SQL directo → o selector de organização aparece na sidebar; ao trocar, o CRM fica vazio
6. Emitir uma factura em cada tenant → confirmar numeração independente
7. Fazer upload de um documento → confirmar no MinIO que a chave começa por `tenants/<uuid>/`

- [ ] **Step 7: Correr tudo uma última vez**

```bash
pnpm lint && pnpm test && pnpm test:isolation && pnpm exec tsc --noEmit && pnpm build
```
Expected: os cinco comandos passam

- [ ] **Step 8: Commit**

```bash
git add prisma/seeds docs/specs
git commit -m "feat(tenancy): seed two tenants with independent lookup data"
```

---

## Verificação final da Onda 1

| Critério | Como confirmar |
|---|---|
| Nenhuma tabela de negócio sem `tenantId` | A query do Task 4 Step 4 devolve exactamente a lista permitida |
| Nenhum utilizador sem membership | A query do Task 9 Step 2 devolve `0` |
| Todos os scopes fixam o tenant, incluindo para admin/manager | `pnpm test -- tenant-scopes` verde |
| Não há leitura cruzada entre tenants | `pnpm test:isolation` verde |
| SQL cru filtra por tenant | `pnpm test -- similarity fulltext search-documents` verde |
| Equipas alargam a visibilidade sem a alargar entre tenants | Passos 3 e 4 da verificação manual (Task 16 Step 6) |
| A build de produção passa | `pnpm build` |

## Fora do âmbito — Onda 2

Registado aqui para não ser confundido com trabalho em falta: fan-out por tenant dos 30+ jobs Inngest; chave Resend e domínio remetente por tenant; resolução por subdomínio em `proxy.ts`; branding por tenant (logo, cor, fuso a partir de `Tenant` em vez de fixo em `i18n/request.ts`); políticas RLS no Postgres; migração dos ~481 imports de `prismadb` para `tenantScopedClient` e subida da regra ESLint de `warn` para `error`; consola de superadmin de plataforma; `TenantInvite` ligado ao fluxo de registo.
