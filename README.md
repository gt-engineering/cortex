# Cortex Workspace

Cortex is a Bun workspace managed with Turborepo.

Active projects:

- `apps/auth-api` — NestJS authentication backend
- `apps/auth-api-e2e` — authentication end-to-end tests
- `apps/web` — Next.js web application
- `libs/auth-nest` and `libs/encoding` — shared workspace packages

## Local development

Install Bun and Docker, then:

```bash
bun install
cp .env.example .env.local  # first-time setup
cp .env.local .env          # activate the local profile
docker compose up -d postgres redis
bun run dev
```

The web application runs at `http://localhost:3000`; the auth API runs at
`http://localhost:3001/api`. The local Compose file provides only PostgreSQL
and Redis. Application containers are built by CI and are not maintained in
the local Compose stack.

Common commands:

```bash
bun run lint
bun run test
bun run build
bun run e2e
bun run db:generate
bun run db:migrate
bun run db:seed
```

Auth E2E tests use Testcontainers and start isolated PostgreSQL and Redis
containers. Optional local configuration can be copied from
`apps/auth-api-e2e/.env.example`.

## CI/CD

Pull requests validate the affected Turborepo package graph and build affected
Docker images without publishing. A passing push to `main` publishes only the
affected `linux/amd64` images:

- `ghcr.io/gena-tokarev/cortex-auth-api`
- `ghcr.io/gena-tokarev/cortex-web`

Each image receives an immutable `sha-<commit>` tag and the movable `main` tag.
Argo CD Image Updater resolves `main` to an immutable digest and commits changed
service references to `gena-tokarev/infra`. Argo CD then reconciles k3s. GitHub
Actions has no infra, VPS, SSH, Kubernetes, or Argo CD credentials.

The development site is `https://cortex-dev.podolog-warsaw.pl`. Runtime
secrets are provisioned from the Ansible Vault in the private infra repository;
they are not built into either image.

Health endpoints:

- `GET /api/health/live` checks only that the API process is alive.
- `GET /api/health` checks PostgreSQL and Redis readiness.

Before reverting a development release, pause that image's updater rule so it
does not immediately select the same `main` digest again. Database migrations
are not reversed and must remain backward-compatible with the preceding
application version.
