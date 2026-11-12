# Documentação da API — CVGWeb

Referência técnica de todos os endpoints da API REST do **Sistema de Gestão de
Campos Experimentais da Embrapa Cenargen (CVGWeb)**.

**Base URL:** `http://localhost:8000/api`

**Autenticação:** Laravel Sanctum — token Bearer retornado no login.

**Content-Type:** `application/json` em todas as requisições e respostas.

---

## Especificação OpenAPI (Swagger)

A especificação formal da API está em **[`swagger.yaml`](swagger.yaml)**
(OpenAPI 3.0.3) — é ela que vale para gerar cliente, importar no Postman ou
abrir uma UI navegável.

```bash
# UI local, sem instalar nada
docker run --rm -p 8081:8080 \
  -e SWAGGER_JSON=/spec/swagger.yaml \
  -v "$PWD/06-api:/spec" swaggerapi/swagger-ui
# abre em http://localhost:8081
```

Também dá para colar o arquivo em [editor.swagger.io](https://editor.swagger.io)
ou importá-lo no Postman/Insomnia.

> Esta página é a referência em prosa: explica cada endpoint, os códigos de
> erro e as regras por trás deles. O `swagger.yaml` é o contrato. Ao mexer em
> rota, mude os **dois**.

---

## Índice

1. [Autenticação](#1-autenticação)
2. [Acesso (convite e recuperação de senha)](#2-acesso-convite-e-recuperação-de-senha)
3. [Funcionários](#3-funcionários)
4. [Projetos](#4-projetos)
5. [Equipe do projeto](#5-equipe-do-projeto)
6. [Casas de vegetação](#6-casas-de-vegetação)
7. [Reservas](#7-reservas)
8. [Assistente (chat LLM)](#8-assistente-chat-llm)
9. [Tabelas de apoio](#9-tabelas-de-apoio)
10. [Auditoria](#10-auditoria)
11. [Painel e contexto](#11-painel-e-contexto)
12. [Códigos de erro](#12-códigos-de-erro)
13. [Rate limiting](#13-rate-limiting)

---

## 1. Autenticação

### `POST /login`

Autentica o usuário e retorna um token Sanctum.

**Rate limit:** `throttle:login` — 5 requisições/minuto por IP + 5/minuto por identificador.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `login` | string | Sim | E-mail ou nick do usuário |
| `senha` | string | Sim | Senha do usuário |

**Respostas:**

| Status | Descrição |
| --- | --- |
| `200` | Login bem-sucedido. Retorna `{ token, funcionario }` |
| `401` | Credenciais inválidas ou conta inativa |
| `422` | Campos obrigatórios ausentes |
| `429` | Rate limit excedido |

---

### `GET /me`

🔒 **Autenticada**

Retorna os dados do usuário autenticado.

**Resposta `200`:** objeto `Funcionario` com relacionamentos carregados.

---

### `GET /me/contexto`

🔒 **Autenticada**

Retorna o contexto completo pós-login: projetos do usuário, casas de vegetação e permissões por projeto.

**Resposta `200`:** `{ projetos, casas_vegetacao, permissoes }`.

---

### `POST /logout`

🔒 **Autenticada**

Revoga o token atual.

**Resposta `200`:** `{ message: "Logout realizado." }`

---

## 2. Acesso (convite e recuperação de senha)

### `POST /esqueci-senha`

Envia e-mail de recuperação de senha. Sempre retorna a mesma resposta, independentemente de o e-mail existir ou não (prevenção de enumeração).

**Rate limit:** `throttle:acesso` — 5 req/min por IP + 3/min por e-mail.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `email` | string | Sim | E-mail do usuário |

**Resposta `200`:** `{ message: "Se o e-mail estiver cadastrado, enviaremos o link." }`

---

### `GET /definir-senha/{token}`

Valida um token de convite ou recuperação de senha.

**Rate limit:** `throttle:acesso`

**Parâmetro de rota:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `token` | string | Token recebido por e-mail (64 caracteres) |

**Respostas:**

| Status | Descrição |
| --- | --- |
| `200` | Token válido. Retorna `{ funcionario: { nome, nick, email } }` |
| `404` | Token inválido, expirado ou já utilizado |

---

### `POST /definir-senha`

Define a senha do usuário a partir de um token válido. Invalida todos os tokens e sessões anteriores.

**Rate limit:** `throttle:acesso`

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `token` | string | Sim | Token de 64 caracteres |
| `senha` | string | Sim | Nova senha (mín. 8, máx. 72 caracteres) |

**Respostas:**

| Status | Descrição |
| --- | --- |
| `200` | Senha definida com sucesso |
| `404` | Token inválido |
| `422` | Validação falhou |

---

### `POST /funcionarios/{id}/convite`

🔒 **Autenticada** · Requer permissão de administração

Reenvia o e-mail de convite para um funcionário que ainda não definiu a senha.

**Resposta `200`:** `{ message, link? }` — o campo `link` só aparece quando o e-mail falha (fallback para o admin copiar manualmente).

---

### `GET /acesso/email`

🔒 **Autenticada**

Retorna a situação atual do serviço de e-mail (se o provedor configurado está acessível).

---

## 3. Funcionários

### `GET /funcionarios`

🔒 **Autenticada** · Requer perfil `FUNCIONARIO-C` ou superior

Lista funcionários com paginação e busca.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `busca` | string | Filtra por nome ou nick (LIKE com escape de wildcards) |
| `ativo` | boolean | Filtra por situação (`1` = ativos, `0` = inativos) |
| `projeto_id` | int | Filtra membros de um projeto específico |
| `page` | int | Página (paginação Laravel padrão) |
| `per_page` | int | Itens por página |

---

### `GET /funcionarios/{id}`

🔒 **Autenticada**

Retorna um funcionário pelo ID com cargo e projetos.

---

### `POST /funcionarios`

🔒 **Autenticada** · Requer perfil `FUNCIONARIO-I` ou `funcionario.criar` no projeto

Cadastra um novo funcionário. O sistema envia automaticamente o e-mail de convite.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `nome` | string | Sim | Nome completo (máx. 150) |
| `nick` | string | Sim | Nome de usuário único (máx. 30) |
| `email` | string | Sim | E-mail válido (RFC strict, máx. 150) |
| `matricula` | string | Não | Matrícula funcional (máx. 20) |
| `cargo_id` | int | Não | FK para `cargos` |
| `obs` | string | Não | Observações |
| `projeto_id` | int | Condicional | Obrigatório para quem cadastra via permissão de projeto |
| `super_usuario` | boolean | Não | Só o SU pode definir |
| `permissoes` | string[] | Não | Tokens de permissão global (só admin) |

---

### `PUT /funcionarios/{id}`

🔒 **Autenticada** · Requer perfil `FUNCIONARIO-A` ou `funcionario.editar` no projeto

Atualiza dados do funcionário. Campos `ativo` e `senha` têm restrições:
- `ativo`: só admin pode alterar.
- `senha`: só o próprio usuário ou admin.

---

### `PATCH /funcionarios/{id}/situacao`

🔒 **Autenticada** · Requer perfil admin

Ativa ou desativa um funcionário. Desativar revoga todos os tokens de acesso.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `ativo` | boolean | Sim | Nova situação |

---

### `DELETE /funcionarios/{id}`

🔒 **Autenticada** · Requer perfil admin

Desativa o funcionário (soft delete) e revoga todos os tokens.

---

## 4. Projetos

### `GET /projetos`

🔒 **Autenticada**

Lista projetos. Admin vê todos; demais veem apenas os seus.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `busca` | string | Filtra por código ou título (LIKE com escape) |
| `status_projeto_id` | int | Filtra por status |
| `situacao` | string | `PENDENTE`, `APROVADO` ou `RECUSADO` — monta a fila de aprovação |

---

### `GET /projetos/{id}`

🔒 **Autenticada**

Retorna um projeto com responsável, status e equipe.

---

### `POST /projetos`

🔒 **Autenticada** — qualquer funcionário ativo

Administrador cria o projeto já `APROVADO`. Qualquer outra pessoa **propõe**:
o projeto nasce `PENDENTE`, com ela como responsável, e não aceita reservas
até ser aprovado.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `codigo` | string | Sim | Código único (máx. 40) |
| `titulo` | string | Sim | Título do projeto (máx. 200) |
| `descricao` | string | Não | Descrição |
| `status_projeto_id` | int | Não | FK para `status_projeto`; em branco, usa o primeiro cadastrado |
| `funcionario_id` | int | Sim | Pesquisador responsável — ignorado para quem não é admin, que é sempre o próprio |
| `data_inicio` | date | Não | Data de início |
| `data_fim` | date | Não | Data de fim (≥ data_inicio) |
| `colaboradores` | int[] | Não | IDs de funcionários para a equipe; só o admin monta equipe na criação |

---

### `POST /projetos/{id}/aprovar`

🔒 **Autenticada** · Requer perfil admin

Aceita a proposta: grava `situacao=APROVADO`, `aprovado_por` e `aprovado_em`, e
concede ao autor, no próprio projeto, `projeto.editar`, `reserva.criar`,
`reserva.editar`, `reserva.cancelar` e `equipe.adicionar`.

**`422`** se o projeto já estiver aprovado.

---

### `POST /projetos/{id}/recusar`

🔒 **Autenticada** · Requer perfil admin

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `motivo` | string | Sim | Por que a proposta não foi aceita (máx. 255) |

Grava `situacao=RECUSADO` e o motivo. O autor continua podendo editar e
propor de novo.

---

### `PUT /projetos/{id}`

🔒 **Autenticada** · Requer autorização via `ProjetoPolicy`

---

### `DELETE /projetos/{id}`

🔒 **Autenticada** · Requer perfil admin

---

## 5. Equipe do projeto

### `GET /projetos/{projeto}/equipe`

🔒 **Autenticada**

Lista membros da equipe com função e permissões.

---

### `POST /projetos/{projeto}/equipe`

🔒 **Autenticada** · Requer autorização no projeto

Adiciona membro à equipe.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `funcionario_id` | int | Sim | ID do funcionário |
| `funcao` | string | Não | Função na equipe (máx. 80) |

---

### `PUT /projetos/{projeto}/equipe/{funcionario}`

🔒 **Autenticada**

Atualiza a função de um membro.

---

### `PUT /projetos/{projeto}/equipe/{funcionario}/permissoes`

🔒 **Autenticada** · Requer perfil admin ou SU

Define as permissões de CRUD de um colaborador dentro do projeto.

---

### `DELETE /projetos/{projeto}/equipe/{funcionario}`

🔒 **Autenticada**

Remove membro da equipe.

---

## 6. Casas de vegetação

### `GET /casas-vegetacao`

🔒 **Autenticada**

Lista casas de vegetação com paginação e busca.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `busca` | string | Filtra por descrição ou tipo (LIKE com escape) |
| `status` | string | `ATIVA`, `MANUTENCAO` ou `INATIVA` |
| `tipo` | string | Tipo do espaço |

---

### `GET /casas-vegetacao/tipos`

🔒 **Autenticada**

Retorna os tipos distintos de espaço cadastrados.

---

### `GET /casas-vegetacao/{id}`

🔒 **Autenticada**

Retorna detalhes de uma casa de vegetação.

---

### `GET /casas-vegetacao/{id}/disponibilidade`

🔒 **Autenticada**

Retorna a capacidade disponível por período.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `data_inicio` | date | Início do período |
| `data_fim` | date | Fim do período |

---

### `POST /casas-vegetacao`

🔒 **Autenticada** · Requer perfil `CASA-I`

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `codigo` | string | Sim | Código único (máx. 20) |
| `descricao` | string | Sim | Descrição (máx. 150) |
| `tipo` | string | Não | Tipo do espaço (máx. 60, padrão: "Casa de vegetação") |
| `localizacao` | string | Não | Localização no campus (máx. 60) |
| `area_m2` | int | Não | Área em metros quadrados |
| `capacidade` | int | Não | Número de bancadas (informativo) |
| `unidade_capacidade` | string | Não | `VASO` ou `BANCADA` |
| `capacidade_total` | int | Não | Capacidade reservável |
| `vasos_3cm` | int | Não | Capacidade de vasos de 3cm |
| `vasos_5cm` | int | Não | Capacidade de vasos de 5cm |
| `vasos_10cm` | int | Não | Capacidade de vasos de 10cm |
| `obs` | string | Não | Observações |
| `status` | string | Não | `ATIVA`, `MANUTENCAO` ou `INATIVA` |

---

### `PUT /casas-vegetacao/{id}`

🔒 **Autenticada** · Requer perfil `CASA-A`

---

### `PATCH /casas-vegetacao/{id}/situacao`

🔒 **Autenticada** · Requer perfil admin

Altera a situação (`ATIVA`, `MANUTENCAO`, `INATIVA`).

---

### `DELETE /casas-vegetacao/{id}`

🔒 **Autenticada** · Requer perfil `CASA-E`

---

## 7. Reservas

### `GET /reservas`

🔒 **Autenticada**

Lista reservas. Admin vê todas; demais veem apenas dos seus projetos.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `casa_vegetacao_id` | int | Filtra por espaço |
| `projeto_id` | int | Filtra por projeto |
| `situacao` | string | `PENDENTE`, `APROVADA`, `CANCELADA` |
| `data_inicio` | date | Início do período |
| `data_fim` | date | Fim do período |

---

### `GET /reservas/{id}`

🔒 **Autenticada**

Retorna detalhes de uma reserva.

---

### `POST /reservas`

🔒 **Autenticada** · Requer perfil `RESERVA-I` ou `reserva.criar` no projeto

Cria uma reserva. Valida se a quantidade cabe na capacidade livre do período.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `casa_vegetacao_id` | int | Sim | FK para casas_vegetacao |
| `funcionario_id` | int | Sim | Pesquisador responsável |
| `projeto_id` | int | Condicional | Obrigatório para não-admin |
| `quantidade` | int | Sim | Quantidade a reservar (≥ 1) |
| `data_inicio` | date | Sim | Data de início |
| `data_fim` | date | Sim | Data de fim (≥ data_inicio) |
| `finalidade` | string | Sim | Finalidade da reserva (máx. 255) |
| `obs` | string | Não | Observações |

---

### `PUT /reservas/{id}`

🔒 **Autenticada** · Requer autorização via `ReservaPolicy`

Atualiza uma reserva pendente. Revalida capacidade.

---

### `POST /reservas/{id}/aprovar`

🔒 **Autenticada** · Requer perfil `RESERVA-A`

Aprova uma reserva pendente. Revalida capacidade no momento da aprovação.

---

### `POST /reservas/{id}/cancelar`

🔒 **Autenticada** · Requer autorização via `ReservaPolicy`

Cancela uma reserva.

---

### `DELETE /reservas/{id}`

🔒 **Autenticada** · Requer autorização via `ReservaPolicy`

---

## 8. Assistente (chat LLM)

### `GET /assistente`

🔒 **Autenticada**

Verifica se o assistente está configurado (chave Groq presente).

**Resposta `200`:** `{ disponivel: boolean }`

---

### `POST /assistente`

🔒 **Autenticada** · **Rate limit:** `throttle:assistente`

Envia uma mensagem ao assistente com LLM.

**Request body:**

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `mensagens` | array | Sim | Array de mensagens (máx. 32) |
| `mensagens[].role` | string | Sim | `user` ou `assistant` |
| `mensagens[].content` | string | Sim | Conteúdo (máx. 2000 chars) |

**Regras:** a última mensagem deve ter `role: "user"`.

---

## 9. Tabelas de apoio

### `GET /cargos`

🔒 **Autenticada**

Lista todos os cargos. Aceita `?busca=` para filtrar por nome.

---

### `GET /status-projeto`

🔒 **Autenticada**

Lista todos os status de projeto com flag `encerrado`.

---

### `GET /permissoes/catalogo`

🔒 **Autenticada**

Retorna o catálogo completo de permissões disponíveis por projeto.

---

## 10. Auditoria

### `GET /auditorias`

🔒 **Autenticada** · Requer perfil admin

Lista registros de auditoria com paginação e filtros.

**Query params:**

| Param | Tipo | Descrição |
| --- | --- | --- |
| `funcionario_id` | int | Filtra por autor |
| `entidade` | string | Filtra por tipo de entidade |
| `acao` | string | Filtra por ação (CRIOU, ALTEROU, EXCLUIU, etc.) |
| `data_inicio` | date | Início do período |
| `data_fim` | date | Fim do período |

---

### `GET /auditorias/filtros`

🔒 **Autenticada** · Requer perfil admin

Retorna os valores distintos de `entidade` e `acao` para montar filtros na UI.

---

## 11. Painel e contexto

### `GET /painel`

🔒 **Autenticada** · Requer perfil admin

Retorna números consolidados do sistema: total de funcionários, projetos, casas, reservas ativas, etc.

---

## 12. Códigos de erro

| Código | Significado |
| --- | --- |
| `200` | Sucesso |
| `201` | Recurso criado |
| `401` | Não autenticado |
| `403` | Sem permissão |
| `404` | Recurso não encontrado |
| `422` | Erro de validação. O corpo retorna `{ message, errors: { campo: [...] } }` |
| `429` | Rate limit excedido |
| `500` | Erro interno |

---

## 13. Rate limiting

| Limiter | Rotas | Limite |
| --- | --- | --- |
| `login` | `POST /login` | 5/min por IP + 5/min por identificador |
| `acesso` | `POST /esqueci-senha`, `GET/POST /definir-senha` | 5/min por IP + 3/min por e-mail |
| `assistente` | `POST /assistente` | Configurável via `GROQ_LIMITE_POR_MINUTO` |
| `api` (global) | Todas as rotas autenticadas | 60/min por usuário (padrão Laravel) |

Quando o limite é excedido, a API retorna `429 Too Many Requests` com headers `Retry-After` e `X-RateLimit-*`.
