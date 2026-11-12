# Documentação de Segurança — CVGWeb

Documentação técnica das medidas de segurança implementadas no **Sistema de
Gestão de Campos Experimentais da Embrapa Cenargen (CVGWeb)**.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [Autenticação](#2-autenticação)
3. [Autorização](#3-autorização)
4. [Rate limiting](#4-rate-limiting)
5. [Proteção contra injeção (SQL/LIKE)](#5-proteção-contra-injeção-sqllike)
6. [Proteção contra XSS](#6-proteção-contra-xss)
7. [Content Security Policy (CSP)](#7-content-security-policy-csp)
8. [Cabeçalhos de segurança HTTP](#8-cabeçalhos-de-segurança-http)
9. [CORS](#9-cors)
10. [Armazenamento no navegador](#10-armazenamento-no-navegador)
11. [Gestão de tokens e convites](#11-gestão-de-tokens-e-convites)
12. [Auditoria](#12-auditoria)
13. [Configurações sensíveis](#13-configurações-sensíveis)
14. [Medidas não implementadas (backlog)](#14-medidas-não-implementadas-backlog)
15. [Checklist de deploy](#15-checklist-de-deploy)

---

## 1. Visão geral

A segurança do CVGWeb é implementada em camadas complementares que protegem
tanto o back-end (Laravel 12) quanto o front-end (TypeScript + Vite):

| Camada | Componente | Proteção |
| --- | --- | --- |
| Transporte | HSTS, HTTPS | Man-in-the-middle |
| Cabeçalhos HTTP | SecurityHeaders middleware | Clickjacking, MIME sniffing |
| CSP | Meta tag no HTML | XSS, injeção de scripts |
| Autenticação | Sanctum 4 (token Bearer) | Acesso não autorizado |
| Autorização | Policies + perfil + permissões por projeto | Escalação de privilégio |
| Rate limiting | Dual-layer (IP + identificador) | Força bruta, DDoS |
| Injeção | Eloquent ORM + SanitizaBusca | SQL injection, LIKE injection |
| XSS | Template tag `html` com escape automático | Cross-site scripting |
| Tokens | SHA-256 + expiração 48h | Roubo de token de convite |
| Auditoria | Trilha completa com diff | Rastreabilidade |

---

## 2. Autenticação

### Mecanismo

O sistema usa **Laravel Sanctum 4** com tokens de API (Bearer tokens). Não
utiliza sessões baseadas em cookies — toda comunicação é stateless via
`Authorization: Bearer {token}`.

### Fluxo de login

```
Cliente                          Servidor
  │                                │
  ├── POST /login ──────────────> │  1. Valida nick + senha (bcrypt 12 rounds)
  │   { nick, senha }             │  2. Verifica se usuário está ativo
  │                                │  3. Verifica se senha foi definida
  │ <── 200 { token, user } ──── │  4. Revoga tokens anteriores (1 ativo por vez)
  │                                │  5. Cria novo token Sanctum
  │                                │  6. Registra auditoria + ultimo_acesso_em
  │                                │
  ├── GET /me ──────────────────> │  Header: Authorization: Bearer {token}
  │   (qualquer rota protegida)   │  Middleware: auth:sanctum
  │                                │
  ├── POST /logout ─────────────> │  Revoga o token atual
  │ <── 200 ────────────────────  │
```

### Primeiro acesso

Usuários são criados por administradores **sem senha**. O fluxo é:

1. Admin cadastra funcionário → sistema gera token de 64 caracteres aleatórios.
2. Token é armazenado como **hash SHA-256** (nunca em texto claro).
3. E-mail de convite é enviado pelo provedor configurado (SMTP em desenvolvimento), com link contendo o token.
4. Funcionário acessa link → sistema valida token + expiração (48h).
5. Funcionário define senha → token é marcado como usado, todas as sessões são revogadas.

### Hashing de senha

- Algoritmo: **bcrypt** com 12 rounds (padrão do Laravel).
- Senhas nunca trafegam em texto claro no banco.
- `password` é `nullable` — nulo indica convite pendente.

---

## 3. Autorização

### Camada 1: Perfil do funcionário

O campo `perfil` armazena tokens de permissão no formato `/ESCOPO/`:

| Perfil | Significado |
| --- | --- |
| `/SU/` | Super usuário — acesso total |
| `/FUNCIONARIO-I/` | Inserir funcionários |
| `/FUNCIONARIO-A/` | Alterar funcionários |
| `/FUNCIONARIO-E/` | Excluir funcionários |
| `/RESERVA-I/` | Inserir reservas |
| `/RESERVA-A/` | Aprovar/cancelar reservas |
| `/CASA-I/` | Inserir casas de vegetação |
| `/CASA-A/` | Alterar casas de vegetação |
| `/CASA-E/` | Excluir casas de vegetação |

### Camada 2: Permissões por projeto

O campo `permissoes` (JSON) na tabela `colaboradores_projeto` permite
controle granular por projeto:

```json
{
  "funcionario.criar": true,
  "funcionario.editar": false,
  "reserva.criar": true,
  "reserva.aprovar": false
}
```

### Policies

Cada modelo principal tem uma Policy registrada no Laravel:

| Policy | Modelo | Verifica |
| --- | --- | --- |
| `FuncionarioPolicy` | Funcionario | Perfil `/FUNCIONARIO-*/ ` + tipo `SU`/`ADMIN` |
| `ProjetoPolicy` | Projeto | Responsável ou colaborador com permissão |
| `ReservaPolicy` | Reserva | Perfil `/RESERVA-*/` + pertencer ao projeto |
| `CasaVegetacaoPolicy` | CasaVegetacao | Perfil `/CASA-*/` |
| `AuditoriaPolicy` | Auditoria | Apenas `SU` e `ADMIN` |

### Proteção nos Models

Todos os models usam `$fillable` (lista explícita de campos preenchíveis).
Nenhum model usa `$guarded = []`, que permitiria mass assignment.

---

## 4. Rate limiting

### Configuração

Três rate limiters definidos em `AppServiceProvider.php`:

#### Login (`throttle:login`)

Dual-layer para impedir força bruta:

```php
RateLimiter::for('login', function (Request $request) {
    $identificador = mb_strtolower(trim($request->input('nick')));
    return [
        Limit::perMinute(5)->by($request->ip()),           // máx 5/min por IP
        Limit::perMinute(5)->by('login|' . $identificador), // máx 5/min por nick
    ];
});
```

**Por que dual-layer:** o limite por IP impede varredura de senhas a partir de
um endereço; o limite por identificador impede que uma botnet distribua
tentativas contra a mesma conta.

#### Acesso público (`throttle:acesso`)

Rotas de esqueci-senha e definir-senha:

```php
RateLimiter::for('acesso', function (Request $request) {
    $email = mb_strtolower(trim($request->input('email')));
    return [
        Limit::perMinute(5)->by($request->ip()),          // impede varredura
        ...($email !== '' ? [Limit::perMinute(3)->by('acesso|' . $email)] : []),
    ];
});
```

**Por que 3 por e-mail:** impede que várias origens inundem a mesma caixa de
entrada com e-mails de recuperação.

#### Assistente (`throttle:assistente`)

```php
RateLimiter::for('assistente', fn ($request) =>
    Limit::perMinute(20)->by($request->user()?->getAuthIdentifier() ?? $request->ip())
);
```

Limita chamadas ao LLM (Groq) por usuário autenticado.

### Aplicação nas rotas

| Rota | Middleware |
| --- | --- |
| `POST /login` | `throttle:login` |
| `POST /esqueci-senha` | `throttle:acesso` |
| `GET /definir-senha/{token}` | `throttle:acesso` |
| `POST /definir-senha` | `throttle:acesso` |
| `POST /assistente` | `throttle:assistente` |

---

## 5. Proteção contra injeção (SQL/LIKE)

### SQL Injection

O sistema usa **Eloquent ORM** com query builder parametrizado em todas as
consultas. Nenhuma query é construída por concatenação de strings.

### LIKE Injection

O Laravel parametriza o valor do `LIKE`, mas os wildcards (`%`, `_`) são
interpretados pelo banco. A classe `SanitizaBusca` escapa esses caracteres:

```php
class SanitizaBusca
{
    public static function escapar(string $valor): string
    {
        return str_replace(
            ['\\', '%', '_'],
            ['\\\\', '\\%', '\\_'],
            $valor,
        );
    }
}
```

**Aplicada em:** `FuncionarioController`, `CasaVegetacaoController`,
`CargoController`, `ProjetoController` — todo controller que aceita busca textual.

**Uso:**
```php
$q->where('nome', 'like', '%' . SanitizaBusca::escapar($termo) . '%')
```

---

## 6. Proteção contra XSS

### Front-end: template tag `html`

O front-end usa uma tagged template literal (`html`) que escapa
automaticamente todas as interpolações:

```typescript
import { html } from '@/lib/html';

// Valores interpolados são escapados antes da inserção no DOM
element.innerHTML = html`<p>${userInput}</p>`;
```

Caracteres escapados: `&`, `<`, `>`, `"`, `'`.

### Back-end: respostas JSON

O back-end retorna exclusivamente JSON — nunca HTML renderizado no servidor.
Isso elimina XSS refletido no back-end, transferindo a responsabilidade de
renderização (com escape) para o front-end.

---

## 7. Content Security Policy (CSP)

CSP implementada via meta tag no `index.html` do front-end:

```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data: https://*.tile.openstreetmap.org;
               connect-src 'self' https:;
               font-src 'self';
               frame-ancestors 'none';
               base-uri 'self';
               form-action 'self';" />
```

### Diretivas explicadas

| Diretiva | Valor | Proteção |
| --- | --- | --- |
| `default-src` | `'self'` | Só carrega recursos do próprio domínio |
| `script-src` | `'self'` | Bloqueia scripts inline e de terceiros |
| `style-src` | `'self' 'unsafe-inline'` | Permite estilos inline (necessário para o mapa) |
| `img-src` | `'self' data: https://*.tile.openstreetmap.org` | Tiles do mapa + imagens locais |
| `connect-src` | `'self' https:` | API própria + conexões HTTPS (Groq LLM) |
| `font-src` | `'self'` | Apenas fontes locais (IBM Plex Sans) |
| `frame-ancestors` | `'none'` | Ninguém embute o sistema em iframe |
| `base-uri` | `'self'` | Impede injeção de `<base>` |
| `form-action` | `'self'` | Formulários só enviam para o próprio domínio |

---

## 8. Cabeçalhos de segurança HTTP

Middleware `SecurityHeaders` aplicado globalmente em todas as respostas do
back-end:

| Cabeçalho | Valor | Proteção |
| --- | --- | --- |
| `X-Content-Type-Options` | `nosniff` | Impede MIME type sniffing |
| `X-Frame-Options` | `DENY` | Impede clickjacking via iframe |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Não vaza path em cross-origin |
| `X-XSS-Protection` | `0` | Desliga filtro legado (criava mais brechas) |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | Desliga APIs desnecessárias |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Força HTTPS por 1 ano (só quando em HTTPS) |

---

## 9. CORS

Configurado em `config/cors.php` do Laravel:

- **Allowed origins:** apenas o `FRONTEND_URL` definido no `.env`.
- **Allowed methods:** GET, POST, PUT, PATCH, DELETE.
- **Credentials:** habilitado (para envio de cookies se necessário).
- **Max age:** pré-configurado para reduzir preflight requests.

O front-end envia `Authorization: Bearer {token}` em todas as requisições
autenticadas — sem cookies de sessão.

---

## 10. Armazenamento no navegador

### sessionStorage (dados sensíveis)

| Chave | Conteúdo | Ciclo de vida |
| --- | --- | --- |
| `cvgweb:token` | Token Bearer Sanctum | Morre ao fechar a aba |
| `cvgweb:sessao` | Dados do usuário logado (nome, tipo, perfil) | Morre ao fechar a aba |

**Por que sessionStorage:** o token de autenticação não sobrevive ao
fechamento da aba, impedindo roubo por outra aba ou extensão maliciosa que
leia o `localStorage`.

### localStorage (dados não sensíveis)

| Chave | Conteúdo | Ciclo de vida |
| --- | --- | --- |
| `cvgweb:rascunho:*` | Rascunhos de reserva em andamento | Expiração automática de 24h |

**Regras:**
- Rascunhos expiram automaticamente após 24 horas.
- Todos os rascunhos são limpos no logout.
- Não contém nenhum dado de autenticação.

---

## 11. Gestão de tokens e convites

### Tokens de convite/recuperação

| Propriedade | Valor |
| --- | --- |
| Tamanho | 64 caracteres aleatórios (`Str::random(64)`) |
| Armazenamento | Hash SHA-256 (nunca texto claro) |
| Validade | 48 horas |
| Uso único | Sim — `usado_em` é marcado |
| Invalidação | Emitir novo token invalida todos os anteriores não-usados |

### Tokens Sanctum

| Propriedade | Valor |
| --- | --- |
| Geração | Laravel Sanctum 4 |
| Um por vez | Sim — login revoga tokens anteriores |
| Revogação | Logout, definição de senha, desativação do usuário |
| Armazenamento | Tabela `personal_access_tokens` (hash SHA-256) |

---

## 12. Auditoria

### O que é registrado

Toda ação de escrita gera registro na tabela `auditorias`:

| Campo | Conteúdo |
| --- | --- |
| `funcionario_id` | Quem fez |
| `funcionario_nome` | Snapshot do nome (não depende de FK) |
| `acao` | `CRIOU`, `ALTEROU`, `EXCLUIU`, `LOGIN`, `LOGOUT` |
| `entidade` | Model afetado (ex: `Funcionario`, `Reserva`) |
| `entidade_id` | ID do registro |
| `rotulo` | Label legível (ex: nome da pessoa) |
| `alteracoes` | Diff JSON: `{ campo: { de: old, para: new } }` |
| `projeto_id` | Projeto relacionado (quando aplicável) |
| `ip` | Endereço IP da requisição |
| `created_at` | Data/hora |

### Quem pode ver

Apenas usuários com tipo `SU` ou `ADMIN` (controlado por `AuditoriaPolicy`).

### Índices de performance

- `(entidade, entidade_id)` — busca por registro.
- `created_at` — filtros por período.
- `funcionario_id` — filtros por autor.

---

## 13. Configurações sensíveis

### Variáveis de ambiente

| Variável | Proteção |
| --- | --- |
| `APP_DEBUG` | Deve ser `false` em produção (padrão no `.env.example`) |
| `APP_KEY` | Chave de criptografia do Laravel — nunca versionada |
| `DB_PASSWORD` | Senha do banco — nunca versionada |
| `RESEND_KEY` | Chave da API de e-mail — nunca versionada |
| `GROQ_API_KEY` | Chave do LLM — fica no servidor, nunca exposta ao front |

### Arquivos `.env`

O sistema tem **três** arquivos `.env` (raiz, backend, frontend), todos listados
no `.gitignore`. Os `.env.example` contêm apenas valores de desenvolvimento.

### Chave do LLM

A `GROQ_API_KEY` fica exclusivamente no back-end. O front-end se comunica com
o assistente via `/assistente` (rota autenticada + rate limited), sem acesso
direto à API do Groq.

---

## 14. Medidas não implementadas (backlog)

As medidas abaixo são recomendadas para implementação futura:

### Prioridade alta

| Medida | Descrição | Motivo |
| --- | --- | --- |
| **HTTPS obrigatório** | Configurar certificado TLS no servidor de produção | HSTS só funciona depois do primeiro acesso HTTPS |
| **Rotação de APP_KEY** | Procedimento documentado para rotação da chave | Em caso de comprometimento |
| **Backup automatizado** | Script de backup do MySQL com retenção | Recuperação de desastres |

### Prioridade média

| Medida | Descrição | Motivo |
| --- | --- | --- |
| **2FA (autenticação em dois fatores)** | TOTP via app autenticador | Proteção contra roubo de credenciais |
| **Política de senha** | Mínimo 12 caracteres, verificar HaveIBeenPwned | Senhas fracas |
| **Expiração de token Sanctum** | TTL de 8 horas nos tokens | Limitar janela de exposição |
| **Logging centralizado** | Enviar logs para serviço externo (ex: Papertrail) | Monitoramento e alertas |
| **Scan de dependências** | `composer audit` e `npm audit` em CI | Vulnerabilidades em pacotes |

### Prioridade baixa

| Medida | Descrição | Motivo |
| --- | --- | --- |
| **CSP via header** | Mover CSP da meta tag para cabeçalho HTTP | Permite `report-uri` para monitorar violações |
| **Subresource Integrity (SRI)** | Hashes nos scripts e estilos | Proteção contra CDN comprometida |
| **Security.txt** | `/.well-known/security.txt` | Canal de reporte de vulnerabilidades |
| **Honeypot em formulários** | Campo oculto para detectar bots | Reduzir spam em convites |

---

## 15. Checklist de deploy

Antes de publicar o sistema em produção, verificar:

- [ ] `APP_DEBUG=false` no `.env` do backend
- [ ] `APP_ENV=production`
- [ ] `APP_KEY` gerada e única para produção (`php artisan key:generate`)
- [ ] Certificado TLS/SSL configurado no servidor
- [ ] `FRONTEND_URL` correto no `.env` do backend (CORS)
- [ ] `VITE_API_URL` correto no `.env` do frontend
- [ ] Senha do super usuário `SU` trocada (padrão é `super123`)
- [ ] `DB_PASSWORD` forte e única
- [ ] `RESEND_KEY` configurada para o domínio de produção
- [ ] `GROQ_API_KEY` configurada
- [ ] `GROQ_LIMITE_POR_MINUTO` ajustado para produção
- [ ] Backup do banco configurado e testado
- [ ] Logs do Laravel direcionados para arquivo rotacionado
- [ ] Firewall: apenas portas 80/443 expostas
- [ ] MySQL: porta 3306 **não** exposta externamente
