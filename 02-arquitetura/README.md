# 2. Arquitetura

O CVGWeb é uma **SPA em TypeScript** que conversa com uma **API REST em Laravel
12**, com persistência em **MySQL 8**. As três partes sobem juntas por um único
`docker compose`, a partir do repositório de orquestração.

---

## 2.1 Os três repositórios

O código não vive num monorepo. São três repositórios independentes, e o de
orquestração clona os outros dois dentro de si:

| Repositório | Pasta | Conteúdo |
| :--- | :--- | :--- |
| [embrapa](https://github.com/Embrapa-Cenargem-CVGweb) (orquestração) | raiz | `docker-compose.yml`, `.env`, README de instalação |
| [Embrapa-backend](https://github.com/Embrapa-Cenargem-CVGweb/Embrapa-backend) | `Embrapa-backend/` | API Laravel 12 + Sanctum |
| [Embrapa-frontend](https://github.com/Embrapa-Cenargem-Dashboard-Interativo/Embrapa-frontend) | `Embrapa-frontend/` | Interface TypeScript + Vite |
| [Documentacao](https://github.com/Embrapa-Cenargem-CVGweb/Documentacao) | `Documentacao/` | Esta documentação |

> As pastas precisam ter exatamente esses nomes: é assim que o
> `docker-compose.yml` as encontra.

---

## 2.2 Visão de containers

```
┌─────────────────────────────────────────────────────────────────┐
│                          Navegador                              │
│   SPA TypeScript (Vite)  ·  token Sanctum em localStorage       │
└───────────────────────────────┬─────────────────────────────────┘
                                │ HTTP/JSON · Bearer token
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API Laravel 12 (:8000)                      │
│  Middleware → Form Request → Controller → Service → Model       │
│  Sanctum · Policies · VerificaPerfil · SecurityHeaders          │
└───────┬─────────────────────────┬──────────────────┬────────────┘
        │                         │                  │
        ▼                         ▼                  ▼
┌───────────────┐        ┌─────────────────┐  ┌─────────────────┐
│  MySQL 8      │        │  SMTP / e-mail  │  │  Groq (LLM)     │
│  (:3306)      │        │  convites       │  │  assistente     │
└───────────────┘        └─────────────────┘  └─────────────────┘
```

| Container | Imagem | Porta no host | Volume |
| :--- | :--- | :--- | :--- |
| `mysql` | `mysql:8.4` | `3307` | `cvgweb_mysql_data` |
| `backend` | build do `Embrapa-backend/Dockerfile` | `8000` | `cvgweb_back_vendor` |
| `frontend` | `node:22-alpine` | `5173` | `cvgweb_front_node_modules` |

Detalhes de subida, variáveis e volumes: [9. Operação](../09-operacao/README.md).

---

## 2.3 Stack

| Camada | Tecnologia | Por quê |
| :--- | :--- | :--- |
| Interface | TypeScript 5 + Vite, **sem framework** | O sistema é um mapa e um punhado de telas CRUD; um framework custaria mais do que entrega |
| Estilo | CSS com tokens (`styles/tokens.css`) | Paleta institucional em um lugar só; nenhum componente escreve cor literal |
| API | Laravel 12 | Ecossistema do time, Eloquent, Form Requests e Policies prontos |
| Autenticação | Laravel Sanctum (tokens pessoais) | API sem estado, consumida por SPA e, no futuro, por app |
| Banco | MySQL 8 | Padrão da infraestrutura do Cenargen |
| E-mail | SMTP (Gmail em desenvolvimento) | Convite e recuperação de senha |
| LLM | Groq, chamada **pelo back-end** | A chave nunca chega ao navegador |
| Testes | Playwright (E2E) + PHPUnit | Ver [8. Testes](../08-testes/README.md) |

---

## 2.4 Caminho de uma requisição

```
Navegador
  └─ fetch() em src/services/api.ts, com Authorization: Bearer <token>
      └─ CORS + SecurityHeaders
          └─ auth:sanctum                     → 401 se o token não vale
              └─ throttle                     → 429 se estourou o limite
                  └─ VerificaPerfil           → 403 se o perfil não permite
                      └─ Form Request         → 422 se o dado é inválido
                          └─ Controller       → orquestra
                              └─ Policy       → 403 se não pode naquele registro
                                  └─ Service  → regra de negócio
                                      └─ Model (Eloquent) → MySQL
                                          └─ Auditoria (trait Auditavel)
```

A resposta sempre volta em JSON. Erros seguem o padrão do Laravel
(`message` + `errors`), documentado em [6. API](../06-api/README.md).

---

## 2.5 Autenticação e autorização

**Autenticação** é por token Sanctum:

```
POST /api/login  { login: "SU", senha: "..." }  →  { token: "1|abc..." }
```

O front guarda o token em `localStorage` (`cvgweb:token`) e o envia em toda
requisição. `POST /api/logout` revoga o token atual.

**Autorização** tem duas camadas, e as duas precisam permitir:

| Camada | Onde | O que controla |
| :--- | :--- | :--- |
| Perfil do sistema | middleware `VerificaPerfil` + `Funcionario::temPerfil()` | O que a pessoa pode fazer **no sistema** (`/SU/`, `/FUNCIONARIO-I/A/E/`) |
| Permissão por projeto | `ColaboradorProjeto` + Policies | O que ela pode fazer **dentro de cada projeto** |

Detalhamento em [3. Back-end](../03-backend/README.md#autorização-em-duas-camadas)
e [7. Segurança](../07-seguranca/README.md).

---

## 2.6 Primeiro acesso: convite por e-mail

Não existe tela de cadastro público nem senha provisória.

```
Admin cadastra a pessoa (sem senha)
   └─ back-end gera ConviteAcesso (token, validade 48 h)
       └─ e-mail com .../definir-senha?token=...
           └─ pessoa abre o link, confere nome/usuário/e-mail
               └─ cria a senha duas vezes
                   └─ resumo do acesso + "Entrar no sistema"
```

"Esqueci minha senha" dispara um link do mesmo tipo. A resposta da API é
sempre a mesma, exista a conta ou não — para não revelar quais e-mails estão
cadastrados.

---

## 2.7 Decisões de arquitetura

| Decisão | Alternativa descartada | Motivo |
| :--- | :--- | :--- |
| Front sem framework | React / Vue | Escopo pequeno; menos dependência, build mais simples, carga menor na rede interna |
| Reserva por **capacidade**, não pela estufa inteira | Bloqueio exclusivo do espaço | Duas pesquisas cabem na mesma estufa; bloquear tudo desperdiçaria o parque |
| Reserva nasce **pendente** | Reserva ativa na hora | Reservar é pedir: a liberação é um ato administrativo, com registro de quem aprovou |
| Projeto proposto pelo pesquisador também nasce **pendente** | Só o adm criar projeto | Quem não está em projeto nenhum não conseguiria nem começar — e a administração continua decidindo o que entra |
| Pendente **já segura** a capacidade | Só a ativa ocupar | Senão dois pedidos para o mesmo período passariam os dois |
| Convite por link no e-mail | OTP a cada login / senha provisória do admin | O admin nunca conhece a senha de ninguém; o link expira sozinho |
| **Migrations** como fonte da verdade do schema | `cvgweb-schema.sql` mantido à mão | Versionamento, rollback e `migrate` idempotente no boot |
| Groq chamada pelo **back-end** | `VITE_GROQ_API_KEY` no front | Tudo que começa com `VITE_` é embutido no JS publicado e fica legível |
| Auditoria por trait no Model | Log no controller | Nenhuma escrita escapa, nem a feita por comando de console |
| Três repositórios | Monorepo | Front e back pertencem a organizações diferentes no GitHub |

---

## 2.8 De onde o sistema veio

O CVGWeb substitui uma aplicação PHP legada (`pcvgweb`), que usava um formato
de transporte próprio (`[[TAG]]..[[/TAG]]`, o "MeuXML") e perfis em token. O
mapa regra-a-regra da migração está em
[Migração do legado](../03-backend/migracao-do-legado.md).
