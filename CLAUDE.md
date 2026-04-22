# CLAUDE.md — Em Alta (Promo Platform)

---

## ⚠️ Regras Críticas — Leia Antes de Qualquer Coisa

### 🌐 Idioma
**Toda comunicação com o usuário deve ser em português brasileiro (pt-BR).** Código, nomes de variáveis e arquivos permanecem em inglês. Apenas a conversa com o usuário é em português.

---

### 📋 Planning Antes de Executar — OBRIGATÓRIO

**Antes de executar qualquer task, feature, fix ou refactor — não importa o tamanho — você DEVE:**

1. Criar um arquivo `PLANNING.md` na raiz do projeto
2. Escrever um plano detalhado em português contendo:
   - O que será feito (passo a passo)
   - Quais arquivos serão criados, modificados ou deletados
   - Quais domínios estão envolvidos
   - Quais dependências ou efeitos colaterais existem
   - Riscos ou decisões que precisam de input do usuário
3. Apresentar o plano ao usuário e **aguardar aprovação explícita**
4. Só após o usuário confirmar (ex: "pode executar", "aprovado", "ok", "sim") — executar
5. Copiar o `PLANNING.md` aprovado para dentro do relatório da task em `.claude/tasks/`
6. Mover a task de `.claude/tasks/pendentes/` para `.claude/tasks/concluidas/` com relatório completo
7. Deletar o `PLANNING.md` da raiz do projeto

**Nunca pule esta etapa.** Mesmo para mudanças de uma linha, um plano mínimo é obrigatório.

**Estrutura obrigatória do PLANNING.md:**
```markdown
# Planejamento — [Nome da Task]

## O que será feito
Descrição clara e objetiva da tarefa.

## Arquivos envolvidos
- `apps/backend/internal/promotions/promotion_service.go` — modificar método X
- `apps/backend/internal/promotions/promotion_dto.go` — adicionar campo Y
- `infra/migrations/005_add_field_y.sql` — nova migration

## Domínios impactados
- promotions (direto)
- notifications (via evento PromotionUpdated)

## Passos de execução
1. Criar migration SQL
2. Atualizar model promotion.go
3. Atualizar DTO
4. Atualizar service
5. Atualizar controller
6. Verificar se algum event handler precisa de ajuste

## Riscos e decisões
- O campo Y deve ser obrigatório ou opcional? → aguardando definição do usuário

## O que NÃO será feito
- Nenhuma alteração em outros domínios
- Nenhuma refatoração não solicitada
```

---

### 🚀 Início de Sessão — Leitura Obrigatória

**Toda vez que uma nova sessão começar, antes de qualquer outra coisa, o Claude DEVE:**

1. Ler `.claude/PROJECT_DOCS.md` — documentação completa de regras de negócio e arquitetura
2. Ler `.claude/PROJECT_STATUS.md` — estado atual da implementação
3. Ler os arquivos em `.claude/tasks/pendentes/` — tasks aguardando execução
4. Somente após essa leitura, responder ao usuário ou iniciar qualquer ação

**Nunca responda ou execute nada antes de ler esses três arquivos.** Eles são a fonte da verdade do projeto.

Se algum dos arquivos não existir ainda, informe o usuário e pergunte se deseja criá-lo antes de continuar.

---

### 📄 PROJECT_DOCS.md — Fonte da Verdade

**Localização:** `.claude/PROJECT_DOCS.md`

Este arquivo é a documentação viva do projeto. Contém todas as regras de negócio, decisões arquiteturais, modelo de dados, fluxos de domínio e contratos de API.

**Quando atualizar obrigatoriamente:**
- Uma regra de negócio foi criada, alterada ou removida
- Uma decisão arquitetural foi tomada (novo domínio, novo padrão, mudança de estrutura)
- Um endpoint foi adicionado, modificado ou removido
- Uma tabela ou campo do banco de dados foi criado, alterado ou removido
- Um evento de domínio foi criado ou modificado
- Uma integração externa foi adicionada ou alterada
- Um fluxo de usuário foi alterado (ex: cadastro, notificações, votação)

**Como atualizar:**
- Atualize apenas a seção afetada — nunca reescreva o documento inteiro
- Registre a mudança no relatório da task correspondente em `.claude/tasks/concluidas/`
- Mantenha o mesmo padrão de formatação existente no documento

---

### 📊 PROJECT_STATUS.md — Estado Dinâmico da Aplicação

**Localização:** `.claude/PROJECT_STATUS.md`

Este arquivo é um **documento vivo** que descreve o estado real e atual da aplicação — o que realmente existe no repositório, quais rotas estão funcionando, quais regras de negócio estão implementadas, quais tabelas existem no banco. Não é uma lista de tarefas nem um planejamento futuro.

**Princípio:** o PROJECT_STATUS deve refletir o repositório como ele está agora. Se alguém ler esse arquivo, deve entender exatamente o que a aplicação faz hoje.

**Formato obrigatório do PROJECT_STATUS.md:**

```markdown
# PROJECT_STATUS.md — Em Alta

**Última atualização:** YYYY-MM-DD HH:MM
**Atualizado pela task:** TASK-[NNN]
**Versão do projeto:** X.Y.Z

---

## 📦 Repositório

Descrição da estrutura real de pastas e arquivos que existem hoje.

---

## 🧠 Resumo da Aplicação (o que a aplicação faz hoje)

Texto corrido descrevendo as funcionalidades reais e disponíveis.
Exemplo: "Usuários podem se cadastrar, confirmar o e-mail e completar o perfil.
O feed exibe promoções ordenadas por trend_score. Votos 🚀/🗑️ estão funcionando."

---

## 🔗 Rotas Implementadas

### Backend — API REST
| Método | Rota | Domínio | Descrição |
|---|---|---|---|
| POST | `/api/v1/auth/register` | users | Cadastro com e-mail, CPF e senha |

### Frontend — Angular
| Rota | Feature | Descrição |
|---|---|---|
| `/` | home | Feed principal de promoções |

### Bot Telegram
| Comando | Descrição |
|---|---|
| `/start` | Boas-vindas e lista de comandos |

---

## 🗃️ Banco de Dados

### Tabelas existentes
| Tabela | Migration | Descrição resumida |
|---|---|---|
| `users` | 001 | Usuários com CPF criptografado e status de conta |

### Migrations aplicadas
- `001_users.sql` — criada em YYYY-MM-DD

---

## 📡 Integrações Ativas

| Serviço | Finalidade | Status |
|---|---|---|
| Resend | Envio de e-mails transacionais | ✅ Configurado |

---

## 🔔 Notificações

Descreve quais canais estão ativos e quais eventos já disparam notificação.

---

## ⚙️ Workers e Jobs

| Worker/Job | Frequência | Fontes | Status |
|---|---|---|---|
| `trend_score_job` | 5min | Redis | ✅ Ativo |

---

## 📝 Observações da Sessão Atual

Notas relevantes sobre decisões tomadas, problemas encontrados ou contexto importante
que não se encaixa nas seções acima.
```

**Quando atualizar obrigatoriamente:**
- Ao concluir qualquer task que adicione algo ao repositório
- Quando uma nova rota for implementada — adicionar na tabela de rotas
- Quando uma migration for criada — adicionar na tabela de banco
- Quando uma integração for configurada
- Quando uma regra de negócio for implementada de fato (não apenas documentada)

**Como atualizar:**
- Escreva em texto corrido na seção "Resumo da Aplicação" — não use listas de checkbox
- Adicione rotas e tabelas nas tabelas correspondentes
- Atualize sempre `Última atualização` e `Atualizado pela task`
- Descreva o que existe, não o que vai existir
- **Ao iniciar uma sessão:** leia o arquivo inteiro e use-o como contexto do estado atual antes de qualquer ação

---

### 📁 Sistema de Tasks — `.claude/tasks/`

#### Estrutura de pastas
```
.claude/
└── tasks/
    ├── pendentes/       # Tasks aguardando execução
    │   └── TASK-001_nome-da-task.md
    └── concluidas/      # Tasks executadas e finalizadas
        └── TASK-001_nome-da-task.md
```

#### Ciclo de vida de uma task

**1. Task criada pelo usuário (status: pendente)**

O usuário pode criar uma task manualmente ou pedir ao Claude para criar. O arquivo fica em `.claude/tasks/pendentes/` com o seguinte formato:

```markdown
# TASK-[NNN] — [Nome da Task]

**Status:** 🟡 Pendente
**Criada em:** YYYY-MM-DD
**Prioridade:** alta | média | baixa

## Descrição
O que precisa ser feito.

## Critérios de conclusão
- [ ] Critério 1
- [ ] Critério 2
```

**2. Claude executa a task (após PLANNING.md aprovado)**

Ao concluir uma task, o Claude DEVE:
1. Mover o arquivo de `pendentes/` para `concluidas/`
2. Atualizar o status e preencher o relatório de execução completo no arquivo

**3. Formato obrigatório após conclusão:**

```markdown
# TASK-[NNN] — [Nome da Task]

**Status:** ✅ Concluída
**Criada em:** YYYY-MM-DD
**Concluída em:** YYYY-MM-DD HH:MM
**Prioridade:** alta | média | baixa

## Descrição
O que precisava ser feito.

## Critérios de conclusão
- [x] Critério 1
- [x] Critério 2

---

## Relatório de Execução

### O que foi feito
Descrição detalhada e objetiva de tudo que foi executado.

### Arquivos criados
- `caminho/do/arquivo.go` — descrição do que foi criado

### Arquivos modificados
- `caminho/do/arquivo.go`
  - Adicionado método `NomeDoMetodo()` — descrição
  - Alterado campo `nomeCampo` de `string` para `*string`

### Arquivos deletados
- `caminho/do/arquivo.go` — motivo da deleção

### Migrations executadas
- `infra/migrations/005_nome.sql` — descrição do que a migration faz

### Eventos de domínio publicados/criados
- `NomeDoEvento` no domínio `x` — quando é disparado

### Decisões tomadas durante a execução
- Decisão X foi tomada porque Y — aprovada pelo usuário em [data]

### O que NÃO foi feito (e por quê)
- Item que estava no escopo mas foi deixado para outra task — motivo

### Observações
Qualquer informação relevante para o futuro.
```

#### Regras do sistema de tasks

- **Nunca execute uma task de `pendentes/` sem antes criar e ter o `PLANNING.md` aprovado**
- **Sempre mova o arquivo** de `pendentes/` para `concluidas/` — nunca apenas edite no lugar
- **O relatório de execução é obrigatório** — uma task concluída sem relatório é considerada incompleta
- **Nunca delete tasks** — mesmo tasks canceladas devem ser movidas para `concluidas/` com status `❌ Cancelada` e motivo explicado
- **Numeração sequencial** — sempre use o próximo número disponível (TASK-001, TASK-002, etc.)
- **Se uma task gerou outra task** — registre na seção "Observações" e crie o novo arquivo em `pendentes/`
- **O `PLANNING.md` aprovado deve ser copiado dentro do relatório** na seção correspondente antes de ser deletado da raiz

---

## Project Overview

**Em Alta** is a real-time promotion aggregator platform (build-to-learn project) similar to Pelando.com. It aggregates deals and coupons from Brazilian e-commerces via affiliate APIs and scraping, with a full notification system (browser push, email, WhatsApp, Telegram), an AI-generated newsletter (Claude API), a Telegram query bot, and a data engineering pipeline downstream.

Full business rules, architecture, data model, API contracts, and domain documentation are in:
→ **[PROJECT_DOCS.md](./PROJECT_DOCS.md)**

Read PROJECT_DOCS.md before making any architectural decisions or creating new domains.

---

## Monorepo Structure

```
promo-platform/
├── apps/
│   ├── backend/      # Go API — Domain-oriented DDD structure
│   ├── frontend/     # Angular SPA — Standalone Components + Signals
│   ├── workers/      # Go — Scraping and affiliate API collectors
│   └── scheduler/    # Go — Cron jobs (trend score, newsletter, expiration)
├── infra/
│   ├── docker/       # Dockerfiles per app
│   ├── nginx/        # API Gateway config
│   └── migrations/   # Shared SQL migrations
├── docker-compose.yml
└── Makefile
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Angular (latest) — Standalone Components, Signals, Lazy Loading |
| Backend | Go (latest) — Gin, domain-oriented structure |
| Database | PostgreSQL 16 |
| Cache | Redis 7 |
| Message broker | RabbitMQ 3 |
| Email | Resend / SendGrid / Amazon SES |
| Push notifications | Web Push API (VAPID) |
| WhatsApp | Meta Cloud API (approved templates) |
| Telegram | Telegram Bot API |
| AI Newsletter | Anthropic Claude API (claude-sonnet-4-20250514) |
| Containerization | Docker + Docker Compose |

---

## Backend Domain Structure

Each domain folder inside `apps/backend/internal/` is **self-contained** and follows this exact convention:

```
{domain}/
├── {domain}.go                  # Model / Entity + internal Value Objects
├── {domain}_dto.go              # All DTOs (request + response)
├── {domain}_repository.go       # Repository interface (contract)
├── {domain}_repository_pg.go    # PostgreSQL implementation
├── {domain}_service.go          # Business rules and orchestration
├── {domain}_controller.go       # HTTP handlers
└── events/                      # Domain events published to RabbitMQ
```

**Existing domains:**
`users` · `promotions` · `categories` · `coupons` · `votes` · `comments` · `favorites` · `metrics` · `trending` · `notifications` · `newsletter` · `moderation` · `collection` · `telegrambot`

**Dependency rule:**
- `controller` → `service` → `repository interface`
- `repository_pg` implements the interface — only file that knows SQL
- Domains **never import each other directly** — communication via domain events on RabbitMQ
- `shared/` is the only package importable by all domains

---

## Coding Standards

### 🚫 Escopo Cirúrgico — Nunca Saia do que Foi Pedido

- **Nunca altere arquivos que não foram mencionados na task.** Se a tarefa é adicionar um campo no `promotion_service.go`, apenas esse arquivo (e seus dependentes diretos obrigatórios) são tocados. Nenhum outro.
- **Nunca refatore código existente** que não seja estritamente necessário para a task solicitada. Viu algo que poderia melhorar? Registre no `PLANNING.md` como sugestão futura — não execute sem aprovação.
- **Nunca renomeie variáveis, métodos ou arquivos existentes** sem solicitação explícita do usuário.
- **Nunca reorganize imports, formate arquivos existentes ou ajuste estilos** de código em arquivos que não são o alvo direto da task.
- **Nunca mova arquivos** de lugar sem solicitação explícita.
- **Nunca adicione dependências, libraries ou packages** sem perguntar antes ao usuário.
- **Nunca crie arquivos extras** além dos listados no `PLANNING.md` aprovado.

> **Regra de ouro:** se a mudança não estava no `PLANNING.md` aprovado, ela não acontece. Ponto.

---

### ❓ Sempre Pergunte — Nunca Suponha

**Sempre que houver dúvida sobre qualquer coisa, PARE e pergunte ao usuário. Nunca faça suposições, inferências ou "achismos".**

Situações que exigem pergunta obrigatória antes de qualquer ação:

- O comportamento esperado não está descrito no `PROJECT_DOCS.md`, `CLAUDE.md` ou `README.md`
- Há mais de uma forma de implementar algo e a documentação não define qual
- Um campo, endpoint ou regra está ausente ou ambíguo na documentação
- A task pode impactar um domínio além dos listados no planejamento
- Uma decisão de schema (nullable, tipo, constraint) não está documentada
- O usuário pediu algo que conflita com uma regra existente no `CLAUDE.md`
- Qualquer integração externa (API, serviço, biblioteca) não está documentada no projeto
- O comportamento em casos de erro não foi definido para a feature em questão

**Formato obrigatório de pergunta:**
```
❓ Tenho uma dúvida antes de continuar:

[Descreva claramente o que está indefinido]

Opções que vejo:
- Opção A: [descrição e consequência]
- Opção B: [descrição e consequência]

Qual você prefere?
```

**O que nunca fazer:**
- Escolher a opção que "parece mais lógica" sem perguntar
- Assumir que o padrão atual de outro domínio se aplica ao novo contexto
- Inferir regras de negócio a partir do código existente sem confirmação
- Implementar e depois perguntar se estava certo

---

### ❌ Never — Zero Tolerance

These are non-negotiable. No exceptions, no excuses:

- **No comments in code** — if the code needs a comment to be understood, rename the variable, method, or type until it is self-explanatory
- **No N+1 queries** — always use JOINs, batch loads (`WHERE id IN (...)`), or Redis. Every loop that touches the database is a red flag
- **No duplicated logic** — extract to a shared helper, service method, or utility before duplicating a single line
- **No unrelated changes in a commit** — one concern per commit, one concern per PR
- **No business logic in controllers** — controllers only receive the request, call the service, and return the response
- **No business logic in repositories** — repositories only query and persist, never decide
- **No multiple loose parameters when a DTO can be used** — always group related parameters into a struct/object
- **No direct imports between domain packages** — domains communicate only via RabbitMQ events

---

### ✅ SOLID Principles — Applied to Every File

| Principle | How to apply in this project |
|---|---|
| **S** — Single Responsibility | Each file does one thing: model, DTO, repository, service, or controller — never mix |
| **O** — Open/Closed | Add behavior via new methods or event handlers, never by modifying existing stable logic |
| **L** — Liskov Substitution | Repository interfaces must be fully replaceable by any implementation (pg, mock, etc.) |
| **I** — Interface Segregation | Keep repository interfaces focused — do not add methods that only one caller uses |
| **D** — Dependency Inversion | Services depend on repository interfaces, never on concrete implementations |

---

### ✅ Clean Code — Applied to Every File

- **Names must reveal intent** — `getUsersByCategory()` not `getUsers2()` or `fetch()`
- **Functions do one thing** — if you need to write "and" to describe what a function does, split it
- **Functions are small** — if a function exceeds ~30 lines, it is doing too much
- **No magic numbers or strings** — extract to named constants or enums
- **Fail fast** — validate inputs at the top of the function and return early; avoid deep nesting
- **Symmetry** — if you have `create`, you have `delete`; if you have `enable`, you have `disable`

---

### ✅ HTML Rules (Angular Templates)

- Use **semantic HTML5 tags** — `<article>`, `<section>`, `<nav>`, `<header>`, `<footer>`, `<main>`, `<aside>`, `<figure>`, `<time>` — never `<div>` for everything
- All `<img>` must have a meaningful `alt` attribute — never `alt=""` unless the image is purely decorative
- All interactive elements must be keyboard accessible — use `<button>` for actions, `<a>` for navigation, never `<div (click)>`
- Use `[attr.aria-*]` bindings for dynamic accessibility attributes
- Never use inline styles — always use SCSS classes
- Forms must use `<label>` with `for` attribute linked to input `id`
- Use `<ng-container>` to avoid unnecessary DOM wrapper elements

---

### ✅ SCSS Rules (Angular Styles)

- Use **CSS custom properties** (`--var-name`) for all design tokens (colors, spacing, typography)
- Follow **BEM naming** for component classes: `.promotion-card__title--featured`
- Never use `!important` — if you need it, the specificity structure is wrong
- Avoid deep nesting — maximum 3 levels of nesting in SCSS
- Use `@use` and `@forward` instead of `@import` (deprecated)
- Mobile-first approach — base styles for mobile, `@media (min-width: ...)` for larger screens
- Use logical properties where applicable: `margin-inline`, `padding-block`, `inset`
- No magic numbers for spacing — always use spacing tokens (multiples of 4px or 8px)

---

### Go Conventions

- Errors bubble up — no wrapping every function in error handling that hides the stack trace
- Use interfaces for all external dependencies (repositories, email senders, event bus, etc.)
- Value Objects are immutable structs with validation in the constructor — return an error if invalid
- Domain events implement the `shared.DomainEvent` interface
- Repository interface: `{Domain}Repository` — implementation: `pg{Domain}Repository`
- Never use `interface{}` or `any` unless absolutely unavoidable — always use typed structs

---

### Angular Conventions

- Standalone Components only — no NgModules
- Signals for all reactive state — no BehaviorSubject, no Subject for state
- Lazy loading per feature module — never eagerly load feature routes
- HTTP calls only inside services — never inside components directly
- All forms use Reactive Forms — no template-driven forms
- Use `OnPush` change detection strategy on all components

---

## Domain Events (RabbitMQ)

Domains communicate exclusively via events. Never call another domain's repository directly.

Key event flows (see PROJECT_DOCS.md for the full map):
```
catalog → PromotionPublished   → trending, notification, moderation
catalog → FlashEnded           → notification (publisher summary + favorites)
engagement → CommentPosted     → notification (publisher alert)
engagement → FavoriteAdded     → notification (creates favorite preferences)
trending → TrendScoreUpdated   → catalog (batch updates is_trending flag)
```

---

## Voting System

Votes are **🚀 Rocket** (positive) and **🗑️ Trash** (negative) — never "hot/cold".

- DB fields: `rocket_votes`, `trash_votes`
- Enum values: `rocket`, `trash`
- Score: `rocket_votes - trash_votes`
- UI: `VoteButtonsComponent` with 🚀 / 🗑️

---

## Promotion Statuses

```
pending_review → active → expired → archived
                       → flash_active → flash_ended
                       → removed
```

Promotions submitted by users always start as `pending_review`.
Bot-collected promotions go directly to `active` after deduplication.

---

## User Account Flow

```
Register (email + CPF + password)
  → pending_confirmation
    → [click email link]
      → pending_profile
        → [complete profile form]
          → active
```

CPF is stored as:
- `cpf_hash` (SHA-256) — used for UNIQUE constraint
- `cpf_encrypted` (AES-256) — used for recovery by data pipeline

CPF is **never** returned in any API response.

---

## Email Templates

All emails use HTML templates stored in `apps/backend/email/templates/`.
**Never send plain text emails.**

Templates are identified by number:
`01_confirm` · `02_welcome` · `03_reset_password` · `04_promotion_alert` · `05_flash_alert` · `06_favorite_expiring` · `06b_*` (favorite event variants) · `07_account_banned` · `08_promotion_removed` · `09_promotion_featured` · `10_new_comment` · `11_flash_summary` · `12_channel_unreachable` · `13_newsletter`

Full template specs (variables, subjects, preview texts) → PROJECT_DOCS.md § RN-022.

---

## Telegram Bot Commands

The bot serves two independent purposes:
1. **Notification channel** — receives automatic alerts (configured in user preferences)
2. **Query interface** — user sends commands or free text to search promotions

Key commands: `/start` `/hoje` `/relampago` `/categoria` `/loja` `/meus` `/conectar` `/pausar` `/ativar`

Free-text messages trigger a full-text, accent-insensitive search.
Results paginated at **5 per page** via inline keyboard callback.
Rate limit: 20 queries/hour (authenticated), 10/hour (anonymous) — enforced in Redis.

---

## Newsletter (AI-Generated)

Generated by the **Claude API** (`claude-sonnet-4-20250514`) using the top 10 trending promotions.
- Daily edition: generated at 8:50am, sent at 9:00am
- Weekly edition: every Monday at 8:50am, sent at 9:00am
- Claude returns HTML body only (no `<html>`, `<head>`, `<body>` tags)
- HTML is sanitized (strip `<script>`, event attributes) before wrapping with platform header/footer
- Final HTML stored in `newsletter_editions.html_content`

Prompt and full spec → PROJECT_DOCS.md § TEMPLATE-13 and § Módulo Newsletter.

---

## Scheduler Jobs

All in `apps/scheduler/jobs/`:

| Job | Frequency | What it does |
|---|---|---|
| `trend_score_job` | Every 5 min | Recalculates trend_score for all active promotions |
| `flash_check_job` | Every 1 min | Expires flash promotions by deadline or stock |
| `expiration_check_job` | Every 10 min | Archives expired regular promotions |
| `newsletter_daily_job` | Daily 8:50am | Generates and sends daily newsletter |
| `newsletter_weekly_job` | Monday 8:50am | Generates and sends weekly newsletter |

---

## Migrations

All SQL migrations live in `infra/migrations/`.
Never modify the database schema directly.
Always create a new numbered migration file.

---

## Makefile Commands

```bash
make dev           # Start full local stack
make backend       # Start backend + dependencies only
make frontend      # Start frontend only
make migrate       # Run pending migrations
make migrate-down  # Roll back last migration
make test          # Run all tests
make lint          # Lint all apps
```

---

## Environment Variables

All required variables are documented in PROJECT_DOCS.md § Variáveis de Ambiente.

Required service credentials:
- `DATABASE_URL`, `REDIS_URL`, `RABBITMQ_URL`
- `JWT_SECRET`, `JWT_REFRESH_SECRET`
- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
- Affiliate APIs: `ML_APP_ID`, `SHOPEE_APP_ID`, `AMAZON_ACCESS_KEY`
- Email: `SMTP_PROVIDER`, `SMTP_HOST`, `EMAIL_FROM`
- Push: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`
- WhatsApp: `WHATSAPP_ACCESS_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`
- Telegram: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`
- AI: `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`
- Platform: `PLATFORM_PRIMARY_COLOR`, `PLATFORM_LOGO_URL`

---

## LGPD Compliance

- CPF, birth date, phone, and gender are **private fields** — never exposed in public endpoints
- Every email must include an unsubscribe link with a unique token
- Users can request full data export: `GET /api/v1/users/me/export`
- Users can request account deletion: `DELETE /api/v1/users/me` (sensitive data deleted, activity data anonymized)
- Data pipeline accesses PII only through masked views

---

## Before Opening a PR

**Escopo:**
- [ ] Apenas arquivos listados no PLANNING.md aprovado foram modificados
- [ ] Nenhum arquivo foi renomeado, movido ou reorganizado sem solicitação
- [ ] Nenhuma refatoração não solicitada foi feita em código existente
- [ ] Nenhuma dependência nova foi adicionada sem aprovação do usuário
- [ ] Todas as dúvidas foram perguntadas ao usuário antes da execução

**Code quality:**
- [ ] Nenhum comentário adicionado ao código
- [ ] Nenhum N+1 introduzido — toda query dentro de loop foi revisada
- [ ] Nenhum domínio importando diretamente o package de outro domínio
- [ ] Nenhuma lógica de negócio em controller ou repository
- [ ] Nenhum parâmetro solto onde um DTO poderia ser usado
- [ ] SOLID aplicado — cada arquivo tem responsabilidade única
- [ ] Funções pequenas e com nome que revela intenção
- [ ] Sem números ou strings mágicas — constantes ou enums usados

**Frontend:**
- [ ] Tags HTML semânticas usadas corretamente
- [ ] Nenhum `<div>` onde existe tag semântica equivalente
- [ ] Todos os `<img>` têm `alt` significativo
- [ ] Nenhum estilo inline — apenas classes SCSS
- [ ] BEM naming seguido nas classes SCSS
- [ ] `@use` no lugar de `@import` nos arquivos SCSS
- [ ] `OnPush` aplicado nos novos componentes

**Projeto:**
- [ ] Todos os novos e-mails usam templates HTML
- [ ] Votos referenciados como 🚀 rocket / 🗑️ trash (nunca hot/cold)
- [ ] CPF nunca retornado em nenhuma response
- [ ] Novos domínios seguem a convenção dos 7 arquivos
- [ ] Migration criada para qualquer mudança de schema
- [ ] `PLANNING.md` deletado da raiz após conclusão
- [ ] Task movida de `pendentes/` para `concluidas/` com relatório completo preenchido
- [ ] `.claude/PROJECT_STATUS.md` atualizado — itens movidos para ✅ Implementado
- [ ] `.claude/PROJECT_DOCS.md` atualizado — se alguma regra de negócio ou arquitetura mudou
- [ ] Relatório lista todos os arquivos criados, modificados e deletados
- [ ] Decisões tomadas durante a execução registradas no relatório
- [ ] `make test` passando
- [ ] `make lint` passando