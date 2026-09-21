# Configuration

REALM reads local environment values from `.env.local` when the command supports Node's env-file loading. `.env.local` is intentionally ignored by Git.

## Database

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | One-shot migration/bootstrap connection |
| `REALM_RUNTIME_DATABASE_URL` | Least-privileged application runtime connection |
| `REALM_TRANSFER_DATABASE_URL` | Optional least-privileged export/import connection |
| `REALM_POSTGRES_BIN` | Optional directory containing PostgreSQL tools |
|
For local development, keep these connections on `127.0.0.1`, `localhost`, or `::1`. Do not paste database URLs containing credentials into issues, logs, screenshots, or model prompts.

## Server binding

| Variable | Purpose |
|---|---|
| `HOST_BIND` | Bind address; loopback is the safe default |
| `PORT` | Local HTTP port; defaults to `9999` |
| `REALM_SESSION_SECRET` | Optional session signing secret; a stable per-install key file is generated when unset |

REALM signs in with an account name plus an optional password (see `docs/development/DEPLOY-AUTH.md`). A leftover `REALM_ACCESS_TOKEN` is ignored — it is no longer a credential or a startup prerequisite.

An explicit LAN bind is an operator decision. A bind address does not provide authentication, encryption, or public-service safety by itself.

## Model providers

| Variable | Purpose |
|---|---|
| `REALM_MODEL_PROVIDER` | Default provider profile |
| `REALM_MODEL_ID` | Optional model identifier |
| `REALM_MODEL_BASE_URL` | Optional OpenAI-compatible endpoint |
| `REALM_MODEL_API_KEY` | Local-only provider key |
| `REALM_OPENROUTER_*` | Optional OpenRouter profile values |
| `REALM_DEEPSEEK_API_KEY` | Optional local provider key |
| `REALM_KIMI_API_KEY` | Optional local provider key |

The settings page is the preferred way to configure profiles for interactive use. Leave provider keys empty in `.env.example` and never commit real values.

## Propagation worker

`REALM_PROPAGATION_WORKSPACES` optionally selects local workspace identifiers for the propagation worker. Keep operational workspace identifiers out of public documentation and source control.
