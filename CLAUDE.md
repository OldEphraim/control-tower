# CLAUDE.md — Pilotbase Control Tower: Project Context

This is your **orientation document**. Read it once at the start of each session to understand what this project is and why it's built the way it is. Then open `STEPS.md`, find the current phase, and work through it sequentially.

Do not skip steps. Do not reorder phases. At the end of every phase, run the listed verification commands before moving on.

---

## What This Project Is

**Pilotbase Control Tower** is a full-stack demonstration app built to show deep familiarity with Pilotbase (formerly Flight Schedule Pro) — their exact technology stack, their product architecture, and their 2025–2026 strategic direction.

Pilotbase is the dominant US flight school management SaaS (1,400+ schools). They recently rebranded from "Flight Schedule Pro" to "Pilotbase" (February 2026) and are building AI-powered scheduling as their flagship new feature. This app is a working simulation of that ecosystem.

The simulated school is **Suncoast Aviation Academy**, located at **KTPA** (Tampa International Airport, Florida). Civil twilight at KTPA is approximately **06:30–19:45**.

---

## Repository Structure

```
pilotbase-control-tower/
├── CLAUDE.md                        ← You are here
├── STEPS.md                         ← Step-by-step implementation work
├── DECISION_LOG.md                  ← Maintained throughout development
├── README.md                        ← Public-facing project description
├── docker-compose.yml               ← Local SQL Server + PostgreSQL + MongoDB
├── .gitignore
├── .github/
│   └── workflows/
│       └── ci.yml
├── backend/
│   └── PilotbaseControlTower.Api/   ← .NET 8 Web API (C#)
│   └── PilotbaseControlTower.Tests/ ← xUnit test project
├── frontend-school/                 ← Angular 18 — school operator dashboard
└── frontend-pilot/                  ← React 18 + Vite — pilot-facing app
```

---

## The Three-Database Architecture

This is the single most important architectural decision in the project. Understand it before touching any code, and preserve it throughout.

**SQL Server** holds `LegacyFlightRecords` — historical flight data from before the platform migration. Pilotbase was founded in 2000 on the classic Microsoft stack (.NET + SQL Server). Their engineering environment is explicitly described as "mostly greenfield with some legacy." SQL Server *is* that legacy. New operational data never goes here — it goes to PostgreSQL. This models the brownfield-to-greenfield migration they are actively living.

**PostgreSQL** holds all new operational data: students, instructors, aircraft, schedule events, financing milestones. This is the "next gen" database — cloud-native, open-source, natural for Azure/AWS deployment.

**MongoDB** holds flight log documents. LogTen (acquired by Pilotbase/FSP in May 2022) is the world's most downloaded pilot logbook app. A flight logbook entry is a rich, variable-schema document — nested objects for conditions, approaches, landings, and instructor endorsements that differ per flight type. Not every flight has instrument approaches. Not every entry has an instructor signature. Multi-engine entries have fields single-engine entries don't. A relational table would be ugly here; MongoDB's flexible document model is the correct choice.

---

## The Two-Frontend Architecture

**Angular 18** (`frontend-school/`) is the **school operator dashboard**. Angular is Pilotbase's confirmed primary frontend framework. The school operator is their primary user persona — 1,400+ flight schools use this surface daily. This maps to the legacy FSP product.

**React 18** (`frontend-pilot/`) is the **pilot-facing Pilotbase app**. The Pilotbase platform launched February 2026 as a new product layer. Newer Pilotbase products use React. This maps to their new strategic product introduced with the rebrand.

Both frontends share a consistent API contract with the .NET 8 backend.

---

## Business Model Signals — Do Not Remove

These details demonstrate that real research was done on Pilotbase. They must survive the entire implementation:

1. **Sallie Mae milestone trigger** — `POST /api/financing/milestone` fires when a student completes a training milestone. Disbursements are tied to verifiable progress, not lump-sum enrollment. This is the specific innovation of Pilotbase's May 2025 Sallie Mae partnership.

2. **Part 141 vs. Part 61 logic** — Students are tagged by training type. Part 141 follows FAA-approved course outlines with mandatory stage checks. Part 61 is more flexible. The AI scheduling engine respects this. This is what separates Pilotbase from generic scheduling software.

3. **AOPA squawk-free metric** — The KPI dashboard shows the percentage of student flights on aircraft with zero open squawks. 53% of 2025 AOPA Flight Training Experience Award winners use Pilotbase. This metric is how they win those awards.

4. **LogTen document schema** — MongoDB logbook entries include: conditions (VFR/IFR, day/night, cross-country), flight times (PIC/SIC/dual/solo split), instrument approaches by type (ILS, RNAV, VOR), landing counts (day/night), instructor endorsements. Any pilot who has used LogTen will recognize this schema.

5. **Civil twilight awareness** — The AI scheduling prompt includes civil twilight times for KTPA. Flights cannot begin before civil twilight unless explicitly night training. This is a real constraint in Pilotbase's Intelligent Scheduling feature.

---

## Technology → Pilotbase Mapping

| Technology | This Project | Why Pilotbase Uses It |
|---|---|---|
| **C# / .NET 8** | Web API backend | Primary backend language since founding (2000) |
| **SQL Server** | Legacy flight records | Original database from the .NET/SQL Server era |
| **PostgreSQL** | New operational data | Cloud-native next-gen database |
| **MongoDB** | Logbook documents | LogTen (acquired 2022) is document-oriented by nature |
| **Angular 18** | School operator dashboard | Primary frontend framework for the FSP product |
| **React 18** | Pilot Pilotbase app | Newer product layer introduced with 2026 rebrand |
| **TypeScript** | Both frontends | Type safety across the full frontend layer |
| **Azure** | API hosting | Primary cloud (natural .NET ecosystem fit) |
| **AWS S3** | Logbook PDF exports | Secondary cloud, likely from Coradine/LogTen infra |
| **CI/CD** | GitHub Actions | Automated build, test, and deploy on push |

---

## Ports and Local Dev URLs

| Service | Port | URL |
|---|---|---|
| .NET 8 API | 5000 | http://localhost:5000 |
| Swagger UI | 5000 | http://localhost:5000/swagger |
| Angular app | 4200 | http://localhost:4200 |
| React app | 5173 | http://localhost:5173 |
| SQL Server | 1433 | localhost,1433 |
| PostgreSQL | 5432 | localhost:5432 |
| MongoDB | 27017 | localhost:27017 |

---

## DECISION_LOG.md

A `DECISION_LOG.md` must be maintained throughout development. Every time a non-trivial decision is made — a package choice, a schema design, a workaround for a compatibility issue, anything a future developer would ask "why did they do it this way?" — add an entry using this format:

```markdown
## [Phase X, Step Y.Z] — Decision Title
**Date**: YYYY-MM-DD
**Decision**: One sentence stating what was decided.
**Alternatives considered**: What else was evaluated.
**Rationale**: Why this choice was made.
**Tradeoff**: What was given up, or what risk this introduces.
```

---

## Before Starting Each Session

1. Run `docker-compose up -d` from the repo root and confirm all three containers are healthy.
2. Check `DECISION_LOG.md` for decisions from previous sessions that affect the current phase.
3. Open `STEPS.md` and find the current phase. Work through it top to bottom.
4. Run the verification commands at the end of the phase before stopping.