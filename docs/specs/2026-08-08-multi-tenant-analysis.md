# Análise: transformação do NextCRM em CRM Multi-Tenant

**Data:** 2026-08-08
**Estado do repo analisado:** branch `main`, v0.21.2, Next.js 16.2.6, Prisma 7.6 / PostgreSQL, better-auth 1.6.22
**Objectivo:** operar um único CRM que serve várias organizações independentes (Sanjila, China Express, e outros projectos futuros), com equipas distribuídas por organização.

---

## 1. Contexto

Hoje a aplicação é **single-tenant**: existe uma única "empresa" implícita, todos os utilizadores partilham o mesmo espaço de dados, e o isolamento que existe é apenas **por papel e por dono do registo** (`admin` / `manager` vêem tudo, `user` vê o que lhe está atribuído).

Confirmado por inspecção: `prisma/schema.prisma` (84 modelos, 2017 linhas) não contém uma única ocorrência de `tenant`, `organization`, `workspace`, `orgId` ou `companyId`. O próprio design doc interno já o admite — `docs/specs/2026-05-01-permission-driven-migration-design.md` §11.4 diz textualmente que o multi-tenant ficaria para depois.

A boa notícia é que o trabalho de segurança já feito (migração permission-driven, auditoria BOLA/IDOR) deixou o código numa forma **muito favorável** a esta transformação:

- Um único acessor de sessão: [lib/auth-server.ts](lib/auth-server.ts)
- Um único ponto onde a identidade é resolvida: `requireAuthenticated()` em [lib/authz/session.ts](lib/authz/session.ts), usado em ~152 sítios
- A lógica de scoping está centralizada em fragmentos `where` compostáveis em [lib/authz/scopes/crm.ts](lib/authz/scopes/crm.ts) (780 linhas)
- Sessão em base de dados (não JWT) → não há problema de re-emissão de tokens
- ~1 ficheiro de teste de scope por server action já existente
- O servidor MCP já reutiliza o mesmo objecto `AuthzUser`

O trabalho é grande, mas é **um trabalho de propagação por costuras que já existem**, não de reconstrução.

---

## 2. Decisões de arquitectura

### 2.1 Modelo de isolamento: coluna discriminadora + RLS como rede de segurança

| Opção | Isolamento | Custo | Veredicto |
|---|---|---|---|
| A. BD partilhada, schema partilhado, coluna `tenantId` | Lógico | Baixo | **Recomendado** |
| B. Schema por tenant (Postgres schemas) | Forte | Alto — Prisma `multiSchema` é estático, migrações ×N | Não |
| C. BD por tenant | Máximo | Muito alto — routing de ligações, ops ×N | Não (talvez para 1 cliente enterprise futuro) |

**Recomendação: A**, reforçada em duas camadas:

1. **Camada ORM (fase 2):** uma *Prisma Client Extension* que injecta `tenantId` em todos os `where` e `data`. Apanha o erro humano em tempo de desenvolvimento e cobre os ~481 ficheiros que chamam `prismadb.`.
2. **Camada BD (fase 6):** **Row Level Security** do Postgres como backstop definitivo. É esta camada que protege os **8 ficheiros de produção que usam `$queryRaw`** — pesquisa pgvector e full-text tsvector — onde a extensão Prisma não chega:
   - `actions/crm/similarity/get-similar-{accounts,contacts,leads,opportunities}.ts`
   - `actions/fulltext/unified-search.ts`
   - `actions/documents/search-documents.ts`
   - `inngest/functions/embed-*.ts`, `inngest/functions/documents/enrich-document.ts`

**Nota de implementação sobre RLS:** [lib/prisma.ts](lib/prisma.ts) usa `@prisma/adapter-pg` com um `pg.Pool`. Como cada query pode apanhar uma ligação diferente do pool, `SET app.tenant_id` ao nível de sessão é **inseguro**. A forma correcta é `SET LOCAL` dentro de uma transacção interactiva (que fixa a ligação), aplicado pela extensão:

```ts
// esboço — lib/db/tenant-client.ts
prismadb.$extends({
  query: {
    $allModels: {
      async $allOperations({ args, query }) {
        const tid = tenantStore.getStore()?.tenantId;
        if (!tid) throw new TenantContextMissingError();
        args.where = { ...args.where, tenantId: tid };   // camada 1
        return query(args);
      },
    },
  },
});
```

Com `AsyncLocalStorage` a transportar o tenant desde `requireAuthenticated()`. Fazer RLS na fase 6 e não na fase 1 — primeiro correcção por construção, depois a rede.

### 2.2 Tenant ≠ Team — são **dois** níveis, e o utilizador pertence a vários tenants

Este é o ponto onde é fácil errar. `Users.tenantId` como coluna única seria um erro: um consultor da Sanjila que também trabalhe na China Express ficaria preso a um dos dois.

A hierarquia correcta:

```
Tenant (Sanjila | China Express | …)
  └── Membership (Users ↔ Tenant, com papel POR tenant)
       └── Team (Comercial, Suporte, Financeiro…)
            └── TeamMembership
```

Consequência directa: **`Users.role` (enum `AppRole`) deixa de ser global.** O papel passa a viver em `Membership.role`. O que fica em `Users` é apenas um papel de **plataforma** (`platformRole: none | superadmin`) — quem gere o CRM inteiro, transversal aos tenants.

Novo modelo de papéis:

| Nível | Papel | Alcance |
|---|---|---|
| Plataforma | `superadmin` | Todos os tenants; cria/suspende tenants; impersonação auditada |
| Tenant | `owner` | Tudo dentro do tenant + facturação + apagar tenant |
| Tenant | `admin` | Tudo dentro do tenant + páginas `/admin` |
| Tenant | `manager` | Todos os dados de negócio do tenant |
| Tenant | `user` | Só os seus registos (+ equipa, se activado) |

Atenção: `requireRole()` usa hoje uma **allowlist plana**, não uma hierarquia — `requireRole(["manager"])` rejeitaria um admin. Hoje todas as chamadas passam `["admin"]`, por isso ainda não deu problema, mas com `owner` no meio isto quebra. Corrigir para hierarquia na fase 2.

### 2.3 Resolução do tenant: subdomínio, verificado no servidor

Três opções avaliadas:

| Opção | Refactor de rotas | Deep-link | Multi-tab |
|---|---|---|---|
| Segmento de path `/[locale]/t/[tenant]/…` | **Enorme** (~100 ficheiros movidos, luta com o matcher do next-intl) | Sim | Sim |
| Só sessão + selector de tenant | Zero | Não | **Quebra** |
| **Subdomínio** `sanjila.crm.ao` | ~10 linhas em `proxy.ts` | Sim | Sim |

**Recomendação: subdomínio.** É a única que não colide com o `next-intl`. O [proxy.ts](proxy.ts) actual termina com `return intlMiddleware(req)` — o subdomínio é lido *antes* e o `intlMiddleware` fica intocado.

Restrição importante: o proxy corre no **Edge** e não pode consultar o Prisma. Solução — o proxy só extrai a *string* do slug e passa-a num header; a validação real (o utilizador tem membership neste tenant?) acontece no servidor, dentro de `requireAuthenticated()`:

```ts
// proxy.ts — antes do intlMiddleware
const host = req.headers.get("host") ?? "";
const slug = host.split(".")[0];            // "sanjila"
const headers = new Headers(req.headers);
headers.set("x-tenant-slug", slug);         // NUNCA confiar nisto sem validar
return intlMiddleware(new NextRequest(req, { headers }));
```

Infra necessária: DNS wildcard `*.crm.ao` + certificado wildcard. Cookie da sessão em `.crm.ao` (partilhado, com switcher) — ou por subdomínio (isolamento mais forte, login por tenant). Recomendo cookie partilhado + switcher, porque o mesmo utilizador em vários tenants é o caso de uso central aqui.

**As rotas `/api/**` estão fora do matcher do proxy** — recebem o tenant da sessão via `requireAuthenticated()`, não do header. Ponto crítico a não esquecer.

---

## 3. Alterações ao modelo de dados

### 3.1 Modelos novos

```prisma
model Tenant {
  id        String   @id @default(uuid()) @db.Uuid
  slug      String   @unique          // "sanjila", "china-express"
  name      String
  status    TenantStatus @default(ACTIVE)   // ACTIVE | SUSPENDED | ARCHIVED
  logoUrl   String?
  primaryColor String?
  defaultLocale Language @default(pt)
  timezone  String   @default("Africa/Luanda")
  createdAt DateTime @default(now())
  // … relações inversas para todos os modelos de negócio
}

model Membership {
  id       String     @id @default(uuid()) @db.Uuid
  userId   String     @db.Uuid
  tenantId String     @db.Uuid
  role     TenantRole @default(user)
  status   ActiveStatus @default(PENDING)
  @@unique([userId, tenantId])
  @@index([tenantId, role])
}

model Team {
  id       String @id @default(uuid()) @db.Uuid
  tenantId String @db.Uuid
  name     String
  @@unique([tenantId, name])
}

model TeamMembership {
  teamId String @db.Uuid
  userId String @db.Uuid
  @@id([teamId, userId])
}

model TenantInvite {
  id        String   @id @default(uuid()) @db.Uuid
  tenantId  String   @db.Uuid
  email     String
  role      TenantRole
  token     String   @unique
  expiresAt DateTime
}
```

### 3.2 Classificação dos 84 modelos existentes

**Grupo A — precisam de `tenantId` directo (~50 modelos raiz).** Todos os `crm_*` de negócio, `Documents`, `Boards`/`Sections`/`Tasks`, `Invoices` e satélites, `crm_Products`/`crm_ProductCategories`, `crm_Activities`, `crm_AuditLog`, `crm_Report_Config`/`Schedule`, `Employees`.

**Grupo B — tabelas de junção e satélites (20 tabelas).** As 13 junções (`DocumentsTo*`, `AccountWatchers`, `ContactsToOpportunities`, `TargetsToTargetLists`, `EmailsTo*`…) e as 7 tabelas de embedding (`crm_Embeddings_*`, `crm_Document_Chunks`, `EmailEmbedding`).

Decisão: **adicionar `tenantId` também aqui**, apesar de tecnicamente derivável do FK. Razões: (a) RLS precisa da coluna em cada tabela protegida; (b) as tabelas de embedding são consultadas por SQL cru — sem `tenantId` na própria tabela o filtro tem de fazer JOIN, o que degrada a pesquisa vectorial.

**Grupo C — hoje user-scoped, passam a tenant+user.** `EmailAccount`, `Email`, `CalendarConnection`, `crm_CalendarEvents`, `ApiToken`, `ApiKeys`, `crm_Contact_Enrichment`, `crm_Target_Enrichment`.

**Grupo D — configuração: singletons que passam a linha-por-tenant.**

| Modelo | Hoje | Depois |
|---|---|---|
| `Invoice_Settings` | 1 linha global com a identidade fiscal da empresa | **1 linha por tenant** — obrigatório, é a identidade fiscal na factura |
| `Invoice_Series` | contador global | **por tenant** — ver §3.4 |
| `Invoice_TaxRates` | global | por tenant |
| `crm_FunnelSettings` | singleton | por tenant |
| `crm_SystemSettings` | PK = `key` | **PK passa a `@@id([tenantId, key])`** |
| Lookups (`crm_Lead_Sources`, `crm_Lead_Statuses`, `crm_Lead_Types`, `crm_Contact_Types`, `crm_Industry_Type`, `crm_Opportunities_Sales_Stages`, `crm_Opportunities_Type`, `Documents_Types`) | globais, editáveis no `/admin` | por tenant, com seed de valores por omissão na criação do tenant |

**Grupo E — ficam genuinamente globais.** `Currency` (PK `code`), `ExchangeRate`, `systemServices`, e as tabelas do better-auth (`Session`, `Account`, `Verification`).

### 3.3 Constraints únicos que colidem entre tenants

Esta lista é a fonte mais provável de erros de migração:

| Constraint | Acção |
|---|---|
| `Users.email @unique` | **Manter global** — a identidade é uma só, o multi-tenant vive no `Membership` |
| `crm_Products.sku @unique` | → `@@unique([tenantId, sku])` |
| `crm_Contact_Types.name`, `crm_Lead_Sources.name`, `crm_Lead_Statuses.name`, `crm_Lead_Types.name` | → `@@unique([tenantId, name])` |
| `crm_SystemSettings.key` (PK) | → `@@id([tenantId, key])` |
| `Invoices @@unique([seriesId, number])` | Seguro **se** a série for por tenant |
| `crm_campaign_sends.unsubscribe_token @unique` | Manter global (token aleatório) |
| `ApiToken.tokenHash @unique` | Manter global, mas **acrescentar `tenantId`** ao token |
| `crm_CalendarEvents @@unique([source, externalId])` | → incluir `tenantId` |

### 3.4 Numeração de facturas — risco legal, não apenas técnico

`Invoice_Series` tem um `counter Int` global. Com dois tenants a facturar do mesmo contador, a Sanjila e a China Express partilhariam a sequência de numeração — o que viola os requisitos da AGT para séries de facturação em Angola.

**`Invoice_Series` tem de ser por tenant, e o incremento do contador tem de ser feito sob `SELECT … FOR UPDATE` dentro da transacção de emissão.** Verificar se já é o caso; se for um `update` optimista, corrigir agora.

### 3.5 Migração de dados

O `prisma/migrations/` tem 55 pastas e o build corre `prisma migrate deploy`. A migração de dados faz-se em três passos:

1. `ALTER TABLE … ADD COLUMN tenant_id uuid NULL` (nullable, sem downtime)
2. Criar o tenant `sanjila` e fazer `UPDATE … SET tenant_id = '<uuid>'` em todas as tabelas — os dados actuais são todos da Sanjila
3. `ALTER … SET NOT NULL` + índices `@@index([tenantId, deletedAt])` compostos

Atenção: **`crm_ActivityLinks` e `crm_AuditLog` são polimórficos** (`entityType` String + `entityId` uuid, sem FK). Não há integridade referencial para apoiar o backfill — o `tenantId` tem de ser preenchido por join manual por `entityType`.

---

## 4. Buracos existentes que vazam entre tenants no dia 1

Estes já são problemas hoje (mostram dados a quem não devia ver); com multi-tenant passam a **fuga de dados entre empresas**. São bloqueantes.

| Ficheiro | Problema |
|---|---|
| [actions/crm/get-crm-data.ts](actions/crm/get-crm-data.ts) | `getAllCrmData()` **não tem qualquer chamada de auth**. Faz 16 `findMany` paralelos só com `{ deletedAt: null }` e devolve todas as contas/oportunidades/leads/contactos. Usado para popular dropdowns em páginas de detalhe. |
| [actions/get-users.ts](actions/get-users.ts) | `getUsers()`, `getActiveUsers()` — sem auth, sem filtro |
| `actions/dashboard/*.ts` | Contagens globais (`crm_Accounts.count({ where: { deletedAt: null } })`) mostradas a todos |
| [app/api/crm/leads/create-lead-from-web/route.ts](app/api/crm/leads/create-lead-from-web/route.ts) | Webhook público autenticado por um **único `NEXTCRM_TOKEN` partilhado**, comparado com `!==`. Cria lead sem dono e sem tenant. Tem de passar a token por tenant, guardado com hash. |
| [lib/mcp/auth.ts](lib/mcp/auth.ts) / [lib/api-tokens.ts](lib/api-tokens.ts) | Tokens `nxtc__` não são ligados a tenant |
| `Boards.sharedWith String[] @db.Uuid` | Array de user ids sem FK — precisa de validação explícita de tenant |
| `app/[locale]/(routes)/crm/accounts/[accountId]/page.tsx:55` | `(session?.user as any)?.role` — cast que contorna a camada authz |

---

## 5. Infra transversal (a parte que se esquece)

### 5.1 Armazenamento (MinIO / S3)

[app/api/upload/presigned-url/route.ts:41](app/api/upload/presigned-url/route.ts#L41) gera `const key = \`${folder}/${randomUUID()}.${ext}\`` — sem prefixo de tenant, num bucket único.

Passar a `tenants/{tenantId}/{folder}/{uuid}.{ext}`, e o endpoint de presigned URL tem de validar que o tenant do pedido corresponde ao prefixo. Nota: os documentos já existentes ficam com chaves antigas — ou se migram os objectos, ou se aceita o prefixo legado só para o tenant Sanjila.

### 5.2 Inngest — 30+ jobs em background

Esta é a maior fatia de trabalho depois dos scopes, e a mais fácil de esquecer, porque **jobs não têm sessão**.

Dois padrões a aplicar:

- **Jobs disparados por evento** (`embed-account`, `enrich-contact`, `send-step`, `process-event`): o `tenantId` entra no payload do evento e é usado explicitamente em todas as queries.
- **Jobs de cron / fan-out** (`emails/sync-all`, `calendar/google-sync-all`, `crm/care-tasks`, `crm/renewal-reminders`, `crm/recycle-targets`, `crm/kill-rule`, `crm/qualified-cadence`, `reports/send-scheduled`, `ecb/sync-exchange-rates`): passam a iterar **por tenant** — o cron dispara um evento de fan-out por tenant activo, cada um processado isoladamente. Excepção: `ecb/sync-exchange-rates` fica global (`ExchangeRate` é dado de referência).

`app/api/inngest` faz early-return no proxy — nunca passa por resolução de tenant. Correcto, mas significa que todo o contexto tem de vir do payload.

### 5.3 Email e campanhas (Resend)

- A chave Resend é hoje global (`actions/admin/system/set-resend-key.ts`) → passa a por tenant, ou global com **domínio remetente e from-address por tenant**. A Sanjila não pode enviar campanhas com o domínio da China Express.
- O webhook `app/api/campaigns/webhooks/resend` tem de resolver o tenant a partir do `resend_message_id` / `crm_campaign_sends`, não da sessão.
- Links de unsubscribe: o token já é único global, mas a página de unsubscribe tem de resolver o tenant a partir do token.

### 5.4 Chaves de LLM

`ApiKeys.scope` é hoje `SYSTEM | USER`. Adicionar `TENANT` — cada empresa pode querer a sua própria chave OpenAI, ou usar a da plataforma.

### 5.5 Branding por tenant

Como o objectivo é que Sanjila e China Express sintam a app como sua: logo, cor primária, nome, locale por omissão e fuso horário no modelo `Tenant`. O `i18n/request.ts` tem hoje `timeZone: "Africa/Luanda"` fixo — passa a vir do tenant. O [app/[locale]/(routes)/layout.tsx](app/[locale]/(routes)/layout.tsx) já é o ponto de estrangulamento por onde todas as páginas passam (linhas 50–66), portanto é aí que o tenant se resolve e se injecta no contexto React.

---

## 6. Rede de segurança: testes e guardas de CI

Sem isto, o multi-tenant degrada-se silenciosamente — basta um `findMany` novo sem `tenantId` para abrir uma fuga.

1. **Harness de isolamento.** Seed com dois tenants (A e B) e dados equivalentes. Um teste que, para cada server action exportada, corre como utilizador do tenant A e afirma que **zero** linhas do tenant B aparecem. Os ~16 ficheiros em `lib/authz/__tests__/` e os `__tests__/*-scope.test.ts` por action dão o molde.
2. **Guarda de lint.** Proibir o import directo de `prismadb` fora de `lib/db/` — tudo passa pelo cliente ligado ao tenant. Regra ESLint `no-restricted-imports` com allowlist explícita para os casos legítimos (seeds, scripts, inngest com tenant explícito).
3. **Guarda de SQL cru.** Uma regra que exija que todo o `$queryRaw` contenha `tenant_id` no texto, ou esteja numa allowlist justificada.
4. **`TenantContextMissingError`.** A extensão Prisma deve **atirar**, nunca fazer fallback silencioso para "sem filtro". Falhar alto é a única postura segura aqui.

---

## 7. Faseamento e esforço

O trabalho divide-se em duas ondas. O caro em tenancy não é o schema — é a fan-out do Inngest, o email por tenant, a UX de switcher, o branding. Mas nada disso é preciso enquanto houver **um** tenant. O que **não** pode esperar é a fronteira, porque retrofitar uma fronteira é que é caro.

**Onda 1 — a fronteira dura + equipas (4–7 semanas).** Plano de implementação task-a-task em [docs/superpowers/plans/2026-08-08-multi-tenant-wave-1.md](../superpowers/plans/2026-08-08-multi-tenant-wave-1.md), 16 tarefas.

| Conteúdo | Esforço |
|---|---|
| `Tenant`/`Membership`/`Team`/`TeamMembership`/`TenantInvite`; `tenantId` em ~70 tabelas; constraints únicos compostos; backfill para o tenant `sanjila` | 1–2 sem |
| `AuthzUser` = `{id, role, tenantId, teamMemberIds, isPlatformAdmin}`; `requireAuthenticated()` valida membership; `requireRole` hierárquico; contexto `AsyncLocalStorage`; extensão Prisma | 1–2 sem |
| Predicado de tenant em todos os `*ScopeWhere` (incluindo os early-returns de `admin`/`manager`); fechar os buracos da §4; filtro de tenant no SQL cru | 2–3 sem |
| Barato agora, caro depois: prefixo `tenants/{id}/` nas chaves S3; `Invoice_Series` por tenant com bloqueio pessimista | incluído |
| Gestão de equipas, switcher de organização, harness de isolamento, guarda de CI | incluído |

**Onda 2 — as funcionalidades por tenant (4–6 semanas), quando a China Express entrar de facto.**

| Conteúdo | Esforço |
|---|---|
| Fan-out por tenant dos 30+ jobs Inngest | 1–2 sem |
| Resend/domínio remetente por tenant; chaves LLM por tenant | 1 sem |
| Resolução por subdomínio em `proxy.ts`; branding (logo, cor, fuso a partir de `Tenant`) | 1 sem |
| Políticas RLS no Postgres; migração dos ~481 imports de `prismadb` para `tenantScopedClient`; regra ESLint de `warn` para `error` | 1–2 sem |
| Consola de superadmin de plataforma; `TenantInvite` no fluxo de registo | 1 sem |

**Ordem de risco:** a Onda 1 não pode ser feita pela metade — um `tenantId` em falta é uma fuga entre empresas. A Onda 2 é incremental por natureza.

---

## 8. Recomendações finais

1. **Não usar `Users.tenantId`.** Membership table desde o início. Refazer isto mais tarde custa dez vezes mais.
2. **Fechar os buracos da §4 antes de introduzir o segundo tenant.** `get-crm-data.ts` sozinho expõe todo o CRM.
3. **Aplicar RLS.** Com 8 ficheiros de SQL cru em produção e uma pesquisa vectorial a crescer, a camada ORM não é suficiente sozinha. É a diferença entre "acreditamos que está isolado" e "a base de dados garante".
4. **Séries de facturação por tenant, com bloqueio pessimista.** É requisito legal, não preferência.
5. **Criar o tenant Sanjila como parte da migração**, não como um passo manual — os dados existentes têm de ficar num tenant nomeado desde o primeiro deploy.
6. **Só depois disto** convidar a China Express.

---

## 9. Verificação

Como validar a transformação ponta a ponta:

```bash
# 1. Reset local + migração + seed de dois tenants
npm run db:up && npm run db:wait && npm run db:migrate && npm run db:seed

# 2. Suite de isolamento (a criar na fase 6)
npm test -- tenant-isolation

# 3. Scopes existentes continuam verdes
npm test -- authz scope
```

Manualmente:
- Login como utilizador só da Sanjila → `sanjila.localhost:3000` funciona, `china-express.localhost:3000` devolve 403/404
- Utilizador com membership em ambos → switcher aparece, dados mudam completamente ao trocar
- Emitir uma factura em cada tenant → numeração independente, identidade fiscal correcta em cada PDF
- Correr `emails/sync-all` no Inngest dev server → confirmar um fan-out por tenant, sem cruzamento
- Query directa à BD como role restrito → confirmar que RLS bloqueia leitura cross-tenant mesmo com SQL cru
