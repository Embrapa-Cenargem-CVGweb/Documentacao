<p align="center">
  <img src="_assets/logo_embrapa.png" alt="Embrapa" width="240">
</p>

<h1 align="center">CVGWeb — Documentação Técnica</h1>

<p align="center">
  Sistema de Gestão de Campos Experimentais · Embrapa Cenargen
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Laravel-12-FF2D20?style=flat-square&logo=laravel&logoColor=white" alt="Laravel 12">
  <img src="https://img.shields.io/badge/TypeScript-5-3178C6?style=flat-square&logo=typescript&logoColor=white" alt="TypeScript 5">
  <img src="https://img.shields.io/badge/MySQL-8-4479A1?style=flat-square&logo=mysql&logoColor=white" alt="MySQL 8">
  <img src="https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white" alt="Docker Compose">
  <img src="https://img.shields.io/badge/OpenAPI-3.0.3-6BA539?style=flat-square&logo=openapiinitiative&logoColor=white" alt="OpenAPI 3.0.3">
</p>

---

Este repositório é a **única fonte de documentação** do CVGWeb. Os READMEs dos
repositórios de código explicam apenas o que o projeto é e como ligá-lo; tudo
que é detalhe técnico mora aqui.

## Por onde começar

| Quem chega | Caminho |
| :--- | :--- |
| **Nunca viu o projeto** | [1. Visão geral](01-visao-geral/README.md) → [2. Arquitetura](02-arquitetura/README.md) |
| **Vai mexer na API** | [3. Back-end](03-backend/README.md) → [6. API](06-api/README.md) → [5. Banco](05-banco-de-dados/README.md) |
| **Vai mexer na tela** | [4. Front-end](04-frontend/README.md) → [6. API](06-api/README.md) |
| **Vai instalar ou operar** | [README da raiz](../README.md) → [9. Operação](09-operacao/README.md) |
| **Vai auditar ou avaliar** | [7. Segurança](07-seguranca/README.md) → [8. Testes](08-testes/README.md) |

---

## Sumário

| # | Seção | Conteúdo |
| :--- | :--- | :--- |
| **01** | [Visão geral](01-visao-geral/README.md) | O problema, a proposta, atores, módulos e glossário. Documento de Visão, Requisitos e Termo de Anuência |
| **02** | [Arquitetura](02-arquitetura/README.md) | Os três repositórios, containers, stack, caminho de uma requisição, autenticação e decisões de arquitetura |
| **03** | [Back-end](03-backend/README.md) | Camadas do Laravel, controllers, models, regras de negócio, autorização, auditoria, assistente e convenções |
| **04** | [Front-end](04-frontend/README.md) | Estrutura, telas, convenções, integração com a API, estado e acessibilidade |
| **05** | [Banco de dados](05-banco-de-dados/README.md) | Schema MySQL, tabelas, relacionamentos, índices e dados iniciais |
| **06** | [API](06-api/README.md) | Referência de todos os endpoints + [`swagger.yaml`](06-api/swagger.yaml) (OpenAPI 3.0.3) |
| **07** | [Segurança](07-seguranca/README.md) | Autenticação, autorização, rate limiting, cabeçalhos, XSS, auditoria e backlog |
| **08** | [Testes](08-testes/README.md) | Suíte E2E com Playwright, cenários automatizados e testes manuais |
| **09** | [Operação](09-operacao/README.md) | Docker, variáveis de ambiente, e-mail, backup e problemas comuns |

Complementos:

- [Migração do legado](03-backend/migracao-do-legado.md) — o que veio da aplicação PHP antiga e onde cada regra ficou
- [`arquivo/`](arquivo/) — documentos históricos em PDF, mantidos só para referência

---

## Estrutura deste repositório

```
Documentacao/
├── README.md                    ← você está aqui
├── 01-visao-geral/
│   ├── README.md                Visão geral, atores, módulos e glossário
│   ├── documento-de-visao.md
│   ├── documento-de-requisitos.md
│   └── termo-de-anuencia.jpg
├── 02-arquitetura/README.md     Containers, stack e decisões
├── 03-backend/
│   ├── README.md                Camadas, regras de negócio e autorização
│   └── migracao-do-legado.md    Mapa regra-a-regra do sistema PHP antigo
├── 04-frontend/README.md        Estrutura, telas e convenções
├── 05-banco-de-dados/README.md  Schema, tabelas e relacionamentos
├── 06-api/
│   ├── README.md                Referência dos endpoints
│   └── swagger.yaml             Especificação OpenAPI 3.0.3
├── 07-seguranca/README.md       Medidas implementadas e backlog
├── 08-testes/README.md          Playwright E2E e testes manuais
├── 09-operacao/README.md        Docker, env, e-mail e troubleshooting
├── _assets/                     Imagens usadas pelos documentos
└── arquivo/                     PDFs históricos (não use como referência)
```

---

## Repositórios do projeto

| Repositório | O que é | README |
| :--- | :--- | :--- |
| **embrapa** (orquestração) | `docker-compose.yml`, `.env` e os dois repos clonados dentro | [README](../README.md) |
| **Embrapa-backend** | API Laravel 12 + Sanctum | [README](../Embrapa-backend/README.md) |
| **Embrapa-frontend** | Interface TypeScript + Vite | [README](../Embrapa-frontend/README.md) |
| **Documentacao** | Este repositório | — |

> Os links para os outros repositórios funcionam quando os quatro estão
> clonados lado a lado, como descrito no [README da raiz](../README.md).

---

## Manutenção

**A documentação anda junto com o projeto.** Mudou o sistema, muda o documento
no mesmo trabalho — não depois.

| Tipo de alteração | Atualize |
| :--- | :--- |
| Nova rota ou endpoint | [06-api/README.md](06-api/README.md) **e** [06-api/swagger.yaml](06-api/swagger.yaml) |
| Migration, coluna ou índice | [05-banco-de-dados](05-banco-de-dados/README.md) |
| Regra de negócio | [03-backend](03-backend/README.md) + [01-visao-geral](01-visao-geral/README.md) se mudar o comportamento visível |
| Tela, componente ou convenção do front | [04-frontend](04-frontend/README.md) |
| Medida de segurança | [07-seguranca](07-seguranca/README.md) |
| Cenário de teste | [08-testes](08-testes/README.md) |
| Variável de ambiente, container ou provedor | [09-operacao](09-operacao/README.md) |
| Decisão de arquitetura | [02-arquitetura](02-arquitetura/README.md) (tabela de decisões) |
| Feature completa | Todas as seções acima que ela tocar |

Convenções deste repositório:

- Cada seção é uma pasta numerada com um `README.md` como entrada.
- Documento novo entra na pasta da seção a que pertence, nunca na raiz.
- Imagem vai para `_assets/` e é referenciada por caminho relativo.
- Nada de segredo: chave, senha e token ficam no `.env`, jamais na documentação.

---

<p align="center">
  <sub>CVGWeb · Embrapa Cenargen · CEUB — documentação atualizada em novembro de 2026</sub>
</p>
