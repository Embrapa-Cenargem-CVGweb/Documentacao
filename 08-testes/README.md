# Documentação de Testes — CVGWeb

Como o **Sistema de Gestão de Campos Experimentais da Embrapa Cenargen
(CVGWeb)** é testado: a suíte automatizada da API (PHPUnit) e os roteiros de
teste manual pela tela.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [Como executar](#2-como-executar)
3. [O que a suíte cobre](#3-o-que-a-suíte-cobre)
4. [Testes manuais](#4-testes-manuais)
5. [Plano de expansão](#5-plano-de-expansão)

---

## 1. Visão geral

| Suíte | Onde | O que cobre | Quantos |
| :--- | :--- | :--- | :--- |
| **PHPUnit** (API) | `Embrapa-backend/tests/Feature` | Regras da API, de ponta a ponta dentro do back-end: requisição HTTP → validação → permissão → banco | 153 testes em 24 arquivos |
| **Manual** | esta página, seção 4 | O que depende de tela, e-mail de verdade ou leitor de tela | 10 roteiros |

Princípios:

- **Cada teste prepara o próprio banco** (`RefreshDatabase`) e não depende de
  outro.
- **Nunca toca no banco de uso:** no SQLite é um banco em memória; no MySQL é
  sempre o `cvg_testes`, com o nome fixo na configuração.
- **E-mail nunca sai:** `MAIL_MAILER=array` forçado no `phpunit.xml`.

---

## 2. Como executar

### PHPUnit — no SQLite e no MySQL

```bash
cd Embrapa-backend
php artisan test                              # SQLite em memória (rápido, o padrão)
php vendor/bin/phpunit -c phpunit.mysql.xml   # os mesmos testes num MySQL/MariaDB de verdade
docker compose exec backend php artisan test  # dentro do Docker
```

O SQLite deixa passar coisas que o MySQL recusa (modo estrito,
`ONLY_FULL_GROUP_BY`, JSON, o id 0 do adm). Antes de mexer em consulta ou
migration, rode também o `phpunit.mysql.xml`. Ele usa host, porta, usuário e
senha do `.env`, mas o banco é sempre `cvg_testes` — o `RefreshDatabase` apaga
tudo nele, então crie um banco só para isso:

```sql
CREATE DATABASE cvg_testes CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON cvg_testes.* TO 'cvgweb'@'localhost';
```

Os 153 testes passam no SQLite, no MySQL 8.4 instalado no Windows, no MySQL
8.0 e no MariaDB 10.4. Testes que dependem do banco:

- `SuperUsuarioIdZeroTest` — no MySQL o `AUTO_INCREMENT` não volta quando o
  teste anterior desfaz a transação; o teste confere o número exato só no
  SQLite e, no MySQL, que a primeira pessoa não caiu no id 0.
- `DemoContasTest` — sete logins seguidos batem no limite de tentativas; o
  teste zera o contador entre uma conta e outra.

---

## 3. O que a suíte cobre

| Área | Arquivos de teste |
| :--- | :--- |
| Login, sessão e senha | `LoginGenerico`, `SessaoSeguranca`, `SuperUsuarioIdZero` |
| Cadastro de usuário e segurança do e-mail | `CadastroPorCargo`, `FuncionarioEmailSeguranca`, `ConviteEnvio` |
| Permissões | `PermissoesGlobais`, `PermissoesProjeto`, `Lideranca` |
| Projetos | `ProjetoCriacao`, `ProjetoDocumento`, `ConviteProjeto` |
| Reservas e capacidade | `ReservaCapacidade`, `ReservaBancadas`, `ReservaEdicao`, `Calendario` |
| Estufas | `CasaTemperaturaVasos`, `CasaHistorico` |
| Sino de avisos | `Notificacao`, `SinoFluxos` |
| Auditoria (logs) | `Auditoria`, `AuditoriaCobertura`, `AuditoriaResumo` |
| Dados fictícios | `DemoContas` (todas as contas entram e veem o que devem, nada duplica, recusa em produção) |

---

## 4. Testes manuais

Os cenários abaixo não estão automatizados e devem ser executados manualmente em
cada release ou mudança significativa.

### TM-01: Fluxo de convite e-mail → definição de senha

| Item | Detalhe |
| --- | --- |
| **Pré-condição** | Serviço de e-mail configurado e acessível (ver 9. Operação) |
| **Passos** | 1. Admin cadastra funcionário com e-mail válido. 2. Funcionário recebe e-mail com link. 3. Funcionário acessa link. 4. Tela de definição de senha carrega com token válido. 5. Funcionário define senha (mín. 8 caracteres). 6. Funcionário faz login com a senha definida. |
| **Resultado esperado** | Login bem-sucedido; `senha_definida_em` preenchido no banco |

### TM-02: Recuperação de senha ("Esqueci minha senha")

| Item | Detalhe |
| --- | --- |
| **Pré-condição** | Usuário com senha já definida |
| **Passos** | 1. Na tela de login, clicar "Esqueci minha senha". 2. Informar o e-mail cadastrado. 3. Receber e-mail com link (validade 48h). 4. Acessar link e definir nova senha. 5. Fazer login com a nova senha. |
| **Resultado esperado** | Login com nova senha; token anterior invalidado; todas as sessões revogadas |

### TM-03: Token expirado

| Item | Detalhe |
| --- | --- |
| **Pré-condição** | Link de convite ou recuperação com mais de 48h |
| **Passos** | 1. Acessar link expirado. 2. Tentar definir senha. |
| **Resultado esperado** | Mensagem clara de token expirado; orientação para solicitar novo link |

### TM-04: Acessibilidade por teclado

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Navegar pelo sistema usando apenas Tab, Shift+Tab, Enter e Escape. 2. Verificar que todos os formulários são acessíveis. 3. Verificar que modais prendem o foco (focus trap). 4. Verificar que Escape fecha modais. |
| **Resultado esperado** | Navegação completa sem mouse; foco visível em todos os elementos interativos; ARIA roles corretos |

### TM-05: Leitor de tela

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Ativar NVDA ou VoiceOver. 2. Navegar pelas telas principais. 3. Verificar que tabelas, botões e modais são anunciados corretamente. |
| **Resultado esperado** | Todos os elementos têm labels (`aria-label`, `aria-labelledby`); tabelas usam `role="grid"` com `aria-checked` nas linhas |

### TM-06: Responsividade

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Acessar o sistema em viewport de 360px (mobile). 2. Acessar em 768px (tablet). 3. Acessar em 1280px (desktop). |
| **Resultado esperado** | Tabelas adaptam para layout de cards; touch targets ≥ 44px; mapa e marcadores funcionais |

### TM-07: Assistente (chatbot Groq)

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Abrir o chat no mapa. 2. Enviar pergunta sobre uma estufa. 3. Verificar que a resposta é relevante. 4. Mudar para outra tela e confirmar que o chat sumiu. |
| **Resultado esperado** | Resposta coerente do LLM; chat visível apenas no mapa |

### TM-08: Auditoria

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Criar, editar e excluir um espaço de teste. 2. Criar e cancelar uma reserva. 3. Acessar a tela de auditoria. 4. Filtrar por entidade e por período. |
| **Resultado esperado** | Cada ação registrada com autor, data, IP e diff (campo `alteracoes` mostrando "de" e "para") |

### TM-09: Rate limiting

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Tentar login errado 5 vezes seguidas. 2. Na 6ª tentativa, observar resposta HTTP 429. 3. Aguardar 1 minuto e tentar novamente. |
| **Resultado esperado** | Bloqueio temporário com mensagem "Too Many Attempts"; acesso restaurado após o período |

### TM-10: Permissões por projeto

| Item | Detalhe |
| --- | --- |
| **Passos** | 1. Criar projeto com colaborador com permissão apenas de `reserva.criar`. 2. Logar como o colaborador. 3. Tentar criar reserva (deve funcionar). 4. Tentar criar funcionário (deve ser negado). |
| **Resultado esperado** | Ação permitida executa; ação não permitida retorna 403 |

---

---

## 5. Plano de expansão

| Alvo | Como |
| :--- | :--- |
| `POST /reservas/lote` (vasos e bancadas num envio) | PHPUnit feature |
| Limites exatos de tentativa (login, acesso, assistente) | PHPUnit |
| Rodar o `phpunit.mysql.xml` no CI, além do SQLite | GitHub Actions com serviço MySQL |
| Validações do front (TypeScript) | Vitest |
