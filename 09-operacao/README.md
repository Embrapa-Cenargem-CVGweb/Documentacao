# 9. Operação

Como o CVGWeb sobe, o que cada variável de ambiente controla, onde ficam os
dados e o que fazer quando algo não funciona.

> O passo a passo de instalação está no
> [README da raiz](../../README.md). Esta página é a referência de operação.

---

## Índice

1. [Topologia](#topologia)
2. [Os três `.env`](#os-três-env)
3. [Variáveis de ambiente](#variáveis-de-ambiente)
4. [E-mail](#e-mail)
5. [Assistente](#assistente)
6. [Comandos do dia a dia](#comandos-do-dia-a-dia)
7. [Onde ficam os dados](#onde-ficam-os-dados)
8. [Backup e restauração](#backup-e-restauração)
9. [Schema do banco](#schema-do-banco)
10. [Problemas comuns](#problemas-comuns)

---

## Topologia

Um único `docker compose`, na raiz do repositório de orquestração, sobe tudo:

| Serviço | Imagem | Endereço |
| :--- | :--- | :--- |
| Interface | `node:22-alpine` + Vite | http://localhost:5173 |
| API | build do `Embrapa-backend/Dockerfile` | http://localhost:8000/api |
| MySQL | `mysql:8.4` | `localhost:3307` (usuário `cvgweb`) |

```bash
docker compose up -d --build
```

No primeiro boot a API instala as dependências do Composer, roda as migrations
e os seeders (cargos, status de projeto, as 51 casas de vegetação e o super
usuário); o front roda `npm install`.

**Primeiro acesso:** e-mail `admin@cvgweb.local`, senha `supersu` — e troque em seguida. É a
única conta criada pelo seeder; as demais são cadastradas pela tela e recebem
link de convite por e-mail.

---

## Os três `.env`

| Arquivo | Quem lê | Quando |
| :--- | :--- | :--- |
| `./.env` | `docker compose` | Sempre que se usa Docker. Injeta tudo nos containers |
| `Embrapa-backend/.env` | `php artisan` | **Só sem Docker.** No Docker é reescrito a cada boot pelo entrypoint — não edite lá |
| `Embrapa-frontend/.env` | `npm run dev` | **Só sem Docker.** Tem uma variável: `VITE_API_URL` |

---

## Variáveis de ambiente

| Variável | Para que serve |
| :--- | :--- |
| `APP_KEY` | Chave de criptografia do Laravel. **Obrigatória** — sem ela o login falha ao assinar o token |
| `APP_DEBUG` | `false` em produção. Nunca `true` com dado real |
| `APP_URL` | Endereço público da API (padrão `http://localhost:8000`) |
| `COMPOSE_PROJECT_NAME` | Nome do projeto no Docker (`cvgweb`). Não mude: os contêineres e volumes são achados por ele |
| `APP_TIMEZONE` | Fuso das datas da API (padrão `America/Sao_Paulo`) |
| `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | Acesso ao MySQL |
| `DB_ROOT_PASSWORD` | Senha do root do MySQL do contêiner. Troque fora do desenvolvimento |
| `DB_PORT_HOST` | Porta do MySQL no host (3307, para não brigar com um MySQL ou XAMPP já instalado) |
| `BACKEND_PORT` / `FRONTEND_PORT` | Portas publicadas da API e do front |
| `FRONTEND_URL` | Origem liberada no CORS da API |
| `VITE_API_URL` | Endereço que o **navegador** usa para chamar a API |
| `RUN_MIGRATIONS` / `RUN_SEED` | Migrations e seeders no boot do container. Pode deixar `true`: são idempotentes |
| `MAIL_*` | Envio do convite e do "esqueci a senha" — ver abaixo |
| `MAIL_TIMEOUT` | Segundos antes de desistir da conexão SMTP (padrão 5). O padrão do PHP é 60, e nesse tempo o cadastro de usuário fica esperando |
| `GROQ_API_KEY` / `GROQ_MODEL` / `GROQ_LIMITE_POR_MINUTO` | Assistente. **Só no back-end** |
| `CVGWEB_INTERVALO_REENVIO` | Segundos de espera antes de reenviar convite para a mesma pessoa (padrão 60) |
| `CVGWEB_LINK_MANUAL` | Quando o e-mail não sai, devolve o link ao administrador na resposta |
| `CVGWEB_LIMITE_ESPACOS` | Limite de casas de vegetação cadastradas (padrão 51) |
| `CVGWEB_SU_EMAIL` / `CVGWEB_SU_SENHA` | E-mail e senha com que o super usuário é **criado** no primeiro boot (padrão `admin@cvgweb.local` / `supersu`). Não alteram uma conta que já existe: depois, a senha se troca pela tela |

> Gere a `APP_KEY` com
> `docker compose run --rm backend php artisan key:generate --show` e cole no
> `.env` da raiz.

---

## E-mail

**Todo cadastro de senha passa pelo e-mail.** Não existe tela de cadastro nem
senha provisória — o fluxo está descrito em
[2. Arquitetura](../02-arquitetura/README.md#26-primeiro-acesso-convite-por-e-mail).

### Desenvolvimento — SMTP do Gmail

```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_SCHEME=smtp
MAIL_USERNAME=<conta@gmail.com>
MAIL_PASSWORD=<senha de app, 16 caracteres>
MAIL_FROM_ADDRESS=<conta@gmail.com>
MAIL_FROM_NAME="CVGWeb - Embrapa Cenargen"
```

A senha é uma **senha de app** do Google (conta com verificação em duas etapas
→ *Senhas de app*), nunca a senha da conta. Entrega para qualquer destinatário,
o que resolve o teste do convite com e-mail de verdade.

### Quando a rede bloqueia a porta de saída

Muita rede corporativa e muito provedor doméstico bloqueiam a saída na porta
**587**. O sintoma é `Connection timed out` no `laravel.log` e o cadastro de
usuário parecendo travado — ele não travou, está esperando o socket desistir.

Duas saídas:

1. **Porta 465 (SSL)**, que costuma passar onde a 587 não passa:

   ```env
   MAIL_PORT=465
   MAIL_SCHEME=smtps
   ```

2. **Modo log** (abaixo), que é o caminho normal de desenvolvimento.

Em qualquer caso, `MAIL_TIMEOUT=5` mantém a tela respondendo em segundos: o
convite já foi gravado e o administrador recebe o link na própria resposta,
num diálogo com botão de copiar.

### Sem enviar nada

```env
MAIL_MAILER=log
```

O link vai para `Embrapa-backend/storage/logs/laravel.log`. Em `APP_ENV=local`
ele também volta na resposta da API de cadastro/reenvio, para testar sem caixa
de entrada.

### Produção

O caminho é um remetente de domínio verificado da Embrapa
(ex.: `cvgweb@cenargen.embrapa.br`), por SMTP institucional ou por provedor de
API. O código traz um transport próprio para a Resend
(`app/Mail/Transport/ResendTransport.php`), usável com `MAIL_MAILER=resend` e
`RESEND_API_KEY` — mas, **sem domínio verificado**, o remetente
`onboarding@resend.dev` só entrega para o e-mail da própria conta Resend — por
isso, em desenvolvimento, use SMTP.

Teste o envio sem mexer na tela:

```bash
docker compose exec backend php artisan cvgweb:testar-email alguem@exemplo.com
```

Depois de mexer no `.env`, reinicie a API: `docker compose up -d backend`.

---

## Assistente

O navegador faz `POST /api/assistente` (autenticado, limitado a
`GROQ_LIMITE_POR_MINUTO` perguntas por usuário) e o Laravel fala com a Groq.
Sem `GROQ_API_KEY` o widget aparece, mas avisa que não está configurado.
Deixe `GROQ_MODEL` vazio para o back-end escolher o melhor modelo que a chave
alcança.

> Não existe nenhuma variável `VITE_GROQ_*`. Tudo que começa com `VITE_` é
> embutido no JavaScript publicado e fica legível para qualquer visitante.

---

## Comandos do dia a dia

```bash
docker compose up -d              # sobe tudo (banco intacto)
docker compose down               # para tudo, sem perder o banco
docker compose down -v            # para tudo e APAGA o banco
docker compose logs -f backend    # acompanha a API
docker compose restart backend    # recarrega a API depois de mexer no .env
docker compose exec backend bash  # um shell dentro da API
```

Artisan e testes:

```bash
docker compose exec backend php artisan migrate
docker compose exec backend php artisan db:seed
docker compose exec backend php artisan test

docker compose exec backend php artisan cvgweb:senha --listar
docker compose exec backend php artisan cvgweb:senha admin@cvgweb.local novasenha
docker compose exec backend php artisan cvgweb:limpar-reservas --espaco=CVG01
```

---

## Onde ficam os dados

Com Docker, tudo em volumes de nome fixo — nada do banco toca a pasta do
projeto:

| Volume | Guarda |
| :--- | :--- |
| `cvgweb_mysql_data` | O banco inteiro |
| `cvgweb_back_vendor` | Dependências do Composer |
| `cvgweb_front_node_modules` | Dependências do npm |

Ligar e desligar os containers não perde nada; só o `-v` destrói. Migrations e
seeders rodam a cada boot, mas são idempotentes: `migrate` aplica só o que
falta e os seeders usam `firstOrCreate`. O que foi cadastrado pela tela não é
tocado.

---

## Backup e restauração

```bash
# salvar
docker compose exec mysql mysqldump -ucvgweb -pcvgweb embrapa_cvg > backup.sql

# restaurar (aqui ou em outro servidor)
docker compose exec -T mysql mysql -ucvgweb -pcvgweb embrapa_cvg < backup.sql
```

Quando o banco definitivo existir, as migrations montam a estrutura lá e o dump
leva o conteúdo.

---

## Schema do banco

O schema é criado e versionado pelas **migrations**, em
`Embrapa-backend/database/migrations`. Para mudar o banco, crie uma migration
nova; o `php artisan migrate` (que o contêiner roda a cada boot) aplica só o
que falta.

### Upload de documentos

Os documentos dos projetos ficam em
`Embrapa-backend/storage/app/private/projetos/`, fora da pasta pública —
inclua essa pasta no backup. A API aceita até 10 MB por arquivo; a imagem do
back-end sobe com `upload_max_filesize=12M` e `post_max_size=16M`
(`Dockerfile`). Depois de atualizar o código, reconstrua a imagem para o
limite valer: `docker compose up -d --build backend`.

Estrutura detalhada: [5. Banco de dados](../05-banco-de-dados/README.md),
incluindo os bancos testados fora do Docker e o que muda neles.

### Dados fictícios

Os dados fictícios (contas de exemplo, projetos, reservas, convites e 30 dias
de logs) ficam **só na branch `apresentacao`** do back-end e do front-end. A
`main` (no back-end, `master`) começa com o banco limpo: cargos, status, as 51
estufas e o super usuário. Para apresentar:

```bash
git -C Embrapa-backend checkout apresentacao
git -C Embrapa-frontend checkout apresentacao
docker compose up -d --build
docker compose exec backend php artisan db:seed --class=DemoSeeder
```

As contas e o que cada uma demonstra estão no README do back-end, na branch
`apresentacao`. Com `APP_ENV=production` o seeder se recusa a rodar.

---

## Problemas comuns

| Sintoma | O que é |
| :--- | :--- |
| Login responde "Unsupported cipher" ou erro 500 | `APP_KEY` vazia no `.env`. Gere com `key:generate --show` e reinicie a API |
| A tela abre mas nada carrega, e o console mostra erro de CORS | `FRONTEND_URL` não bate com o endereço de onde o front está sendo servido |
| O mapa sobe vazio e o login recusa tudo | `VITE_API_URL` em branco ou API fora do ar. Confira `docker compose logs backend` |
| `docker compose up` falha dizendo que a porta está em uso | Outro MySQL, Apache ou XAMPP ocupa a porta. Mude `DB_PORT_HOST`, `BACKEND_PORT` ou `FRONTEND_PORT` |
| O convite de acesso não chega | `MAIL_MAILER` ainda é `log` (o link está no `laravel.log`), ou o remetente não tem domínio verificado |
| O cadastro de usuário fica "travado" ao salvar | A porta SMTP está bloqueada na rede. O usuário **foi** cadastrado; confira `Connection timed out` no `laravel.log`. Use `MAIL_TIMEOUT=5`, a porta 465 ou `MAIL_MAILER=log` |
| Gmail recusa o envio com erro de autenticação | A senha precisa ser uma **senha de app**, não a senha da conta |
| **Sem Docker:** `SQLSTATE[HY000] [1045] Access denied` | Usuário/senha do `.env` não batem com o MySQL instalado, ou o usuário foi criado só para `'localhost'` e o `.env` usa `127.0.0.1` (ou o contrário). Crie para os dois ou troque o `DB_HOST` |
| **Sem Docker:** `SQLSTATE[HY000] [2002]` | MySQL instalado não está rodando, ou a porta é outra (o do Docker é 3307, o instalado costuma ser 3306) |
| **Sem Docker:** acentos viram `Conclu�do` no terminal | Só o console do Windows. O dado está certo; use `mysql --default-character-set=utf8mb4` |
| **Sem Docker:** o super usuário foi parar no id 1 | O banco foi montado por uma ferramenta que não ligou `NO_AUTO_VALUE_ON_ZERO`. Monte o banco com `php artisan migrate --seed`; a migration `2026_11_13_000001` coloca o adm no id 0 num banco que o tenha em outro id |
| **Sem Docker:** toda tela demora quase meio segundo para responder | OPcache desligado: o PHP recompila o Laravel a cada chamada. No `php.ini`, ligue `zend_extension=opcache`, `opcache.enable=1` e `opcache.enable_cli=1` (com ele cada chamada cai para centésimos de segundo) |
| **Sem Docker:** a API fica lenta com várias telas abertas | No Windows o `php artisan serve` atende uma requisição por vez (`PHP_CLI_SERVER_WORKERS` só funciona em Linux/macOS). Use o Docker, ou Nginx/Apache + PHP-FPM |
| O banco "sumiu" depois de renomear a pasta | Os volumes têm nome fixo (`cvgweb_*`). Confira se o `.env` ainda tem `COMPOSE_PROJECT_NAME=cvgweb` |
