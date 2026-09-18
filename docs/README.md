# REALM documentation

Documents for players come first; engineering references are kept in their own section.

## 开始游玩 / Start here

1. [Web bootstrap](./WEB-BOOTSTRAP.md) — one-command launcher for macOS, Windows, and Linux
2. [Getting started](./GETTING-STARTED.md) — verify the checkout and run the test suite
3. [Configuration](./CONFIGURATION.md) — local database, server binding, and model provider settings
4. [Self-hosting boundaries](./SELF-HOSTING.md) — safe local and LAN operating boundaries
5. [Desktop installation](./DESKTOP-INSTALLATION.md) — unsigned preview packages
6. [System design](./architecture/SYSTEM-DESIGN.md) — how the world, memory, and rules actually work

## 工程参考 / Engineering references

These are working documents for contributors. They record design decisions and internal contracts; they are not player guides.

- [Technical architecture](./architecture/TECHNICAL-ARCHITECTURE.md) — engineering design review (module boundaries, concurrency, contracts)
- [PostgreSQL runtime contract](./architecture/POSTGRESQL-RUNTIME-CONTRACT.md) — storage and migration invariants
- [Local Record API](./architecture/LOCAL-RECORD-API.md) — record-level runtime interface
- [Model and memory runtime](./architecture/MODEL-AND-MEMORY-RUNTIME.md) — inference and memory pipeline notes
