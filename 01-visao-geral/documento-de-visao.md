![logo_ceub](../_assets/logo_ceub.png)

# Documento de Visão - Embrapa Cenargen: CVGWeb 

**Centro Universitário de Brasília (CEUB)**  
**Data:** 27/08/2026  
**Disciplina:** Projeto Integrador IV  
**Semestre:** 8º semestre  

**Professor/Orientador:** Tiago Leite Pereira

### Alunos

- Felipe Tolentino Soares
- Lucas Daniel Paiva de Sá
- Luis Guilherme Andrade Palhares de Melo
- Miguel Artur de Castro Miranda
- Lucas Andrade Fonseca

![logo_embrapa](../_assets/logo_embrapa.png)

---

## Sumário

1. [Problema](#1-problema)
2. [Visão Geral](#2-visão-geral)
3. [Restrições](#3-restrições)
4. [Arquitetura Básica](#4-arquitetura-básica)
5. [Usuários](#5-usuários)
6. [Partes Interessadas](#6-partes-interessadas)
7. [Futuro](#7-futuro)
8. [Referências](#8-referências)

---

## Introdução

Este documento, elaborado no âmbito da disciplina Projeto Integrador IV do Centro Universitário de Brasília (CEUB), contém as informações gerais do projeto **Embrapa Cenargen: CVGWeb**, que será utilizado para apoiar o processo de controle e planejamento da utilização dos campos experimentais e casas de vegetação da unidade.

O objetivo deste documento é apresentar uma visão geral do sistema proposto, permitindo o entendimento do problema a ser resolvido, da solução proposta e dos principais elementos envolvidos no projeto.

Este documento é composto pelos seguintes itens:

- **Problema** – detalhamento da situação atual que motiva o desenvolvimento do sistema.
- **Visão Geral** – descrição do sistema proposto, seu objetivo e seus principais recursos.
- **Restrições** – limitações relacionadas ao sistema e ao projeto.
- **Arquitetura Básica** – visão geral da estrutura tecnológica da solução.
- **Usuários** – descrição dos perfis que utilizarão o sistema.
- **Partes Interessadas** – pessoas ou setores relacionados ao projeto.
- **Futuro** – possíveis evoluções do sistema.
- **Referências** – fontes utilizadas no desenvolvimento deste documento.

## 1. Problema

O **Centro Nacional de Recursos Genéticos e Biotecnologia (Cenargen)**, unidade da **Empresa Brasileira de Pesquisa Agropecuária (Embrapa)**, possui uma infraestrutura composta por diversas casas de vegetação e áreas de campos experimentais utilizados para a realização de experimentos científicos.

Esses espaços são utilizados por diferentes pesquisadores e projetos de pesquisa ao longo de diversos períodos de tempo. Atualmente, o controle da utilização desses espaços é realizado pelo **Núcleo de Apoio à Pesquisa (NAP)**, porém esse processo não conta com um sistema informatizado específico que permita visualizar de forma centralizada a ocupação atual, as reservas futuras e a disponibilidade dos espaços.

A ausência de uma ferramenta dedicada dificulta o planejamento da utilização da infraestrutura, aumenta a possibilidade de conflitos na alocação de espaços e limita a geração de informações gerenciais sobre a utilização desses recursos.

Dessa forma, surge a necessidade de desenvolver uma solução informatizada que permita melhorar a organização, o controle e o planejamento do uso dos campos experimentais e casas de vegetação do Cenargen.

## 2. Visão Geral

O projeto **Embrapa Cenargen: CVGWeb** consiste em uma aplicação web destinada ao controle da utilização dos campos experimentais e casas de vegetação da unidade.

O sistema terá como objetivo centralizar as informações relacionadas à ocupação dos espaços experimentais, permitindo que pesquisadores e gestores visualizem de forma clara e organizada a disponibilidade desses recursos.

Para isso, o sistema contará com uma interface baseada em um **mapa interativo do Cenargen**, permitindo que os usuários visualizem os diferentes espaços experimentais e acessem informações detalhadas sobre sua utilização.

### Principais recursos

- Visualização dos espaços experimentais em mapa interativo;
- Consulta da disponibilidade dos espaços;
- Registro de reservas para utilização dos espaços;
- Cadastro de pesquisadores;
- Cadastro de projetos de pesquisa;
- Associação entre pesquisadores, projetos e espaços utilizados;
- Geração de relatórios sobre a utilização da infraestrutura.

O escopo inicial do sistema contempla as funcionalidades necessárias para o desenvolvimento de um **protótipo funcional**, permitindo validar o conceito da solução e apoiar a gestão dos espaços experimentais.

## 3. Restrições

As seguintes restrições devem ser consideradas no desenvolvimento do sistema:

- O sistema será desenvolvido inicialmente como um protótipo acadêmico.
- Não haverá integração inicial com outros sistemas institucionais da Embrapa.
- O sistema será acessado por meio de navegadores web.
- As informações sobre pesquisadores, projetos e espaços deverão ser cadastradas manualmente no sistema.

## 4. Arquitetura Básica

O sistema será desenvolvido como uma aplicação web, composta por uma interface de usuário acessível por meio de navegador e um sistema de gerenciamento de dados responsável pelo armazenamento das informações relacionadas aos pesquisadores, projetos e espaços experimentais.

De forma geral, a arquitetura será composta por:

- **Interface web** para interação com os usuários;
- **Camada de aplicação** responsável pela lógica do sistema;
- **Banco de dados** para armazenamento das informações.

## 5. Usuários

O sistema será utilizado principalmente pelos seguintes perfis de usuários:

### Pesquisadores

Responsáveis pela condução de experimentos científicos e pela solicitação de utilização de espaços experimentais.

### Equipe do Núcleo de Apoio à Pesquisa (NAP)

Responsável pela gestão, organização e controle da utilização dos campos experimentais e casas de vegetação.

### Administração do Cenargen

Interessada em obter informações consolidadas sobre o uso da infraestrutura de pesquisa.

## 6. Partes Interessadas

Diretamente relacionadas: **Cenargen (Embrapa)** e **Equipe de desenvolvimento do projeto**.

### Cenargen

| Representante | Atuação no Projeto | E-mail |
|---|---|---|
| Sr. Luis Alberto Martins Palhares de Melo | Cliente / Usuário do sistema | palhares65@gmail.com |

### Equipe do Projeto 

| Representante | Atuação no Projeto | E-mail |
|---|---|---|
| Felipe Tolentino Soares | Coordenador - Documentador | felipe.tolentino@sempreceub.com |
| Lucas Andrade Fonseca | Arquiteto (Dev Team) — Front/Back | lucas.afonseca@sempreceub.com |
| Lucas Daniel Paiva de Sá | AD/DBA (Dev Team) — Front/Back | lucas.psa@sempreceub.com |
| Luis Guilherme Andrade Palhares de Melo | PO (Dev Team) — Front/Back | luis.pmelo@sempreceub.com |
| Miguel Artur de Castro Miranda | Documentador | miguel.artur@sempreceub.com |

## 7. Futuro

Possíveis evoluções do sistema incluem:

- Integração com sistemas institucionais da Embrapa;
- Inclusão de funcionalidades de análise avançada de uso dos espaços;
- Automatização do processo de solicitação e aprovação de reservas;
- Expansão do sistema para outras unidades da Embrapa.

## 8. Referências

- **EMBRAPA – Empresa Brasileira de Pesquisa Agropecuária**  
  https://www.embrapa.br

- **EMBRAPA Recursos Genéticos e Biotecnologia (Cenargen)**  
  https://www.embrapa.br/recursos-geneticos-e-biotecnologia

- **Documento de Visão Chatbot – SERPRO**
