# ESPECIFICAÇÃO DE STACK, ARQUITETURA E INFRAESTRUTURA
## Plataforma SaaS Multi-Tenant de Gestão Escolar — "Instituto Musical Ricardo"

---

## 1. VISÃO GERAL DA ARQUITETURA

A plataforma **Instituto Musical Ricardo** foi projetada sob o modelo de **Monólito Modular** com isolamento lógico multi-tenant. Essa abordagem prioriza simplicidade operacional, baixo custo de infraestrutura inicial e alto nível de coesão interna por domínios de negócio, permitindo fácil evolução e futura extração de microserviços caso o volume de franquias assim exija.

```
+-----------------------------------------------------------------------+
|                             CLIENT LAYER                              |
|   +---------------------------------+ +-----------------------------+ |
|   | Desktop Web (React / Tailwind)  | | PWA Mobile (React Mobile)   | |
|   | Admin, Secretaria, Financeiro   | | Alunos & Professores        | |
|   +---------------------------------+ +-----------------------------+ |
+-----------------------------------++----------------------------------+
                                    || HTTPS / TLS 1.3
+-----------------------------------vv----------------------------------+
|                            BACKEND MONOLITH                           |
|   +---------------------------------------------------------------+   |
|   | Nginx / Reverse Proxy & TLS Termination                       |   |
|   +---------------------------------------------------------------+   |
|   | Express.js / NestJS Modular Engine                            |   |
|   |   ├── Tenant & RBAC Guard Middleware                          |   |
|   |   ├── Identity Module (JWT / Auth)                            |   |
|   |   ├── Agenda Module (Anti-Overbooking Engine)                 |   |
|   |   ├── Academic Module (Diário, Chamada, Alunos)               |   |
|   |   ├── Financial Module (Faturamento, DRE, Adapter Gateway)    |   |
|   |   └── Notification Module (WhatsApp / E-mail Queues)          |   |
|   +---------------------------------------------------------------+   |
+-----------------------------------++----------------------------------+
                                    || PostgreSQL Connection Pool
+-----------------------------------vv----------------------------------+
|                            DATA STORAGE LAYER                         |
|   +---------------------------------------------------------------+   |
|   | PostgreSQL 16+ (Row-level tenant isolation, Indexes & Locks)  |   |
|   +---------------------------------------------------------------+   |
+-----------------------------------------------------------------------+
```

---

## 2. STACK TECNOLÓGICA RECOMENDADA

| Camada | Tecnologia | Justificativa Técnica |
| :--- | :--- | :--- |
| **Backend** | **Node.js + TypeScript (Express / NestJS)** | Modelo de concorrência não bloqueante ideal para requisições assíncronas (Webhooks, mensagens de WhatsApp). Tipagem estática forte garantindo robustez para contratos financeiros e cálculo de repasses. |
| **Frontend Web** | **React.js + Vite + Tailwind CSS** | Desempenho superior no carregamento client-side, componentização reutilizável para tabelas de alta densidade administrativa e estilização ágil via Utility-First CSS. |
| **Mobile / PWA** | **React.js PWA (Web Mobile)** | Evita custos extras de desenvolvimento e publicação em lojas de aplicativos (iOS/Android), mantendo usabilidade fluida para marcação de chamadas por professores e reagendamentos por alunos. |
| **Banco de Dados** | **PostgreSQL (v16+)** | Consistência ACID indispensável para operações financeiras e transações atômicas de bloqueio anti-overbooking. Suporte a queries complexas de DRE. |
| **Fila & Processamento** | **Redis + BullMQ** | Gerenciamento de tarefas assíncronas em segundo plano (disparos de régua de cobrança por WhatsApp/E-mail, expiração de faturas e atualização de relatórios). |
| **Infra / Container** | **Docker & Docker Compose** | Padronização dos ambientes de desenvolvimento, homologação e produção com isolamento de dependências. |
| **Proxy / Web Server** | **Nginx** | Reverse proxy, rescisão TLS 1.3, rate limiting contra ataques de negação de serviço e servidor estático do frontend buildado. |

---

## 3. ESTRUTURA DE DIRETÓRIOS DO PROJETO

```text
backend/
├── src/
│   ├── @shared/                      # Código compartilhado entre múltiplos domínios
│   │   ├── domain/                   # Entity base, Value Objects genéricos
│   │   ├── errors/                   # Tratamento centralizado de erros (AppError)
│   │   ├── middlewares/              # TenantId, RBAC Guard, AuthGuard, RateLimiter
│   │   └── utils/                    # Criptografia, formatadores de datas e moeda
│   ├── config/                       # Variáveis de ambiente (env.ts), logger (Winston/Pino)
│   ├── database/                     # Migrations PostgreSQL e Seeds institucionais
│   ├── modules/                      # Domínios isolados do monólito modular
│   │   ├── tenant/                   # Gestão de franquias, escolas e unidades
│   │   ├── identity/                 # Autenticação, gestão de usuários e papéis (RBAC)
│   │   ├── academic/                 # Alunos, responsáveis, matrículas e chamadas
│   │   ├── agenda/                   # Motor Anti-overbooking, salas e reposições
│   │   │   ├── domain/               # Regras do motor anti-overbooking (RN-003, RN-001)
│   │   │   ├── infrastructure/       # Queries SQL atômicas com bloqueio (FOR UPDATE)
│   │   │   └── presentation/         # Endpoints REST de agendamento/cancelamento
│   │   ├── financial/                # Faturas, DRE, Folha Docente e Adapter Gateway
│   │   │   └── adapters/             # Implementações (AsaasAdapter, MercadoPagoAdapter)
│   │   └── notifications/            # Processadores de fila (WhatsApp / SES / Sendgrid)
│   ├── server.ts                     # Bootstrap da aplicação
└── docker-compose.yml
```

---

## 4. MODELAGEM RELACIONAL DO BANCO DE DADOS (DDL)

```sql
-- Habilitação da extensão de UUID nativo
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. TENANTS E UNIDADES
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    razao_social VARCHAR(200) NOT NULL,
    nome_fantasia VARCHAR(200) NOT NULL,
    cnpj VARCHAR(14) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'ATIVO',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE unidades (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    nome VARCHAR(150) NOT NULL,
    endereco TEXT,
    telefone VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE salas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    unidade_id UUID NOT NULL REFERENCES unidades(id) ON DELETE CASCADE,
    identificacao VARCHAR(100) NOT NULL,
    capacidade INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. USUÁRIOS E PERMISSÕES (RBAC)
CREATE TABLE usuarios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    nome_completo VARCHAR(150) NOT NULL,
    email VARCHAR(150) NOT NULL,
    senha_hash VARCHAR(255) NOT NULL,
    perfil VARCHAR(30) NOT NULL, -- SUPERADMIN, ADMIN, GERENTE, COORDENADOR, SECRETARIA, FINANCEIRO, PROFESSOR, ALUNO
    status VARCHAR(20) NOT NULL DEFAULT 'ATIVO',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_usuario_tenant_email UNIQUE(tenant_id, email)
);

-- 3. PROFESSORES E ALUNOS
CREATE TABLE professores (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    usuario_id UUID NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
    cpf VARCHAR(11) NOT NULL,
    regime_contrato VARCHAR(20) NOT NULL, -- PJ_HORA_AULA, PERCENTUAL, CLT
    valor_hora_aula NUMERIC(10, 2),
    percentual_repasse NUMERIC(5, 2),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE alunos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    usuario_id UUID NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
    cpf VARCHAR(11),
    data_nascimento DATE NOT NULL,
    responsavel_financeiro_id UUID REFERENCES usuarios(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. MATRÍCULAS E CONTRATOS
CREATE TABLE planos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    nome VARCHAR(100) NOT NULL,
    periodicidade VARCHAR(20) NOT NULL, -- MENSAL, SEMESTRAL, ANUAL
    valor_mensal NUMERIC(10, 2) NOT NULL,
    fidelidade_meses INT DEFAULT 0
);

CREATE TABLE matriculas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    aluno_id UUID NOT NULL REFERENCES alunos(id),
    plano_id UUID NOT NULL REFERENCES planos(id),
    status VARCHAR(20) NOT NULL DEFAULT 'ATIVA', -- ATIVA, TRANCADA, CANCELADA
    data_inicio DATE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 5. AGENDA E REGISTRO DE AULAS (ANTI-OVERBOOKING CRÍTICO)
CREATE TABLE aulas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    matricula_id UUID REFERENCES matriculas(id),
    sala_id UUID NOT NULL REFERENCES salas(id),
    professor_id UUID NOT NULL REFERENCES professores(id),
    aluno_id UUID NOT NULL REFERENCES alunos(id),
    data_inicio TIMESTAMP WITH TIME ZONE NOT NULL,
    data_fim TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'AGENDADA', -- AGENDADA, REALIZADA, CANCELADA_ALUNO, CANCELADA_ESCOLA, REPOSICAO
    is_reposicao BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT check_intervalo_valido CHECK (data_fim > data_inicio)
);

CREATE TABLE diários_frequencia (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    aula_id UUID NOT NULL UNIQUE REFERENCES aulas(id) ON DELETE CASCADE,
    status_presenca VARCHAR(30) NOT NULL, -- PRESENTE, FALTA_JUSTIFICADA, FALTA_INJUSTIFICADA, AULA_CANCELADA
    conteudo_ministrado TEXT,
    tarefa_casa TEXT,
    registrado_em TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- ÍNDICES ESTRATÉGICOS PARA O MOTOR ANTI-OVERBOOKING E PERFORMANCE MULTI-TENANT
CREATE INDEX idx_aulas_anti_overbooking_sala ON aulas(sala_id, data_inicio, data_fim) WHERE status != 'CANCELADA_ALUNO';
CREATE INDEX idx_aulas_anti_overbooking_prof ON aulas(professor_id, data_inicio, data_fim) WHERE status != 'CANCELADA_ALUNO';
CREATE INDEX idx_aulas_anti_overbooking_aluno ON aulas(aluno_id, data_inicio, data_fim) WHERE status != 'CANCELADA_ALUNO';
CREATE INDEX idx_tenant_isolation ON usuarios(tenant_id);
```

---

## 5. LÓGICA CRÍTICA DE MOTOR ANTI-OVERBOOKING (TRANSAÇÃO ATÔMICA)

Para atender ao requisito **RN-003 (Restrição Anti-Overbooking)** e conter condições de corrida (*race conditions*), o procedimento de validação e gravação de agendamento utiliza bloqueios temporários de tabela/linha no PostgreSQL:

```typescript
// Exemplo conceitual do repositório de agendamento
import { PoolClient } from 'pg';

export async function agendarAulaComValidacaoAtomiica(
  client: PoolClient,
  params: {
    tenantId: string;
    salaId: string;
    professorId: string;
    alunoId: string;
    dataInicio: Date;
    dataFim: Date;
    matriculaId?: string;
  }
) {
  // 1. Inicia bloqueio explícito das entidades afetadas no intervalo de tempo
  const conflitoQuery = `
    SELECT id FROM aulas 
    WHERE tenant_id = $1
      AND status NOT IN ('CANCELADA_ALUNO', 'CANCELADA_ESCOLA')
      AND (data_inicio < $5 AND data_fim > $4)
      AND (sala_id = $2 OR professor_id = $3 OR aluno_id = $6)
    FOR UPDATE;
  `;

  const { rows } = await client.query(conflitoQuery, [
    params.tenantId,
    params.salaId,
    params.professorId,
    params.dataInicio,
    params.dataFim,
    params.alunoId
  ]);

  if (rows.length > 0) {
    throw new Error('CONFLITO_AGENDA: Sala, Professor ou Aluno já possuem compromisso no horário selecionado.');
  }

  // 2. Insere a aula garantindo a atomicidade dentro da mesma transação
  const insertQuery = `
    INSERT INTO aulas (tenant_id, sala_id, professor_id, aluno_id, data_inicio, data_fim, matricula_id)
    VALUES ($1, $2, $3, $4, $5, $6, $7)
    RETURNING id;
  `;

  const result = await client.query(insertQuery, [
    params.tenantId,
    params.salaId,
    params.professorId,
    params.alunoId,
    params.dataInicio,
    params.dataFim,
    params.matriculaId || null
  ]);

  return result.rows[0].id;
}
```

---

## 6. PLANO DE EXECUÇÃO E FASES DE ENTREGA

1. **FASE 1 (Fundação & Core Multi-tenant):**
   - Configuração da estrutura backend/frontend e containerização com Docker Compose.
   - Implementação dos Schemas do Postgres, Migrations e Middlewares de isolamento `tenant_id` e RBAC.
   - Telas administrativas de cadastro de Unidades, Salas, Professores e Alunos.

2. **FASE 2 (Motor de Agenda & Autosserviço):**
   - Desenvolvimento da API com validação atômica Anti-Overbooking.
   - PWA Mobile do Professor (registro de presença em 1 clique e diário de classe).
   - Fluxo self-service de cancelamento (regrados com antecedência de 24h) e geração de crédito de reposição para o Aluno.

3. **FASE 3 (Financeiro, Gateway e Notificações):**
   - Adapters para Gateways de Pagamento (Pix Dinâmico / Cartão Recorrente).
   - Integração com provedor de WhatsApp API para régua de cobrança automática e confirmações.
   - Apuração automatizada do extrato de repasse docente.