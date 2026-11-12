# Documentação do Banco de Dados — CVGWeb

Documentação técnica do schema relacional do **Sistema de Gestão de Campos
Experimentais da Embrapa Cenargen (CVGWeb)**.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [SGBD e configuração](#2-sgbd-e-configuração)
3. [Diagrama de relacionamentos](#3-diagrama-de-relacionamentos)
4. [Tabelas do domínio](#4-tabelas-do-domínio)
5. [Tabelas de infraestrutura](#5-tabelas-de-infraestrutura)
6. [Dados iniciais (seeders)](#6-dados-iniciais-seeders)
7. [Migrations](#7-migrations)
8. [Plano de migração: MySQL + MongoDB](#8-plano-de-migração-mysql--mongodb)

---


## Aprovação de projeto

A tabela `projetos` carrega, além do andamento da pesquisa
(`status_projeto_id`), a situação da **proposta**:

| Coluna | Tipo | Para que serve |
| :--- | :--- | :--- |
| `situacao` | `varchar(20)`, padrão `APROVADO`, indexada | `PENDENTE`, `APROVADO` ou `RECUSADO` |
| `aprovado_por` | FK → `funcionarios`, nula | Quem aceitou a proposta |
| `aprovado_em` | `timestamp`, nulo | Quando foi aceita |
| `motivo_recusa` | `varchar(255)`, nulo | Por que não foi aceita |

O padrão `APROVADO` cobre os projetos que já existiam: foram criados pelo
administrador, o único que podia criá-los antes desta mudança.

Migration: `2026_11_12_000001_add_aprovacao_to_projetos_table.php`.

---

## 1. Visão geral

O CVGWeb utiliza **MySQL 8** como banco relacional, gerenciado pelo Eloquent ORM
do Laravel 12. Toda a estrutura é criada e versionada via migrations.

O arquivo `cvgweb-schema.sql` na raiz do repositório de orquestração contém o
schema completo gerado a partir das migrations, para consulta e importação
manual. **Nunca edite este arquivo à mão** — use `scripts/exportar-schema.sh`
para regerar.

### Números atuais

| Métrica | Valor |
| --- | --- |
| Tabelas de domínio | 8 |
| Tabelas de infraestrutura | 7 |
| Migrations | 21 |
| Casas de vegetação pré-cadastradas | 50 |
| Cargos pré-cadastrados | 5 |
| Status de projeto | 5 |

---

## 2. SGBD e configuração

| Parâmetro | Valor |
| --- | --- |
| SGBD | MySQL 8.0+ |
| Charset | `utf8mb4` |
| Collation | `utf8mb4_unicode_ci` |
| Engine | InnoDB |
| Fuso horário | Configurável via `APP_TIMEZONE` (padrão `America/Sao_Paulo`) |
| Porta padrão (Docker) | 3307 no host, 3306 no container |

### Conexão padrão (Docker)

```env
DB_CONNECTION=mysql
DB_HOST=mysql          # nome do serviço no docker-compose
DB_PORT=3306
DB_DATABASE=embrapa_cvg
DB_USERNAME=cvgweb
DB_PASSWORD=cvgweb
```

---

## 3. Diagrama de relacionamentos

```
┌─────────────┐       ┌──────────────────┐       ┌───────────────┐
│   cargos    │───┐   │  funcionarios    │───┐   │  projetos     │
│             │   └──>│  (usuarios)      │   │   │               │
│ id          │       │                  │   │   │ id            │
│ nome        │       │ id               │   └──>│ funcionario_id│
│ descricao   │       │ nome, cargo_id   │       │ codigo, titulo│
└─────────────┘       │ email, matricula │       │ status_proj_id│──┐
                      │ tipo, perfil     │       │ data_inicio   │  │
                      │ password         │       │ data_fim      │  │
                      │ cargo_id ────────┘       └───────┬───────┘  │
                      │ ativo            │               │          │
                      └────────┬─────────┘               │          │
                               │                         │          │
                               │    ┌────────────────────┘          │
                               │    │                               │
                      ┌────────┴────┴────────┐        ┌─────────────┴──┐
                      │colaboradores_projeto │        │ status_projeto │
                      │                     │        │                │
                      │ id                  │        │ id             │
                      │ projeto_id          │        │ descricao      │
                      │ funcionario_id      │        │ encerrado      │
                      │ funcao             │        └────────────────┘
                      │ permissoes (json)   │
                      └─────────────────────┘

┌──────────────────┐       ┌──────────────────┐
│ casas_vegetacao  │       │    reservas      │
│                  │<──────│                  │
│ id               │       │ id               │
│ codigo, descricao│       │ casa_vegetacao_id│
│ tipo, localizacao│       │ funcionario_id   │──> funcionarios
│ area_m2          │       │ projeto_id       │──> projetos
│ capacidade       │       │ quantidade       │
│ capacidade_total │       │ data_inicio      │
│ unidade_capac.   │       │ data_fim         │
│ status           │       │ finalidade       │
└──────────────────┘       │ situacao         │
                           └──────────────────┘

┌──────────────────┐       ┌──────────────────┐
│ convites_acesso  │       │   auditorias     │
│                  │       │                  │
│ id               │       │ id               │
│ funcionario_id   │──>    │ funcionario_id   │──> funcionarios
│ email            │ func. │ funcionario_nome │
│ tipo             │       │ acao             │
│ token_hash       │       │ entidade         │
│ expira_em        │       │ entidade_id      │
│ usado_em         │       │ rotulo           │
│ criado_por       │──>    │ alteracoes (JSON)│
└──────────────────┘ func. │ projeto_id       │
                           │ ip               │
                           └──────────────────┘
```

---

## 4. Tabelas do domínio

### 4.1 `cargos`

Cargos do quadro de funcionários do Cenargen.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `nome` | varchar(100) | UNIQUE, NOT NULL | Nome do cargo |
| `descricao` | varchar(255) | nullable | Descrição |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

---

### 4.2 `funcionarios`

Usuários do sistema. O campo `password` nulo indica que o usuário foi cadastrado
mas ainda não definiu a senha via convite.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `nome` | varchar(150) | NOT NULL | Nome completo |
| `matricula` | varchar(20) | UNIQUE, nullable | Identificador interno (rótulo na tela) |
| `email` | varchar(150) | UNIQUE, NOT NULL | E-mail — único identificador de entrada (o `nick` foi removido em `2026_11_12_000002`) |
| `email_verified_at` | timestamp | nullable | Data de verificação |
| `tipo` | varchar(20) | NOT NULL, default `FUNCIONARIO` | `SU`, `ADMIN`, `FUNCIONARIO` |
| `perfil` | varchar(255) | NOT NULL | Tokens de permissão: `/SU/`, `/FUNCIONARIO-I/A/E/`, `/RESERVA-I/A/`, `/CASA-I/A/E/` |
| `password` | varchar(255) | nullable | Hash bcrypt (12 rounds). Nulo = aguardando convite |
| `convite_enviado_em` | timestamp | nullable | Data do último convite enviado |
| `senha_definida_em` | timestamp | nullable | Data em que a senha foi definida |
| `ultimo_acesso_em` | timestamp | nullable | Último login |
| `cargo_id` | bigint unsigned | FK → `cargos.id`, ON DELETE SET NULL | |
| `ativo` | tinyint(1) | NOT NULL, default `1` | 1 = ativo, 0 = desativado |
| `obs` | text | nullable | Observações |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

**Índices:** `tipo` (INDEX).

---

### 4.3 `status_projeto`

Situações possíveis de um projeto de pesquisa.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `descricao` | varchar(60) | UNIQUE, NOT NULL | Ex: "Em andamento", "Concluído" |
| `encerrado` | tinyint(1) | NOT NULL, default `0` | 1 = não aceita novas reservas |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

---

### 4.4 `projetos`

Projetos de pesquisa vinculados a pesquisadores e casas de vegetação.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `codigo` | varchar(40) | UNIQUE, NOT NULL | Código do projeto |
| `titulo` | varchar(200) | NOT NULL | Título |
| `descricao` | text | nullable | Descrição |
| `status_projeto_id` | bigint unsigned | FK → `status_projeto.id`, ON DELETE RESTRICT | |
| `funcionario_id` | bigint unsigned | FK → `funcionarios.id`, ON DELETE RESTRICT | Responsável |
| `data_inicio` | date | nullable | |
| `data_fim` | date | nullable | |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

---

### 4.5 `colaboradores_projeto`

Equipe de cada projeto (além do responsável). Cada colaborador pode ter permissões de CRUD específicas dentro do projeto.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `projeto_id` | bigint unsigned | FK → `projetos.id`, ON DELETE CASCADE | |
| `funcionario_id` | bigint unsigned | FK → `funcionarios.id`, ON DELETE CASCADE | |
| `funcao` | varchar(80) | nullable | Função na equipe |
| `permissoes` | json | nullable | Permissões CRUD: `funcionario.criar`, `funcionario.editar`, `reserva.criar`, `reserva.aprovar` |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

**Unique constraint:** `(projeto_id, funcionario_id)`.

---

### 4.6 `casas_vegetacao`

Estufas e espaços experimentais do Cenargen. A capacidade reservável é definida
por `capacidade_total` + `unidade_capacidade`.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `codigo` | varchar(20) | UNIQUE, NOT NULL | Ex: CVG01 |
| `descricao` | varchar(150) | NOT NULL | Nome descritivo |
| `tipo` | varchar(60) | NOT NULL, default `Casa de vegetação` | Tipo do espaço |
| `localizacao` | varchar(60) | nullable | Setor no campus |
| `area_m2` | int unsigned | nullable | Área em m² |
| `capacidade` | smallint unsigned | NOT NULL, default `0` | Nº de bancadas (informativo) |
| `unidade_capacidade` | varchar(10) | NOT NULL, default `VASO` | `VASO` ou `BANCADA` |
| `capacidade_total` | int unsigned | NOT NULL, default `0` | Capacidade reservável |
| `vasos_3cm` | int unsigned | nullable | |
| `vasos_5cm` | int unsigned | nullable | |
| `vasos_10cm` | int unsigned | nullable | |
| `obs` | text | nullable | |
| `status` | varchar(15) | NOT NULL, default `ATIVA` | `ATIVA`, `MANUTENCAO`, `INATIVA` |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

**Índices:** `status` (INDEX).

---

### 4.7 `reservas`

Reservas de capacidade em casas de vegetação, com período e aprovação.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `casa_vegetacao_id` | bigint unsigned | FK → `casas_vegetacao.id`, ON DELETE RESTRICT | |
| `funcionario_id` | bigint unsigned | FK → `funcionarios.id`, ON DELETE RESTRICT | Pesquisador |
| `projeto_id` | bigint unsigned | FK → `projetos.id`, ON DELETE RESTRICT, nullable | |
| `quantidade` | int unsigned | NOT NULL, default `0` | Quantidade reservada |
| `data_inicio` | date | NOT NULL | |
| `data_fim` | date | NOT NULL | ≥ data_inicio |
| `finalidade` | varchar(255) | NOT NULL | |
| `obs` | text | nullable | |
| `situacao` | varchar(15) | NOT NULL, default `PENDENTE` | `PENDENTE`, `APROVADA`, `CANCELADA` |
| `aprovado_por` | bigint unsigned | FK → `funcionarios.id`, nullable | Quem aprovou |
| `aprovado_em` | timestamp | nullable | |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

---

### 4.8 `convites_acesso`

Tokens para convite de primeiro acesso e recuperação de senha.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `funcionario_id` | bigint unsigned | FK → `funcionarios.id`, ON DELETE CASCADE | |
| `email` | varchar(150) | NOT NULL | E-mail de destino |
| `tipo` | varchar(15) | NOT NULL, default `CONVITE` | `CONVITE` ou `RECUPERACAO` |
| `token_hash` | varchar(64) | UNIQUE, NOT NULL | SHA-256 do token de 64 chars |
| `expira_em` | timestamp | NOT NULL | Expiração (48h após criação) |
| `usado_em` | timestamp | nullable | Data de uso |
| `criado_por` | bigint unsigned | FK → `funcionarios.id`, ON DELETE SET NULL | Admin que criou |
| `created_at` | timestamp | nullable | |
| `updated_at` | timestamp | nullable | |

**Índices:** `(funcionario_id, tipo)` (INDEX).

**Regras de negócio:**
- Token: 64 caracteres aleatórios (`Str::random(64)`), armazenado como hash SHA-256.
- Emitir um novo token invalida todos os anteriores não-usados do mesmo usuário (transação).
- Validade: 48 horas.
- Uso: marca `usado_em`, define a senha e revoga todas as sessões.

---

### 4.9 `auditorias`

Trilha de auditoria: quem criou, alterou ou excluiu o quê.

| Coluna | Tipo | Restrições | Descrição |
| --- | --- | --- | --- |
| `id` | bigint unsigned | PK, auto_increment | |
| `funcionario_id` | bigint unsigned | FK → `funcionarios.id`, ON DELETE SET NULL | Autor da ação |
| `funcionario_nome` | varchar(150) | nullable | Nome do autor (snapshot) |
| `acao` | varchar(10) | NOT NULL | Ex: `CRIOU`, `ALTEROU`, `EXCLUIU`, `LOGIN`, `LOGOUT` |
| `entidade` | varchar(60) | NOT NULL | Ex: `Funcionario`, `Reserva`, `CasaVegetacao` |
| `entidade_id` | bigint unsigned | nullable | ID do registro afetado |
| `rotulo` | varchar(150) | nullable | Label legível (ex: nome do funcionário) |
| `alteracoes` | json | nullable | Diff: `{ campo: { de: old, para: new } }` |
| `projeto_id` | bigint unsigned | nullable | Projeto relacionado |
| `ip` | varchar(45) | nullable | IP da requisição |
| `created_at` | timestamp | nullable | |

**Índices:** `(entidade, entidade_id)`, `created_at`, `funcionario_id`.

---

## 5. Tabelas de infraestrutura

Tabelas gerenciadas automaticamente pelo Laravel:

| Tabela | Finalidade |
| --- | --- |
| `personal_access_tokens` | Tokens Sanctum (autenticação de API) |
| `sessions` | Sessões (`SESSION_DRIVER=database`) |
| `cache` | Cache (`CACHE_STORE=database`) |
| `cache_locks` | Travas de cache |
| `jobs` | Fila de jobs (`QUEUE_CONNECTION=database`) |
| `job_batches` | Lotes de jobs |
| `failed_jobs` | Jobs que falharam |
| `migrations` | Controle de versão do schema |

---

## 6. Dados iniciais (seeders)

Os seeders são executados automaticamente no `migrate --seed` e são idempotentes.

### Cargos

| ID | Nome | Descrição |
| --- | --- | --- |
| 1 | Pesquisador | Pesquisador responsável por projetos e reservas |
| 2 | Analista | Analista de laboratório e apoio à pesquisa |
| 3 | Técnico | Técnico de campo e de casas de vegetação |
| 4 | Assistente | Assistente administrativo |
| 5 | Administrador | Administração do sistema |

### Status de projeto

| ID | Descrição | Encerrado |
| --- | --- | --- |
| 1 | Em elaboração | Não |
| 2 | Em andamento | Não |
| 3 | Suspenso | Sim |
| 4 | Concluído | Sim |
| 5 | Cancelado | Sim |

### Casas de vegetação

51 estufas pré-cadastradas (CVG01 a CVG51) com área, capacidade de bancadas e
vasos medidas em campo. Distribuídas pelos setores Norte, Leste, Central e Sul.

### Super usuário

| E-mail | Senha | Tipo | Perfil |
| --- | --- | --- | --- |
| `admin@cvgweb.local` (`CVGWEB_SU_EMAIL`) | `super123` | SU | `/SU/` |

> **Troque a senha no primeiro acesso.**

---

## 7. Migrations

As migrations estão em `database/migrations/` e são aplicadas em ordem
cronológica pelo Laravel. O histórico completo:

| Migration | Descrição |
| --- | --- |
| `create_users_table` | Tabela `sessions` (infra Laravel) |
| `create_cache_table` | Tabelas `cache` e `cache_locks` |
| `create_jobs_table` | Tabelas `jobs`, `job_batches` e `failed_jobs` |
| `create_cargos_table` | Tabela de cargos |
| `create_funcionarios_table` | Usuários do sistema |
| `create_status_projeto_table` | Status de projeto |
| `create_projetos_table` | Projetos de pesquisa |
| `create_colaboradores_projeto_table` | Equipe por projeto |
| `create_casas_vegetacao_table` | Estufas e espaços |
| `create_reservas_table` | Reservas de capacidade |
| `create_personal_access_tokens_table` | Tokens Sanctum |
| `add_acesso_to_funcionarios_table` | Campos de controle de acesso |
| `add_capacidade_to_casas_vegetacao_table` | Capacidade reservável e vasos |
| `add_quantidade_to_reservas_table` | Quantidade na reserva |
| `create_convites_acesso_table` | Tokens de convite e recuperação |
| `create_auditorias_table` | Trilha de auditoria |
| `remove_tabelas_sem_uso` | Remove tabelas obsoletas |
| `add_permissoes_to_colaboradores_projeto` | Permissões por projeto (JSON) |
| `add_projeto_e_ip_to_auditorias` | Projeto e IP na auditoria |
| `add_indices_to_auditorias_table` | Índices de performance |
| `add_envio_to_convites_acesso_table` | Controle de envio do convite |
| `alarga_acao_das_auditorias` | Amplia campo ação |

---

## 8. Plano de migração: MySQL + MongoDB

### Motivação

O sistema está planejado para operar com **arquitetura híbrida**:
- **MySQL** — dados transacionais e relacionais (usuários, projetos, reservas, casas de vegetação).
- **MongoDB** — dados não-estruturados e de alta volumetria (logs de auditoria, histórico de conversas do assistente, dados de sensores/IoT futuros).

### O que migra para o MongoDB

| Dados | Motivo |
| --- | --- |
| **Auditorias** | Volume crescente, schema flexível (campo `alteracoes` já é JSON), consultas agregadas por período |
| **Histórico de conversas** | Mensagens do assistente LLM: arrays de objetos variáveis, sem relacionamento com outras entidades |
| **Dados de sensores (futuro)** | Leituras de temperatura, umidade e luminosidade das estufas — séries temporais de alta frequência |

### O que permanece no MySQL

| Dados | Motivo |
| --- | --- |
| Funcionários, cargos | Integridade referencial, transações, constraints UNIQUE |
| Projetos, equipe, permissões | Relacionamentos complexos (N:N com atributos) |
| Casas de vegetação | Foreign keys com reservas |
| Reservas | Validação de capacidade transacional (requer ACID) |
| Convites de acesso | Tokens com expiração e FK para funcionário |
| Tabelas do Laravel (sessions, cache, jobs) | Driver `database` do framework |

### Estratégia de migração

1. **Fase 1 — Preparação:** instalar o pacote `mongodb/laravel-mongodb`, configurar a conexão no `.env` e `config/database.php`.
2. **Fase 2 — Auditoria:** criar modelo `Auditoria` com connection `mongodb`, migrar dados existentes da tabela MySQL com script artisan.
3. **Fase 3 — Assistente:** implementar persistência das conversas no MongoDB (atualmente não há persistência).
4. **Fase 4 — Validação:** rodar em paralelo (dual-write) por um período, conferir integridade, desativar escrita no MySQL para as coleções migradas.

### Configuração planejada

```env
# .env
MONGO_DB_HOST=127.0.0.1
MONGO_DB_PORT=27017
MONGO_DB_DATABASE=cvgweb
MONGO_DB_USERNAME=
MONGO_DB_PASSWORD=
```

```php
// config/database.php
'mongodb' => [
    'driver' => 'mongodb',
    'host' => env('MONGO_DB_HOST', '127.0.0.1'),
    'port' => env('MONGO_DB_PORT', 27017),
    'database' => env('MONGO_DB_DATABASE', 'cvgweb'),
    'username' => env('MONGO_DB_USERNAME'),
    'password' => env('MONGO_DB_PASSWORD'),
],
```

### Impacto no Docker

O `docker-compose.yml` receberá um novo serviço:

```yaml
mongo:
  image: mongo:7
  ports:
    - "${MONGO_PORT_HOST:-27017}:27017"
  volumes:
    - mongo-data:/data/db
```
