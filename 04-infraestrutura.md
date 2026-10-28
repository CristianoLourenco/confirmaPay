# 04 — Infra-estrutura

← [Índice](00-indice.md) · [Arquitectura](01-arquitectura.md) · [Back-end](02-backend.md) · [Front-end](03-frontend.md)

> Pressuposto por confirmar (Q-02): web na Vercel, API e Postgres na Render, região UE. A arquitectura não depende destes fornecedores.

## 1. Ambientes

| Ambiente | Finalidade | Web | API | Base de dados |
|---|---|---|---|---|
| `local` | Desenvolvimento | `pnpm dev` (Vite) | Docker Compose | Postgres 16 em Compose |
| `demo` | Demonstração e testes de usabilidade | Vercel (produção) | Render (contentor Docker) | Postgres gerido (Render) |

Não há ambiente de *staging* na v1. Pré-visualizações de PR do front-end (Vercel) são opcionais e exigem acrescentar a origem a `CORS_ORIGINS`.

## 2. Implantação

```mermaid
flowchart TB
    subgraph DEV["«device» Telemóvel ou PC do utilizador"]
        B["«execution environment»<br/>Browser + Service Worker"]
    end

    subgraph VC["«node» Vercel — CDN"]
        W["«artifact»<br/>web (estáticos Vite)"]
    end

    subgraph RN["«node» Render — região UE"]
        A["«artifact»<br/>api (contentor Docker)"]
        DB[("«database»<br/>PostgreSQL 16")]
    end

    B -->|"HTTPS GET (shell)"| W
    B -->|"HTTPS JSON /v1"| A
    A -->|"TLS"| DB
```

## 3. Docker

### 3.1 `apps/api/Dockerfile`

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY apps/api/pyproject.toml apps/api/uv.lock ./
RUN uv sync --frozen --no-dev
COPY apps/api/ .
EXPOSE 8000
CMD ["sh", "-c", "uv run alembic upgrade head && uv run python -m app.seed && uv run uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

O seed é idempotente (02 §10), por isso corre em cada arranque.

### 3.2 `docker-compose.yml` (local)

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: confirmapay
      POSTGRES_PASSWORD: confirmapay
      POSTGRES_DB: confirmapay
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U confirmapay"]
      interval: 5s
      retries: 10

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    environment:
      APP_ENV: local
      DATABASE_URL: postgresql+psycopg://confirmapay:confirmapay@db:5432/confirmapay
      CORS_ORIGINS: http://localhost:5173
      ADMIN_API_KEY: dev-admin-key
      SIMULATED_LATENCY_MS: "400"
    ports: ["8000:8000"]
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata: {}
```

### 3.3 `Makefile`

| Alvo | Acção |
|---|---|
| `make up` | `docker compose up -d --build` |
| `make down` | `docker compose down` |
| `make api-test` | Pytest contra Postgres local |
| `make web-dev` | `pnpm --filter web dev` |
| `make gen-api` | Exporta `openapi.json` e regenera `schema.d.ts` |
| `make e2e` | Playwright contra Compose e `vite preview` |
| `make seed` | Corre o seed |

## 4. Variáveis de ambiente

### 4.1 API

| Variável | Obrigatória | Por omissão | Descrição |
|---|---|---|---|
| `APP_ENV` | Sim | — | `local` ou `demo` |
| `DATABASE_URL` | Sim | — | `postgresql+psycopg://...` |
| `CORS_ORIGINS` | Sim | — | Origens permitidas, separadas por vírgula |
| `ADMIN_API_KEY` | Não | — | Se ausente, `/v1/admin/*` desactivado |
| `SIMULATED_LATENCY_MS` | Não | `400` | Latência artificial do provider |
| `SIMULATE_FAILURE_RATE` | Não | `0.0` | Probabilidade de falha aleatória (0–1) |
| `PROVIDER_TIMEOUT_S` | Não | `2.0` | Timeout do provider |
| `RATE_LIMIT` | Não | `60/minute` | Limite por IP |
| `LOG_LEVEL` | Não | `INFO` | Nível de logs |
| `PORT` | Não | `8000` | Porta (injectada pela Render) |

### 4.2 Web (variáveis de *build*, públicas)

| Variável | Descrição |
|---|---|
| `VITE_API_URL` | URL base da API |
| `VITE_APP_ENV` | `local` ou `demo` |
| `VITE_APP_VERSION` | Versão ou SHA do commit |

Nenhum segredo vai em `VITE_*`: tudo o que começa por `VITE_` fica visível no browser.

## 5. CI/CD

```mermaid
flowchart LR
    PR["Pull request"] --> CW["ci-web<br/>lint, typecheck, test, build"]
    PR --> CA["ci-api<br/>ruff, mypy, pytest, openapi, pip-audit"]
    CW --> E2E["e2e<br/>Compose + Playwright + axe"]
    CA --> E2E
    E2E --> MG["Merge em main"]
    MG --> DW["Deploy web<br/>Vercel"]
    MG --> DA["Deploy API<br/>Render deploy hook"]
    DW --> SM["Smoke test<br/>GET /v1/health"]
    DA --> SM
```

| Workflow | Disparo | Passos |
|---|---|---|
| `ci-web` | PR e `main`, `apps/web/**` | install `--frozen-lockfile`, lint, typecheck, `gen-api` + `git diff --exit-code`, testes, build |
| `ci-api` | PR e `main`, `apps/api/**` | Postgres de serviço, ruff, mypy, migrações, pytest com cobertura, export OpenAPI + `git diff --exit-code`, `pip-audit` |
| `e2e` | PR | Compose (API + Postgres), `vite preview`, Playwright, axe |
| `deploy` | `main`, após CI verde | Deploy hook da Render; Vercel por integração Git; smoke test |

### 5.1 `ci-api.yml` (esqueleto)

```yaml
name: ci-api
on:
  pull_request:
    paths: ["apps/api/**"]
  push:
    branches: [main]
    paths: ["apps/api/**"]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U test"
          --health-interval 5s
          --health-retries 10
    env:
      APP_ENV: local
      DATABASE_URL: postgresql+psycopg://test:test@localhost:5432/test
      CORS_ORIGINS: http://localhost:5173
    defaults:
      run:
        working-directory: apps/api
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --frozen
      - run: uv run ruff check . && uv run ruff format --check .
      - run: uv run mypy app
      - run: uv run alembic upgrade head
      - run: uv run pytest --cov=app
      - run: uv run python -m app.export_openapi && git diff --exit-code openapi.json
      - run: uv run pip-audit
```

### 5.2 `ci-web.yml` (esqueleto)

```yaml
name: ci-web
on:
  pull_request:
    paths: ["apps/web/**", "apps/api/openapi.json"]
  push:
    branches: [main]
    paths: ["apps/web/**", "apps/api/openapi.json"]

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: apps/web
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm gen:api && git diff --exit-code src/shared/api/schema.d.ts
      - run: pnpm test
      - run: pnpm build
```

### 5.3 Regras de repositório

| Item | Regra |
|---|---|
| Ramos | *Trunk-based*: ramos curtos a partir de `main` |
| Commits | Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`) |
| `main` | Protegido: PR obrigatória, CI verde, sem *push* directo |
| Segredos | GitHub Secrets e painéis da Vercel/Render; nunca no repositório |

## 6. Cabeçalhos de segurança do web (`vercel.json`)

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Content-Security-Policy", "value": "default-src 'self'; connect-src 'self' https://API_ORIGIN; img-src 'self' data:; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "no-referrer" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
        { "key": "X-Robots-Tag", "value": "noindex" }
      ]
    }
  ]
}
```

Substituir `API_ORIGIN` pelo domínio real da API (Q-06). O Service Worker e o manifest são servidos da própria origem.

## 7. Observabilidade

| Tema | Solução |
|---|---|
| Logs | JSON para *stdout*; consultados no painel da Render |
| Saúde | `GET /v1/health` devolve estado e versão, e executa `SELECT 1` (503 se a BD falhar) |
| Disponibilidade | Monitor externo (UptimeRobot ou equivalente) a cada 5 minutos |
| Erros | Sentry opcional com `send_default_pii=False` e filtro que remove corpos de pedido |
| Métricas de teste | `GET /v1/admin/metrics` e exportação CSV (02 §3.7) |

## 8. Dados: retenção, cópias e reposição

| Tema | Regra |
|---|---|
| Natureza dos dados | Fictícios ou anónimos (`P-NNN`); risco baixo |
| Cópias | Backups diários do Postgres gerido, se o plano os incluir |
| Retenção | Eliminar `test_runs`, `lookups`, `decisions` e `survey_responses` 90 dias após o fim dos testes (`python -m app.maintenance purge --days 90`) |
| Repor a demo | `python -m app.seed --reset` apaga dados de teste e repõe os beneficiários |
| Exportar antes de repor | `GET /v1/admin/export.csv` |

## 9. Runbook

| Situação | Acção |
|---|---|
| Deploy | Merge em `main`; CI verde dispara os deploys; verificar o smoke test |
| Rollback web | Vercel: promover o deploy anterior |
| Rollback API | Render: *redeploy* da versão anterior; migrações devem ser compatíveis com a versão anterior |
| API lenta ou a falhar | Ver `/v1/health`, logs por `request_id`, estado da BD |
| Limite de pedidos a bloquear testes | Aumentar `RATE_LIMIT` e reiniciar |
| Simular falha na demonstração | Usar `DEMO-9999` ou definir `SIMULATE_FAILURE_RATE` |
| Rodar `ADMIN_API_KEY` | Alterar na Render e reiniciar |

## 10. Lista de verificação antes de construir

- [ ] Repositório criado e `main` protegido
- [ ] Contas Vercel e Render criadas (Q-02)
- [ ] Domínio ou subdomínios definidos (Q-06)
- [ ] Segredos criados: `ADMIN_API_KEY`, deploy hook
- [ ] Figma com os cinco ecrãs e quatro estados (03 §6)
- [ ] Lista de nomes fictícios aprovada (Q-03)
- [ ] Protocolo de teste e consentimento revistos (05 §5)
- [ ] Decisão sobre variante de controlo (Q-01)
