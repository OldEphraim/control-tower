# Pilotbase Control Tower

A full-stack AI-powered flight school management simulation built to demonstrate deep familiarity with [Pilotbase](https://www.flightschedulepro.com/) (formerly Flight Schedule Pro) — their technology stack, product architecture, and current strategic direction.

---

## Why This Exists

Pilotbase is the dominant US flight school management SaaS platform, serving 1,400+ flight schools and recently rebranding (February 25, 2026) from Flight Schedule Pro to position itself as the world's first "Pilot Experience Platform." Their product has evolved from scheduling software into a full-lifecycle ecosystem connecting training management, digital logbooks, student financing, and career placement.

This project is a working simulation of that ecosystem — built in their actual tech stack, using the same architectural decisions they've made, and demonstrating the specific product bets they're making in 2025–2026.

---

## What It Demonstrates

**Intelligent Scheduling** — Pilotbase's flagship 2024 AI feature. Given a set of students (at various Part 141/61 curriculum stages), instructors (with ratings and availability windows), and aircraft (with maintenance schedules), an LLM generates an optimized weekly schedule with natural-language reasoning — explaining *why* it prioritized each booking and flagging risks like upcoming checkrides or squawk-grounded aircraft.

**Three-database architecture** — SQL Server for legacy historical flight records (simulating the "brownfield-to-greenfield" migration they're actively living), PostgreSQL for new operational data (students, instructors, aircraft, scheduling events), and MongoDB for flight log documents (mirroring the LogTen logbook data model they acquired in 2022).

**Dual-frontend product split** — An Angular app for the **flight school operator** (the FSP side: scheduling board, student roster, billing, reporting) and a React app for the **pilot** (the Pilotbase side: personal logbook, training progress, financing milestones, career pathway). This mirrors the actual product architecture Pilotbase announced with their February 2026 rebrand.

**The Sallie Mae financing integration** — When a student completes a curriculum milestone (first solo, instrument checkride, etc.), the system fires a `FinancingMilestoneEvent` that simulates notifying the lender for progress-based loan disbursement — the specific innovation of Pilotbase's May 2025 Sallie Mae partnership.

**The LogTen document model** — Flight log documents stored in MongoDB with a schema that maps to what LogTen actually records: aircraft type/registration, route, conditions (VFR/IFR, day/night), approaches, landings, hobbs/tach time, instructor endorsements. Exported as PDF to AWS S3.

**Part 141 vs. Part 61 curriculum logic** — Students are tagged by training type, which affects curriculum progression, stage check requirements, and scheduling priority rules — the regulatory distinction that separates Pilotbase from generic scheduling tools.

---

## Architecture

```
pilotbase-control-tower/
├── backend/                         # .NET 8 Web API (C#)
│   └── PilotbaseControlTower.Api/
│       ├── Controllers/             # REST endpoints
│       ├── Models/                  # Domain entities
│       ├── Data/                    # EF Core contexts (SQL Server + Postgres)
│       ├── Services/                # Business logic + AI scheduling
│       └── Mongo/                   # MongoDB collections (LogTen-style logbooks)
├── frontend-school/                 # Angular 18 — school operator dashboard
│   └── src/
│       ├── app/schedule/            # Weekly scheduling board
│       ├── app/students/            # Student roster + curriculum tracking
│       ├── app/instructors/         # Instructor management
│       ├── app/billing/             # Financing milestones + invoicing
│       └── app/reporting/           # KPI dashboard (AOPA-style metrics)
├── frontend-pilot/                  # React 18 + Vite — pilot-facing Pilotbase app
│   └── src/
│       ├── pages/Dashboard/         # Hours summary + next milestone
│       ├── pages/Logbook/           # LogTen-style flight log (MongoDB)
│       ├── pages/Training/          # Curriculum progress + stage checks
│       └── pages/Career/            # Hours-to-ATP tracker + job match simulation
├── docker-compose.yml               # Local SQL Server + Postgres + MongoDB
└── .github/workflows/ci.yml         # GitHub Actions CI/CD pipeline
```

### Technology → Pilotbase Mapping

| Technology | This Project | Why Pilotbase Uses It |
|---|---|---|
| **C# / .NET 8** | Web API | Primary backend language since founding |
| **SQL Server** | Legacy historical flight records | Original database (founded 2000, .NET/SQL Server era) |
| **PostgreSQL** | New operational data | Modern cloud-native database for next-gen platform |
| **MongoDB** | Flight log documents | LogTen logbook data is document-oriented, not relational |
| **Angular 18** | School operator dashboard | Primary frontend framework for FSP application |
| **React 18** | Pilot Pilotbase app | Newer product (Pilotbase platform, 2026) uses React |
| **TypeScript** | Shared DTOs across both frontends | Type safety across the entire frontend layer |
| **Azure** | API hosting (App Service) | Primary cloud (natural .NET ecosystem fit) |
| **AWS** | Logbook PDF export (S3) | Secondary cloud, likely from Coradine/LogTen infrastructure |
| **CI/CD** | GitHub Actions pipeline | Automated build, test, and deploy on push |

---

## Local Development

### Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Node.js 20+](https://nodejs.org/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (for local databases)
- [Angular CLI](https://angular.dev/tools/cli): `npm install -g @angular/cli`

### Start Local Databases

```bash
docker-compose up -d
```

This starts:
- **SQL Server 2022** on port 1433
- **PostgreSQL 16** on port 5432
- **MongoDB 7** on port 27017

### Run the API

```bash
cd backend/PilotbaseControlTower.Api
dotnet restore
dotnet run
```

API is available at `http://localhost:5000`. Swagger UI at `http://localhost:5000/swagger`.

Database migrations and seed data run automatically on first start.

### Run the School Operator Dashboard (Angular)

```bash
cd frontend-school
npm install
ng serve
```

Available at `http://localhost:4200`.

### Run the Pilot App (React)

```bash
cd frontend-pilot
npm install
npm run dev
```

Available at `http://localhost:5173`.

---

## Key API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/students` | All students with curriculum stage |
| `GET` | `/api/instructors` | All instructors with ratings and availability |
| `GET` | `/api/aircraft` | Fleet with maintenance status |
| `GET` | `/api/schedule/week` | Current week's schedule |
| `POST` | `/api/schedule/generate` | AI-powered schedule generation |
| `GET` | `/api/logbook/{pilotId}` | Pilot's logbook (MongoDB) |
| `POST` | `/api/logbook/{pilotId}/entries` | Add flight log entry |
| `POST` | `/api/logbook/{pilotId}/export` | Export logbook PDF to S3 |
| `POST` | `/api/financing/milestone` | Trigger Sallie Mae milestone event |
| `GET` | `/api/career/pathway/{pilotId}` | Hours-to-ATP + job match analysis |
| `GET` | `/api/reporting/kpis` | School KPI dashboard data |
| `GET` | `/api/legacy/flights` | Historical records (SQL Server) |

---

## The AI Scheduling Engine

The `POST /api/schedule/generate` endpoint accepts a `SchedulingRequest` containing the week's constraints and calls an LLM to generate an optimized schedule. The model receives:

- Student list with current curriculum stage, recent flight frequency, and upcoming checkride deadlines
- Instructor availability windows and certificate ratings (CFI, CFII, MEI)
- Aircraft availability with squawk status and upcoming maintenance
- Civil twilight times for the school's airport
- Part 141 stage check requirements vs. Part 61 flexibility

The response includes a full weekly schedule plus a natural-language `reasoning` field per booking — for example: *"Prioritized Student Martinez for the Cessna 172 Tuesday morning: she is 2 flights from her first solo endorsement, the aircraft has no open squawks, and civil twilight begins at 6:44am giving a full IFR margin before her 9am class."*

---

## Business Model Signals

**Sallie Mae Integration**: `POST /api/financing/milestone` fires whenever a student completes a milestone (first solo, instrument rating, commercial checkride). The payload mirrors a progress-based loan disbursement trigger — the innovation of Pilotbase's May 2025 Sallie Mae deal, where loan disbursements are tied to verifiable training progress rather than traditional credit applications.

**AOPA Top School Metrics**: The reporting dashboard displays a KPI tile showing percentage of student flights on aircraft with zero open squawks — the kind of operational excellence metric that correlates with AOPA Flight Training Experience Award recognition. 53% of 2025 AOPA award winners use Pilotbase.

**LogTen Document Schema**: MongoDB flight log entries use a schema matching what LogTen (acquired by FSP in 2022) actually stores — not just "flight date and hours" but conditions (VFR/IFR, day/night), instrument approaches by type (ILS, RNAV, VOR), landings (day/night), cross-country time, SIC/PIC/dual distinction, and instructor digital endorsement.

---

## About the Builder

Built by [Alan](https://github.com/OldEphraim) as part of the [Gauntlet AI](https://gauntletai.com/) cohort (Spring 2026) — a 10-week elite AI engineering program in Austin, TX. Alan is a full-stack software engineer (TypeScript, React, Next.js, Go, PostgreSQL, AWS) with a JD from Washington University in St. Louis and 3+ years of professional experience building production systems.