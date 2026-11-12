# Documentação de Testes — CVGWeb

Documentação técnica da suíte de testes end-to-end (E2E) do **Sistema de Gestão
de Campos Experimentais da Embrapa Cenargen (CVGWeb)**.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [Tecnologias e dependências](#2-tecnologias-e-dependências)
3. [Como executar](#3-como-executar)
4. [Arquitetura dos testes](#4-arquitetura-dos-testes)
5. [Cenários de teste](#5-cenários-de-teste)
6. [Helpers e utilitários](#6-helpers-e-utilitários)
7. [Convenções de segurança](#7-convenções-de-segurança)
8. [Relatórios e artefatos de falha](#8-relatórios-e-artefatos-de-falha)
9. [Cenários de teste manuais](#9-cenários-de-teste-manuais)
10. [Plano de expansão](#10-plano-de-expansão)

---

## 1. Visão geral

O CVGWeb possui uma suíte de **testes de ponta a ponta** que exercita o sistema
como um usuário real: entra pela tela de login, clica nos elementos, preenche
formulários e confere que os dados persistem no banco de dados — não apenas na
memória do navegador.

### Princípios

- **Ponta a ponta:** cada teste cobre front-end, API e banco de dados juntos.
- **Independência:** cada cenário prepara seu próprio estado e não depende de
  nenhum outro teste.
- **Segurança do cadastro real:** dados de teste usam o prefixo `E2E` e nunca
  alteram o cadastro das estufas reais do Cenargen.
- **Execução sequencial:** os testes rodam em série (`workers: 1`) porque o
  estado do banco é compartilhado.

---

## 2. Tecnologias e dependências

| Componente | Tecnologia | Versão |
| --- | --- | --- |
| Framework de teste | Playwright | 1.63.0 |
| Linguagem | TypeScript | |
| Navegador | Chromium (headless) | Via Playwright |
| Reporter | list (terminal) + HTML (CI) | |

### Artefatos em caso de falha

| Artefato | Configuração |
| --- | --- |
| Screenshot | `only-on-failure` |
| Vídeo | `retain-on-failure` |
| Trace | `retain-on-failure` |

---

## 3. Como executar

### Pré-requisitos

1. O sistema deve estar rodando (Docker ou manual):
   ```bash
   docker compose up -d
   ```
2. O front-end deve estar servindo localmente:
   ```bash
   cd Embrapa-frontend
   npm run dev
   ```

### Comandos

```bash
cd Embrapa-frontend

# Rodar todos os testes
npm run test:e2e

# Rodar com UI interativa do Playwright
npm run test:e2e:ui

# Ver relatório HTML da última execução
npm run test:e2e:report
```

### Variáveis de ambiente

| Variável | Padrão | Descrição |
| --- | --- | --- |
| `CVG_URL` | `http://localhost:5173` | Endereço do front-end |
| `CVG_API` | `http://localhost:8000/api` | Endereço da API |
| `CVG_NICK` | `SU` | Usuário para login nos testes |
| `CVG_SENHA` | `super123` | Senha do usuário |
| `CVG_RUN` | `T1` | Identificador da execução (prefixo dos dados criados) |
| `CVG_ESTUFA` | `CVG50` | Código da estufa real usada para reservas |

### Configuração Playwright

```typescript
// playwright.config.ts — resumo
{
  testDir: './tests/e2e',
  fullyParallel: false,       // sequencial por necessidade
  workers: 1,                  // uma thread de teste
  retries: 0,                  // sem retry automático
  timeout: 60_000,             // 60s por teste
  expect: { timeout: 10_000 }, // 10s para assertions
  use: {
    baseURL: 'http://localhost:5173',
    locale: 'pt-BR',
    timezoneId: 'America/Sao_Paulo',
  }
}
```

---

## 4. Arquitetura dos testes

### Estrutura de arquivos

```
tests/
└── e2e/
    ├── .auth/
    │   └── sessao.json         ← sessão compartilhada (gerada)
    ├── sessao.setup.ts         ← login único, reaproveitado
    ├── cvgweb.spec.ts          ← todos os cenários E2E
    └── apoio.ts                ← helpers, localizadores, API calls
```

### Fluxo de execução

```
┌──────────────────────┐
│  sessao.setup.ts     │   1. Faz login com SU/super123
│  (projeto "sessão")  │   2. Salva storageState em .auth/sessao.json
└──────────┬───────────┘
           │ depende
           ▼
┌──────────────────────┐
│  cvgweb.spec.ts      │   Todos os testes reusam a sessão
│  (projeto "chromium") │   Cada teste monta e limpa seu cenário
└──────────────────────┘
```

O login é feito uma única vez no projeto "sessão". Isso evita estourar o rate
limiter da rota `/login` (que limita tentativas por IP) e acelera a suíte.

### Como cada teste funciona

1. **Prepara o cenário** — pela API, cria estufas e reservas com prefixo `E2E`.
2. **Exercita pela tela** — preenche formulários, clica botões, navega.
3. **Confere na tela** — verifica que a informação aparece corretamente.
4. **Confere pela API** — faz `GET` na API para validar que o banco gravou.
5. **Recarrega a página** — prova que o dado não estava só no navegador.
6. **Limpa** — cancela reservas e inativa estufas de teste.

---

## 5. Cenários de teste

### CT-01: Carregamento inicial

| Item | Detalhe |
| --- | --- |
| **Teste** | `entra no sistema e carrega os espaços do banco` |
| **Objetivo** | Verificar que o login funciona e os marcadores do mapa aparecem |
| **Passos** | Login → verifica marcadores → consulta API para confirmar |
| **Validação** | Marcadores visíveis no mapa; API retorna espaços com `data.length > 0` |

### CT-02: Cadastro de espaço

| Item | Detalhe |
| --- | --- |
| **Teste** | `cadastra um espaço pela tela e ele fica gravado` |
| **Objetivo** | Cadastrar uma nova estufa e confirmar persistência |
| **Passos** | Admin → abrir modal → preencher código, descrição, tipo, capacidade → enviar |
| **Validação** | Modal fecha; aviso com código; tabela mostra "20 de 20 vasos livres"; API confirma dados |

### CT-03: Alteração de espaço (código existente)

| Item | Detalhe |
| --- | --- |
| **Teste** | `código que já existe passa a ALTERAR aquele espaço, não duplicar` |
| **Objetivo** | Garantir que inserir código existente edita em vez de duplicar |
| **Passos** | Admin → abrir modal → digitar código existente → alterar descrição e capacidade → enviar |
| **Validação** | Formulário detecta modo "alterar"; contagem total de espaços não muda; dados atualizados no banco |

### CT-04: Reserva pela tela

| Item | Detalhe |
| --- | --- |
| **Teste** | `reserva parte da capacidade pela tela e o resto continua disponível` |
| **Objetivo** | Reservar vasos numa estufa real e verificar cálculo de disponibilidade |
| **Passos** | Mapa → abrir estufa CVG50 → reservar 10 vasos por 7 dias → confirmar |
| **Validação** | Disponibilidade reduz em 10; reserva gravada no banco com quantidade correta; cleanup no final |

### CT-05: Persistência da reserva

| Item | Detalhe |
| --- | --- |
| **Teste** | `a reserva sobrevive ao recarregamento (está no banco, não no navegador)` |
| **Objetivo** | Provar que os dados estão no banco de dados, não no localStorage |
| **Passos** | Criar reserva pela API → recarregar página → verificar → recarregar novamente → verificar |
| **Validação** | Capacidade mostra "12 de 20 vasos livres" após dois recarregamentos |

### CT-06: Limite de capacidade

| Item | Detalhe |
| --- | --- |
| **Teste** | `não deixa reservar mais do que sobrou` |
| **Objetivo** | Validar que a regra de negócio de capacidade é cumprida |
| **Passos** | Criar estufa com 20 vasos → reservar os 20 → tentar reservar mais 5 |
| **Validação** | API retorna 422; mensagem menciona "capacidade cheia"; tela mostra "0 de 20 vasos livres" |

### CT-07: Espaço em manutenção

| Item | Detalhe |
| --- | --- |
| **Teste** | `espaço em manutenção não aceita reserva` |
| **Objetivo** | Verificar que estufas em manutenção bloqueiam novas reservas |
| **Passos** | Criar estufa → mudar status para MANUTENCAO pela tela → tentar reservar pela API |
| **Validação** | Status gravado como "MANUTENCAO" no banco; API retorna 422 com motivo |

### CT-08: Proteção de capacidade em uso

| Item | Detalhe |
| --- | --- |
| **Teste** | `não reduz a capacidade abaixo do que já está reservado` |
| **Objetivo** | Impedir redução de capacidade que tornaria reservas existentes inválidas |
| **Passos** | Criar estufa com 20 vasos → reservar 12 → tentar reduzir capacidade para 5 |
| **Validação** | Modal mostra erro; capacidade permanece 20 no banco |

### CT-09: Painel de métricas

| Item | Detalhe |
| --- | --- |
| **Teste** | `o painel mostra os números que a API calcula` |
| **Objetivo** | Confirmar que o painel usa dados do back-end, não cálculos locais |
| **Passos** | Admin → abrir modal de métricas → ler total, livres e manutenção → comparar com `/painel` |
| **Validação** | Valores da tela coincidem exatamente com os da API; total ≥ 50 |

### CT-10: Cadastro de usuário (convite)

| Item | Detalhe |
| --- | --- |
| **Teste** | `cadastra usuário sem senha e com convite pendente` |
| **Objetivo** | Criar usuário e verificar fluxo de convite |
| **Passos** | Usuários → novo usuário → preencher nome e e-mail → enviar |
| **Validação** | Aviso menciona "48 horas" ou "link"; banco marca `CONVITE_PENDENTE`; lista mostra "Sem senha" |

### CT-11: Login sem senha definida

| Item | Detalhe |
| --- | --- |
| **Teste** | `login é recusado para quem ainda não criou a senha` |
| **Objetivo** | Impedir acesso antes do convite ser aceito |
| **Passos** | Criar usuário sem senha → limpar sessão → tentar login |
| **Validação** | Erro visível na tela mencionando "convite"; app não carrega |

### CT-12: Assistente (chatbot)

| Item | Detalhe |
| --- | --- |
| **Teste** | `o assistente aparece no mapa e em nenhuma outra tela` |
| **Objetivo** | Verificar visibilidade do assistente por tela |
| **Passos** | Mapa (visível) → Admin (oculto) → Reservas (oculto) → Usuários (oculto) → Mapa (visível) |
| **Validação** | `#chat` visível apenas na tela do mapa |

### CT-13: Aprovação e cancelamento de reserva

| Item | Detalhe |
| --- | --- |
| **Teste** | `aprovar e cancelar reserva muda o banco e devolve a capacidade` |
| **Objetivo** | Testar o ciclo completo PENDENTE → ATIVA → CANCELADA |
| **Passos** | Criar reserva → aprovar pela tela → verificar banco → cancelar → verificar capacidade restaurada |
| **Validação** | Após aprovação: status `ATIVA` com `aprovada_em` preenchido. Após cancelamento: capacidade volta a 20/20; reserva aparece na lista de canceladas |

### CT-14: Limpeza

| Item | Detalhe |
| --- | --- |
| **Teste** | `remove o que a suíte criou` |
| **Objetivo** | Garantir que a suíte não deixa lixo no banco |
| **Passos** | Cancela reservas E2E → remove/inativa espaços E2E → remove funcionário de teste |
| **Validação** | Nenhuma reserva aberta com marca `E2E`; espaços de teste inativos |

---

## 6. Helpers e utilitários

O arquivo `apoio.ts` contém toda a infraestrutura compartilhada pelos testes.

### Sessão e navegação

| Função | Descrição |
| --- | --- |
| `entrar(page)` | Faz login se ainda não estiver autenticado |
| `abrirSistema(page)` | Navega para `/`, faz login e arma o coletor de avisos |
| `recarregar(page)` | Recarrega a página inteira (testa persistência) |
| `irParaAdmin(page)` | Navega para a aba de administração |
| `irParaReservas(page)` | Navega para a aba de reservas |
| `irParaUsuarios(page)` | Navega para a aba de usuários |
| `irParaMapa(page)` | Navega para a aba do mapa |

### Avisos (toasts)

| Função | Descrição |
| --- | --- |
| `coletarAvisos(page)` | Injeta observer que captura textos dos toasts em `window.__avisos` |
| `avisos(page)` | Retorna array com textos coletados |
| `limparAvisos(page)` | Zera o array de avisos |
| `esperarAviso(page, trecho)` | Espera (até 15s) um aviso que contenha o trecho |

### Localizadores

| Função | Descrição |
| --- | --- |
| `linhaDoEspaco(page, codigo)` | Localiza a linha da tabela pelo código da estufa |
| `capacidadeDoEspaco(page, codigo)` | Localiza a célula de capacidade |
| `nomeDoEspaco(page, codigo)` | Retorna o nome exibido na tabela |
| `abrirEspacoNoMapa(page, nome)` | Clica no marcador do mapa e abre o painel |
| `metrica(page, id)` | Lê um número do painel de métricas |

### API (preparar cenário e conferir resultado)

| Função | Descrição |
| --- | --- |
| `pelaApi<T>(page, caminho)` | GET na API com token da sessão |
| `espacoPorCodigo(page, codigo)` | Busca um espaço pelo código |
| `garantirEspacoDeTeste(page, codigo, cap)` | Cria ou ajusta espaço de teste (prefixo `E2E`) |
| `disponibilidade(page, codigo)` | Consulta disponibilidade de hoje |
| `reservarPelaApi(page, codigo, qtd)` | Cria reserva para montar cenário |
| `tentarReservarPelaApi(page, codigo, qtd)` | Tenta reservar e retorna status HTTP (para testar recusa) |
| `limparReservas(page, codigo)` | Cancela reservas com marca `E2E` |
| `apagarEspacoDeTeste(page, codigo)` | Remove ou inativa espaço de teste |
| `funcionarioPorEmail(page, email)` | Busca funcionário pelo e-mail |
| `criarFuncionarioSemSenha(page, email)` | Cria usuário com convite pendente |
| `apagarFuncionario(page, email)` | Remove funcionário de teste |

### Datas

| Função | Descrição |
| --- | --- |
| `iso(data)` | Formata `Date` para `YYYY-MM-DD` |
| `hoje()` | Data de hoje no formato ISO |
| `emDias(n)` | Data daqui a `n` dias no formato ISO |

---

## 7. Convenções de segurança

A suíte foi projetada para rodar contra um banco de dados com dados reais sem
causar danos:

1. **Prefixo `E2E`:** todo espaço criado pelos testes tem código começando com
   `E2E` (constante `MARCA`). Nenhuma operação de escrita ou exclusão é feita em
   espaços sem esse prefixo.

2. **Estufas reais intocadas:** a única interação com estufas reais (ex: CVG50) é
   criar reservas de teste — o cadastro (nome, capacidade, tipo) nunca é alterado.

3. **Reservas marcadas:** toda reserva criada pela suíte tem `E2E` na
   finalidade. A limpeza só cancela reservas com essa marca.

4. **Limpeza automática:** o último teste (`remove o que a suíte criou`)
   cancela todas as reservas `E2E`, inativa os espaços de teste e remove o
   funcionário de teste.

5. **Identificador de execução (`RUN`):** cada execução pode usar um `CVG_RUN`
   diferente, evitando conflito entre execuções simultâneas.

---

## 8. Relatórios e artefatos de falha

### Terminal

O reporter padrão é `list`, que mostra cada teste com status em tempo real.

### HTML (CI)

Em ambiente de CI (`process.env.CI`), o reporter HTML gera um relatório
navegável:

```bash
npm run test:e2e:report
```

### Artefatos por teste

Em caso de falha, o Playwright salva automaticamente:

| Artefato | Localização padrão |
| --- | --- |
| Screenshot da falha | `test-results/<nome-do-teste>/` |
| Vídeo da execução | `test-results/<nome-do-teste>/` |
| Trace (timeline) | `test-results/<nome-do-teste>/` |

O trace pode ser aberto com:
```bash
npx playwright show-trace test-results/<nome-do-teste>/trace.zip
```

---

## 9. Cenários de teste manuais

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

## 10. Plano de expansão

### Testes E2E a implementar

| Cenário | Prioridade | Complexidade |
| --- | --- | --- |
| Fluxo de convite completo (e-mail mock) | Alta | Média |
| Recuperação de senha com token expirado | Alta | Baixa |
| CRUD completo de projetos | Média | Média |
| Gestão de equipe (adicionar/remover colaborador) | Média | Média |
| Filtros de auditoria | Baixa | Baixa |
| Permissões por projeto (colaborador com permissão limitada) | Alta | Alta |
| Chatbot: envio de mensagem e resposta | Baixa | Média |

### Testes unitários (sugestão futura)

| Alvo | Framework sugerido |
| --- | --- |
| Validators do front (TypeScript) | Vitest |
| Form Requests do Laravel | PHPUnit (já incluso no Laravel) |
| Policies (autorização) | PHPUnit |
| SanitizaBusca (escape de LIKE) | PHPUnit |
| Cálculo de disponibilidade | PHPUnit |

### Testes de integração (sugestão futura)

| Alvo | Framework sugerido |
| --- | --- |
| Endpoints da API (feature tests) | PHPUnit + Laravel HTTP Tests |
| Validação de cascata de FK | PHPUnit + RefreshDatabase |
| Rate limiting (thresholds) | PHPUnit + withoutMiddleware seletivo |
