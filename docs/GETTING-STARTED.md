# Getting started

This guide runs REALM locally with PostgreSQL. It assumes a Linux/macOS development machine; Windows users may use Docker or an equivalent PostgreSQL 17 setup.

## 1. Install prerequisites

- Node.js 22.13 or newer;
- PostgreSQL 17 with the `pgvector` extension;
- Docker, for the disposable integration-test cluster;
- Git.

A live model provider is optional for deterministic tests. For interactive generation, configure an OpenAI-compatible provider through the local settings page or `.env.local`.

## 2. Install dependencies

```bash
npm ci
cp .env.example .env.local
```

Review `.env.local` before starting. Keep it untracked and use loopback database addresses for local development.

## 3. Bootstrap PostgreSQL

```bash
npm run db:postgres:bootstrap
```

The bootstrap path starts the local PostgreSQL service, applies migrations, and loads the demo seed. If you manage PostgreSQL yourself, set the connection variables in `.env.local` and run the migration/seed commands separately.

## 4. Start REALM

```bash
npm run dev
```

Open the printed local URL. The default is `http://127.0.0.1:9999`.

## 5. Verify the checkout

```bash
npm test
npm run lint
npm run typecheck
```

The full test command uses an isolated scratch PostgreSQL cluster for integration coverage. It should not connect to a personal or production database.

## Troubleshooting

- **PostgreSQL connection errors:** check that PostgreSQL 17 is running, `pgvector` is installed, and the three connection variables point to the same local database.
- **Model errors:** verify provider settings and the local API endpoint. A missing key or authentication failure should be reported as a configuration failure, not retried forever.
- **Stale test data:** stop the local development database and use a disposable scratch cluster for tests. Do not delete data from a database that contains personal worlds without a backup and an explicit decision.
