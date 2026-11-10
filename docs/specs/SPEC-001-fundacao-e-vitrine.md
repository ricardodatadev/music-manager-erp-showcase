# SPEC-001 — Fundação Segura & Vitrine do Produto

> **Status:** ✅ Concluída · **Versão:** 1.0.9 · **Data:** 08/11/2026
> **Referências:** [Especificação de Requisitos (SRS)](./especificacao_requisitos.md) · [Especificação de Stack](./especificacao_stack.md)

---

## 1. Objetivo

Entregar a **primeira versão apresentável** do Instituto Musical Ricardo para portfólio no GitHub:

1. Uma **landing page** que apresente a proposta do sistema e seus módulos principais.
2. Um **painel administrativo funcional** com login, dashboard com indicadores e CRUDs simples dos módulos principais.
3. Uma **fundação técnica segura e extensível** (multi-tenant, RBAC, validação, testes, CI), para que as próximas specs só aprofundem os módulos, sem precisar refazer a base.

> **Princípio da SPEC-001:** _raso nas funcionalidades, sólido na fundação._ Funcionalidades podem ser simples, mas segurança, isolamento de dados e arquitetura **não** são simplificados.

## 2. Escopo

### 2.1 Dentro do escopo

| Módulo (SRS)        | O que entra nesta spec                                                         | Requisitos relacionados                                     |
| :------------------ | :----------------------------------------------------------------------------- | :---------------------------------------------------------- |
| Tenant & Unidades   | Tenant demo, CRUD de Unidades e Salas                                          | RNF-002                                                     |
| Identity            | Login/logout, sessão segura, perfis e RBAC por módulo                          | RNF-001, Matriz 3.2                                         |
| Alunos & Matrículas | CRUD de Alunos (com Responsável quando menor), Planos e Matrículas simples     | RF-001 (parcial), RF-002 (parcial, sem contrato/assinatura) |
| Professores         | CRUD de Professores (regime, valor hora/aula, instrumentos)                    | RF-005                                                      |
| Agenda              | Cadastro e listagem de Aulas com **anti-overbooking** (sala, professor, aluno) | RF-008, RN-003                                              |
| Financeiro          | Faturas simples (gerar, listar, marcar paga/cancelar)                          | RF-010 (parcial, sem gateway)                               |
| Dashboard           | KPIs e gráficos básicos                                                        | Objetivos 2.3                                               |
| Vitrine             | Landing page, README de portfólio, capturas de tela                            | —                                                           |

### 2.2 Fora do escopo (próximas specs)

Contrato eletrônico e assinatura com OTP, pipeline de aulas experimentais, trancamento/rescisão com multa, diário de classe/chamada, PWA do professor e portal do aluno, reposição self-service, folha docente, gateway de pagamento, régua de cobrança (Redis/BullMQ/WhatsApp), DRE, NFS-e, ferramentas de LGPD (exportação/anonimização), escopo por unidade (Gerente), SuperAdmin, Nginx/TLS de produção. Ver o roadmap na **seção 10**.

## 3. Decisões de Arquitetura

| #     | Decisão                                                                                                                                                        | Justificativa                                                                                                                                                                                                                                                                                                                            |
| :---- | :------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AD-01 | **Monorepo com npm workspaces**: `apps/api`, `apps/web`, `packages/shared`                                                                                     | Contratos (schemas Zod, enums, matriz RBAC) compartilhados entre frontend e backend, com uma única fonte de verdade.                                                                                                                                                                                                                     |
| AD-02 | **API: Node.js 24 + TypeScript + Express 5**, em monólito modular                                                                                              | Segue a stack definida e a estrutura `modules/*` da especificação de stack.                                                                                                                                                                                                                                                              |
| AD-03 | **PostgreSQL 16+ com Drizzle ORM** para queries tipadas; **migrations em SQL puro** versionadas, com executor próprio (checksum + advisory lock)               | Drizzle é leve e próximo do SQL. As migrations são SQL porque RLS, _exclusion constraints_ e funções `SECURITY DEFINER` não são geradas por ferramentas de ORM; isso também evitou o `drizzle-kit`, que trazia dependência vulnerável (`esbuild` antigo via `@esbuild-kit`). Um teste garante que o schema Drizzle não diverge do banco. |
| AD-04 | **Isolamento multi-tenant em duas camadas**: filtro por `tenant_id` na aplicação **+ Row-Level Security (RLS) no Postgres**                                    | Defesa em profundidade: um bug na aplicação não vaza dados de outro tenant. A API conecta com um papel sem `BYPASSRLS`.                                                                                                                                                                                                                  |
| AD-05 | **Anti-overbooking garantido pelo banco** via _exclusion constraints_ (`btree_gist` + `tstzrange`), com pré-checagem na aplicação para gerar mensagem amigável | O `SELECT ... FOR UPDATE` da especificação de stack não impede a corrida quando ainda **não existe** nenhuma linha conflitante (duas inserções simultâneas passam). A constraint torna a garantia atômica.                                                                                                                               |
| AD-06 | **Web: React + Vite + Tailwind CSS + React Router + TanStack Query + React Hook Form + Zod** (gráficos em SVG próprio)                                         | Segue a stack definida, com o mínimo de bibliotecas extras e todas bem mantidas. Em desenvolvimento o Vite faz **proxy de `/api`** para a API: frontend e API ficam na mesma origem (cookies SameSite=Strict e `connect-src 'self'`), espelhando o Nginx de produção.                                                                    |
| AD-07 | **Sessão por cookies httpOnly**: access token JWT curto (15 min) + refresh token opaco rotativo (7 dias, armazenado como hash)                                 | Tokens fora do alcance de JavaScript (mitiga roubo via XSS). A rotação detecta reuso de refresh token.                                                                                                                                                                                                                                   |
| AD-08 | **Hash de senhas com Argon2id**                                                                                                                                | Recomendado pelo OWASP; supera o mínimo de bcrypt pedido no RNF-001.                                                                                                                                                                                                                                                                     |
| AD-09 | **Valores monetários trafegam em centavos (inteiro)** e são armazenados como `NUMERIC(10,2)`                                                                   | Evita erros de ponto flutuante em cálculos financeiros.                                                                                                                                                                                                                                                                                  |
| AD-10 | **Nomenclatura:** termos de domínio em português (`aluno`, `matricula`), termos técnicos em inglês (`repository`, `service`)                                   | Mantém a rastreabilidade com o SRS e o padrão técnico do ecossistema.                                                                                                                                                                                                                                                                    |
| AD-11 | **Arquivar em vez de excluir** alunos e professores (`status = INATIVO`); FKs com `RESTRICT` em dados financeiros/agenda                                       | Preserva histórico e retenção legal (RNF-003), sem perda acidental de dados.                                                                                                                                                                                                                                                             |

### 3.1 Estrutura de diretórios alvo

```text
music-manager-erp/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── @shared/            # errors, middlewares (auth, tenant, rbac, rate-limit), utils
│   │   │   ├── config/             # env.ts (validação Zod), logger (pino)
│   │   │   ├── database/           # schema Drizzle, migrations SQL, seed demo, conexão
│   │   │   ├── modules/
│   │   │   │   ├── identity/       # auth, sessões, usuários
│   │   │   │   ├── tenant/         # unidades, salas
│   │   │   │   ├── academic/       # alunos, responsáveis, planos, matrículas
│   │   │   │   ├── teachers/       # professores
│   │   │   │   ├── agenda/         # aulas + anti-overbooking
│   │   │   │   ├── financial/      # faturas
│   │   │   │   └── dashboard/      # agregações de KPIs
│   │   │   ├── app.ts              # composição do Express (sem listen, testável)
│   │   │   └── server.ts           # bootstrap
│   │   └── test/
│   └── web/
│       └── src/
│           ├── app/                # router, providers, layout
│           ├── features/           # landing, auth, dashboard, alunos, professores, ...
│           ├── components/ui/      # design system (Button, Input, Table, Modal, ...)
│           └── lib/                # api client, formatters
├── packages/shared/                # schemas Zod, enums, matriz RBAC, tipos
├── docs/specs/
├── docker-compose.yml
└── .github/workflows/ci.yml
```

Cada módulo da API segue o padrão **`routes → controller → service → repository`**, com schemas Zod importados de `packages/shared`.

### 3.2 Ajustes ao modelo de dados da especificação de stack

| Ajuste                                                                                                                                                  | Motivo                                                               |
| :------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------- |
| `tenant_id NOT NULL` em **todas** as tabelas de domínio (`salas`, `alunos`, `professores`, `responsaveis`, `aulas`, `faturas`, `diarios_frequencia`...) | RNF-002 e RLS exigem a chave em toda consulta.                       |
| `tenants.slug` (único)                                                                                                                                  | Identificar a escola no login sem precisar consultar outros tenants. |
| `alunos.usuario_id` e `professores.usuario_id` **opcionais**; nome, e-mail e WhatsApp na própria entidade                                               | Nem todo aluno ou professor tem login no MVP (portais virão depois). |
| Tabela `responsaveis` própria (N alunos → 1 responsável)                                                                                                | Segue o ER do SRS e o critério de aceite do RF-001.                  |
| CPF único por tenant (`UNIQUE(tenant_id, cpf)`) e validação de dígitos verificadores                                                                    | Exceção "CPF duplicado" do RF-001.                                   |
| `diários_frequencia` → `diarios_frequencia` (sem acento)                                                                                                | Identificadores sem caracteres especiais.                            |
| Novas tabelas: `sessoes` (refresh tokens com hash) e `faturas`                                                                                          | Requisitos de sessão segura e do módulo financeiro.                  |
| `professores.instrumentos TEXT[]`                                                                                                                       | Especialidades do RF-005.                                            |

## 4. Requisitos de Segurança (obrigatórios nesta spec)

| ID     | Requisito                                                                                                                                                                                                       |
| :----- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SEC-01 | Nenhum segredo versionado: `.env` no `.gitignore`, apenas `.env.example` com valores fictícios. A API **falha ao iniciar** se uma variável obrigatória faltar ou for fraca (ex.: `JWT_SECRET` < 32 caracteres). |
| SEC-02 | Senhas com Argon2id; mensagem genérica em falha de login ("credenciais inválidas"), sem revelar se o e-mail existe.                                                                                             |
| SEC-03 | Cookies `httpOnly`, `Secure` (em produção) e `SameSite=Strict`; refresh token rotativo e invalidado no logout; reuso de refresh token revoga toda a família de sessões.                                         |
| SEC-04 | Proteção CSRF: CORS restrito à origem do frontend (`credentials: true`) + header obrigatório `X-Requested-With` em métodos mutáveis.                                                                            |
| SEC-05 | `helmet` com CSP restritiva no frontend servido; `x-powered-by` desabilitado.                                                                                                                                   |
| SEC-06 | Rate limiting global e reforçado em `/auth/login` (ex.: 5 tentativas/min por IP + e-mail).                                                                                                                      |
| SEC-07 | Toda entrada validada por Zod (body, params e query), com limite de tamanho do body (100 kb), UUIDs validados e paginação com `limit` máximo de 100.                                                            |
| SEC-08 | `tenant_id` vem **sempre** da sessão autenticada, nunca do cliente. RLS ativo em todas as tabelas de domínio, com testes automatizados de isolamento entre tenants.                                             |
| SEC-09 | RBAC aplicado **no backend** em toda rota (o frontend só oculta menus). Negação por padrão: rota sem permissão declarada não é exposta.                                                                         |
| SEC-10 | Erros padronizados (`{ error: { code, message } }`) sem stack trace ou detalhes de SQL em produção; logs estruturados (pino) com _redaction_ de `password`, `cookie`, `authorization` e `cpf`.                  |
| SEC-11 | Dados pessoais minimizados nas listagens (CPF mascarado: `***.456.789-**`); CPF completo só no detalhe, para perfis autorizados.                                                                                |
| SEC-12 | Postgres no Docker exposto apenas em `127.0.0.1`; API conecta com papel de aplicação sem privilégios de superusuário/owner.                                                                                     |
| SEC-13 | CI executa `npm audit --audit-level=high`, lint, typecheck e testes; Dependabot habilitado.                                                                                                                     |
| SEC-14 | Seed de demonstração **recusa rodar** com `NODE_ENV=production`; dados 100% fictícios.                                                                                                                          |
| SEC-15 | React sem `dangerouslySetInnerHTML`; nenhuma informação de sessão em `localStorage`.                                                                                                                            |

### 4.1 Verificação final (item 8.6)

Revisão executada em 08/11/2026: cada requisito foi conferido no código e possui teste automatizado ou controle de CI.

| ID     | Status | Onde está                                                                      | Como é verificado                                                                                        |
| :----- | :----: | :----------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| SEC-01 |   ✅   | `config/env.ts`, `database-env.ts`, `.gitignore`, `scripts/gerar-env.mjs`      | `test/http/env.test.ts` (segredo fraco/exemplo, mensagem sem valor); varredura de segredos a cada commit |
| SEC-02 |   ✅   | `@shared/utils/password.ts`, `auth.service.ts` (hash fictício anti-timing)     | `test/http/auth.test.ts` (resposta idêntica para 4 tipos de falha)                                       |
| SEC-03 |   ✅   | `@shared/http/cookies.ts`, `auth.service.ts` (rotação + família)               | `auth.test.ts` (cookies, reuso revoga família, logout revoga na hora)                                    |
| SEC-04 |   ✅   | `middlewares/anti-csrf.ts`, CORS em `app.ts`                                   | `test/http/seguranca-http.test.ts`                                                                       |
| SEC-05 |   ✅   | `helmet` (API: `default-src 'none'`), CSP no build do web (`vite.config.ts`)   | `seguranca-http.test.ts`; inspeção do `dist/index.html`                                                  |
| SEC-06 |   ✅   | `middlewares/rate-limit.ts`                                                    | `auth.test.ts` (5 falhas → 429 sem afetar outra conta)                                                   |
| SEC-07 |   ✅   | `@shared/http/rota.ts`, schemas em `@imr/shared`                               | `rota.test.ts`, `seguranca-http.test.ts` (413, JSON malformado, campos extras)                           |
| SEC-08 |   ✅   | `0002_seguranca_rls.sql`, `tenant-context.ts`, FKs compostas                   | `test/database/rls.test.ts`, `test/modulos/isolamento-rbac.test.ts`                                      |
| SEC-09 |   ✅   | `rotas.protegida({ permissao })`, `MATRIZ_RBAC`                                | `isolamento-rbac.test.ts` (matriz perfil × endpoint), `rota.test.ts`                                     |
| SEC-10 |   ✅   | `error-handler.ts`, `errors/postgres.ts`, `config/logger.ts`                   | `test/http/logger.test.ts`, `seguranca-http.test.ts`                                                     |
| SEC-11 |   ✅   | `mascararCpf` + regra de detalhe em alunos/professores/responsáveis            | `test/modulos/cadastros.test.ts`, `packages/shared/test/cpf.test.ts`                                     |
| SEC-12 |   ✅   | `docker-compose.yml` (127.0.0.1), papel `imr_app` sem privilégios              | `rls.test.ts` (não superusuário, sem BYPASSRLS, sem DELETE em faturas)                                   |
| SEC-13 |   ✅   | `.github/workflows/ci.yml`, `codeql.yml`, `dependabot.yml` (Actions por SHA)   | CI no GitHub                                                                                             |
| SEC-14 |   ✅   | `setup.ts` (`bloquearEmProducao`), dados `.example`                            | `test/database/seed.test.ts`                                                                             |
| SEC-15 |   ✅   | Regra ESLint proíbe `dangerouslySetInnerHTML`; `localStorage` só guarda o tema | `npm run lint`; revisão por busca no código                                                              |

**Riscos residuais conhecidos (tratados em specs futuras):** rate limit em memória (trocar por Redis com múltiplas instâncias — SPEC-006/008); `frame-ancestors` e TLS dependem do Nginx de produção (SPEC-008); exportação/anonimização de dados pessoais (LGPD) na SPEC-007.

## 5. Perfis no MVP (subconjunto da Matriz RBAC 3.2)

A matriz fica em `packages/shared/src/rbac.ts` no formato `perfil → módulo → nível` (`none | read | write`) e é usada pelo backend (guard) e pelo frontend (menus).

| Perfil      | Unidades/Salas | Alunos/Matrículas | Professores | Agenda |     Financeiro      |    Dashboard    |
| :---------- | :------------: | :---------------: | :---------: | :----: | :-----------------: | :-------------: |
| ADMIN       |     write      |       write       |    write    | write  |        write        |  todos os KPIs  |
| COORDENADOR |      read      |       read        |    write    |  read  |        none         |  alunos, aulas  |
| SECRETARIA  |      read      |       write       |    read     | write  | write (operacional) |  todos os KPIs  |
| FINANCEIRO  |      none      |       read        |    none     |  none  |        write        | alunos, receita |

O dashboard exibe apenas os KPIs dos módulos em que o perfil tem ao menos `read` (regra derivada da matriz, sem exceções manuais).

Os perfis `SUPERADMIN`, `GERENTE`, `PROFESSOR` e `ALUNO` já existem no enum, mas **não acessam o painel** nesta spec (login retorna 403 com mensagem "Portal em breve").

## 6. Contratos da API (v1)

Base: `/api/v1`. Todas as rotas, exceto `auth/login`, `auth/refresh` e `health`, exigem sessão.

| Recurso      | Endpoints                                                                                                                                                     |
| :----------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Health       | `GET /health`                                                                                                                                                 |
| Auth         | `POST /auth/login` · `POST /auth/refresh` · `POST /auth/logout` · `GET /auth/me`                                                                              |
| Unidades     | `GET/POST /unidades` · `GET/PATCH/DELETE /unidades/:id`                                                                                                       |
| Salas        | `GET/POST /salas` · `GET/PATCH/DELETE /salas/:id`                                                                                                             |
| Alunos       | `GET/POST /alunos` · `GET/PATCH /alunos/:id` · `POST /alunos/:id/arquivar` · `POST /alunos/:id/reativar`                                                      |
| Responsáveis | `GET/POST /responsaveis` · `GET/PATCH /responsaveis/:id`                                                                                                      |
| Professores  | `GET/POST /professores` · `GET/PATCH /professores/:id` · `POST /professores/:id/arquivar` · `POST /professores/:id/reativar`                                  |
| Planos       | `GET/POST /planos` · `GET/PATCH /planos/:id`                                                                                                                  |
| Matrículas   | `GET/POST /matriculas` · `GET/PATCH /matriculas/:id`                                                                                                          |
| Aulas        | `GET /aulas?de=&ate=&professorId=&salaId=&alunoId=` (janela ≤ 62 dias) · `GET /aulas/:id` · `POST /aulas` (409 `CONFLITO_AGENDA`) · `PATCH /aulas/:id/status` |
| Faturas      | `GET/POST /faturas` · `POST /faturas/:id/pagar` · `POST /faturas/:id/cancelar`                                                                                |
| Dashboard    | `GET /dashboard/resumo`                                                                                                                                       |

Listagens retornam `{ data: [], meta: { page, limit, total } }` e aceitam `?page=&limit=&q=` (a agenda retorna `{ data: [] }` pela janela de datas). Valores monetários trafegam em centavos (`...Centavos`). Erros de banco viram respostas amigáveis: `409 REGISTRO_DUPLICADO`, `409 REGISTRO_EM_USO`, `400 REFERENCIA_INVALIDA`, `409 CONFLITO_AGENDA` com `details.recursos`, `409 TRANSICAO_INVALIDA`.

## 7. Vitrine (frontend)

### 7.1 Landing page (`/`)

- **Hero:** nome Instituto Musical Ricardo, proposta em uma frase, CTAs "Ver demonstração" (vai para o login com credenciais demo pré-preenchidas) e "Ver no GitHub".
- **Problemas que resolve:** os 4 problemas da seção 2.2 do SRS, em cards.
- **Módulos:** um card por módulo (Alunos & Matrículas, Professores & Remuneração, Agenda & Salas, Financeiro, Pedagógico, Multi-tenant/SaaS), cada um com 2–3 recursos e um selo **"Disponível"** ou **"Em breve"**.
- **Destaques técnicos:** multi-tenant com RLS, anti-overbooking atômico, RBAC, stack.
- **Rodapé:** links do repositório e do autor.

### 7.2 Painel (`/app/*`)

- Layout com sidebar filtrada pelo RBAC, topbar com escola, usuário e logout.
- **Dashboard:** KPIs (alunos ativos, aulas na semana, receita do mês, inadimplência %), gráfico de receita dos últimos 6 meses e gráfico de aulas por dia da semana.
- **CRUDs:** tabela paginada com busca, formulário em modal ou página com validação Zod compartilhada, feedback com toasts, estados de _loading_, vazio e erro.
- **Agenda:** listagem semanal simples (sem calendário drag-and-drop) e formulário de nova aula exibindo o conflito retornado pela API.
- Responsivo (mobile-first), acessível (labels, foco visível, contraste AA), interface em PT-BR, datas em `America/Sao_Paulo` e valores em BRL.

## 8. Critérios de Aceite Globais

- [x] `docker compose up -d` + `npm install` + `npm run db:setup` + `npm run dev` sobem o projeto do zero, seguindo o README.
- [x] Um usuário do tenant A **nunca** lê nem altera dados do tenant B (teste automatizado na aplicação **e** no banco via RLS).
- [x] Duas aulas conflitantes (mesma sala, professor ou aluno) **nunca** são gravadas, inclusive em requisições simultâneas (teste automatizado).
- [x] Cada perfil só acessa o que a matriz da seção 5 permite (teste automatizado por perfil).
- [ ] CI verde: lint, typecheck, testes e `npm audit`. — os mesmos comandos passam localmente; confirmação no GitHub após o primeiro push
- [x] README com descrição, capturas de tela, arquitetura, como rodar e roadmap.

## 9. Plano de Implementação (Checklist)

> Marque `[x]` ao concluir cada item. Cada **fase** termina com um commit.

### Fase 0 — Especificação

- [x] 0.1 Analisar SRS e especificação de stack
- [x] 0.2 Redigir a SPEC-001 com decisões, segurança e checklist

### Fase 1 — Fundação do monorepo

- [x] 1.1 `package.json` raiz com workspaces (`apps/*`, `packages/*`) e scripts (`dev`, `build`, `lint`, `typecheck`, `test`, `db:*`) — `dev` e `db:migrate/seed` entram nas Fases 2 e 3, quando os apps existirem
- [x] 1.2 `tsconfig` base estrito (`strict`, `noUncheckedIndexedAccess`) e por pacote
- [x] 1.3 ESLint + Prettier + `.editorconfig`
- [x] 1.4 `.gitignore` (node_modules, dist, `.env*` exceto `.env.example`, coverage, logs) e `.gitattributes`
- [x] 1.5 `docker-compose.yml` com Postgres 16+ (porta em `127.0.0.1`, volume nomeado, healthcheck) e banco de testes separado
- [x] 1.6 `.env.example` documentado (SEC-01)
- [x] 1.7 `packages/shared`: enums (perfis, status), matriz RBAC, utilitários de CPF e dinheiro (com testes)

### Fase 2 — Banco de dados

- [x] 2.1 Schema de todas as tabelas (seção 3.2) com FKs **compostas** `(id, tenant_id)`, índices e `CHECK`s (SQL) + mapeamento Drizzle
- [x] 2.2 Migration SQL: extensão `btree_gist`, _exclusion constraints_ anti-overbooking (sala, professor e aluno) (AD-05)
- [x] 2.3 Migration SQL: papel `imr_app`, `ENABLE`/`FORCE ROW LEVEL SECURITY` e policies por `current_setting('app.tenant_id')` (AD-04)
- [x] 2.4 Helper `withTenant(tenantId, fn)`: transação + `set_config('app.tenant_id', ..., true)`
- [x] 2.5 Seed demo: 2 tenants (para provar o isolamento), 2 unidades, salas, ~8 professores, ~40 alunos, planos, matrículas, aulas da semana, 6 meses de faturas, 1 usuário por perfil (SEC-14)
- [x] 2.6 Scripts `db:migrate`, `db:seed`, `db:setup`, `db:reset`
- [x] 2.7 Testes de banco: RLS e isolamento entre tenants, FK composta, privilégio mínimo, anti-overbooking (incl. concorrência), migrator (idempotência e checksum), schema Drizzle × banco, seed

### Fase 3 — API: núcleo e segurança

- [x] 3.1 `config/env.ts` com validação Zod e _fail fast_ (SEC-01)
- [x] 3.2 Logger pino com redaction e request id (SEC-10)
- [x] 3.3 `app.ts`: helmet, CORS restrito, limite de body, rate limit, header anti-CSRF (SEC-04/05/06/07)
- [x] 3.4 `AppError` + error handler padronizado + 404 (SEC-10)
- [x] 3.5 Middleware `validate(schema)` para body, params e query (SEC-07) — implementado no construtor de rotas `@shared/http/rota.ts` (dados validados chegam tipados ao handler)
- [x] 3.6 Módulo identity: login (slug + e-mail + senha), refresh rotativo, logout, me (SEC-02/03)
- [x] 3.7 Middlewares `authenticate`, `tenantContext` e `authorize(modulo, nivel)` (SEC-08/09) — `rotas.protegida({ permissao })` aplica autenticar → autorizar → validar; `ctx.tenant()` executa com RLS. Rota protegida sem `permissao` não compila
- [x] 3.8 `GET /health` (verifica o banco)
- [x] 3.9 Testes: auth (sucesso, falha genérica, rate limit, rotação e reuso), validação de env, cabeçalhos/CORS/CSRF, tokens forjados e `alg: none`, revogação imediata (logout, usuário inativo), RBAC no construtor de rotas

### Fase 4 — API: módulos

- [x] 4.1 Unidades e Salas (CRUD)
- [x] 4.2 Professores (CRUD + arquivar)
- [x] 4.3 Responsáveis e Alunos (CRUD + arquivar; menor de idade exige responsável; CPF validado e único; CPF mascarado em listas) (SEC-11)
- [x] 4.4 Planos e Matrículas (transições de status; trancar/cancelar cancela as aulas futuras — RF-004)
- [x] 4.5 Aulas com pré-checagem e tradução da violação de constraint para 409 `CONFLITO_AGENDA`
- [x] 4.6 Faturas (gerar, listar com filtros, pagar, cancelar; status `VENCIDA` calculado por data)
- [x] 4.7 Dashboard `/resumo` (agregações SQL respeitando o RBAC: sem dados financeiros para quem não tem acesso)
- [x] 4.8 Testes de integração: isolamento entre tenants, RBAC por perfil, anti-overbooking concorrente, CRUD feliz e de erro

### Fase 5 — Web: base

- [x] 5.1 Vite + React + TS + Tailwind; tokens de design (cores, tipografia), tema claro/escuro; CSP injetada no build de produção; fontes auto-hospedadas (sem CDN externo)
- [x] 5.2 Componentes UI: Button, Input, Select, Table, Modal, Badge, Card, Toast, EmptyState, Skeleton
- [x] 5.3 Cliente HTTP (`credentials: 'include'`, header anti-CSRF, refresh automático em 401, tratamento de erros) — renovação "single-flight" (várias requisições 401 compartilham um único refresh, evitando falso positivo de reuso de token)
- [x] 5.4 Router com rotas públicas e protegidas e layout do painel com sidebar filtrada pelo RBAC; painel e login carregados sob demanda (code splitting)
- [x] 5.5 Tela de login (slug da escola + e-mail + senha; credenciais demo visíveis apenas em modo demo); redirecionamento pós-login restrito a rotas internas (anti open redirect)
- [x] 5.6 Testes do frontend: cliente HTTP (CSRF, refresh único, sessão expirada, caminho inválido), menu por perfil, destino seguro e formulário de login

### Fase 6 — Web: landing page

- [x] 6.1 Hero + CTAs — prévia ilustrativa do painel em HTML/CSS; link do GitHub via `VITE_GITHUB_URL` (validado: só https://github.com)
- [x] 6.2 Seção de problemas resolvidos
- [x] 6.3 Cards de módulos com status Disponível / Em breve
- [x] 6.4 Destaques técnicos + rodapé; seções extras: anti-overbooking passo a passo, matriz de permissões gerada da mesma fonte da API e roadmap
- [x] 6.5 SEO básico (title, meta description, Open Graph) e favicon, `theme-color`, `robots.txt` (bloqueia `/app` e `/api`); `og:image` entra com as capturas de tela (8.5)
- [x] 6.6 Testes da landing: estrutura de títulos, CTAs, status dos módulos, matriz RBAC e validação do link do GitHub

### Fase 7 — Web: dashboard e CRUDs

- [x] 7.1 Dashboard com KPIs e 2 gráficos — paleta das séries validada (contraste, luminosidade e daltonismo) nos temas claro e escuro; blocos condicionados ao RBAC
- [x] 7.2 Unidades e Salas
- [x] 7.3 Professores
- [x] 7.4 Alunos e Responsáveis
- [x] 7.5 Planos e Matrículas
- [x] 7.6 Agenda semanal + nova aula com exibição de conflito; navegação por semana, filtros por professor/sala e mudança de status das aulas
- [x] 7.7 Faturas (listar, filtrar por status, gerar, pagar, cancelar)
- [x] 7.8 Testes: conversão monetária, semana no fuso de São Paulo, gráfico (legenda/tabela), dashboard por perfil e conflito de agenda exibido no formulário

### Fase 8 — Qualidade, CI e vitrine no GitHub

- [x] 8.1 GitHub Actions: lint, typecheck, testes (com serviço Postgres), build e `npm audit` (SEC-13) — Actions fixadas por SHA, token somente leitura; + CodeQL (`security-extended`) semanal e em PRs
- [x] 8.2 Dependabot (npm e GitHub Actions)
- [x] 8.3 `SECURITY.md` (política de reporte) e `LICENSE` (MIT)
- [x] 8.4 README de portfólio: badges, descrição, capturas de tela, diagrama de arquitetura (Mermaid), como rodar, credenciais demo, decisões técnicas e roadmap; + `npm run setup:env` gera o `.env` com segredos aleatórios
- [x] 8.5 Capturas de tela em `docs/assets/` (landing, dashboard, CRUD, conflito de agenda) — geradas por script (`npm run capturas`, Playwright) + `og:image` 1200×630; a revisão visual corrigiu 9 problemas de interface (ver histórico 1.0.9)
- [x] 8.6 Revisão de segurança final (checklist da seção 4 inteiro marcado) — ver seção 4.1 (rastreabilidade SEC × teste); + teste de redaction do logger

## 10. Roadmap de Próximas Specs

| Spec     | Tema                                                                                                                   | Requisitos SRS                 |
| :------- | :--------------------------------------------------------------------------------------------------------------------- | :----------------------------- |
| SPEC-002 | Agenda avançada: grade recorrente, calendário visual, cancelamento e reposição self-service                            | RF-008, RF-009, RN-001         |
| SPEC-003 | Portal do Professor (PWA): agenda própria, chamada em 1 clique, diário de classe                                       | RF-006, RN-002                 |
| SPEC-004 | Matrícula digital: contrato dinâmico, assinatura com OTP, aulas experimentais, trancamento e rescisão                  | RF-002, RF-003, RF-004, RN-004 |
| SPEC-005 | Financeiro avançado: contas a pagar, DRE (caixa/competência), folha docente                                            | RF-007, RF-012                 |
| SPEC-006 | Integrações: adapter de gateway de pagamento, webhooks idempotentes, Redis + BullMQ, régua de cobrança WhatsApp/e-mail | RF-010, RF-011                 |
| SPEC-007 | Portal do Aluno/Responsável e LGPD (exportação, anonimização, auditoria)                                               | RNF-003                        |
| SPEC-008 | Multi-unidade (Gerente), SuperAdmin SaaS, Nginx/TLS 1.3 e deploy                                                       | RNF-001, RNF-005               |

## 11. Histórico

| Versão |    Data    | Descrição                                                                                                                                                                                                                                                                                                                                                                                                 |
| :----: | :--------: | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0.0  | 08/11/2026 | Criação da SPEC-001.                                                                                                                                                                                                                                                                                                                                                                                      |
| 1.0.1  | 08/11/2026 | Fase 1 concluída; regra do dashboard derivada da matriz RBAC (corrige coluna da SECRETARIA).                                                                                                                                                                                                                                                                                                              |
| 1.0.2  | 08/11/2026 | Fase 2 concluída; migrations em SQL puro (AD-03), FKs compostas por tenant, item 2.7 de testes de banco.                                                                                                                                                                                                                                                                                                  |
| 1.0.3  | 08/11/2026 | Fase 3 concluída; sessão revalidada no banco a cada requisição (revogação imediata); construtor de rotas com permissão obrigatória.                                                                                                                                                                                                                                                                       |
| 1.0.4  | 08/11/2026 | Fase 4 concluída; rotas `reativar`, `GET /aulas/:id`, mapeamento de erros do Postgres e contrato de erros documentado.                                                                                                                                                                                                                                                                                    |
| 1.0.5  | 08/11/2026 | Fase 5 concluída; proxy do Vite (mesma origem), CSP no build, NODE_ENV fora do .env compartilhado, item 5.6 de testes.                                                                                                                                                                                                                                                                                    |
| 1.0.6  | 08/11/2026 | Fase 6 concluída; landing com seções de diferencial (agenda e permissões) e roadmap; item 6.6 de testes.                                                                                                                                                                                                                                                                                                  |
| 1.0.7  | 08/11/2026 | Fase 7 concluída; telas de todos os módulos, gráficos em SVG próprio (AD-06), item 7.8 de testes.                                                                                                                                                                                                                                                                                                         |
| 1.0.8  | 08/11/2026 | Fase 8 (exceto 8.5): CI, CodeQL, Dependabot, SECURITY.md, LICENSE, README e revisão de segurança (seção 4.1).                                                                                                                                                                                                                                                                                             |
| 1.0.9  | 08/11/2026 | Item 8.5 concluído e SPEC-001 encerrada. A revisão visual corrigiu: barras ausentes na prévia da landing; eixo monetário cortado e eixo de contagem com rótulos fracionados arredondados; layout de indicadores; competência "Nov. De 26"; resposta de refresh não consumida (conexão presa); toast duplicado no conflito de agenda; nomes com sobrenome repetido e falta de matrículas recentes no seed. |
