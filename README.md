# G7 OS — Sistema Operacional Interno

Sistema interno da G7 Soluções Digitais para centralizar a operação completa da agência: clientes, projetos, demandas, planejamento, CRM, comunicação, financeiro e IA.

**Status:** Fundação profissional pronta. Auth, RBAC, banco e telas iniciais funcionais.

---

## Stack

- **Next.js 15** (App Router, Server Components, Server Actions)
- **TypeScript** strict
- **Tailwind CSS** + **shadcn/ui** (dark/light)
- **Supabase** (Postgres + Auth + Storage)
- **Vercel** (deploy)
- Preparado para **n8n** (automações futuras)

---

## Estrutura do projeto

```
g7-os/
├── src/
│   ├── app/
│   │   ├── (auth)/                    # Rotas públicas (sem sidebar)
│   │   │   └── login/                 # Login + server action signIn/signOut
│   │   ├── (app)/                     # Rotas autenticadas (com sidebar/topbar)
│   │   │   ├── layout.tsx             # Carrega AuthContext, sidebar, topbar
│   │   │   ├── dashboard/             # KPIs e atividade recente
│   │   │   ├── clients/               # Lista + detalhe (tabs)
│   │   │   └── projects/              # Lista de projetos
│   │   ├── globals.css                # Tokens shadcn + paleta sidebar
│   │   ├── layout.tsx                 # Layout raiz (ThemeProvider)
│   │   └── page.tsx                   # Redirect para /dashboard
│   ├── components/
│   │   ├── ui/                        # shadcn primitives (button, card, etc.)
│   │   ├── sidebar.tsx                # Navegação principal (RBAC-aware)
│   │   ├── topbar.tsx                 # Workspace switcher + user menu
│   │   ├── theme-toggle.tsx
│   │   ├── theme-provider.tsx
│   │   └── page-header.tsx
│   ├── config/
│   │   └── navigation.ts              # Fonte única dos itens de menu
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts              # Cliente browser
│   │   │   ├── server.ts              # Cliente server + service role
│   │   │   └── middleware.ts          # Renovação de sessão + proteção
│   │   ├── actions/
│   │   │   └── workspace.ts           # Trocar workspace ativo
│   │   ├── auth.ts                    # getAuthContext + assertPermission
│   │   ├── rbac.ts                    # Permissões por role (UI)
│   │   ├── labels.ts                  # Mapeamento status -> label/cor
│   │   └── utils.ts                   # cn, formatBRL, formatDate, initials
│   ├── types/
│   │   └── database.ts                # Tipos do banco (sync com SQL)
│   └── middleware.ts                  # Hook do middleware do Supabase
├── supabase/
│   ├── migrations/
│   │   ├── 001_init_core.sql          # Orgs, profiles, members, audit, helpers
│   │   ├── 002_domain_tables.sql      # Clients, projects, tasks, content, CRM
│   │   ├── 003_finance.sql            # Tabelas financeiras + audit trigger
│   │   └── 004_rls_policies.sql       # 46 policies de Row Level Security
│   └── seeds/
│       └── seed.sql                   # Workspaces G7 + categorias financeiras
├── package.json
├── tailwind.config.ts
├── tsconfig.json
├── next.config.ts
├── postcss.config.js
├── .env.example
└── .gitignore
```

---

## Modelo de dados

**19 tabelas** organizadas em domínios:

| Domínio | Tabelas |
|---------|---------|
| Identidade | `organizations`, `profiles`, `organization_members` |
| Auditoria | `audit_logs` |
| Clientes | `clients`, `client_links`, `client_briefings` |
| Projetos | `projects`, `project_members` |
| Tarefas | `tasks` |
| Conteúdo | `content_items` |
| CRM | `crm_leads`, `crm_lead_activities` |
| Comunicação | `comments`, `attachments`, `notifications` |
| Financeiro | `finance_accounts`, `finance_categories`, `finance_records` |

**Convenções:**
- Toda tabela tem `id uuid PK`, `created_at`, `updated_at`
- Toda tabela de domínio tem `organization_id` (single-tenant preparado para multi-tenant)
- Trigger `touch_updated_at` em todas as tabelas
- `audit_logs` recebe registro automático de toda operação em tabelas financeiras

---

## Sistema de permissões (RBAC)

**8 roles:** `adm`, `gestao`, `trafego`, `social_media`, `designer`, `videomaker`, `comercial`, `financeiro`.

**Camadas:**

1. **Banco (RLS)** — segurança real. Funções `is_member_of()`, `has_role_in()`, `is_manager_in()`, `can_see_finance()` usadas em 46 policies.
2. **Server (`@/lib/auth`)** — `getAuthContext()` resolve user + profile + workspace ativo + role + memberships. `assertPermission()` para Server Actions.
3. **UI (`@/lib/rbac`)** — função `can()` para esconder menus/botões. **Não é segurança**, só UX.

**Regras-chave:**
- ADM vê tudo
- Gestão vê tudo, exceto financeiro
- Time operacional vê apenas projetos onde é membro + suas tarefas
- Financeiro: somente ADM e role `financeiro`. Toda operação é auditada.

---

## Setup local

### 1. Criar projeto no Supabase

1. Acesse [supabase.com](https://supabase.com), crie um projeto novo (região: South America).
2. Em **Project Settings → API**, copie a `URL`, `anon key` e `service_role key`.

### 2. Aplicar migrations

No SQL Editor do Supabase (ou via `supabase db push`), rode na ordem:

```bash
1. supabase/migrations/001_init_core.sql
2. supabase/migrations/002_domain_tables.sql
3. supabase/migrations/003_finance.sql
4. supabase/migrations/004_rls_policies.sql
5. supabase/seeds/seed.sql
```

### 3. Criar primeiro usuário (ADM)

Em **Authentication → Users → Add user**, crie sua conta com senha. Pegue o `id` (UUID) gerado.

No SQL Editor, rode (substituindo o UUID):

```sql
insert into public.organization_members (organization_id, user_id, role, is_default)
values
  ('b9803815-ec5b-507f-91e7-43137b625e86', 'SEU_USER_UUID', 'adm', true),
  ('6eeaff63-a7a2-51fc-805e-169fe438787b', 'SEU_USER_UUID', 'adm', false);
```

### 4. Configurar variáveis de ambiente

```bash
cp .env.example .env.local
# preencha NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY
```

### 5. Instalar e rodar

```bash
npm install
npm run dev
```

Acesse `http://localhost:3000`, faça login com a conta criada.

---

## Deploy no Vercel

1. Suba o repositório no GitHub.
2. Em [vercel.com/new](https://vercel.com/new), importe o repo.
3. Configure as variáveis de ambiente (mesmas do `.env.local`).
4. Deploy automático a cada push na branch `main`.

A `SUPABASE_SERVICE_ROLE_KEY` deve ser marcada como **Sensitive** no Vercel.

---

## Roadmap — próximas fases

### Fase 2 — CRUD completo dos módulos atuais
- [ ] `/clients/new` e `/clients/[id]/edit` — formulários com server actions
- [ ] `/projects/new` e `/projects/[id]` — detalhe com tarefas vinculadas
- [ ] `/tasks` — Kanban drag-and-drop (dnd-kit)
- [ ] Sistema de upload de arquivos (Supabase Storage)
- [ ] Sistema de comentários com menções (`@user`)
- [ ] Notificações em tempo real (Supabase Realtime)

### Fase 3 — Conteúdo, CRM, Calendário
- [ ] `/content` — calendário editorial com drag-and-drop
- [ ] `/crm` — pipeline visual com kanban de leads
- [ ] `/calendar` — visão consolidada (tarefas + conteúdo + reuniões)

### Fase 4 — Automações n8n
- Hooks já preparados em `tasks.metadata.n8n_workflow_id`
- Webhook receiver em `/api/n8n/webhook` (HMAC validado)
- Workflows: novo lead → CRM, nova tarefa → notificação, contrato fechado → criar cliente

### Fase 5 — IA Assistente
- Endpoint `/api/ai/chat` com Claude API
- Skills: gerar relatório de tráfego, escrever roteiro, analisar briefing
- Sumarização automática de briefings antigos

### Fase 6 — Financeiro completo
- Dashboard de fluxo de caixa
- DRE simplificado por workspace
- Conciliação bancária (importação OFX)
- Dashboard de auditoria (somente ADM)

### Fase 7 — Multi-tenant (se for vender pra outras agências)
- Painel de admin global (super_admin role)
- Onboarding de novos workspaces
- Billing (Stripe)

---

## Convenções de desenvolvimento

- **Server Components por padrão.** Use `"use client"` só quando precisar de interatividade.
- **Server Actions** para mutations (não criar route handlers à toa).
- **Toda query de domínio passa por `organization_id` + RLS.** Nunca confie só na UI.
- **Permissões:** RLS é a verdade, RBAC client é só UX.
- **Tipos:** ao adicionar tabelas no SQL, atualizar `src/types/database.ts` (ou rodar `supabase gen types`).
- **Comentários:** Padrão G7 — formal, consultivo, sem emojis em código de produção.

---

## Validação técnica realizada

Esta fundação foi validada antes de entregar:

- ✅ As 4 migrations SQL aplicam sem erro num Postgres 16 real
- ✅ 19 tabelas, 46 RLS policies, todas as funções de autorização criadas
- ✅ TypeScript compila sem erros (`tsc --noEmit`)
- ✅ Build de produção do Next.js passa (`next build`)
- ✅ Middleware ativo, todas as rotas roteando corretamente

---

**Construído com cuidado por Guilherme Alves Candido (Sócio Administrador G7)** sobre a fundação técnica preparada nesta sessão.
