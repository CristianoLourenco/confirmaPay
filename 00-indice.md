# ConfirmaPay — Documentação técnica

**Versão:** 0.1 · **Data:** 17 de Outubro de 2026 · **Estado:** pronta para iniciar a construção do protótipo PWA

Complementa o [README de concepção](../../README.md) (v0.2). Em caso de conflito, o README prevalece sobre âmbito e requisitos (RF-01 a RF-09); esta documentação prevalece sobre decisões técnicas.

> Localização prevista no repositório: `docs/technical/`.

---

## 1. Âmbito

Documentação técnica para construir o **protótipo PWA com dados simulados**. Não cobre integração real com a EMIS, bancos ou sistemas de pagamento (ver README §7, "Excluído").

## 2. Índice

| # | Ficheiro | Conteúdo |
|---|---|---|
| 00 | [00-indice.md](00-indice.md) | Âmbito, decisões (ADR), convenções, glossário, questões em aberto |
| 01 | [01-arquitectura.md](01-arquitectura.md) | Contexto, componentes, camadas, fluxo principal, requisitos não funcionais, stack, monorepo |
| 02 | [02-backend.md](02-backend.md) | Domínio, modelo de dados, contrato da API, regras, sequências, estados, erros, segurança |
| 03 | [03-frontend.md](03-frontend.md) | Rotas, ecrãs, componentes, estados de resultado, PWA, acessibilidade, testes |
| 04 | [04-infraestrutura.md](04-infraestrutura.md) | Ambientes, implantação, Docker, variáveis, CI/CD, observabilidade, runbook |
| 05 | [05-testes-e-metricas.md](05-testes-e-metricas.md) | Rastreabilidade RF→testes, casos de teste, cenários, protocolo e métricas de usabilidade |
| 06 | [06-backlog.md](06-backlog.md) | Marcos, critérios de saída, riscos, definição de pronto |

## 3. Decisões de arquitectura (ADR)

| ID | Decisão | Consequência |
|---|---|---|
| ADR-01 | Cliente **PWA** em React + Vite + TypeScript | Demo acessível por link, sem instalação |
| ADR-02 | API em **FastAPI** com **PostgreSQL** em todos os ambientes | Paridade dev/produção; sem SQLite |
| ADR-03 | Consulta de beneficiário atrás de uma interface (`BeneficiaryProvider`) | Trocar o simulado por um real não altera router, serviço nem UI |
| ADR-04 | Contrato **OpenAPI** como fonte única; tipos TypeScript gerados | Front e back não divergem; CI falha se o contrato mudar sem regenerar |
| ADR-05 | Resultado sempre explícito em quatro estados: `MATCH`, `MISMATCH`, `NOT_FOUND`, `UNAVAILABLE` | Falha de consulta nunca é apresentada como verificação bem-sucedida |
| ADR-06 | Só são aceites identificadores `DEMO-NNNN` | Impossível recolher identificadores reais |
| ADR-07 | Sem autenticação de utilizador; endpoints de administração por chave (`X-Admin-Key`) | Fluxo de demonstração sem fricção; métricas protegidas |
| ADR-08 | **Monorepo** (`apps/web`, `apps/api`, `docs`) | Uma PR altera contrato, API e UI em conjunto |
| ADR-09 | Nomes fornecidos pelo participante (`expected_name`) não são persistidos | Minimização de dados pessoais |
| ADR-10 | Erros da API em formato `application/problem+json` (RFC 9457) | Tratamento uniforme no cliente |

## 4. Convenções

### 4.1 Diagramas (Mermaid + UML)

| Diagrama UML | Sintaxe Mermaid | Onde se usa |
|---|---|---|
| Classes | `classDiagram` | Domínio e serviços (02) |
| Sequência | `sequenceDiagram` | Interacções front↔API↔provider↔BD (02, 03) |
| Estados | `stateDiagram-v2` | Ciclo de vida da consulta (02), fluxo de ecrãs (03) |
| Entidade-relação | `erDiagram` | Modelo de dados (02) |
| Actividade | `flowchart` | Fluxo principal (01) |
| Componentes | `flowchart` com estereótipos `«component»` | Arquitectura (01, 03) |
| Implantação | `flowchart` com estereótipos `«node»`, `«artifact»` | Infra (04) |

O Mermaid não tem sintaxe nativa para diagramas de componentes, implantação e actividade; usa-se `flowchart` com os estereótipos UML no texto dos nós.

### 4.2 Identificadores

| Prefixo | Significado |
|---|---|
| `RF-xx` | Requisito funcional (README §8) |
| `NFR-xx` | Requisito não funcional (01 §6) |
| `TC-xx` | Caso de teste (05) |
| `SC-xx` | Cenário de usabilidade (05) |
| `ADR-xx` | Decisão de arquitectura (este ficheiro) |
| `Q-xx` | Questão em aberto (§6) |
| `M-x` | Marco (06) |

### 4.3 Idioma

Documentação em português. Código, nomes de rotas, campos JSON e commits em inglês. Textos de interface em português (`pt-AO`).

## 5. Glossário

| Termo | Definição |
|---|---|
| Beneficiário | Pessoa ou entidade fictícia que recebe a transferência simulada |
| Identificador | Código `DEMO-NNNN` que identifica o beneficiário no protótipo |
| Consulta (*lookup*) | Pedido ao serviço para resolver um identificador e obter os dados do beneficiário |
| Provider | Implementação que resolve identificadores (simulada hoje; real no futuro) |
| Cenário | Situação de teste pré-definida (`SC-xx`) com resultado esperado |
| Execução de teste (*test run*) | Uma participação de um participante num cenário |
| Simulação | Qualquer operação do protótipo: nenhum dinheiro é movimentado e nenhum dado é real |

## 6. Questões em aberto

| ID | Questão | Impacto |
|---|---|---|
| Q-01 | Variante de controlo (fluxo sem consulta) para medir o "tempo acrescentado ao fluxo" (README §4.2) | Sem ela, só se mede o tempo absoluto por cenário |
| Q-02 | Hosting confirmado? Pressuposto: Vercel (web) + Render (API e Postgres, região UE) | Secção 04 |
| Q-03 | Aprovação da lista de nomes e instituições fictícias (02 §10) | Seed e cenários |
| Q-04 | Dimensão da amostra e valores-alvo das métricas (README §10) | Protocolo (05 §5) |
| Q-05 | Marca, nome final e ícones (o nome "ConfirmaPay" é provisório) | Manifest PWA, copy |
| Q-06 | Domínio da demo | CORS, CSP, manifest |
