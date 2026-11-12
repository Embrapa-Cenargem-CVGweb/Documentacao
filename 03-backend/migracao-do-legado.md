# Migração da regra de negócio (Aplicação PHP → Laravel)

Regras de negócio do CVGWeb portadas do app PHP legado (`pcvgweb`) para este
back-end Laravel 12 (API + Sanctum).

## O que foi migrado

| Regra legada (PHP) | Onde ficou no Laravel |
|---|---|
| Login por `nick` case-insensitive + senha (`fdao000_Usuario_Existe`, `js001_acessar.js`) | `AuthController@login` (agora com hash + token Sanctum) |
| Perfil de permissões `/SU/`, `/FUNCIONARIO-I/A/E/` (`gPerfilUsr`, `js002_funcionarioH.js`) | `Funcionario::temPerfil()` / `isSuperUsuario()` + middleware `perfil:` |
| SU zera as permissões I/A/E (`f002_LigarOuDesligarOpcoes`) | `FuncionarioController::montarPerfil()` |
| CRUD de Funcionário (`fdao001..004`) | `FuncionarioController` + `Store/UpdateFuncionarioRequest` |
| Listagem de Cargo (`fdao001_Listar_Cargo`, `CVGD002Cargo`) | `CargoController` + `CargoSeeder` |
| Geração de ID via `CVGD000aChaveUnica` | auto-increment do Eloquent |
| Formatação de matrícula (`futil01_FormataMatriculaEmbrapa`) | `App\Support\Matricula::formatar()` |
| Formato de transporte `[[TAG]]..[[/TAG]]` ("MeuXML") | substituído por JSON |
| Status de projeto (`CVGD005StatusProj`) | `StatusProjeto` + `StatusProjetoSeeder` |
| Projeto / colaboradores (`CVGD003/004`) | `Projeto`, `ProjetoController` |

## O que foi implementado a partir do README do back-end

O domínio de reservas ainda não existia no PHP (só a intenção na `ChaveUnica`).
Foi modelado conforme o README:

- **Casa de Vegetação** (espaço experimental): `CasaVegetacao`, `CasaVegetacaoController`.
- **Reserva**: `Reserva`, `ReservaController`.
- **Regra central — capacidade por período**: a reserva não toma a estufa
  inteira. Cada espaço tem `capacidade_total` numa unidade (VASO ou BANCADA) e
  a soma das quantidades das reservas NÃO CANCELADAS (ativas **e** pendentes)
  que se cruzam no período não pode passar dela; o resto fica livre para outro
  pesquisador. Implementada em `ReservaService` e aplicada em
  `Store/UpdateReservaRequest` (retorna 422). Sobreposição:
  `inicio_existente <= fim_novo E fim_existente >= inicio_novo`.
- **Fluxo de aprovação**: toda reserva nova nasce `PENDENTE`, inclusive a de
  quem tem `reserva.aprovar` e a do adm — reservar é pedir. Liberar é um ato
  separado (`POST /api/reservas/{id}/aprovar`), que reconfere a capacidade e
  grava `aprovada_por` / `aprovada_em`. A pendente já segura o espaço, senão
  dois pedidos para o mesmo período passariam os dois.
- **Edicao de reserva**: mexer no espaco, no periodo ou na quantidade de uma
  reserva ja ATIVA a devolve para `PENDENTE` e zera `aprovada_em`/`aprovada_por`
  (`ReservaController@update`). A aprovacao valia para o que foi aprovado;
  corrigir finalidade ou observacao nao reabre a fila. `DELETE` apaga a linha e
  libera a capacidade — diferente de cancelar, que preserva o registro e o motivo.
- **Disponibilidade**: `GET /api/casas-vegetacao/{id}/disponibilidade`, com
  `ignorar_reserva` para a tela de edicao nao contar a propria reserva duas vezes.

## Decisões

- Banco em **convenção Laravel** (snake_case, tabelas novas) — não conecta ao
  MySQL legado `dbcvg`. Mapeamento dos nomes antigos está nos comentários das migrations.
- Autenticação via **Sanctum** (token Bearer). A senha, que no legado era texto
  puro (`D_SENHAUSR`), agora é armazenada com hash.
- `Funcionario` é o modelo autenticável (era `CVGD001Funcionario`). O `User`
  padrão do Laravel foi mantido, mas não é usado pela API.

## Setup

O caminho normal é o docker-compose na raiz de `Projetos/embrapa`, que sobe
banco, API e front juntos e roda migrations e seeders no primeiro boot:

```bash
docker compose up -d --build
```

Sem docker, dentro de `Embrapa-backend`:

```bash
composer install
php artisan key:generate
php artisan migrate --seed    # cria tabelas + cargos, status e super usuário
php artisan serve
```

Login inicial: **nick `SU` / senha `super123`** (troque em produção).

## Endpoints principais

```
POST   /api/login                                   { nick, senha }
POST   /api/esqueci-senha                           { email }
POST   /api/definir-senha                           { token, senha }
GET    /api/me
GET    /api/me/contexto                             projetos, casas e permissoes
POST   /api/logout

GET    /api/cargos
GET    /api/status-projeto
GET    /api/permissoes/catalogo
GET    /api/painel                                  metricas da administracao

apiResource /api/funcionarios
PATCH  /api/funcionarios/{id}/situacao              { ativo }
POST   /api/funcionarios/{id}/convite               reenvia o link de senha

apiResource /api/projetos
GET    /api/projetos/{id}/equipe
POST   /api/projetos/{id}/equipe
PUT    /api/projetos/{id}/equipe/{funcionario}
PUT    /api/projetos/{id}/equipe/{funcionario}/permissoes
DELETE /api/projetos/{id}/equipe/{funcionario}

apiResource /api/casas-vegetacao                    ?com_ocupacao=1 na listagem
GET    /api/casas-vegetacao/tipos                   sugestoes de tipo e setor
GET    /api/casas-vegetacao/{id}/disponibilidade    ?de=&ate=&quantidade=
PATCH  /api/casas-vegetacao/{id}/situacao           { status }

apiResource /api/reservas
POST   /api/reservas/{id}/aprovar
POST   /api/reservas/{id}/cancelar

GET    /api/auditorias                              trilha de alteracoes
GET    /api/auditorias/filtros
```

Todas as rotas (exceto `login`) exigem header `Authorization: Bearer <token>`.
