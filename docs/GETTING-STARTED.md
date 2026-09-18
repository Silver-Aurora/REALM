# Getting started

This guide runs REALM locally. It assumes Node.js 22.13+ and Git; PostgreSQL/pgvector can be installed locally or supplied through the Docker fallback. A live model provider is optional for deterministic tests.

## 1. Bootstrap and start

```bash
node scripts/setup-web.mjs
```

The CLI asks before installing missing prerequisites, creates `.env.local`, installs npm dependencies, starts local PostgreSQL or Docker pgvector, applies migrations, seeds the demo world, starts REALM, and opens the browser. Use `node scripts/setup-web.mjs --check` for a read-only probe.

See [Web bootstrap](./WEB-BOOTSTRAP.md) for the supported modes and safety boundaries.


```bash
npm test
npm run lint
npm run typecheck
```

The full test command uses an isolated scratch PostgreSQL cluster for integration coverage. It should not connect to a personal or production database.

## Desktop and source deployment

For unsigned macOS/Windows preview packages, see [Desktop installation](./DESKTOP-INSTALLATION.md). For a source checkout, the current web path still requires Node.js and a local PostgreSQL/pgvector setup; it is not yet a clean-machine one-click installer.

## Troubleshooting

- **PostgreSQL connection errors:** check that PostgreSQL 17 is running, `pgvector` is installed, and the three connection variables point to the same local database.
- **Model errors:** verify provider settings and the local API endpoint. A missing key or authentication failure should be reported as a configuration failure, not retried forever.
- **Stale test data:** stop the local development database and use a disposable scratch cluster for tests. Do not delete data from a database that contains personal worlds without a backup and an explicit decision.
