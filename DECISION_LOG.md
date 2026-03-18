# Decision Log

This file tracks every non-trivial architectural and implementation decision made during development of Pilotbase Control Tower.

Format:
## [Phase X, Step Y.Z] — Decision Title
**Date**: YYYY-MM-DD
**Decision**: One sentence.
**Alternatives considered**: ...
**Rationale**: ...
**Tradeoff**: ...

---

## [Phase 0, Step 0.1] — Three-database architecture
**Date**: 2026-03-18
**Decision**: Use SQL Server for legacy data, PostgreSQL for operational data, and MongoDB for logbook documents.
**Alternatives considered**: Single PostgreSQL database with JSONB columns for logbook data; single SQL Server database for everything.
**Rationale**: Mirrors Pilotbase's actual architecture. SQL Server is their legacy database (founded 2000, .NET/SQL Server era). PostgreSQL is their modern cloud-native choice. MongoDB matches the document-oriented shape of LogTen logbook entries (acquired 2022) — entries have variable nested schemas (some have instrument approaches, some don't; multi-engine entries differ from single-engine).
**Tradeoff**: Three database connections increase local setup complexity and add infrastructure overhead. Justified because the multi-database story is itself a demonstration of the architecture.

## [Phase 0, Step 0.1b] — Angular for school operator app, React for pilot app
**Date**: 2026-03-18
**Decision**: Use Angular 18 for the school operator dashboard and React 18 for the pilot-facing Pilotbase app.
**Alternatives considered**: Single React app for both user personas; single Angular app for both.
**Rationale**: Mirrors Pilotbase's actual frontend split. Angular is their confirmed primary framework for the legacy FSP product used by 1,400+ flight schools daily. The Pilotbase platform (launched February 2026) is a new product layer — newer Pilotbase products use React. Having both frameworks in the same repo demonstrates familiarity with both and reflects the real architectural tension they are managing after the rebrand.
**Tradeoff**: Two separate frontend projects increase build complexity and double the npm dependency surface. Justified for the same reason as the three-database choice: the split is itself the signal.

## [Phase 0, Step 0.3] — Apple Silicon platform override for SQL Server
**Date**: 2026-03-18
**Decision**: Add `platform: linux/amd64` to the `sqlserver` service in docker-compose.yml.
**Alternatives considered**: Using an ARM-native SQL Server image; using a third-party SQL Server compatible container.
**Rationale**: The official Microsoft SQL Server 2022 image (`mcr.microsoft.com/mssql/server:2022-latest`) does not publish a native ARM64 image. Docker Desktop on Apple Silicon (M-series) can run amd64 images via Rosetta 2 emulation. This machine was confirmed as `arm64` at setup time (`uname -m` output: `arm64`).
**Tradeoff**: Rosetta 2 emulation adds a small performance overhead for SQL Server queries in local dev. Acceptable because SQL Server is used only for legacy read queries in this project — not for high-throughput operations.

## [Phase 0, Step 0.3b] — SQL Server docker exec password quoting workaround
**Date**: 2026-03-18
**Decision**: When running sqlcmd via `docker exec` from a macOS/zsh host, read the password from the container's env var rather than passing it as a literal argument.
**Alternatives considered**: Escaping `!` with backslash; using single quotes; wrapping in `bash -c` with various quoting strategies.
**Rationale**: The `!` character in `PilotbaseDev123!` is consumed by zsh history expansion when passed as a shell argument to `docker exec`, even inside single quotes in some contexts. Reading via `PASS=$(printenv SA_PASSWORD)` inside the container avoids the host shell entirely.
**Tradeoff**: The STEPS.md verification command is not runnable verbatim from zsh. The workaround is: `docker exec pct-sqlserver bash -c 'PASS=$(printenv SA_PASSWORD) && /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$PASS" -Q "SELECT '"'"'SQL Server OK'"'"'" -No'`

## [Phase 0, Pre-Phase 1] — Externalize credentials to .env file
**Date**: 2026-03-18
**Decision**: Extract all database credentials and API keys from docker-compose.yml into a `.env` file, with a `.env.example` committed to the repo as a template.
**Alternatives considered**: Leaving credentials hardcoded in docker-compose.yml (acceptable for a purely local-dev demo); using Docker secrets (overkill for local dev).
**Rationale**: Hardcoded credentials in source control are a bad habit regardless of project purpose. The `.env` + `.env.example` pattern is the standard for Docker Compose projects, keeps secrets out of git history, and makes the template self-documenting for anyone cloning the repo. The `.gitignore` already had `.env` covered and `!.env.example` whitelisted.
**Tradeoff**: Adds one setup step for new developers (copy `.env.example` to `.env` and fill in values). Mitigated by the example file documenting every required variable.
