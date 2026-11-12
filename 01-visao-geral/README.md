# 1. Visão geral

O **CVGWeb** é o Sistema de Gestão de Campos Experimentais da **Embrapa
Cenargen**. Ele substitui o controle manual das casas de vegetação (estufas) do
campus por um sistema web onde a equipe enxerga o parque inteiro sobre a foto
aérea, pede reserva de capacidade por período e a administração aprova.

---

## 1.1 O problema

O parque do Cenargen tem **51 casas de vegetação**. Antes do CVGWeb, saber quem
estava usando qual espaço, até quando e para qual projeto dependia de planilha,
e-mail e conversa de corredor. Dois pesquisadores podiam combinar o mesmo
espaço sem saber, e não havia registro de quem autorizou o quê.

## 1.2 A proposta

| Frente | O que o sistema faz |
| :--- | :--- |
| **Visualização** | Mapa aéreo do campus com cada estufa marcada, situação e ocupação atual |
| **Reserva por capacidade** | A reserva não toma a estufa inteira: pede-se uma quantidade de vasos ou bancadas e o restante continua disponível |
| **Aprovação** | Reservar é **pedir**. Toda reserva nasce pendente e um administrador libera |
| **Projetos e equipes** | Projetos com líder, equipe e permissões de CRUD concedidas por projeto |
| **Acesso por convite** | Não existe tela de cadastro: o administrador cadastra, a pessoa recebe um link por e-mail e cria a própria senha |
| **Auditoria** | Todo ato registrado — quem fez, o que fez, quando e de qual IP |
| **Assistente** | Chat que responde, em linguagem natural, quem está usando cada espaço, por quanto tempo e para qual projeto |

---

## 1.3 Atores

| Ator | O que pode fazer |
| :--- | :--- |
| **Super usuário (SU)** | Tudo. Conta técnica criada pelo seeder, usada em implantação e teste |
| **Administrador** | Cadastra pessoas, espaços e projetos; concede permissões; aprova e cancela reservas; lê os logs |
| **Líder de projeto** | Enxerga e conduz o próprio projeto; só faz CRUD onde recebeu permissão |
| **Pesquisador / colaborador** | Consulta o mapa, **propõe projeto**, pede reserva e acompanha o que é seu |

> O líder **não** herda poder de administrador: sem concessão explícita por
> projeto, só o administrador faz CRUD. A exceção é o autor de uma proposta
> aceita, que recebe o mínimo para tocar o próprio projeto.

### Como alguém começa a usar

```
Administração cadastra a pessoa → convite por e-mail → ela cria a senha
   └─ entra e vê o mapa, as próprias reservas e os próprios projetos
       ├─ já está na equipe de um projeto?  → pede reserva
       └─ não está?                         → propõe um projeto
                                               └─ administração aprova
                                                   └─ pede reserva
```

---

## 1.4 Módulos do sistema

| Módulo | Tela | Documentação técnica |
| :--- | :--- | :--- |
| Mapa | Foto aérea com os marcadores das 51 estufas | [Front-end](../04-frontend/README.md#telas) |
| Reservas | Pedido, aprovação, cancelamento e histórico | [Back-end](../03-backend/README.md#regras-de-negócio) |
| Projetos | Projeto, equipe, líder e permissões | [Back-end](../03-backend/README.md#autorização-em-duas-camadas) |
| Usuários | Cadastro, convite, situação e perfil | [Segurança](../07-seguranca/README.md) |
| Espaços | Alteração das 51 casas de vegetação | [Banco de dados](../05-banco-de-dados/README.md) |
| Logs | Auditoria de tudo que acontece | [Segurança](../07-seguranca/README.md) |
| Assistente | Chat com LLM sobre o estado do parque | [Back-end](../03-backend/README.md#assistente) |

---

## 1.5 Glossário

| Termo | Significado |
| :--- | :--- |
| **Casa de vegetação / estufa / espaço** | Ambiente controlado de cultivo. O parque é fechado em 50 — o cadastro **altera** uma delas, nunca cria uma nova |
| **Unidade de capacidade** | `VASO` ou `BANCADA`: a grandeza em que a estufa é medida e reservada |
| **Capacidade total** | Quantas unidades a estufa comporta |
| **Reserva** | Pedido de uma quantidade de unidades, num período, para um projeto |
| **Pendente / Ativa / Cancelada** | Situações da reserva: aguardando aprovação, aprovada, encerrada ou recusada |
| **Perfil** | Permissão global do usuário no sistema (`/SU/`, `/FUNCIONARIO-I/A/E/`) |
| **Permissão por projeto** | Autorização de CRUD válida apenas dentro de um projeto |
| **Convite de acesso** | Token com prazo de 48 h enviado por e-mail para a pessoa criar a própria senha |
| **Auditoria / Log** | Registro imutável de uma ação do usuário |

---

## 1.6 Documentos de origem

| Documento | Conteúdo |
| :--- | :--- |
| [Documento de Visão](documento-de-visao.md) | Escopo, objetivos, público-alvo e justificativa do projeto |
| [Documento de Requisitos](documento-de-requisitos.md) | Requisitos funcionais, não funcionais e casos de uso |
| [Termo de Anuência](termo-de-anuencia.jpg) | Autorização da Embrapa Cenargen para o desenvolvimento |

Versões anteriores em PDF, mantidas só para referência histórica, estão em
[`arquivo/`](../arquivo/).

---

## 1.7 Contexto acadêmico

O CVGWeb é o projeto integrador do curso de Ciência da Computação do **CEUB**,
desenvolvido em parceria com a Embrapa Cenargen e entregue por sprints.
