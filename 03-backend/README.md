# 3. Back-end (Laravel 12)

API REST que concentra **todas** as regras de negócio do CVGWeb: persistência,
controle de acesso, fluxo de reserva, convite por e-mail, auditoria e a
conversa com a LLM do assistente.

Repositório: [Embrapa-backend](https://github.com/Embrapa-Cenargem-CVGweb/Embrapa-backend) ·
Como rodar: [README do repositório](../../Embrapa-backend/README.md) ·
Endpoints: [6. API](../06-api/README.md)

---

## Índice

1. [Camadas](#camadas)
2. [Estrutura de pastas](#estrutura-de-pastas)
3. [Controllers](#controllers)
4. [Models e relacionamentos](#models-e-relacionamentos)
5. [Regras de negócio](#regras-de-negócio)
6. [Autorização em duas camadas](#autorização-em-duas-camadas)
7. [Auditoria](#auditoria)
8. [Acesso e e-mail](#acesso-e-e-mail)
9. [Assistente](#assistente)
10. [Comandos de console](#comandos-de-console)
11. [Configuração](#configuração)
12. [Convenções de código](#convenções-de-código)

---

## Camadas

Cada responsabilidade tem um lugar. Nada de regra de negócio em controller,
nada de validação em model.

| Camada | Pasta | Responsabilidade |
| :--- | :--- | :--- |
| **Middleware** | `app/Http/Middleware` | Autenticação, perfil, cabeçalhos de segurança, throttle |
| **Form Request** | `app/Http/Requests` | Validação de entrada e as regras que dependem só do pedido |
| **Controller** | `app/Http/Controllers/Api` | Orquestra: autoriza, chama o serviço, devolve JSON |
| **Policy** | `app/Policies` | "Esta pessoa pode fazer isso **neste registro**?" |
| **Service** | `app/Services` | Regra de negócio de verdade (capacidade, envio de link, LLM) |
| **Model** | `app/Models` | Persistência, relacionamentos, escopos e auditoria |
| **Support** | `app/Support` | Utilitários puros (matrícula, permissões, sanitização) |

---

## Estrutura de pastas

```
app/
├── Console/Commands/          4 comandos de operação (senha, limpeza, e-mail)
├── Http/
│   ├── Controllers/Api/       14 controllers REST
│   ├── Middleware/
│   │   ├── SecurityHeaders    CSP, X-Frame-Options, nosniff, Referrer-Policy
│   │   └── VerificaPerfil     Autorização por perfil do sistema
│   └── Requests/              8 Form Requests (Store/Update × 4 domínios)
├── Mail/
│   ├── LinkDeAcessoMail       E-mail de convite e de nova senha
│   └── Transport/             Transport próprio sobre a API HTTP da Resend (alternativa ao SMTP)
├── Models/
│   ├── Concerns/Auditavel     Trait que registra toda escrita na auditoria
│   └── …                      10 modelos Eloquent
├── Policies/                  5 policies
├── Providers/
├── Services/
│   ├── Acesso/                Geração e envio do link de acesso
│   ├── Assistente/            Cliente Groq, contexto e tratamento de erro
│   └── ReservaService         A regra central de capacidade por período
└── Support/
    ├── Matricula              Formatação da matrícula Embrapa
    ├── Permissoes             Catálogo de permissões por projeto
    └── SanitizaBusca          Escape de wildcards em buscas LIKE

bootstrap/app.php              Middleware, aliases, rate limiters e exceções
config/cvgweb.php              Regras do CVGWeb que não são segredo
database/
├── migrations/                Fonte da verdade do schema
└── seeders/                   Cargos, status, as 50 estufas e o SU
routes/api.php                 Todas as rotas da API
```

---

## Controllers

| Controller | Domínio |
| :--- | :--- |
| `AuthController` | Login, logout e `me` |
| `AcessoController` | Convite, "esqueci a senha", consulta de token e definição de senha |
| `ContextoController` | Pacote inicial que o front carrega depois do login |
| `FuncionarioController` | CRUD de usuários e mudança de situação |
| `CargoController` | Lista de cargos |
| `ProjetoController` | CRUD de projetos |
| `StatusProjetoController` | Lista de status de projeto |
| `EquipeProjetoController` | Equipe do projeto e permissões por membro |
| `PermissaoController` | Catálogo de permissões concedíveis |
| `CasaVegetacaoController` | CRUD dos espaços, tipos, situação e disponibilidade |
| `ReservaController` | CRUD de reservas, aprovação e cancelamento |
| `AuditoriaController` | Consulta dos logs e dos filtros disponíveis |
| `PainelController` | Números do painel administrativo |
| `AssistenteController` | Status e resposta do assistente |

---

## Models e relacionamentos

| Model | Tabela | Papel |
| :--- | :--- | :--- |
| `Funcionario` | `funcionarios` | Usuário do sistema; autenticável via Sanctum |
| `Cargo` | `cargos` | Cargo do quadro (Pesquisador, Analista, Técnico…) |
| `Projeto` | `projetos` | Projeto de pesquisa com código, título e datas |
| `StatusProjeto` | `status_projeto` | Situação do projeto |
| `ColaboradorProjeto` | `colaboradores_projeto` | Vínculo pessoa × projeto + permissões concedidas |
| `CasaVegetacao` | `casas_vegetacao` | Espaço experimental, com unidade e capacidade |
| `Reserva` | `reservas` | Quantidade reservada num período, com aprovação |
| `ConviteAcesso` | `convites_acesso` | Token de convite/redefinição e o estado do envio |
| `Auditoria` | `auditorias` | Registro imutável de uma ação |

Colunas, índices, chaves e o diagrama entidade-relacionamento estão em
[5. Banco de dados](../05-banco-de-dados/README.md).

---

## Regras de negócio

### Capacidade por período — a regra central

A reserva **não** toma a estufa inteira. Cada espaço tem uma unidade
(`VASO` ou `BANCADA`) e uma `capacidade_total`; a reserva pede uma
`quantidade`. A soma das quantidades das reservas **não canceladas** — ativas
**e** pendentes — que se cruzam no período não pode passar da capacidade.

```
sobreposição:  inicio_existente <= fim_novo  E  fim_existente >= inicio_novo
```

Implementada em `ReservaService` e aplicada pelos `Store/UpdateReservaRequest`,
que respondem **422** com a quantidade ainda disponível. Os mesmos Form
Requests recusam `projeto_id` de projeto que não esteja `APROVADO`.

> A pendente já segura a capacidade. Se só a ativa ocupasse, dois pedidos para
> o mesmo período passariam os dois e o conflito só apareceria na aprovação.

### Proposta de projeto — qualquer pesquisador propõe, o adm decide

Reservar exige um projeto, e quem não está em projeto nenhum precisa de um
caminho para começar. Por isso **qualquer funcionário ativo propõe** um
projeto: ele nasce `PENDENTE` e não vale para nada até a administração
decidir.

| Situação | Como se chega nela | O que ela permite |
| :--- | :--- | :--- |
| `PENDENTE` | Proposta criada por quem não é administrador | Nada: a API recusa reserva nesse projeto |
| `APROVADO` | `POST /projetos/{id}/aprovar`, ou criação pelo próprio adm | Reservar, montar equipe, o fluxo inteiro |
| `RECUSADO` | `POST /projetos/{id}/recusar`, com motivo obrigatório | Nada; o autor lê o motivo, corrige e propõe de novo |

Ao aprovar, o autor recebe no próprio projeto o mínimo para tocá-lo —
`projeto.editar`, `reserva.criar`, `reserva.editar`, `reserva.cancelar` e
`equipe.adicionar`. Sem isso ele seria líder de um projeto em que não poderia
reservar. Qualquer permissão além dessas continua sendo concessão caso a caso
do administrador.

Enquanto a proposta não é aceita, o autor pode editá-la (`ProjetoPolicy::update`)
para corrigir e reenviar. Excluir projeto segue exclusivo do adm.

### Fluxo de aprovação — reservar é pedir

| Situação | Como se chega nela |
| :--- | :--- |
| `PENDENTE` | Toda reserva nova nasce aqui — inclusive a de quem tem `reserva.aprovar` e a do administrador |
| `ATIVA` | `POST /reservas/{id}/aprovar`, que reconfere a capacidade e grava `aprovada_por` e `aprovada_em` |
| `CANCELADA` | `POST /reservas/{id}/cancelar`, por recusa ou desistência |

Editar espaço, período ou quantidade de uma reserva já aprovada a devolve para
avaliação: a alteração não se aprova sozinha.

### Parque fechado em 50 espaços

`config/cvgweb.php` fixa `espacos.limite = 50`. O cadastro de espaço serve para
**alterar** uma das 50 casas de vegetação do Cenargen; criar a 51ª é recusado
pela API.

### Conta sem senha

O cadastro de uma pessoa grava o registro **sem senha**. Só o link do convite
cria a primeira senha — nem o administrador a conhece.

---

## Autorização em duas camadas

As duas precisam permitir. Uma barra o que a pessoa pode fazer **no sistema**;
a outra, o que ela pode fazer **naquele projeto**.

### 1. Perfil do sistema

Herdado do sistema legado e preservado: tokens no campo `perfil` do
funcionário (`/SU/`, `/FUNCIONARIO-I/A/E/`), verificados pelo middleware
`perfil:` e por `Funcionario::temPerfil()`. O super usuário zera as demais
permissões porque já pode tudo.

### 2. Permissões por projeto

`ColaboradorProjeto` guarda, por par **(funcionário × projeto)**, o que o
administrador concedeu. A mesma pessoa pode ter permissões diferentes em
projetos diferentes, e **nem o líder tem CRUD por padrão**.

Catálogo (`App\Support\Permissoes`):

| Grupo | Permissões |
| :--- | :--- |
| Projeto | `projeto.editar` |
| Equipe | `equipe.adicionar`, `equipe.editar`, `equipe.remover` |
| Funcionário | `funcionario.criar`, `funcionario.editar` |
| Reserva | `reserva.criar`, `reserva.editar`, `reserva.cancelar`, `reserva.aprovar`, `reserva.excluir` |
| Espaço | `casa.configurar` |
| Auditoria | `auditoria.ver` |

As Policies (`FuncionarioPolicy`, `ProjetoPolicy`, `ReservaPolicy`,
`CasaVegetacaoPolicy`, `AuditoriaPolicy`) cruzam as duas camadas em cada
registro.

---

## Auditoria

A trait `App\Models\Concerns\Auditavel` registra criação, alteração e exclusão
direto no model. Como está na camada de persistência, **nada escapa** — nem a
escrita feita por comando de console.

Cada registro guarda quem fez, o que fez, em qual recurso, quando, de qual IP
e, quando se aplica, em qual projeto. A consulta sai em
`GET /api/auditorias`, com filtros em `GET /api/auditorias/filtros`, e aparece
na tela **Logs**.

---

## Acesso e e-mail

`App\Services\Acesso\EnvioDeLinkDeAcesso` cuida do convite de primeiro acesso e
do "esqueci minha senha":

- gera o `ConviteAcesso` com token e prazo de **48 h**;
- envia o `LinkDeAcessoMail` e grava o estado do envio no próprio convite;
- segura reenvio para a mesma pessoa por `CVGWEB_INTERVALO_REENVIO` segundos
  (padrão 60), contra clique duplo e contra encher a caixa de alguém;
- quando o e-mail **não sai**, devolve o link ao administrador na resposta,
  para entrega por outro canal (`CVGWEB_LINK_MANUAL`, ligado só em local).

A resposta do "esqueci minha senha" é sempre a mesma, exista a conta ou não.

Configuração do provedor: [9. Operação](../09-operacao/README.md#e-mail).

---

## Assistente

O navegador faz `POST /api/assistente` (autenticado, com limite por usuário) e
é o **Laravel** que fala com a Groq:

```
AssistenteController
  └─ ContextoAssistente   lê o estado real do parque no banco
      └─ base-conhecimento.md  regras de resposta, com marcadores {{...}}
          └─ GroqClient    chama a API da Groq com GROQ_API_KEY
```

As regras do que o assistente responde ficam em
`resources/assistente/base-conhecimento.md` — para mudar o comportamento,
edite o `.md`, não o código. `GROQ_MODEL` vazio faz o back-end perguntar à Groq
quais modelos a chave alcança e escolher o melhor.

> A chave **nunca** vai para o front. Não existe variável `VITE_GROQ_*`: tudo
> que começa com `VITE_` é embutido no JavaScript publicado.

---

## Comandos de console

```bash
php artisan cvgweb:senha --listar              # lista os usuários
php artisan cvgweb:senha admin@cvgweb.local novasenha  # define senha pela linha de comando
php artisan cvgweb:limpar-reservas --espaco=CVG01
php artisan cvgweb:testar-email alguem@exemplo.com
```

`cvgweb:senha` é a saída quando ninguém mais consegue entrar. Todas as
escritas desses comandos também caem na auditoria.

---

## Configuração

| Arquivo | O que guarda |
| :--- | :--- |
| `config/cvgweb.php` | Regras que não são segredo: limite de espaços, tipos sugeridos, prazo de reenvio, link manual |
| `config/services.php` | Credenciais de serviços externos (Groq, e-mail) |
| `bootstrap/app.php` | Middleware, aliases, rate limiters (`login`, `acesso`, `assistente`, `api`) e tratamento de exceções |

A lista completa de variáveis de ambiente está em
[9. Operação](../09-operacao/README.md#variáveis-de-ambiente).

---

## Convenções de código

- Todo model declara `$fillable` — nunca `$guarded = []`.
- Senha é hasheada pelo cast `'hashed'` do Eloquent (bcrypt, 12 rounds).
- Busca com `LIKE` passa por `SanitizaBusca::escapar()`.
- Resposta de erro nunca revela se um e-mail existe no sistema.
- Regra de negócio mora em Service; controller só orquestra.
- Validação mora em Form Request; nunca dentro do controller.
- Nome de rota, campo e tabela em **português**, em `snake_case`.
