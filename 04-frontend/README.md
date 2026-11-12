# 4. Front-end (TypeScript + Vite)

Interface do CVGWeb: consulta dos espaços sobre a foto aérea do campus, pedido
de reserva e administração de espaços, reservas, projetos, usuários e logs.

HTML, CSS e TypeScript **sem framework**, empacotados com Vite.

Repositório: [Embrapa-frontend](https://github.com/Embrapa-Cenargem-Dashboard-Interativo/Embrapa-frontend) ·
Como rodar: [README do repositório](../../Embrapa-frontend/README.md)

---

## Índice

1. [Estrutura de pastas](#estrutura-de-pastas)
2. [Telas](#telas)
3. [Convenções](#convenções)
4. [Integração com a API](#integração-com-a-api)
5. [Estado e atualização de tela](#estado-e-atualização-de-tela)
6. [Acessibilidade](#acessibilidade)
7. [Assistente](#assistente)
8. [Build](#build)

---

## Estrutura de pastas

```
index.html              Marcação completa; o sprite de ícones entra inline
assets/
├── MapEstufa.webp      Foto aérea do campus (fallback .jpg)
├── icons.svg           Sprite dos ícones — fonte do que é injetado no HTML
└── fonts/              IBM Plex Sans e Mono, subconjunto latin, .woff2
scripts/
└── build-icons.mjs     Injeta assets/icons.svg no index.html
styles/
├── tokens.css          Cor, tipografia, espaço, raio, sombra e motion
├── base.css            @font-face, reset, foco, utilidades de acessibilidade
├── layout.css          Casca: lateral, área principal, troca de views
├── components.css      Botão, campo, pill, chip, tabela, modal, toast, card
├── telas.css           Telas de listagem (reservas, projetos, usuários, logs)
├── login.css           Tela de entrada
├── map.css             Foto, marcadores, legenda e ficha do espaço
├── admin.css           Tela de espaços
└── chat.css            Assistente
src/
├── main.ts             Ponto de entrada
├── app.ts              Navegação, menu da conta, registro das ações
├── types.ts            Tipos de domínio
├── lib/
│   ├── actions.ts      Delegação de eventos por data-action
│   ├── bus.ts          Aviso de mudança nos dados
│   ├── dom.ts          Atalhos de DOM
│   ├── format.ts       Status, datas e plurais
│   ├── html.ts         Template de HTML com escape automático
│   ├── modal.ts        Modais acessíveis e diálogo de confirmação
│   ├── navegacao.ts    Troca de seção e estado da rota
│   ├── senha.ts        Medidor e regras de senha
│   ├── telas.ts        Casca comum das listagens
│   └── toast.ts        Avisos temporários
├── data/
│   ├── estufas.ts      Leitura do parque a partir da API
│   └── hotspots.ts     Coordenadas dos marcadores sobre a foto
├── services/
│   ├── api.ts          Todas as chamadas à API Laravel
│   └── llm.ts          Chamada ao assistente (via back-end)
└── views/
    ├── login.ts        Entrada
    ├── acesso.ts       Definir senha pelo link do convite
    ├── mapa.ts         Foto aérea, marcadores e ficha do espaço
    ├── reservas.ts     Pedidos, aprovação e histórico
    ├── projetos.ts     Projetos, equipe e permissões
    ├── admin.ts        Espaços
    ├── auditoria.ts    Logs
    ├── chat.ts         Assistente
    └── falha.ts        Erro de carregamento
```

---

## Telas

| Seção | View | O que faz |
| :--- | :--- | :--- |
| **Mapa** | `mapa.ts` | Foto aérea com um marcador por estufa; clique abre a ficha com situação, ocupação, responsável e período |
| **Reservas** | `reservas.ts` | Pedir, aprovar, cancelar e acompanhar reservas |
| **Projetos** | `projetos.ts` | Projetos, equipe, líder e concessão de permissões |
| **Usuários** | `admin.ts` / `telas.ts` | Cadastro, convite, situação e perfil |
| **Espaços** | `admin.ts` | Alteração das 50 casas de vegetação |
| **Logs** | `auditoria.ts` | Auditoria de tudo que acontece, com filtros |
| **Entrada** | `login.ts`, `acesso.ts` | Login, "esqueci minha senha" e criação da primeira senha pelo link |

Reservas, Projetos, Usuários e Logs são seções próprias do menu — saíram dos
modais. O botão do assistente aparece **só na tela do mapa**.

### Quem vê o quê

O menu não olha mais para um booleano `admin`: ele pergunta ao back-end.
`src/data/contexto.ts` guarda a resposta de `GET /me/contexto` e é a única
fonte que as telas consultam.

| Seção | Condição |
| :--- | :--- |
| Mapa, Reservas, Projetos | Todo mundo — a API devolve só o que é da pessoa |
| Usuários | `pode.cadastrar_funcionario` |
| Espaços | `pode.configurar_casa` |
| Logs | `pode.ver_auditoria` |

O mesmo módulo responde se dá para reservar agora (`projetosParaReservar`) e,
quando não dá, por quê (`motivoParaNaoReservar`) — a ficha do espaço mostra
esse motivo abaixo do botão em vez de só desabilitá-lo.

---

## Convenções

**Tokens.** Nenhum arquivo além de `styles/tokens.css` escreve um valor de cor
literal. Para mudar a paleta, muda-se o token e os componentes acompanham. O
sistema tem um tema só, o claro: `color-scheme` fixo em `light`.

**Sem `onclick` e sem `window`.** Todo controle declara o que faz num atributo
`data-action`, e um único listener no documento (`lib/actions.ts`) encaminha
para a função registrada. Selects usam `data-change` do mesmo jeito. Para
acrescentar um comportamento: ponha o `data-action` no HTML e registre o
handler em `registrarTodasAsAcoes()`, em `src/app.ts`.

**Escape.** Texto vindo da API ou digitado por alguém entra pelo template
`html` de `lib/html.ts`, que escapa toda interpolação. Marcação montada de
propósito usa `raw()`.

**Ícones.** Lucide (licença ISC), só o subconjunto usado, como sprite inline.
Para acrescentar um: pegue o SVG em [lucide.dev](https://lucide.dev), cole um
`<symbol id="i-nome" …>` em `assets/icons.svg` e rode `npm run icons` — o
script avisa se algum `<use>` aponta para símbolo inexistente.

**Tipografia.** IBM Plex Sans e Mono auto-hospedadas em `.woff2`. Nenhuma
requisição a CDN: o app funciona offline e na rede interna.

---

## Integração com a API

`VITE_API_URL` é **obrigatória**. O front não tem fonte de dados local: sem a
API no ar, o login não autentica e o mapa sobe vazio.

| Arquivo | Papel |
| :--- | :--- |
| `services/api.ts` | Toda chamada HTTP; monta o header `Authorization` e trata erro |
| `data/estufas.ts` | Lê `/casas-vegetacao` e `/reservas` e traduz para o modelo da interface |

A autenticação é real: `/login` devolve um token Sanctum, guardado em
`localStorage` na chave `cvgweb:token` e enviado em toda requisição. Quem foi
cadastrado pela administração e ainda não criou a senha pelo link recebe uma
mensagem explicando isso, em vez do erro genérico.

**Toda escrita vai para a API** — reservar, aprovar, cancelar, mudar a situação
do espaço, cadastrar espaço e cadastrar, desativar ou excluir usuário.

---

## Estado e atualização de tela

Depois de gravar, o estado é **relido do banco**: a tela nunca fica adiantada
em relação ao que foi salvo. `lib/bus.ts` avisa as views interessadas de que os
dados mudaram, e cada uma se redesenha.

Se a releitura falhar no meio da sessão, o app mantém o que já estava na tela e
registra o motivo no console — melhor um dado de um minuto atrás do que uma
tela vazia.

---

## Acessibilidade

Meta: **WCAG 2.1 AA**, conferida com `npx axe` nas telas principais, no desktop
e no celular, sem violações apontadas.

- Nenhum estado é comunicado só por cor: as pills trazem ícone e palavra.
- Os marcadores do mapa têm nome acessível completo.
- Modais aplicam `inert` ao resto da página, devolvem o foco ao fechar e fecham
  com `Esc`.
- O anel de foco é um só, definido em `base.css`, e nunca é removido.

---

## Assistente

Botão no canto inferior direito da tela do mapa, visível depois da entrada.
Responde quem está usando cada espaço, por quanto tempo e qual projeto está lá
dentro.

A chamada passa pelo back-end (`POST /api/assistente`) — o front **não** tem
chave de LLM. As regras de resposta e o contexto do parque ficam no back-end,
em `resources/assistente/base-conhecimento.md`
(ver [3. Back-end](../03-backend/README.md#assistente)).

---

## Build

```bash
npm run dev         # desenvolvimento em http://localhost:5173
npm run typecheck   # só a checagem de tipos
npm run build       # typecheck + pacote de produção em dist/
npm run preview     # serve o dist/ para conferir o pacote
npm run icons       # regenera o sprite de ícones dentro do index.html
npm run test:e2e    # Playwright (exige a API no ar) — ver 8. Testes
```

> Toda variável `VITE_` é embutida no JavaScript publicado e fica legível para
> qualquer visitante. Segredo nenhum entra no front.
