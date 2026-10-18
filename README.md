# Vendor Risk Copilot

A full-stack **procurement & third-party-risk** application with an **AI agent** that
drafts risk assessments and a **human approval** step. Built as a portfolio project to
demonstrate the stack used in modern procurement/risk technology teams: React + TypeScript,
a secure REST API, SQL, Microsoft Azure, SSO with role-based access, and a Model Context
Protocol (MCP) agentic workflow.

**Author:** Nousheen Fathima · [LinkedIn](https://linkedin.com/in/nousheen)

> This is a runnable demonstrator, not a production system. The data store is in-memory
> and the MCP server ships a small sample directory so it runs with no cloud account.
> The auth, RBAC, approval gate, risk engine, and Azure IaC are real. See
> [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for what is demo vs production.

## What it shows

| Requirement (from a typical procurement/risk SWE role) | Where it's demonstrated |
| --- | --- |
| React + backend services + SQL | `packages/web` (React/TS), `packages/api` (Express), `packages/shared` + Prisma seam |
| Deploy & operate in Azure | `infra/main.bicep` (App Service, PostgreSQL, Key Vault, App Insights) |
| Secure REST APIs, enterprise integration | `packages/api` routes + middleware |
| SSO, authentication, authorization, RBAC | `packages/api/src/auth.ts`, `packages/shared/src/rbac.ts` |
| AI-enabled app, MCP integration, agentic workflow | `packages/mcp`, approval gate in `packages/api` |
| Automated testing, CI/CD | `vitest` suites, `.github/workflows/ci.yml` |

## Monorepo layout

```
packages/
  shared/   deterministic risk engine + RBAC rules (framework-free, unit-tested)
  api/      Express REST API: Entra ID auth, RBAC, the human approval gate
  mcp/      Model Context Protocol server: lookup / score / draft tools (no approvals)
  web/      React + TypeScript dashboard and approvals UI
infra/      Azure Bicep
docs/       ARCHITECTURE.md
```

## Run it locally

Requires Node 22+.

```bash
npm install
npm run build --workspace packages/shared   # other packages import its built output

# API (dev auth mode: present an identity via a header instead of Entra)
VRC_AUTH_MODE=dev npm run dev:api            # http://localhost:3001

# Web (proxies /api to the API)
npm run dev:web                              # http://localhost:5173
```

Try the API in dev mode:

```bash
# list vendors as a requester
curl localhost:3001/api/vendors \
  -H 'X-Dev-User: {"id":"u1","email":"r@x.co","roles":["requester"]}'

# draft (reviewer) then approve (reviewer)
curl -XPOST localhost:3001/api/vendors/v-1001/assessment \
  -H 'X-Dev-User: {"id":"rev1","email":"v@x.co","roles":["reviewer"]}' \
  -H 'Content-Type: application/json' -d '{"draftedBy":"agent"}'
```

## Run the MCP server

```bash
npm run build --workspace packages/shared
npm run -w packages/mcp build && npm run mcp
```

Point any MCP client (e.g. Claude Desktop) at the built `packages/mcp/dist/server.js`
over stdio. It exposes `lookup_vendor`, `score_vendor_risk`, and `draft_assessment`.

## Test & typecheck

```bash
npm run build --workspace packages/shared
npm run typecheck     # all packages
npm test              # vitest across packages
```

## Deploy to Azure

```bash
az group create -n vrc-rg -l canadacentral
az deployment group create -g vrc-rg -f infra/main.bicep \
  -p namePrefix=vrc pgAdminUser=vrcadmin pgAdminPassword=<strong-password>
```

Then set `VRC_AUTH_MODE=entra`, register the app in Microsoft Entra ID, and configure
the `VRC_ENTRA_*` settings. CI (`.github/workflows/ci.yml`) typechecks, builds, tests,
and validates the Bicep on every push.

## Roadmap (honest next steps)

- Replace the in-memory `Store` with a Prisma schema against `DATABASE_URL`.
- Wire MSAL into the React app for real Entra sign-in.
- Add an agent host that calls the MCP tools end-to-end and writes drafts via the API.
- Add integration tests and a deploy workflow (OIDC to Azure).
