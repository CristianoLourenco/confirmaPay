# 05 — Testes e métricas

← [Índice](00-indice.md)

## 1. Estratégia

| Nível | Ferramenta | Onde corre |
|---|---|---|
| Unitário (domínio) | Pytest | CI `ci-api` |
| Integração (API + Postgres) | Pytest + httpx | CI `ci-api` |
| Contrato do provider | Pytest (suite parametrizada) | CI `ci-api` |
| Unitário e componente (web) | Vitest, Testing Library, MSW | CI `ci-web` |
| E2E | Playwright | CI `e2e` |
| Acessibilidade | `@axe-core/playwright` | CI `e2e` |
| PWA e desempenho | Lighthouse CI | CI `e2e` |

## 2. Rastreabilidade RF → componentes → testes

| Requisito | Back-end | Front-end | Testes |
|---|---|---|---|
| RF-01 Introduzir identificador | `POST /v1/lookups`, validação | `LookupForm` | TC-01, TC-02 |
| RF-02 Consultar dados simulados | `SimulatedProvider`, `beneficiaries` | `useCreateLookup` | TC-03 |
| RF-03 Apresentar correspondência | Estado `MATCH` | `MatchView` | TC-03, TC-04 |
| RF-04 Identificador inexistente | Estado `NOT_FOUND` | `NotFoundView` | TC-05 |
| RF-05 Divergência | Estado `MISMATCH`, normalização | `MismatchView` | TC-06, TC-12 |
| RF-06 Indisponibilidade | Estado `UNAVAILABLE`, timeout | `UnavailableView`, erro local | TC-07, TC-08, TC-09 |
| RF-07 Confirmar ou cancelar | `POST .../decision`, regras 4.4 | `DecisionBar`, diálogo | TC-10, TC-11, TC-12 |
| RF-08 Não executar pagamentos | Sem rotas de pagamento nem saída de rede | Sem chamadas externas | TC-13 |
| RF-09 Identificar a simulação | `is_simulation`, `disclaimer` | `SimulationBanner` | TC-14 |

## 3. Casos de teste

| ID | Req. | Tipo | Dado e acção | Resultado esperado |
|---|---|---|---|---|
| TC-01 | RF-01 | E2E | Introduzir `DEMO-1001` e submeter | Pedido enviado; navega para o resultado |
| TC-02 | RF-01, NFR-03 | API | `identifier = "923456789"` | 422 `IDENTIFIER_FORMAT_INVALID`; nada guardado em `lookups` |
| TC-03 | RF-02, RF-03 | API | `DEMO-1001` sem `expected_name` | 201 `MATCH`, `NOT_PERFORMED`, `is_simulation = true` |
| TC-04 | RF-03 | API | `DEMO-1001` com `expected_name = "maria exemplo  teste"` | `MATCH`, `PERFORMED` (insensível a maiúsculas, acentos e espaços) |
| TC-05 | RF-04 | API | `DEMO-4040` | `NOT_FOUND`, `beneficiary = null`, `allowed_decisions = ["CANCELLED"]` |
| TC-06 | RF-05 | API | `DEMO-1002` com `expected_name = "Joana Fictícia Santos"` | `MISMATCH`, `beneficiary` preenchido |
| TC-07 | RF-06 | API | `DEMO-9999` | `UNAVAILABLE`; nunca `MATCH` |
| TC-08 | RF-06, NFR-02 | API | Provider substituído por um que demora mais que `PROVIDER_TIMEOUT_S` | `UNAVAILABLE` |
| TC-09 | RF-06 | E2E | Browser offline ao submeter | `UnavailableView`; botão "Confirmar" ausente |
| TC-10 | RF-07 | API | `CONFIRMED` sobre `MATCH`, depois repetir | 1.º: 200; 2.º: 409 `DECISION_ALREADY_RECORDED` |
| TC-11 | RF-07 | API | `CONFIRMED` sobre `NOT_FOUND` e sobre `UNAVAILABLE` | 409 `DECISION_NOT_ALLOWED` |
| TC-12 | RF-05, RF-07 | E2E | `MISMATCH` e clicar "Confirmar mesmo assim" | Abre diálogo; só regista após confirmação no diálogo |
| TC-13 | RF-08 | API + E2E | Bloquear sockets (excepto BD) nos testes; Playwright falha em pedidos a domínios externos | Nenhuma chamada externa; não existem rotas de pagamento |
| TC-14 | RF-09 | E2E + API | Percorrer todas as rotas; inspeccionar respostas | Banner visível em todas as rotas; `is_simulation = true` nas respostas |
| TC-15 | NFR-03 | API | Pedido com `expected_name` e captura de logs | O nome não aparece em logs nem na BD |
| TC-16 | NFR-07 | API | Provocar 422 e 404 | `request_id` no corpo e em `X-Request-ID` |
| TC-17 | NFR-04 | E2E | axe nos quatro estados, formulário e conclusão | Sem violações críticas ou graves |
| TC-18 | NFR-05 | CI | Lighthouse móvel na página inicial e no formulário | Desempenho ≥ 90 e PWA ≥ 90 |
| TC-19 | NFR-08 | API | Suite de contrato executada com `SimulatedProvider` e com um provider falso | Ambos passam: `None` só para inexistente; falhas lançam `ProviderUnavailable` |
| TC-20 | ADR-07 | API | `/v1/admin/metrics` sem chave, com chave errada, com chave correcta | 401, 401, 200 |
| TC-21 | 02 §3.6 | API | Dois inquéritos para a mesma execução | 2.º: 409 `SURVEY_ALREADY_RECORDED` |
| TC-22 | NFR-01 | API | 200 pedidos sequenciais com `SIMULATED_LATENCY_MS=0` | p95 ≤ 300 ms |
| TC-23 | NFR-06 | API | `GET /v1/health` com BD ligada e desligada | 200; 503 |

## 4. Testes de contrato do provider

Suite parametrizada por implementação de `BeneficiaryProvider`:

1. Identificador existente devolve `ProviderResult` com `beneficiary` e máscara correcta.
2. Identificador inexistente devolve `beneficiary = None`.
3. Falha de infra-estrutura lança `ProviderUnavailable` (nunca `None`).
4. `resolve` não altera dados.

## 5. Cenários de usabilidade

O catálogo vive em `app/scenarios.py`. `expected_status` e `expected_decision` são internos e nunca expostos pela API.

| Código | Título | Identificador | Nome esperado | Estado | Decisão correcta |
|---|---|---|---|---|---|
| SC-01 | Destinatário correcto | `DEMO-1001` | Maria Exemplo Teste | `MATCH` | `CONFIRMED` |
| SC-02 | Nome diferente do esperado | `DEMO-1002` | Joana Fictícia Santos | `MISMATCH` | `CANCELLED` |
| SC-03 | Erro de dígito (pretendia `DEMO-1004`) | `DEMO-1005` | Ana Simulada Costa | `MISMATCH` | `CANCELLED` |
| SC-04 | Beneficiário inexistente | `DEMO-4040` | Pedro Inexistente Demo | `NOT_FOUND` | `CANCELLED` |
| SC-05 | Consulta indisponível | `DEMO-9999` | Maria Exemplo Teste | `UNAVAILABLE` | `CANCELLED` |
| SC-06 | Nome coincide, mas há pretexto suspeito | `DEMO-1003` | Empresa Demo Lda | `MATCH` | `CANCELLED` |

Narrativa do `SC-06`: o participante recebe uma mensagem urgente a pedir um pagamento para uma "conta nova". O nome coincide, mas o pedido tem sinais de alerta. Serve para medir falsos sinais de confiança (§6).

### 5.1 Protocolo

| Item | Regra |
|---|---|
| Participantes | Código anónimo `P-NNN`; sem nome, telefone ou identificação |
| Consentimento | Informado antes do teste: dados fictícios, tarefa simulada, direito de parar a qualquer momento |
| Ordem dos cenários | Rotativa entre participantes para reduzir efeito de ordem |
| Tarefa | Ler a instrução do cenário, usar o protótipo, decidir |
| Inquérito | Duas perguntas após cada cenário (§6) |
| Dados reais | Proibidos: o sistema rejeita qualquer identificador fora de `DEMO-NNNN` |
| Amostra | A definir (Q-04) |

## 6. Métricas

Alinhadas com o README §10. Os valores-alvo só são definidos depois do protocolo e da amostra (Q-04). Os resultados não provam redução de fraude em produção.

| Métrica | Definição | Fonte |
|---|---|---|
| Detecção de divergências | Nos `SC-02` e `SC-03`: execuções com decisão `CANCELLED` ÷ execuções com decisão | `decisions`, cenário |
| Decisão correcta | Execuções com `decision = expected_decision` ÷ execuções com decisão, por cenário | `decisions`, catálogo |
| Tempo de tarefa | Mediana e P90 de `decided_at − started_at`, por cenário | `test_runs`, `decisions` |
| Compreensão | Respostas com `understood_result = true` ÷ respostas | `survey_responses` |
| Falsos sinais de confiança | Respostas com `match_implies_safe = true` ÷ respostas (relevante em `SC-01` e `SC-06`) | `survey_responses` |
| Tempo acrescentado ao fluxo | Diferença face a uma variante de controlo | Depende de Q-01 |

Perguntas do inquérito:

1. "Percebeu o que o ecrã de resultado lhe estava a dizer?" → `understood_result`.
2. "Se o nome coincide, o pagamento é seguro?" → `match_implies_safe` (resposta "sim" = `true`).

Regras de reporte: indicar sempre o número de participantes (n) por cenário; não extrapolar para o sistema real.

## 7. Critérios de aceitação da v1

- TC-01 a TC-23 passam no CI.
- Nenhuma violação crítica no axe; Lighthouse móvel ≥ 90.
- Todas as rotas mostram o banner de simulação.
- Exportação CSV de uma execução completa (cenário + decisão + inquérito) confere com a BD.
- Documentação sincronizada com `openapi.json`.
