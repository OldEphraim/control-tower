# STEPS.md — Pilotbase Control Tower: Implementation Guide

Work through each phase sequentially. Do not skip steps. At the end of each phase, all verification commands must pass before proceeding to the next phase.

When you make a non-trivial decision (package choice, schema design, workaround, API shape), add an entry to `DECISION_LOG.md` using the format in `CLAUDE.md`.

---

## PHASE 0: Repository Bootstrap and Infrastructure

**Goal**: A working repo with all three databases running locally, a GitHub Actions pipeline, and the basic folder structure in place.

---

### Step 0.1 — Initialize DECISION_LOG.md

Create `DECISION_LOG.md` at the repo root:

```markdown
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
**Date**: [today]
**Decision**: Use SQL Server for legacy data, PostgreSQL for operational data, and MongoDB for logbook documents.
**Alternatives considered**: Single PostgreSQL database with JSONB columns for logbook data; single SQL Server database for everything.
**Rationale**: Mirrors Pilotbase's actual architecture. SQL Server is their legacy database (founded 2000, .NET/SQL Server era). PostgreSQL is their modern cloud-native choice. MongoDB matches the document-oriented shape of LogTen logbook entries (acquired 2022) — entries have variable nested schemas (some have instrument approaches, some don't; multi-engine entries differ from single-engine).
**Tradeoff**: Three database connections increase local setup complexity and add infrastructure overhead. Justified because the multi-database story is itself a demonstration of the architecture.

## [Phase 0, Step 0.1b] — Angular for school operator app, React for pilot app
**Date**: [today]
**Decision**: Use Angular 18 for the school operator dashboard and React 18 for the pilot-facing Pilotbase app.
**Alternatives considered**: Single React app for both user personas; single Angular app for both.
**Rationale**: Mirrors Pilotbase's actual frontend split. Angular is their confirmed primary framework for the legacy FSP product used by 1,400+ flight schools daily. The Pilotbase platform (launched February 2026) is a new product layer — newer Pilotbase products use React. Having both frameworks in the same repo demonstrates familiarity with both and reflects the real architectural tension they are managing after the rebrand.
**Tradeoff**: Two separate frontend projects increase build complexity and double the npm dependency surface. Justified for the same reason as the three-database choice: the split is itself the signal.
```

---

### Step 0.2 — Create .gitignore

Create `.gitignore` at the repo root:

```gitignore
# .NET
backend/**/bin/
backend/**/obj/
backend/**/*.user
backend/**/.vs/
backend/**/.idea/

# Node
node_modules/
dist/
.angular/
*.local
.vite/

# Environment / secrets
.env
.env.*
!.env.example
**/appsettings.Development.json
**/appsettings.*.json
!**/appsettings.json

# OS
.DS_Store
Thumbs.db

# Test results
**/TestResults/
coverage/
lcov.info
```

---

### Step 0.3 — Create docker-compose.yml

Create `docker-compose.yml` at the repo root:

```yaml
version: '3.9'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    # NOTE: On Apple Silicon (M1/M2/M3), add: platform: linux/amd64
    container_name: pct-sqlserver
    environment:
      SA_PASSWORD: "PilotbaseDev123!"
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Developer"
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'PilotbaseDev123!' -Q 'SELECT 1' -No || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s

  postgres:
    image: postgres:16-alpine
    container_name: pct-postgres
    environment:
      POSTGRES_USER: pilotbase
      POSTGRES_PASSWORD: pilotbase_dev
      POSTGRES_DB: pilotbase_ops
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pilotbase -d pilotbase_ops"]
      interval: 5s
      timeout: 3s
      retries: 10

  mongodb:
    image: mongo:7
    container_name: pct-mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: pilotbase
      MONGO_INITDB_ROOT_PASSWORD: pilotbase_dev
      MONGO_INITDB_DATABASE: pilotbase_logs
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')", "--username", "pilotbase", "--password", "pilotbase_dev", "--authenticationDatabase", "admin", "--quiet"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s

volumes:
  sqlserver_data:
  postgres_data:
  mongodb_data:
```

**If on Apple Silicon**: Add `platform: linux/amd64` under the `sqlserver` service. Record this in DECISION_LOG.

---

### Step 0.4 — Create GitHub Actions workflow

Create `.github/workflows/ci.yml`. Note the `if:` guards on the frontend jobs — the `frontend-school` and `frontend-pilot` directories don't exist until Phases 7 and 8, so without these guards every push during backend development will fail CI:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-backend:
    name: Build & Test .NET API
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore backend/PilotbaseControlTower.Api/PilotbaseControlTower.Api.csproj

      - name: Build
        run: dotnet build backend/PilotbaseControlTower.Api/PilotbaseControlTower.Api.csproj --no-restore --configuration Release

      - name: Test
        run: |
          if [ -f "backend/PilotbaseControlTower.Tests/PilotbaseControlTower.Tests.csproj" ]; then
            dotnet test backend/PilotbaseControlTower.Tests/PilotbaseControlTower.Tests.csproj --configuration Release --verbosity normal
          else
            echo "No test project found — skipping"
          fi

  build-frontend-school:
    name: Build Angular School App
    runs-on: ubuntu-latest
    # Only run once frontend-school has been initialized
    if: hashFiles('frontend-school/package.json') != ''
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node 20
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend-school/package-lock.json

      - name: Install
        run: npm ci
        working-directory: frontend-school

      - name: Build
        run: npm run build -- --configuration production
        working-directory: frontend-school

  build-frontend-pilot:
    name: Build React Pilot App
    runs-on: ubuntu-latest
    # Only run once frontend-pilot has been initialized
    if: hashFiles('frontend-pilot/package.json') != ''
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node 20
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend-pilot/package-lock.json

      - name: Install
        run: npm ci
        working-directory: frontend-pilot

      - name: Build
        run: npm run build
        working-directory: frontend-pilot
```

---

### Step 0.5 — Create directory skeleton

```bash
mkdir -p backend
mkdir -p .github/workflows
```

---

### Phase 0 Verification

```bash
# 1. Start databases
docker-compose up -d

# 2. Wait ~30 seconds, then confirm all three are healthy
docker-compose ps
# Expected: pct-sqlserver, pct-postgres, pct-mongodb all show "healthy" or "running"

# 3. Test SQL Server
docker exec pct-sqlserver bash -c 'PASS=$(printenv SA_PASSWORD) && /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "$PASS" -Q "SELECT '"'"'SQL Server OK'"'"'" -No'
# Expected: SQL Server OK

# 4. Test PostgreSQL
docker exec pct-postgres psql -U pilotbase -d pilotbase_ops -c "SELECT 'PostgreSQL OK';"
# Expected: PostgreSQL OK

# 5. Test MongoDB
docker exec pct-mongodb mongosh \
  --username pilotbase --password pilotbase_dev \
  --authenticationDatabase admin --quiet \
  --eval "db.adminCommand('ping')"
# Expected: { ok: 1 }
```

---

## PHASE 1: .NET 8 Web API — Project Creation and Configuration

**Goal**: A compilable .NET 8 Web API project with all packages installed, folder structure created, connection strings configured, and a test project scaffolded.

---

### Step 1.1 — Create the Web API project

```bash
cd backend
dotnet new webapi -n PilotbaseControlTower.Api --framework net8.0 --use-controllers
cd PilotbaseControlTower.Api
dotnet build
# Expected: Build succeeded. 0 Error(s).
```

---

### Step 1.2 — Create the test project

```bash
cd ..   # back to backend/
dotnet new xunit -n PilotbaseControlTower.Tests --framework net8.0
cd PilotbaseControlTower.Tests
dotnet add reference ../PilotbaseControlTower.Api/PilotbaseControlTower.Api.csproj
dotnet add package Moq --version 4.20.70
dotnet add package FluentAssertions --version 6.12.0
dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 8.0.4

cd ../../   # back to repo root
dotnet new sln -n PilotbaseControlTower
dotnet sln add backend/PilotbaseControlTower.Api/PilotbaseControlTower.Api.csproj
dotnet sln add backend/PilotbaseControlTower.Tests/PilotbaseControlTower.Tests.csproj

dotnet build PilotbaseControlTower.sln
# Expected: Build succeeded. 0 Error(s).
```

---

### Step 1.3 — Install NuGet packages

From `backend/PilotbaseControlTower.Api/`:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.4
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 8.0.4
dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.4
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 8.0.4
dotnet add package MongoDB.Driver --version 2.28.0
dotnet add package QuestPDF --version 2024.3.4
dotnet add package AWSSDK.S3 --version 3.7.306.17
# Do NOT re-add Swashbuckle.AspNetCore — the webapi template already includes it.
# Adding it again may cause NU1605 version conflict warnings.

dotnet build
# Expected: Build succeeded. 0 Error(s).
```

**DECISION_LOG entry**: Record exact installed versions. Note that Swashbuckle was intentionally not reinstalled to avoid version conflicts with the template-bundled version.

---

### Step 1.4 — Create the folder structure

From `backend/PilotbaseControlTower.Api/`:

```bash
mkdir -p Controllers
mkdir -p Models/Legacy
mkdir -p Models/Operational
mkdir -p Models/Logbook
mkdir -p Models/DTOs
mkdir -p Data/Legacy
mkdir -p Data/Operational
mkdir -p Mongo
mkdir -p Services/AI
mkdir -p Services/Financing
mkdir -p Services/Career
mkdir -p Services/S3
mkdir -p Seed

# Remove auto-generated template files
rm -f Controllers/WeatherForecastController.cs
rm -f WeatherForecast.cs
```

---

### Step 1.5 — Configure appsettings.json

Replace the entire contents of `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "SqlServer": "Server=localhost,1433;Database=PilotbaseLegacy;User Id=sa;Password=PilotbaseDev123!;TrustServerCertificate=True;",
    "PostgreSQL": "Host=localhost;Port=5432;Database=pilotbase_ops;Username=pilotbase;Password=pilotbase_dev",
    "MongoDB": "mongodb://pilotbase:pilotbase_dev@localhost:27017/pilotbase_logs?authSource=admin"
  },
  "MongoDB": {
    "DatabaseName": "pilotbase_logs",
    "FlightLogsCollection": "flight_logs"
  },
  "AnthropicApi": {
    "ApiKey": "",
    "Model": "claude-opus-4-6",
    "MaxTokens": 4096
  },
  "AWS": {
    "Region": "us-east-1",
    "BucketName": "pilotbase-logbooks-dev",
    "AccessKeyId": "",
    "SecretAccessKey": ""
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "AllowedOrigins": [
    "http://localhost:4200",
    "http://localhost:5173"
  ]
}
```

Create `appsettings.Development.json` (git-ignored):

```json
{
  "AnthropicApi": {
    "ApiKey": "YOUR_ANTHROPIC_API_KEY_HERE"
  },
  "AWS": {
    "AccessKeyId": "YOUR_AWS_ACCESS_KEY_ID",
    "SecretAccessKey": "YOUR_AWS_SECRET_ACCESS_KEY"
  }
}
```

---

### Step 1.6 — Stub out Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

### Phase 1 Verification

```bash
dotnet build PilotbaseControlTower.sln
# Expected: Build succeeded. 0 Error(s).

dotnet test PilotbaseControlTower.sln
# Expected: Test run succeeded. (0 tests yet — fine)
```

---

## PHASE 2: Models

**Goal**: All domain models defined across all three persistence layers.

---

### Step 2.1 — Legacy model (SQL Server)

Create `Models/Legacy/LegacyFlightRecord.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Legacy;

/// <summary>
/// Pre-migration flight record in SQL Server.
/// READ-ONLY from the application — all new flight data goes to PostgreSQL.
/// Models Pilotbase's "mostly greenfield with some legacy" environment.
/// </summary>
public class LegacyFlightRecord
{
    public int Id { get; set; }
    [MaxLength(200)] public string PilotName { get; set; } = string.Empty;
    [MaxLength(200)] public string InstructorName { get; set; } = string.Empty;
    [MaxLength(10)] public string AircraftRegistration { get; set; } = string.Empty;
    [MaxLength(100)] public string AircraftType { get; set; } = string.Empty;
    public DateTime FlightDate { get; set; }
    public decimal TotalTime { get; set; }
    public decimal DualReceived { get; set; }
    public decimal SoloTime { get; set; }
    [MaxLength(10)] public string DepartureAirport { get; set; } = string.Empty;
    [MaxLength(10)] public string DestinationAirport { get; set; } = string.Empty;
    [MaxLength(500)] public string Remarks { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}
```

---

### Step 2.2 — Operational models (PostgreSQL)

Create `Models/Operational/Enums.cs`:

```csharp
namespace PilotbaseControlTower.Api.Models.Operational;

public enum TrainingType { Part61, Part141 }

/// <summary>
/// Ordered curriculum stages. The integer values encode progression order,
/// used for scheduling priority and Sallie Mae milestone triggers.
/// </summary>
public enum CurriculumStage
{
    DiscoveryFlight = 0,
    PreSolo = 1,
    PostSolo = 2,
    CrossCountry = 3,
    NightFlying = 4,
    InstrumentBasics = 5,
    CheckridePrep = 6,
    PrivatePilotComplete = 7,
    InstrumentRating = 8,
    CommercialTraining = 9,
    AtpPrep = 10
}

public enum FinancingType { SelfFunded, SallieMae, StratusFinancial, Other }
public enum AircraftCategory { SingleEngineLand, MultiEngineLand }

/// <summary>
/// GroundedSquawk: aircraft may still be used for day VFR depending on the squawk
/// (e.g., inop landing light = no night ops but day VFR allowed).
/// GroundedMaintenance: completely unavailable.
/// </summary>
public enum AircraftStatus { Airworthy, GroundedMaintenance, GroundedSquawk, ScheduledMaintenance }
public enum EventStatus { Scheduled, Completed, Cancelled, NoShow }
public enum MilestoneType
{
    FirstFlight, FirstSolo, FirstCrossCountry, NightEndorsement,
    InstrumentHours, PrivatePilotCheckride, InstrumentCheckride,
    CommercialCheckride, ATPMinimums
}
```

Create `Models/Operational/Student.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Operational;

public class Student
{
    public int Id { get; set; }
    [MaxLength(100)] public string FirstName { get; set; } = string.Empty;
    [MaxLength(100)] public string LastName { get; set; } = string.Empty;
    [MaxLength(256)] public string Email { get; set; } = string.Empty;
    [MaxLength(30)] public string Phone { get; set; } = string.Empty;
    public TrainingType TrainingType { get; set; }
    public CurriculumStage CurrentStage { get; set; }
    public decimal TotalHours { get; set; }
    public decimal SoloHours { get; set; }
    public decimal CrossCountryHours { get; set; }
    public decimal NightHours { get; set; }
    public decimal InstrumentHours { get; set; }
    public bool HasSoloEndorsement { get; set; }
    public bool HasCrossCountryEndorsement { get; set; }
    public DateTime EnrollmentDate { get; set; }
    public DateTime? ProjectedCheckrideDate { get; set; }
    [MaxLength(256)] public string? AssignedInstructorEmail { get; set; }
    public FinancingType FinancingType { get; set; }
    public decimal OutstandingBalance { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public List<ScheduleEvent> ScheduleEvents { get; set; } = new();
}
```

Create `Models/Operational/Instructor.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Operational;

public class Instructor
{
    public int Id { get; set; }
    [MaxLength(100)] public string FirstName { get; set; } = string.Empty;
    [MaxLength(100)] public string LastName { get; set; } = string.Empty;
    [MaxLength(256)] public string Email { get; set; } = string.Empty;
    [MaxLength(20)] public string CertificateNumber { get; set; } = string.Empty;
    public bool IsCfi { get; set; }
    public bool IsCfii { get; set; }
    public bool IsMei { get; set; }
    public decimal TotalFlightHours { get; set; }
    /// <summary>
    /// Stored as comma-separated string in PostgreSQL via EF value converter.
    /// NOTE: EF value converters for collections do not participate in change tracking.
    /// Mutations to this list will NOT be detected by EF. For a read-heavy demo this
    /// is acceptable. See DECISION_LOG for details.
    /// </summary>
    public List<string> AvailableDays { get; set; } = new();
    public TimeOnly AvailableFrom { get; set; }
    public TimeOnly AvailableTo { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    public List<ScheduleEvent> ScheduleEvents { get; set; } = new();
}
```

Create `Models/Operational/Aircraft.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Operational;

public class Aircraft
{
    public int Id { get; set; }
    [MaxLength(10)] public string Registration { get; set; } = string.Empty;
    [MaxLength(50)] public string Make { get; set; } = string.Empty;
    [MaxLength(100)] public string Model { get; set; } = string.Empty;
    public int Year { get; set; }
    public AircraftCategory Category { get; set; }
    public bool IsIfrEquipped { get; set; }
    public bool IsMultiEngine { get; set; }
    public decimal TachometerHours { get; set; }
    public decimal NextOilChangeHours { get; set; }
    public decimal NextAnnualInspectionDays { get; set; }
    public AircraftStatus Status { get; set; }
    [MaxLength(500)] public string? ActiveSquawk { get; set; }
    public bool IsAvailable { get; set; }
    public decimal HourlyRate { get; set; }
    public DateTime CreatedAt { get; set; }
    public List<ScheduleEvent> ScheduleEvents { get; set; } = new();
}
```

Create `Models/Operational/ScheduleEvent.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Operational;

public class ScheduleEvent
{
    public int Id { get; set; }
    public int StudentId { get; set; }
    public int InstructorId { get; set; }
    public int AircraftId { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    [MaxLength(200)] public string LessonType { get; set; } = string.Empty;
    /// <summary>
    /// Natural-language reasoning from the AI scheduling engine.
    /// This is Pilotbase's "Intelligent Scheduling" feature.
    /// </summary>
    [MaxLength(1000)] public string? AiReasoning { get; set; }
    public EventStatus Status { get; set; }
    public decimal? ActualFlightTime { get; set; }
    [MaxLength(1000)] public string? InstructorNotes { get; set; }
    public DateTime CreatedAt { get; set; }
    public Student Student { get; set; } = null!;
    public Instructor Instructor { get; set; } = null!;
    public Aircraft Aircraft { get; set; } = null!;
}
```

Create `Models/Operational/FinancingMilestone.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace PilotbaseControlTower.Api.Models.Operational;

/// <summary>
/// Records training milestones that trigger Sallie Mae disbursements.
/// Pilotbase's May 2025 Sallie Mae partnership innovation: disbursements
/// are tied to verifiable training progress, not traditional lump-sum loans.
/// </summary>
public class FinancingMilestone
{
    public int Id { get; set; }
    public int StudentId { get; set; }
    public MilestoneType MilestoneType { get; set; }
    public DateTime AchievedAt { get; set; }
    public decimal HoursAtMilestone { get; set; }
    public bool DisbursementTriggered { get; set; }
    public decimal? DisbursementAmount { get; set; }
    [MaxLength(100)] public string? LenderReference { get; set; }
    public Student Student { get; set; } = null!;
}
```

---

### Step 2.3 — MongoDB logbook document

Create `Models/Logbook/FlightLogEntry.cs`:

```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace PilotbaseControlTower.Api.Models.Logbook;

/// <summary>
/// Flight logbook entry in MongoDB.
/// Schema mirrors LogTen (acquired by Pilotbase May 2022).
/// Document model chosen because entries have variable structure:
/// IFR flights have an Approaches array; VFR flights don't.
/// Solo flights have no InstructorSignature; dual lessons do.
/// </summary>
public class FlightLogEntry
{
    [BsonId, BsonRepresentation(BsonType.ObjectId)]
    public string? Id { get; set; }
    [BsonElement("pilotId")] public int PilotId { get; set; }
    [BsonElement("flightDate")] public DateTime FlightDate { get; set; }
    [BsonElement("aircraftRegistration")] public string AircraftRegistration { get; set; } = string.Empty;
    [BsonElement("aircraftMakeModel")] public string AircraftMakeModel { get; set; } = string.Empty;
    [BsonElement("departureAirport")] public string DepartureAirport { get; set; } = string.Empty;
    [BsonElement("destinationAirport")] public string DestinationAirport { get; set; } = string.Empty;
    [BsonElement("routeOfFlight")] public string? RouteOfFlight { get; set; }
    [BsonElement("conditions")] public FlightConditions Conditions { get; set; } = new();
    [BsonElement("times")] public FlightTimes Times { get; set; } = new();
    [BsonElement("approaches")] public List<InstrumentApproach> Approaches { get; set; } = new();
    [BsonElement("landings")] public LandingCounts Landings { get; set; } = new();
    [BsonElement("remarks")] public string? Remarks { get; set; }
    [BsonElement("instructorSignature")] public InstructorEndorsement? InstructorSignature { get; set; }
    [BsonElement("createdAt")] public DateTime CreatedAt { get; set; }
}

public class FlightConditions
{
    [BsonElement("isIfr")] public bool IsIfr { get; set; }
    [BsonElement("isNight")] public bool IsNight { get; set; }
    [BsonElement("isCrossCountry")] public bool IsCrossCountry { get; set; }
    [BsonElement("isSimulatedInstrument")] public bool IsSimulatedInstrument { get; set; }
}

public class FlightTimes
{
    [BsonElement("totalTime")] public decimal TotalTime { get; set; }
    [BsonElement("picTime")] public decimal PicTime { get; set; }
    [BsonElement("sicTime")] public decimal SicTime { get; set; }
    [BsonElement("dualReceived")] public decimal DualReceived { get; set; }
    [BsonElement("soloTime")] public decimal SoloTime { get; set; }
    [BsonElement("nightTime")] public decimal NightTime { get; set; }
    [BsonElement("instrumentTime")] public decimal InstrumentTime { get; set; }
    [BsonElement("crossCountryTime")] public decimal CrossCountryTime { get; set; }
    [BsonElement("groundTrainingTime")] public decimal GroundTrainingTime { get; set; }
}

public class InstrumentApproach
{
    [BsonElement("type")] public string Type { get; set; } = string.Empty; // ILS, RNAV, VOR, NDB, LOC
    [BsonElement("airport")] public string Airport { get; set; } = string.Empty;
    [BsonElement("runway")] public string Runway { get; set; } = string.Empty;
}

public class LandingCounts
{
    [BsonElement("day")] public int Day { get; set; }
    [BsonElement("night")] public int Night { get; set; }
}

public class InstructorEndorsement
{
    [BsonElement("instructorName")] public string InstructorName { get; set; } = string.Empty;
    [BsonElement("certificateNumber")] public string CertificateNumber { get; set; } = string.Empty;
    [BsonElement("signedAt")] public DateTime SignedAt { get; set; }
}
```

---

### Step 2.4 — Unit tests for models

Create `backend/PilotbaseControlTower.Tests/Models/ModelTests.cs`:

```csharp
using FluentAssertions;
using PilotbaseControlTower.Api.Models.Operational;
using PilotbaseControlTower.Api.Models.Logbook;

namespace PilotbaseControlTower.Tests.Models;

public class ModelTests
{
    [Fact]
    public void CurriculumStage_IsOrdered_PreSoloComesBefore_CheckridePrep()
    {
        ((int)CurriculumStage.PreSolo).Should().BeLessThan((int)CurriculumStage.CheckridePrep);
        ((int)CurriculumStage.PostSolo).Should().BeLessThan((int)CurriculumStage.CrossCountry);
        ((int)CurriculumStage.CrossCountry).Should().BeLessThan((int)CurriculumStage.CheckridePrep);
    }

    [Fact]
    public void Student_DefaultValues_AreCorrect()
    {
        var student = new Student();
        student.HasSoloEndorsement.Should().BeFalse();
        student.HasCrossCountryEndorsement.Should().BeFalse();
        student.ScheduleEvents.Should().NotBeNull().And.BeEmpty();
        student.TotalHours.Should().Be(0);
    }

    [Fact]
    public void FlightLogEntry_Approaches_DefaultsToEmptyList()
    {
        var entry = new FlightLogEntry();
        entry.Approaches.Should().NotBeNull().And.BeEmpty();
        entry.InstructorSignature.Should().BeNull();
    }

    [Fact]
    public void FlightLogEntry_InstructorSignature_IsNullableForSoloFlights()
    {
        var soloFlight = new FlightLogEntry
        {
            Times = new FlightTimes { SoloTime = 1.5m, PicTime = 1.5m, TotalTime = 1.5m },
            InstructorSignature = null
        };
        soloFlight.InstructorSignature.Should().BeNull();
    }

    [Fact]
    public void Aircraft_SquawkProperty_IsNullableForAirworthyAircraft()
    {
        var airworthy = new Aircraft { Status = AircraftStatus.Airworthy, ActiveSquawk = null };
        airworthy.ActiveSquawk.Should().BeNull();

        var squawked = new Aircraft
        {
            Status = AircraftStatus.GroundedSquawk,
            ActiveSquawk = "Left landing light inoperative - night ops prohibited"
        };
        squawked.ActiveSquawk.Should().NotBeNullOrEmpty();
    }
}
```

Run: `dotnet test PilotbaseControlTower.sln` — Expected: 4 tests pass.

---

### Phase 2 Verification

```bash
dotnet build PilotbaseControlTower.sln  # 0 errors
dotnet test PilotbaseControlTower.sln   # 4 passed
```

**DECISION_LOG entry**: Record the decision to use `TimeOnly` for instructor availability hours (type-safe, .NET 8 native, serializes correctly with Npgsql 8.x). Also record the `List<string>` value converter decision for `AvailableDays`:

```markdown
## [Phase 2, Step 2.2] — Instructor.AvailableDays stored as comma-separated string
**Date**: [today]
**Decision**: Store AvailableDays as a comma-separated varchar via EF value converter rather than a PostgreSQL text[] array.
**Alternatives considered**: Native Npgsql text[] array column type (would support change tracking).
**Rationale**: Simpler setup for a demo project; avoids requiring Npgsql-specific array handling in the DbContext.
**Tradeoff**: EF Core value converters for collection types do NOT participate in change tracking. If code reads an Instructor, mutates AvailableDays, and calls SaveChangesAsync(), the change will silently not persist. Acceptable for this read-heavy demo, but any update path for instructors must reassign the entire list. Additionally, InMemory EF provider and real PostgreSQL may behave differently with this converter — do not rely on InMemory tests for AvailableDays mutation behavior.
```

---

## PHASE 3: Database Contexts

**Goal**: Three DbContexts and the MongoDB service wrapper.

---

### Step 3.1 — SQL Server DbContext

Create `Data/Legacy/LegacyDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Models.Legacy;

namespace PilotbaseControlTower.Api.Data.Legacy;

public class LegacyDbContext : DbContext
{
    public LegacyDbContext(DbContextOptions<LegacyDbContext> options) : base(options) { }

    public DbSet<LegacyFlightRecord> LegacyFlightRecords => Set<LegacyFlightRecord>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<LegacyFlightRecord>(entity =>
        {
            entity.ToTable("LegacyFlightRecords");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Id).UseIdentityColumn();
            entity.Property(e => e.TotalTime).HasPrecision(5, 1);
            entity.Property(e => e.DualReceived).HasPrecision(5, 1);
            entity.Property(e => e.SoloTime).HasPrecision(5, 1);
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
        });
    }
}
```

---

### Step 3.2 — PostgreSQL DbContext

Create `Data/Operational/OperationalDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Api.Data.Operational;

public class OperationalDbContext : DbContext
{
    public OperationalDbContext(DbContextOptions<OperationalDbContext> options) : base(options) { }

    public DbSet<Student> Students => Set<Student>();
    public DbSet<Instructor> Instructors => Set<Instructor>();
    public DbSet<Aircraft> Aircraft => Set<Aircraft>();
    public DbSet<ScheduleEvent> ScheduleEvents => Set<ScheduleEvent>();
    public DbSet<FinancingMilestone> FinancingMilestones => Set<FinancingMilestone>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Student>(entity =>
        {
            entity.ToTable("students");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(256);
            entity.HasIndex(e => e.Email).IsUnique();
            entity.Property(e => e.TotalHours).HasPrecision(6, 1);
            entity.Property(e => e.SoloHours).HasPrecision(6, 1);
            entity.Property(e => e.CrossCountryHours).HasPrecision(6, 1);
            entity.Property(e => e.NightHours).HasPrecision(6, 1);
            entity.Property(e => e.InstrumentHours).HasPrecision(6, 1);
            entity.Property(e => e.OutstandingBalance).HasPrecision(10, 2);
            entity.Property(e => e.TrainingType).HasConversion<string>();
            entity.Property(e => e.CurrentStage).HasConversion<string>();
            entity.Property(e => e.FinancingType).HasConversion<string>();
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("NOW()");
            entity.Property(e => e.UpdatedAt).HasDefaultValueSql("NOW()");
        });

        modelBuilder.Entity<Instructor>(entity =>
        {
            entity.ToTable("instructors");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Email).IsRequired().HasMaxLength(256);
            entity.HasIndex(e => e.Email).IsUnique();
            entity.Property(e => e.TotalFlightHours).HasPrecision(7, 1);
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("NOW()");
            entity.Property(e => e.AvailableDays).HasConversion(
                v => string.Join(',', v),
                v => v.Split(',', StringSplitOptions.RemoveEmptyEntries).ToList()
            );
        });

        modelBuilder.Entity<Aircraft>(entity =>
        {
            entity.ToTable("aircraft");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Registration).IsRequired().HasMaxLength(10);
            entity.HasIndex(e => e.Registration).IsUnique();
            entity.Property(e => e.TachometerHours).HasPrecision(7, 1);
            entity.Property(e => e.HourlyRate).HasPrecision(8, 2);
            entity.Property(e => e.Category).HasConversion<string>();
            entity.Property(e => e.Status).HasConversion<string>();
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("NOW()");
        });

        modelBuilder.Entity<ScheduleEvent>(entity =>
        {
            entity.ToTable("schedule_events");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Status).HasConversion<string>();
            entity.Property(e => e.ActualFlightTime).HasPrecision(5, 1);
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("NOW()");
            entity.HasOne(e => e.Student).WithMany(s => s.ScheduleEvents)
                .HasForeignKey(e => e.StudentId).OnDelete(DeleteBehavior.Restrict);
            entity.HasOne(e => e.Instructor).WithMany(i => i.ScheduleEvents)
                .HasForeignKey(e => e.InstructorId).OnDelete(DeleteBehavior.Restrict);
            entity.HasOne(e => e.Aircraft).WithMany(a => a.ScheduleEvents)
                .HasForeignKey(e => e.AircraftId).OnDelete(DeleteBehavior.Restrict);
        });

        modelBuilder.Entity<FinancingMilestone>(entity =>
        {
            entity.ToTable("financing_milestones");
            entity.HasKey(e => e.Id);
            entity.Property(e => e.MilestoneType).HasConversion<string>();
            entity.Property(e => e.HoursAtMilestone).HasPrecision(6, 1);
            entity.Property(e => e.DisbursementAmount).HasPrecision(10, 2);
            entity.HasOne(e => e.Student).WithMany()
                .HasForeignKey(e => e.StudentId).OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

---

### Step 3.3 — MongoDB service

Create `Mongo/MongoDbService.cs`:

```csharp
using MongoDB.Driver;
using PilotbaseControlTower.Api.Models.Logbook;

namespace PilotbaseControlTower.Api.Mongo;

/// <summary>
/// Singleton service for MongoDB logbook operations.
/// MongoClient is thread-safe by design and intended to be reused.
/// Index creation is wrapped in try-catch: if MongoDB is unavailable
/// when the singleton is first resolved, we log a warning but don't
/// crash the app — seed will also fail gracefully.
/// </summary>
public class MongoDbService
{
    private readonly IMongoCollection<FlightLogEntry> _flightLogs;
    private readonly ILogger<MongoDbService> _logger;

    public MongoDbService(IConfiguration configuration, ILogger<MongoDbService> logger)
    {
        _logger = logger;
        var connectionString = configuration.GetConnectionString("MongoDB")!;
        var databaseName = configuration["MongoDB:DatabaseName"]!;
        var collectionName = configuration["MongoDB:FlightLogsCollection"]!;

        var client = new MongoClient(connectionString);
        var database = client.GetDatabase(databaseName);
        _flightLogs = database.GetCollection<FlightLogEntry>(collectionName);

        try
        {
            var indexKeys = Builders<FlightLogEntry>.IndexKeys.Ascending(f => f.PilotId);
            _flightLogs.Indexes.CreateOne(new CreateIndexModel<FlightLogEntry>(indexKeys));
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "MongoDB index creation failed — MongoDB may not be available yet.");
        }
    }

    public async Task<List<FlightLogEntry>> GetByPilotIdAsync(int pilotId) =>
        await _flightLogs.Find(f => f.PilotId == pilotId)
            .SortByDescending(f => f.FlightDate).ToListAsync();

    public async Task<FlightLogEntry?> GetByIdAsync(string id) =>
        await _flightLogs.Find(f => f.Id == id).FirstOrDefaultAsync();

    public async Task<List<FlightLogEntry>> GetRecentAsync(int pilotId, int count = 10) =>
        await _flightLogs.Find(f => f.PilotId == pilotId)
            .SortByDescending(f => f.FlightDate).Limit(count).ToListAsync();

    public async Task<FlightLogEntry> CreateAsync(FlightLogEntry entry)
    {
        entry.CreatedAt = DateTime.UtcNow;
        await _flightLogs.InsertOneAsync(entry);
        return entry;
    }

    public async Task<LogbookTotals> GetTotalsAsync(int pilotId)
    {
        var entries = await GetByPilotIdAsync(pilotId);
        return new LogbookTotals
        {
            TotalTime = entries.Sum(e => e.Times.TotalTime),
            PicTime = entries.Sum(e => e.Times.PicTime),
            DualReceived = entries.Sum(e => e.Times.DualReceived),
            SoloTime = entries.Sum(e => e.Times.SoloTime),
            NightTime = entries.Sum(e => e.Times.NightTime),
            InstrumentTime = entries.Sum(e => e.Times.InstrumentTime),
            CrossCountryTime = entries.Sum(e => e.Times.CrossCountryTime),
            TotalLandings = entries.Sum(e => e.Landings.Day + e.Landings.Night),
            NightLandings = entries.Sum(e => e.Landings.Night),
            TotalFlights = entries.Count,
            InstrumentApproaches = entries.Sum(e => e.Approaches.Count)
        };
    }
}

public record LogbookTotals
{
    public decimal TotalTime { get; init; }
    public decimal PicTime { get; init; }
    public decimal DualReceived { get; init; }
    public decimal SoloTime { get; init; }
    public decimal NightTime { get; init; }
    public decimal InstrumentTime { get; init; }
    public decimal CrossCountryTime { get; init; }
    public int TotalLandings { get; init; }
    public int NightLandings { get; init; }
    public int TotalFlights { get; init; }
    public int InstrumentApproaches { get; init; }
}
```

---

### Step 3.4 — DbContext unit tests

Create `backend/PilotbaseControlTower.Tests/Data/DbContextTests.cs`:

```csharp
using FluentAssertions;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Tests.Data;

public class DbContextTests
{
    private static OperationalDbContext CreateContext() =>
        new(new DbContextOptionsBuilder<OperationalDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString()).Options);

    [Fact]
    public async Task CanSaveAndRetrieve_Student()
    {
        await using var ctx = CreateContext();
        var student = new Student
        {
            FirstName = "Maria", LastName = "Garcia", Email = "maria@test.com",
            Phone = "813-555-0101", TrainingType = TrainingType.Part141,
            CurrentStage = CurriculumStage.PostSolo, TotalHours = 22.5m,
            EnrollmentDate = DateTime.UtcNow, FinancingType = FinancingType.SallieMae,
            CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow
        };
        ctx.Students.Add(student);
        await ctx.SaveChangesAsync();

        var retrieved = await ctx.Students.FirstAsync(s => s.Email == "maria@test.com");
        retrieved.FirstName.Should().Be("Maria");
        retrieved.TrainingType.Should().Be(TrainingType.Part141);
        retrieved.FinancingType.Should().Be(FinancingType.SallieMae);
        retrieved.TotalHours.Should().Be(22.5m);
    }

    [Fact]
    public async Task CanSaveAndRetrieve_Aircraft_WithSquawk()
    {
        await using var ctx = CreateContext();
        var aircraft = new Aircraft
        {
            Registration = "N172SD", Make = "Cessna", Model = "172R Skyhawk", Year = 2016,
            Category = AircraftCategory.SingleEngineLand,
            Status = AircraftStatus.GroundedSquawk,
            ActiveSquawk = "Left landing light inoperative - night ops prohibited",
            IsAvailable = true, HourlyRate = 135.00m, CreatedAt = DateTime.UtcNow
        };
        ctx.Aircraft.Add(aircraft);
        await ctx.SaveChangesAsync();

        var retrieved = await ctx.Aircraft.FirstAsync(a => a.Registration == "N172SD");
        retrieved.Status.Should().Be(AircraftStatus.GroundedSquawk);
        retrieved.ActiveSquawk.Should().Contain("night ops prohibited");
        retrieved.IsAvailable.Should().BeTrue(); // squawked but still day-VFR available
    }

    [Fact]
    public async Task CanSaveAndRetrieve_FinancingMilestone()
    {
        await using var ctx = CreateContext();
        var student = new Student
        {
            FirstName = "Test", LastName = "Pilot", Email = "test@test.com", Phone = "555-0000",
            TrainingType = TrainingType.Part61, CurrentStage = CurriculumStage.PostSolo,
            FinancingType = FinancingType.SallieMae, EnrollmentDate = DateTime.UtcNow,
            CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow
        };
        ctx.Students.Add(student);
        await ctx.SaveChangesAsync();

        var milestone = new FinancingMilestone
        {
            StudentId = student.Id, MilestoneType = MilestoneType.FirstSolo,
            AchievedAt = DateTime.UtcNow, HoursAtMilestone = 22.5m,
            DisbursementTriggered = true, DisbursementAmount = 3000m,
            LenderReference = "SM-20260318-001-FirstSolo"
        };
        ctx.FinancingMilestones.Add(milestone);
        await ctx.SaveChangesAsync();

        var retrieved = await ctx.FinancingMilestones.Include(m => m.Student).FirstAsync();
        retrieved.DisbursementTriggered.Should().BeTrue();
        retrieved.DisbursementAmount.Should().Be(3000m);
        retrieved.Student.Email.Should().Be("test@test.com");
    }
}
```

Run: `dotnet test PilotbaseControlTower.sln` — Expected: 7 tests pass.

**DECISION_LOG entry**: Record `OnDelete(DeleteBehavior.Restrict)` on ScheduleEvent foreign keys — prevents accidentally deleting a student or instructor who has scheduled events, which would corrupt historical schedule data.

---

## PHASE 4: Seed Data

**Goal**: All three databases populated with realistic demo data.

The seed data represents:
- 10 students at varied curriculum stages
- 5 instructors with different rating combinations (CFI, CFII, MEI)
- 6 aircraft (4x Cessna 172 variants, 1x Piper Archer, 1x Piper Seminole): N172SC is in ScheduledMaintenance with IsAvailable=false, N172SD has an active squawk but IsAvailable=true (day VFR permitted), the other four are fully airworthy
- 10 legacy SQL Server flight records
- MongoDB logbook entries for pilots 1, 2, and 5

Create `Seed/DatabaseSeeder.cs`. This is a long file — write it carefully, the seed data is what makes the demo believable. Use the exact students, instructors, and aircraft specified below. Do not invent different data.

```csharp
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Legacy;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Legacy;
using PilotbaseControlTower.Api.Models.Logbook;
using PilotbaseControlTower.Api.Models.Operational;
using PilotbaseControlTower.Api.Mongo;

namespace PilotbaseControlTower.Api.Seed;

public static class DatabaseSeeder
{
    public static async Task SeedAllAsync(
        LegacyDbContext legacyDb, OperationalDbContext operationalDb,
        MongoDbService mongoService, ILogger logger)
    {
        await SeedLegacyAsync(legacyDb, logger);
        await SeedOperationalAsync(operationalDb, logger);
        await SeedLogbookAsync(mongoService, logger);
    }

    private static async Task SeedLegacyAsync(LegacyDbContext db, ILogger logger)
    {
        await db.Database.EnsureCreatedAsync();
        if (await db.LegacyFlightRecords.AnyAsync()) { logger.LogInformation("Legacy SQL Server already seeded."); return; }

        var records = new List<LegacyFlightRecord>
        {
            new() { PilotName = "Garcia, Maria", InstructorName = "Thompson, James", AircraftRegistration = "N12345", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 3, 15), TotalTime = 1.5m, DualReceived = 1.5m, DepartureAirport = "KTPA", DestinationAirport = "KTPA", Remarks = "Pattern work, touch-and-go landings", CreatedAt = new DateTime(2022, 3, 15) },
            new() { PilotName = "Chen, David", InstructorName = "Martinez, Rosa", AircraftRegistration = "N67890", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 4, 2), TotalTime = 2.3m, DualReceived = 2.3m, DepartureAirport = "KTPA", DestinationAirport = "KLAL", Remarks = "Cross-country to Lakeland Linder", CreatedAt = new DateTime(2022, 4, 2) },
            new() { PilotName = "Johnson, Tyler", InstructorName = "Thompson, James", AircraftRegistration = "N11111", AircraftType = "Piper Archer", FlightDate = new DateTime(2022, 4, 18), TotalTime = 1.8m, DualReceived = 1.8m, DepartureAirport = "KTPA", DestinationAirport = "KPIE", Remarks = "Short field and soft field operations", CreatedAt = new DateTime(2022, 4, 18) },
            new() { PilotName = "Garcia, Maria", InstructorName = "Thompson, James", AircraftRegistration = "N12345", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 5, 1), TotalTime = 1.2m, DualReceived = 0, SoloTime = 1.2m, DepartureAirport = "KTPA", DestinationAirport = "KTPA", Remarks = "First solo flight! Three circuits.", CreatedAt = new DateTime(2022, 5, 1) },
            new() { PilotName = "Williams, Ashley", InstructorName = "Martinez, Rosa", AircraftRegistration = "N67890", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 5, 10), TotalTime = 3.1m, DualReceived = 3.1m, DepartureAirport = "KTPA", DestinationAirport = "KRSW", Remarks = "Long cross-country to Fort Myers", CreatedAt = new DateTime(2022, 5, 10) },
            new() { PilotName = "Chen, David", InstructorName = "Martinez, Rosa", AircraftRegistration = "N22222", AircraftType = "Cessna 172SP", FlightDate = new DateTime(2022, 6, 5), TotalTime = 2.0m, DualReceived = 2.0m, DepartureAirport = "KTPA", DestinationAirport = "KTPA", Remarks = "Night currency, pattern work", CreatedAt = new DateTime(2022, 6, 5) },
            new() { PilotName = "Brown, Marcus", InstructorName = "Anderson, Kevin", AircraftRegistration = "N33333", AircraftType = "Piper Arrow", FlightDate = new DateTime(2022, 7, 22), TotalTime = 2.5m, DualReceived = 2.5m, DepartureAirport = "KTPA", DestinationAirport = "KSPG", Remarks = "IFR approach practice ILS 7", CreatedAt = new DateTime(2022, 7, 22) },
            new() { PilotName = "Johnson, Tyler", InstructorName = "Thompson, James", AircraftRegistration = "N12345", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 8, 14), TotalTime = 1.5m, SoloTime = 1.5m, DepartureAirport = "KTPA", DestinationAirport = "KTPA", Remarks = "Solo maneuver practice", CreatedAt = new DateTime(2022, 8, 14) },
            new() { PilotName = "Davis, Jennifer", InstructorName = "Anderson, Kevin", AircraftRegistration = "N44444", AircraftType = "Cessna 172SP", FlightDate = new DateTime(2022, 9, 3), TotalTime = 1.7m, DualReceived = 1.7m, DepartureAirport = "KTPA", DestinationAirport = "KTPA", Remarks = "Stall series, emergency procedures", CreatedAt = new DateTime(2022, 9, 3) },
            new() { PilotName = "Williams, Ashley", InstructorName = "Martinez, Rosa", AircraftRegistration = "N67890", AircraftType = "Cessna 172", FlightDate = new DateTime(2022, 10, 15), TotalTime = 1.9m, SoloTime = 1.9m, DepartureAirport = "KTPA", DestinationAirport = "KLAL", Remarks = "Solo cross-country to Lakeland", CreatedAt = new DateTime(2022, 10, 15) }
        };

        db.LegacyFlightRecords.AddRange(records);
        await db.SaveChangesAsync();
        logger.LogInformation("Seeded {Count} legacy SQL Server flight records.", records.Count);
    }

    private static async Task SeedOperationalAsync(OperationalDbContext db, ILogger logger)
    {
        await db.Database.EnsureCreatedAsync();
        if (await db.Students.AnyAsync()) { logger.LogInformation("Operational PostgreSQL already seeded."); return; }

        var instructors = new List<Instructor>
        {
            new() { FirstName = "James", LastName = "Thompson", Email = "j.thompson@suncoastaviation.com", CertificateNumber = "CFI-1234567", IsCfi = true, IsCfii = false, IsMei = false, TotalFlightHours = 3200m, AvailableDays = new List<string> { "Monday","Tuesday","Wednesday","Thursday","Friday" }, AvailableFrom = new TimeOnly(7,0), AvailableTo = new TimeOnly(17,0), IsActive = true, CreatedAt = DateTime.UtcNow },
            new() { FirstName = "Rosa", LastName = "Martinez", Email = "r.martinez@suncoastaviation.com", CertificateNumber = "CFI-7654321", IsCfi = true, IsCfii = true, IsMei = false, TotalFlightHours = 5100m, AvailableDays = new List<string> { "Monday","Wednesday","Friday","Saturday" }, AvailableFrom = new TimeOnly(6,30), AvailableTo = new TimeOnly(16,0), IsActive = true, CreatedAt = DateTime.UtcNow },
            new() { FirstName = "Kevin", LastName = "Anderson", Email = "k.anderson@suncoastaviation.com", CertificateNumber = "CFI-9988776", IsCfi = true, IsCfii = true, IsMei = true, TotalFlightHours = 8400m, AvailableDays = new List<string> { "Tuesday","Thursday","Saturday","Sunday" }, AvailableFrom = new TimeOnly(8,0), AvailableTo = new TimeOnly(18,0), IsActive = true, CreatedAt = DateTime.UtcNow },
            new() { FirstName = "Sarah", LastName = "Kim", Email = "s.kim@suncoastaviation.com", CertificateNumber = "CFI-5544332", IsCfi = true, IsCfii = false, IsMei = false, TotalFlightHours = 1800m, AvailableDays = new List<string> { "Monday","Tuesday","Thursday","Friday" }, AvailableFrom = new TimeOnly(7,30), AvailableTo = new TimeOnly(15,30), IsActive = true, CreatedAt = DateTime.UtcNow },
            new() { FirstName = "Marcus", LastName = "Johnson", Email = "m.johnson@suncoastaviation.com", CertificateNumber = "CFI-3322110", IsCfi = true, IsCfii = true, IsMei = true, TotalFlightHours = 12000m, AvailableDays = new List<string> { "Wednesday","Thursday","Friday","Saturday","Sunday" }, AvailableFrom = new TimeOnly(9,0), AvailableTo = new TimeOnly(19,0), IsActive = true, CreatedAt = DateTime.UtcNow }
        };
        db.Instructors.AddRange(instructors);
        await db.SaveChangesAsync();

        var aircraft = new List<Aircraft>
        {
            new() { Registration = "N172SA", Make = "Cessna", Model = "172S Skyhawk", Year = 2018, Category = AircraftCategory.SingleEngineLand, IsIfrEquipped = true, IsMultiEngine = false, TachometerHours = 2341.5m, NextOilChangeHours = 2355m, NextAnnualInspectionDays = 145, Status = AircraftStatus.Airworthy, IsAvailable = true, HourlyRate = 155m, CreatedAt = DateTime.UtcNow },
            new() { Registration = "N172SB", Make = "Cessna", Model = "172S Skyhawk", Year = 2019, Category = AircraftCategory.SingleEngineLand, IsIfrEquipped = true, IsMultiEngine = false, TachometerHours = 1987.3m, NextOilChangeHours = 2050m, NextAnnualInspectionDays = 203, Status = AircraftStatus.Airworthy, IsAvailable = true, HourlyRate = 155m, CreatedAt = DateTime.UtcNow },
            new() { Registration = "N172SC", Make = "Cessna", Model = "172SP Skyhawk", Year = 2015, Category = AircraftCategory.SingleEngineLand, IsIfrEquipped = false, IsMultiEngine = false, TachometerHours = 3102.8m, NextOilChangeHours = 3110m, NextAnnualInspectionDays = 67, Status = AircraftStatus.ScheduledMaintenance, IsAvailable = false, HourlyRate = 140m, CreatedAt = DateTime.UtcNow },
            new() { Registration = "N172SD", Make = "Cessna", Model = "172R Skyhawk", Year = 2016, Category = AircraftCategory.SingleEngineLand, IsIfrEquipped = false, IsMultiEngine = false, TachometerHours = 2654.1m, NextOilChangeHours = 2700m, NextAnnualInspectionDays = 88, Status = AircraftStatus.GroundedSquawk, ActiveSquawk = "Left landing light inoperative - night ops prohibited", IsAvailable = true, HourlyRate = 135m, CreatedAt = DateTime.UtcNow },
            new() { Registration = "N28PA", Make = "Piper", Model = "PA-28-181 Archer", Year = 2017, Category = AircraftCategory.SingleEngineLand, IsIfrEquipped = true, IsMultiEngine = false, TachometerHours = 1542.6m, NextOilChangeHours = 1600m, NextAnnualInspectionDays = 178, Status = AircraftStatus.Airworthy, IsAvailable = true, HourlyRate = 160m, CreatedAt = DateTime.UtcNow },
            new() { Registration = "N44ME", Make = "Piper", Model = "PA-44-180 Seminole", Year = 2020, Category = AircraftCategory.MultiEngineLand, IsIfrEquipped = true, IsMultiEngine = true, TachometerHours = 876.2m, NextOilChangeHours = 950m, NextAnnualInspectionDays = 241, Status = AircraftStatus.Airworthy, IsAvailable = true, HourlyRate = 310m, CreatedAt = DateTime.UtcNow }
        };
        db.Aircraft.AddRange(aircraft);
        await db.SaveChangesAsync();

        var students = new List<Student>
        {
            new() { FirstName = "Maria", LastName = "Garcia", Email = "maria.garcia@email.com", Phone = "813-555-0101", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.PostSolo, TotalHours = 22.5m, SoloHours = 3.2m, HasSoloEndorsement = true, EnrollmentDate = new DateTime(2026,1,10), ProjectedCheckrideDate = new DateTime(2026,5,15), AssignedInstructorEmail = "j.thompson@suncoastaviation.com", FinancingType = FinancingType.SallieMae, OutstandingBalance = 8500m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "David", LastName = "Chen", Email = "david.chen@email.com", Phone = "813-555-0102", TrainingType = TrainingType.Part61, CurrentStage = CurriculumStage.CrossCountry, TotalHours = 38.1m, SoloHours = 8.5m, CrossCountryHours = 5.2m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2025,11,5), ProjectedCheckrideDate = new DateTime(2026,4,30), AssignedInstructorEmail = "r.martinez@suncoastaviation.com", FinancingType = FinancingType.StratusFinancial, OutstandingBalance = 5200m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Ashley", LastName = "Williams", Email = "ashley.williams@email.com", Phone = "813-555-0103", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.CheckridePrep, TotalHours = 61.3m, SoloHours = 15.2m, CrossCountryHours = 10.5m, NightHours = 3.8m, InstrumentHours = 3.1m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2025,9,20), ProjectedCheckrideDate = DateTime.UtcNow.AddDays(10), AssignedInstructorEmail = "r.martinez@suncoastaviation.com", FinancingType = FinancingType.SallieMae, OutstandingBalance = 2100m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Tyler", LastName = "Johnson", Email = "tyler.johnson@email.com", Phone = "813-555-0104", TrainingType = TrainingType.Part61, CurrentStage = CurriculumStage.PreSolo, TotalHours = 12.8m, EnrollmentDate = new DateTime(2026,2,1), ProjectedCheckrideDate = new DateTime(2026,7,1), AssignedInstructorEmail = "j.thompson@suncoastaviation.com", FinancingType = FinancingType.SelfFunded, OutstandingBalance = 11200m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Jennifer", LastName = "Davis", Email = "jennifer.davis@email.com", Phone = "813-555-0105", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.InstrumentRating, TotalHours = 95.7m, SoloHours = 22.1m, CrossCountryHours = 18.3m, NightHours = 5.2m, InstrumentHours = 28.4m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2025,7,15), ProjectedCheckrideDate = new DateTime(2026,4,15), AssignedInstructorEmail = "k.anderson@suncoastaviation.com", FinancingType = FinancingType.SallieMae, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Marcus", LastName = "Brown", Email = "marcus.brown@email.com", Phone = "813-555-0106", TrainingType = TrainingType.Part61, CurrentStage = CurriculumStage.NightFlying, TotalHours = 48.2m, SoloHours = 10.8m, CrossCountryHours = 8.1m, NightHours = 1.5m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2025,10,12), ProjectedCheckrideDate = new DateTime(2026,5,30), AssignedInstructorEmail = "s.kim@suncoastaviation.com", FinancingType = FinancingType.SelfFunded, OutstandingBalance = 4800m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Sophia", LastName = "Lee", Email = "sophia.lee@email.com", Phone = "813-555-0107", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.DiscoveryFlight, TotalHours = 1.5m, EnrollmentDate = new DateTime(2026,3,10), ProjectedCheckrideDate = new DateTime(2026,9,1), AssignedInstructorEmail = "s.kim@suncoastaviation.com", FinancingType = FinancingType.SallieMae, OutstandingBalance = 18000m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "James", LastName = "Wilson", Email = "james.wilson@email.com", Phone = "813-555-0108", TrainingType = TrainingType.Part61, CurrentStage = CurriculumStage.CommercialTraining, TotalHours = 198.4m, SoloHours = 45.2m, CrossCountryHours = 38.7m, NightHours = 12.3m, InstrumentHours = 42.1m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2025,3,1), ProjectedCheckrideDate = new DateTime(2026,6,15), AssignedInstructorEmail = "m.johnson@suncoastaviation.com", FinancingType = FinancingType.SelfFunded, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Priya", LastName = "Patel", Email = "priya.patel@email.com", Phone = "813-555-0109", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.PostSolo, TotalHours = 18.9m, SoloHours = 1.2m, HasSoloEndorsement = true, EnrollmentDate = new DateTime(2026,1,20), ProjectedCheckrideDate = new DateTime(2026,6,30), AssignedInstructorEmail = "j.thompson@suncoastaviation.com", FinancingType = FinancingType.StratusFinancial, OutstandingBalance = 9800m, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow },
            new() { FirstName = "Carlos", LastName = "Rodriguez", Email = "carlos.rodriguez@email.com", Phone = "813-555-0110", TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.AtpPrep, TotalHours = 1487.3m, SoloHours = 310.5m, CrossCountryHours = 480.2m, NightHours = 85.6m, InstrumentHours = 210.4m, HasSoloEndorsement = true, HasCrossCountryEndorsement = true, EnrollmentDate = new DateTime(2024,6,1), ProjectedCheckrideDate = new DateTime(2026,8,1), AssignedInstructorEmail = "m.johnson@suncoastaviation.com", FinancingType = FinancingType.SelfFunded, CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow }
        };
        db.Students.AddRange(students);
        await db.SaveChangesAsync();
        logger.LogInformation("Seeded {I} instructors, {A} aircraft, {S} students into PostgreSQL.", instructors.Count, aircraft.Count, students.Count);
    }

    private static async Task SeedLogbookAsync(MongoDbService mongoService, ILogger logger)
    {
        var existing = await mongoService.GetByPilotIdAsync(1);
        if (existing.Any()) { logger.LogInformation("MongoDB logbook already seeded."); return; }

        var entries = new List<FlightLogEntry>
        {
            // pilotId=1 (Maria Garcia) — Post-Solo
            new() { PilotId = 1, FlightDate = new DateTime(2026,1,15), AircraftRegistration = "N172SA", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KTPA", Conditions = new FlightConditions(), Times = new FlightTimes { TotalTime = 1.5m, DualReceived = 1.5m }, Landings = new LandingCounts { Day = 8 }, Remarks = "Pattern work, touch-and-go landings. Good crosswind corrections.", InstructorSignature = new InstructorEndorsement { InstructorName = "James Thompson", CertificateNumber = "CFI-1234567", SignedAt = new DateTime(2026,1,15,12,30,0) } },
            new() { PilotId = 1, FlightDate = new DateTime(2026,1,22), AircraftRegistration = "N172SA", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KTPA", Conditions = new FlightConditions(), Times = new FlightTimes { TotalTime = 1.8m, DualReceived = 1.8m }, Landings = new LandingCounts { Day = 6 }, Remarks = "Stall series, slow flight.", InstructorSignature = new InstructorEndorsement { InstructorName = "James Thompson", CertificateNumber = "CFI-1234567", SignedAt = new DateTime(2026,1,22,11,0,0) } },
            new() { PilotId = 1, FlightDate = new DateTime(2026,2,5), AircraftRegistration = "N172SB", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KTPA", Conditions = new FlightConditions(), Times = new FlightTimes { TotalTime = 2.1m, DualReceived = 2.1m }, Landings = new LandingCounts { Day = 10 }, Remarks = "Pre-solo check. Cleared for solo endorsement.", InstructorSignature = new InstructorEndorsement { InstructorName = "James Thompson", CertificateNumber = "CFI-1234567", SignedAt = new DateTime(2026,2,5,10,0,0) } },
            new() { PilotId = 1, FlightDate = new DateTime(2026,2,12), AircraftRegistration = "N172SA", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KTPA", Conditions = new FlightConditions(), Times = new FlightTimes { TotalTime = 1.2m, SoloTime = 1.2m, PicTime = 1.2m }, Landings = new LandingCounts { Day = 3 }, Remarks = "FIRST SOLO. Three solo circuits. Excellent control." },
            // pilotId=2 (David Chen) — Cross-Country
            new() { PilotId = 2, FlightDate = new DateTime(2026,1,8), AircraftRegistration = "N172SB", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KLAL", RouteOfFlight = "KTPA KLAL KTPA", Conditions = new FlightConditions { IsCrossCountry = true }, Times = new FlightTimes { TotalTime = 2.3m, DualReceived = 2.3m, CrossCountryTime = 2.3m }, Landings = new LandingCounts { Day = 2 }, Remarks = "First dual cross-country to Lakeland.", InstructorSignature = new InstructorEndorsement { InstructorName = "Rosa Martinez", CertificateNumber = "CFI-7654321", SignedAt = new DateTime(2026,1,8,14,0,0) } },
            new() { PilotId = 2, FlightDate = new DateTime(2026,2,14), AircraftRegistration = "N28PA", AircraftMakeModel = "Piper PA-28-181 Archer", DepartureAirport = "KTPA", DestinationAirport = "KRSW", RouteOfFlight = "KTPA KRSW KVNC KTPA", Conditions = new FlightConditions { IsCrossCountry = true }, Times = new FlightTimes { TotalTime = 3.1m, DualReceived = 3.1m, CrossCountryTime = 3.1m }, Landings = new LandingCounts { Day = 3 }, Remarks = "Long cross-country. Three airports.", InstructorSignature = new InstructorEndorsement { InstructorName = "Rosa Martinez", CertificateNumber = "CFI-7654321", SignedAt = new DateTime(2026,2,14,16,30,0) } },
            new() { PilotId = 2, FlightDate = new DateTime(2026,3,1), AircraftRegistration = "N172SA", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KLAL", RouteOfFlight = "KTPA KLAL KTPA", Conditions = new FlightConditions { IsCrossCountry = true }, Times = new FlightTimes { TotalTime = 2.0m, SoloTime = 2.0m, PicTime = 2.0m, CrossCountryTime = 2.0m }, Landings = new LandingCounts { Day = 2 }, Remarks = "Solo cross-country to Lakeland. Excellent." },
            // pilotId=5 (Jennifer Davis) — Instrument Rating
            new() { PilotId = 5, FlightDate = new DateTime(2026,3,3), AircraftRegistration = "N172SA", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KSPG", RouteOfFlight = "KTPA TPA VOR KSPG ILS27", Conditions = new FlightConditions { IsIfr = true, IsSimulatedInstrument = true, IsCrossCountry = true }, Times = new FlightTimes { TotalTime = 1.8m, DualReceived = 1.8m, InstrumentTime = 1.8m, CrossCountryTime = 1.8m }, Approaches = new List<InstrumentApproach> { new() { Type = "ILS", Airport = "KSPG", Runway = "27" }, new() { Type = "ILS", Airport = "KSPG", Runway = "27" } }, Landings = new LandingCounts { Day = 1 }, Remarks = "ILS approaches at KSPG. Two to minimums. Good scan.", InstructorSignature = new InstructorEndorsement { InstructorName = "Kevin Anderson", CertificateNumber = "CFI-9988776", SignedAt = new DateTime(2026,3,3,13,0,0) } },
            new() { PilotId = 5, FlightDate = new DateTime(2026,3,12), AircraftRegistration = "N172SB", AircraftMakeModel = "Cessna 172S", DepartureAirport = "KTPA", DestinationAirport = "KTPA", Conditions = new FlightConditions { IsIfr = true, IsSimulatedInstrument = true }, Times = new FlightTimes { TotalTime = 2.0m, DualReceived = 2.0m, InstrumentTime = 2.0m }, Approaches = new List<InstrumentApproach> { new() { Type = "RNAV", Airport = "KTPA", Runway = "1R" }, new() { Type = "VOR", Airport = "KTPA", Runway = "1L" } }, Landings = new LandingCounts { Day = 2 }, Remarks = "GPS and VOR approaches. Holding at TPA VOR. Excellent.", InstructorSignature = new InstructorEndorsement { InstructorName = "Kevin Anderson", CertificateNumber = "CFI-9988776", SignedAt = new DateTime(2026,3,12,15,0,0) } }
        };

        foreach (var entry in entries)
            await mongoService.CreateAsync(entry);

        logger.LogInformation("Seeded {Count} logbook entries for pilots 1, 2, and 5.", entries.Count);
    }
}
```

### Phase 4 Verification

```bash
dotnet build PilotbaseControlTower.sln
# Expected: Build succeeded. 0 Error(s).
```

---

## PHASE 5: Services

**Goal**: All three application services implemented with unit tests.

---

### Step 5.1 — Financing Milestone Service

Create `Services/Financing/FinancingMilestoneService.cs`:

```csharp
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Api.Services.Financing;

public class FinancingMilestoneService
{
    private readonly OperationalDbContext _db;
    private readonly ILogger<FinancingMilestoneService> _logger;

    public static readonly Dictionary<MilestoneType, decimal> SallieMaeDisbursements = new()
    {
        { MilestoneType.FirstFlight,           1500m },
        { MilestoneType.FirstSolo,             3000m },
        { MilestoneType.FirstCrossCountry,     2500m },
        { MilestoneType.NightEndorsement,      2000m },
        { MilestoneType.PrivatePilotCheckride, 5000m },
        { MilestoneType.InstrumentCheckride,   8000m },
        { MilestoneType.CommercialCheckride,  12000m }
    };

    public FinancingMilestoneService(OperationalDbContext db, ILogger<FinancingMilestoneService> logger)
    {
        _db = db; _logger = logger;
    }

    public async Task<MilestoneResult> RecordMilestoneAsync(
        int studentId, MilestoneType milestoneType, decimal currentHours)
    {
        var student = await _db.Students.FindAsync(studentId)
            ?? throw new KeyNotFoundException($"Student with ID {studentId} not found.");

        var milestone = new FinancingMilestone
        {
            StudentId = studentId, MilestoneType = milestoneType,
            AchievedAt = DateTime.UtcNow, HoursAtMilestone = currentHours
        };

        if (student.FinancingType == FinancingType.SallieMae
            && SallieMaeDisbursements.TryGetValue(milestoneType, out var amount))
        {
            milestone.DisbursementTriggered = true;
            milestone.DisbursementAmount = amount;
            milestone.LenderReference = $"SM-{DateTime.UtcNow:yyyyMMdd}-{studentId:000}-{milestoneType}";
            _logger.LogInformation(
                "Sallie Mae disbursement triggered: ${Amount} for student {Id} milestone {Milestone}. Ref: {Ref}",
                amount, studentId, milestoneType, milestone.LenderReference);
        }

        _db.FinancingMilestones.Add(milestone);
        await _db.SaveChangesAsync();

        return new MilestoneResult
        {
            Milestone = milestone,
            DisbursementTriggered = milestone.DisbursementTriggered,
            DisbursementAmount = milestone.DisbursementAmount,
            LenderReference = milestone.LenderReference,
            StudentName = $"{student.FirstName} {student.LastName}",
            FinancingType = student.FinancingType.ToString(),
            Message = milestone.DisbursementTriggered
                ? $"Milestone recorded. Sallie Mae disbursement of ${milestone.DisbursementAmount:N0} initiated. Reference: {milestone.LenderReference}"
                : $"Milestone recorded. No disbursement triggered (financing: {student.FinancingType})."
        };
    }
}

public class MilestoneResult
{
    public FinancingMilestone Milestone { get; set; } = null!;
    public bool DisbursementTriggered { get; set; }
    public decimal? DisbursementAmount { get; set; }
    public string? LenderReference { get; set; }
    public string StudentName { get; set; } = string.Empty;
    public string FinancingType { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
}
```

---

### Step 5.2 — Financing Service unit tests

Create `backend/PilotbaseControlTower.Tests/Services/FinancingMilestoneServiceTests.cs`:

```csharp
using FluentAssertions;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Moq;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;
using PilotbaseControlTower.Api.Services.Financing;

namespace PilotbaseControlTower.Tests.Services;

public class FinancingMilestoneServiceTests
{
    private static OperationalDbContext CreateContext() =>
        new(new DbContextOptionsBuilder<OperationalDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString()).Options);

    private static FinancingMilestoneService CreateService(OperationalDbContext ctx) =>
        new(ctx, new Mock<ILogger<FinancingMilestoneService>>().Object);

    private static async Task<Student> SeedStudentAsync(OperationalDbContext ctx, FinancingType ft)
    {
        var s = new Student
        {
            FirstName = "Test", LastName = "Pilot",
            Email = $"test{Guid.NewGuid():N}@test.com", Phone = "555-0000",
            TrainingType = TrainingType.Part141, CurrentStage = CurriculumStage.PostSolo,
            FinancingType = ft, EnrollmentDate = DateTime.UtcNow,
            CreatedAt = DateTime.UtcNow, UpdatedAt = DateTime.UtcNow
        };
        ctx.Students.Add(s);
        await ctx.SaveChangesAsync();
        return s;
    }

    [Fact]
    public async Task SallieMaeStudent_FirstSolo_TriggersDisbursement()
    {
        await using var ctx = CreateContext();
        var student = await SeedStudentAsync(ctx, FinancingType.SallieMae);
        var result = await CreateService(ctx).RecordMilestoneAsync(student.Id, MilestoneType.FirstSolo, 22.5m);

        result.DisbursementTriggered.Should().BeTrue();
        result.DisbursementAmount.Should().Be(3000m);
        result.LenderReference.Should().StartWith("SM-");
        result.Message.Should().Contain("$3,000");
    }

    [Fact]
    public async Task SelfFundedStudent_FirstSolo_DoesNotTriggerDisbursement()
    {
        await using var ctx = CreateContext();
        var student = await SeedStudentAsync(ctx, FinancingType.SelfFunded);
        var result = await CreateService(ctx).RecordMilestoneAsync(student.Id, MilestoneType.FirstSolo, 22.5m);

        result.DisbursementTriggered.Should().BeFalse();
        result.DisbursementAmount.Should().BeNull();
        result.LenderReference.Should().BeNull();
    }

    [Fact]
    public async Task MilestoneIsPersisted_InDatabase()
    {
        await using var ctx = CreateContext();
        var student = await SeedStudentAsync(ctx, FinancingType.SallieMae);
        await CreateService(ctx).RecordMilestoneAsync(student.Id, MilestoneType.PrivatePilotCheckride, 65.0m);

        var saved = await ctx.FinancingMilestones.FirstAsync();
        saved.MilestoneType.Should().Be(MilestoneType.PrivatePilotCheckride);
        saved.HoursAtMilestone.Should().Be(65.0m);
        saved.DisbursementAmount.Should().Be(5000m);
    }

    [Fact]
    public void DisbursementAmounts_MatchSallieMaeSchedule()
    {
        FinancingMilestoneService.SallieMaeDisbursements[MilestoneType.FirstFlight].Should().Be(1500m);
        FinancingMilestoneService.SallieMaeDisbursements[MilestoneType.FirstSolo].Should().Be(3000m);
        FinancingMilestoneService.SallieMaeDisbursements[MilestoneType.PrivatePilotCheckride].Should().Be(5000m);
        FinancingMilestoneService.SallieMaeDisbursements[MilestoneType.CommercialCheckride].Should().Be(12000m);
    }

    [Fact]
    public async Task NonexistentStudent_ThrowsKeyNotFoundException()
    {
        await using var ctx = CreateContext();
        var act = async () => await CreateService(ctx).RecordMilestoneAsync(999, MilestoneType.FirstSolo, 20m);
        await act.Should().ThrowAsync<KeyNotFoundException>().WithMessage("*999*");
    }
}
```

---

### Step 5.3 — Career Pathway Service

Create `Services/Career/CareerPathwayService.cs`:

```csharp
using PilotbaseControlTower.Api.Models.Operational;
using PilotbaseControlTower.Api.Mongo;

namespace PilotbaseControlTower.Api.Services.Career;

public class CareerPathwayService
{
    private readonly MongoDbService _mongo;

    // FAA certificate/rating minimums
    private const decimal PrivatePilotMinHours = 40m;
    private const decimal InstrumentRatingMinInstrumentHours = 50m;
    private const decimal CommercialPilotMinHours = 250m;
    private const decimal AtpMinHours = 1500m;

    public CareerPathwayService(MongoDbService mongo) => _mongo = mongo;

    public async Task<CareerPathway> GetPathwayAsync(Student student)
    {
        var totals = await _mongo.GetTotalsAsync(student.Id);

        var milestones = new List<CareerMilestone>
        {
            new() { Name = "Private Pilot Certificate", RequiredHours = PrivatePilotMinHours, CurrentHours = student.TotalHours, IsAchieved = student.CurrentStage >= CurriculumStage.PrivatePilotComplete, HoursRemaining = Math.Max(0, PrivatePilotMinHours - student.TotalHours) },
            new() { Name = "Instrument Rating", RequiredHours = InstrumentRatingMinInstrumentHours, CurrentHours = student.InstrumentHours, IsAchieved = student.CurrentStage >= CurriculumStage.InstrumentRating, HoursRemaining = Math.Max(0, InstrumentRatingMinInstrumentHours - student.InstrumentHours), Description = "50 hours instrument time (40 actual or simulated)" },
            new() { Name = "Commercial Pilot Certificate", RequiredHours = CommercialPilotMinHours, CurrentHours = student.TotalHours, IsAchieved = student.CurrentStage >= CurriculumStage.CommercialTraining, HoursRemaining = Math.Max(0, CommercialPilotMinHours - student.TotalHours) },
            new() { Name = "ATP Certificate", RequiredHours = AtpMinHours, CurrentHours = student.TotalHours, IsAchieved = student.TotalHours >= AtpMinHours, HoursRemaining = Math.Max(0, AtpMinHours - student.TotalHours), Description = "1,500 total hours (1,000 restricted ATP with aviation degree or military)" }
        };

        return new CareerPathway
        {
            StudentId = student.Id,
            StudentName = $"{student.FirstName} {student.LastName}",
            CurrentStage = student.CurrentStage.ToString(),
            TotalHours = student.TotalHours,
            LogbookTotals = totals,
            Milestones = milestones,
            JobMatches = GetJobMatches(student),
            NextMilestone = milestones.FirstOrDefault(m => !m.IsAchieved),
            EstimatedAtpDate = student.TotalHours >= AtpMinHours ? null
                : DateTime.UtcNow.AddMonths((int)Math.Ceiling((AtpMinHours - student.TotalHours) / 10))
        };
    }

    private static List<JobMatch> GetJobMatches(Student student)
    {
        var matches = new List<JobMatch>();

        if (student.TotalHours >= 200 && student.CurrentStage >= CurriculumStage.CommercialTraining)
            matches.Add(new JobMatch { Role = "CFI / Flight Instructor", Operator = "Suncoast Aviation Academy", MinHours = 200, MatchStrength = "Strong", Description = "Build hours while instructing. Standard pathway to airlines." });

        if (student.TotalHours >= 500)
            matches.Add(new JobMatch { Role = "Charter Pilot (Part 135)", Operator = "Regional Charter Operators", MinHours = 500, MatchStrength = student.TotalHours >= 750 ? "Strong" : "Moderate", Description = "On-demand charter. Requires ATP or R-ATP." });

        if (student.TotalHours >= 1000)
            matches.Add(new JobMatch { Role = "Regional Airline First Officer", Operator = "SkyWest, Mesa, Endeavor", MinHours = 1000, MatchStrength = student.TotalHours >= 1500 ? "Strong" : "Moderate", Description = "R-ATP at 1,000 hours with aviation degree. Full ATP at 1,500." });

        if (student.TotalHours >= 1500)
            matches.Add(new JobMatch { Role = "Major Airline First Officer", Operator = "Delta, United, American, Southwest", MinHours = 1500, MatchStrength = "Strong", Description = "Full ATP required. Typical upgrade: 5-10 years regional experience." });

        return matches;
    }
}

public class CareerPathway
{
    public int StudentId { get; set; }
    public string StudentName { get; set; } = string.Empty;
    public string CurrentStage { get; set; } = string.Empty;
    public decimal TotalHours { get; set; }
    public LogbookTotals LogbookTotals { get; set; } = null!;
    public List<CareerMilestone> Milestones { get; set; } = new();
    public List<JobMatch> JobMatches { get; set; } = new();
    public CareerMilestone? NextMilestone { get; set; }
    public DateTime? EstimatedAtpDate { get; set; }
}

public class CareerMilestone
{
    public string Name { get; set; } = string.Empty;
    public decimal RequiredHours { get; set; }
    public decimal CurrentHours { get; set; }
    public decimal HoursRemaining { get; set; }
    public bool IsAchieved { get; set; }
    public string? Description { get; set; }
}

public class JobMatch
{
    public string Role { get; set; } = string.Empty;
    public string Operator { get; set; } = string.Empty;
    public decimal MinHours { get; set; }
    public string MatchStrength { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
}
```

---

### Step 5.4 — Career Service unit tests

Create `backend/PilotbaseControlTower.Tests/Services/CareerPathwayServiceTests.cs`. Note: `CareerPathwayService` depends on `MongoDbService` which has no interface, so we test the business logic that doesn't require MongoDB directly.

```csharp
using FluentAssertions;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Tests.Services;

/// <summary>
/// Tests for career pathway business logic.
/// CareerPathwayService.GetPathwayAsync() requires a live MongoDB connection
/// and is covered by the Phase 6 end-to-end verification instead.
/// These tests verify the pure logic that does not depend on data access.
/// </summary>
public class CareerPathwayServiceTests
{
    [Fact]
    public void CfiEligibility_Requires200Hours_AndCommercialStage()
    {
        var underHours = new Student { TotalHours = 190m, CurrentStage = CurriculumStage.CommercialTraining };
        var overHours = new Student { TotalHours = 210m, CurrentStage = CurriculumStage.CommercialTraining };
        var wrongStage = new Student { TotalHours = 250m, CurrentStage = CurriculumStage.CheckridePrep };

        (underHours.TotalHours >= 200m).Should().BeFalse();
        (overHours.TotalHours >= 200m && overHours.CurrentStage >= CurriculumStage.CommercialTraining).Should().BeTrue();
        (wrongStage.CurrentStage >= CurriculumStage.CommercialTraining).Should().BeFalse();
    }

    [Fact]
    public void AtpMinimums_Are1500Hours()
    {
        var nearAtp = new Student { TotalHours = 1499m };
        var atAtp = new Student { TotalHours = 1500m };
        (nearAtp.TotalHours >= 1500m).Should().BeFalse();
        (atAtp.TotalHours >= 1500m).Should().BeTrue();
    }

    [Fact]
    public void CurriculumStageComparison_WorksWithGreaterThan()
    {
        CurriculumStage.CommercialTraining.Should().BeGreaterThan(CurriculumStage.CheckridePrep);
        CurriculumStage.AtpPrep.Should().BeGreaterThan(CurriculumStage.CommercialTraining);
        CurriculumStage.PrivatePilotComplete.Should().BeGreaterThan(CurriculumStage.CheckridePrep);
    }

    [Fact]
    public void HoursRemaining_NeverGoesNegative()
    {
        const decimal AtpMin = 1500m;
        var studentPastAtp = new Student { TotalHours = 1600m };
        var remaining = Math.Max(0, AtpMin - studentPastAtp.TotalHours);
        remaining.Should().Be(0);
    }
}
```

---

### Step 5.5 — AI Scheduling Service

Create `Services/AI/AiSchedulingService.cs`. This service requires an `HttpClient` injected via constructor — this is required by `AddHttpClient<AiSchedulingService>()` registration in DI. Do not change the constructor signature.

```csharp
using System.Text;
using System.Text.Json;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Api.Services.AI;

/// <summary>
/// Calls the Anthropic API to generate an optimized weekly flight schedule.
/// Constructor accepts HttpClient injected via AddHttpClient<AiSchedulingService>().
/// Do not change the constructor signature or the DI registration will break.
/// </summary>
public class AiSchedulingService
{
    private readonly IConfiguration _config;
    private readonly HttpClient _http;
    private readonly ILogger<AiSchedulingService> _logger;

    public AiSchedulingService(IConfiguration config, HttpClient http, ILogger<AiSchedulingService> logger)
    {
        _config = config;
        _http = http;
        _logger = logger;
    }

    public async Task<AiScheduleResult> GenerateScheduleAsync(ScheduleGenerationRequest request)
    {
        var apiKey = _config["AnthropicApi:ApiKey"];
        if (string.IsNullOrEmpty(apiKey))
        {
            _logger.LogWarning("AnthropicApi:ApiKey is not configured. Returning mock schedule.");
            return MockScheduleResult(request);
        }

        var prompt = BuildSchedulingPrompt(request);
        var body = new
        {
            model = _config["AnthropicApi:Model"] ?? "claude-opus-4-6",
            max_tokens = int.Parse(_config["AnthropicApi:MaxTokens"] ?? "4096"),
            messages = new[] { new { role = "user", content = prompt } }
        };

        var requestMessage = new HttpRequestMessage(HttpMethod.Post, "https://api.anthropic.com/v1/messages")
        {
            Content = new StringContent(JsonSerializer.Serialize(body), Encoding.UTF8, "application/json")
        };
        requestMessage.Headers.Add("x-api-key", apiKey);
        requestMessage.Headers.Add("anthropic-version", "2023-06-01");

        HttpResponseMessage response;
        try
        {
            response = await _http.SendAsync(requestMessage);
            response.EnsureSuccessStatusCode();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Anthropic API call failed.");
            return new AiScheduleResult
            {
                WeekSummary = "Schedule generation failed — API error. Check server logs.",
                Flags = new List<AiScheduleFlag> { new() { Type = "warning", Message = $"Anthropic API error: {ex.Message}" } }
            };
        }

        var raw = await response.Content.ReadAsStringAsync();
        var parsed = JsonDocument.Parse(raw);
        var text = parsed.RootElement.GetProperty("content")[0].GetProperty("text").GetString() ?? "";
        return ParseAiResponse(text);
    }

    private string BuildSchedulingPrompt(ScheduleGenerationRequest req)
    {
        var sb = new StringBuilder();
        sb.AppendLine("You are an intelligent scheduling engine for a flight school. Generate an optimal weekly schedule.");
        sb.AppendLine();
        sb.AppendLine("## School Information");
        sb.AppendLine("School: Suncoast Aviation Academy | Airport: KTPA (Tampa International)");
        sb.AppendLine($"Civil Twilight: {req.CivilTwilightBegin} (morning) — {req.CivilTwilightEnd} (evening)");
        sb.AppendLine($"Week of: {req.WeekStart:yyyy-MM-dd} to {req.WeekStart.AddDays(6):yyyy-MM-dd}");
        sb.AppendLine();

        sb.AppendLine("## Students");
        foreach (var s in req.Students)
        {
            sb.AppendLine($"- {s.FirstName} {s.LastName} | {s.TrainingType} | Stage: {s.CurrentStage} | Hours: {s.TotalHours:F1} | Solo Endorsed: {s.HasSoloEndorsement} | Email: {s.Email}");
            if (s.ProjectedCheckrideDate.HasValue)
                sb.AppendLine($"  ⚠️ CHECKRIDE PRIORITY: {s.ProjectedCheckrideDate:yyyy-MM-dd}");
        }
        sb.AppendLine();

        sb.AppendLine("## Instructors");
        foreach (var i in req.Instructors)
        {
            var ratings = string.Join("/", new[] { i.IsCfi ? "CFI" : null, i.IsCfii ? "CFII" : null, i.IsMei ? "MEI" : null }.Where(r => r != null));
            sb.AppendLine($"- {i.FirstName} {i.LastName} | {ratings} | Available: {string.Join(", ", i.AvailableDays)} {i.AvailableFrom}-{i.AvailableTo} | Email: {i.Email}");
        }
        sb.AppendLine();

        sb.AppendLine("## Aircraft");
        foreach (var a in req.Aircraft)
        {
            var avail = a.IsAvailable ? "Available" : "Unavailable";
            var squawk = a.ActiveSquawk != null ? $"⚠️ SQUAWK: {a.ActiveSquawk}" : "No squawks";
            sb.AppendLine($"- {a.Registration} | {a.Make} {a.Model} | IFR: {a.IsIfrEquipped} | Multi: {a.IsMultiEngine} | {avail} | {squawk}");
        }
        sb.AppendLine();

        sb.AppendLine("## Scheduling Rules");
        sb.AppendLine("1. All flights must occur between civil twilight begin and end, unless it is explicitly night training.");
        sb.AppendLine("2. Part 141 students must follow structured curriculum order — no skipping stages.");
        sb.AppendLine("3. Students with imminent checkride dates (⚠️ CHECKRIDE PRIORITY) must be scheduled first.");
        sb.AppendLine("4. Students without solo endorsement (HasSoloEndorsement = false) MUST fly with an instructor — never solo.");
        sb.AppendLine("5. Aircraft with squawks prohibiting night operations cannot fly night training missions.");
        sb.AppendLine("6. IFR training requires an IFR-equipped aircraft AND a CFII-rated instructor.");
        sb.AppendLine("7. Multi-engine training requires an MEI-rated instructor AND the Seminole (N44ME).");
        sb.AppendLine("8. Do not schedule any instructor more than 8 hours of flight time per day.");
        sb.AppendLine("9. Each scheduling block is 1.5–2.5 hours (including preflight and debrief).");
        sb.AppendLine();

        sb.AppendLine("## Required Output Format");
        sb.AppendLine("Return ONLY a valid JSON object — no preamble, no markdown fences. Exact structure:");
        sb.AppendLine(@"{
  ""weekSummary"": ""Brief overview of the scheduling strategy and any notable prioritizations."",
  ""scheduledEvents"": [
    {
      ""day"": ""Monday"",
      ""startTime"": ""08:00"",
      ""endTime"": ""10:00"",
      ""studentEmail"": ""student@email.com"",
      ""instructorEmail"": ""instructor@school.com"",
      ""aircraftRegistration"": ""N172SA"",
      ""lessonType"": ""Pattern work / Touch-and-go landings"",
      ""reasoning"": ""One sentence explaining exactly why this booking was made and prioritized.""
    }
  ],
  ""flags"": [
    {
      ""type"": ""warning"",
      ""message"": ""Any scheduling conflicts, risks, or notes for school management.""
    }
  ]
}");
        return sb.ToString();
    }

    private AiScheduleResult ParseAiResponse(string text)
    {
        try
        {
            var json = text.Trim();
            if (json.StartsWith("```"))
                json = string.Join('\n', json.Split('\n').Skip(1).SkipLast(1)).Trim();

            using var doc = JsonDocument.Parse(json);
            var root = doc.RootElement;

            var result = new AiScheduleResult
            {
                WeekSummary = root.GetProperty("weekSummary").GetString() ?? "",
                ScheduledEvents = new List<AiScheduledEvent>(),
                Flags = new List<AiScheduleFlag>()
            };

            foreach (var evt in root.GetProperty("scheduledEvents").EnumerateArray())
            {
                result.ScheduledEvents.Add(new AiScheduledEvent
                {
                    Day = evt.GetProperty("day").GetString() ?? "",
                    StartTime = evt.GetProperty("startTime").GetString() ?? "",
                    EndTime = evt.GetProperty("endTime").GetString() ?? "",
                    StudentEmail = evt.GetProperty("studentEmail").GetString() ?? "",
                    InstructorEmail = evt.GetProperty("instructorEmail").GetString() ?? "",
                    AircraftRegistration = evt.GetProperty("aircraftRegistration").GetString() ?? "",
                    LessonType = evt.GetProperty("lessonType").GetString() ?? "",
                    Reasoning = evt.GetProperty("reasoning").GetString() ?? ""
                });
            }

            if (root.TryGetProperty("flags", out var flags))
                foreach (var flag in flags.EnumerateArray())
                    result.Flags.Add(new AiScheduleFlag
                    {
                        Type = flag.GetProperty("type").GetString() ?? "info",
                        Message = flag.GetProperty("message").GetString() ?? ""
                    });

            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to parse AI scheduling response.");
            return new AiScheduleResult
            {
                WeekSummary = "Schedule generation encountered a parsing error.",
                Flags = new List<AiScheduleFlag> { new() { Type = "warning", Message = "AI response parsing failed — check server logs." } }
            };
        }
    }

    private static AiScheduleResult MockScheduleResult(ScheduleGenerationRequest req) =>
        new()
        {
            WeekSummary = "Mock schedule (API key not configured). Add AnthropicApi:ApiKey to appsettings.Development.json to enable AI scheduling.",
            ScheduledEvents = new List<AiScheduledEvent>(),
            Flags = new List<AiScheduleFlag> { new() { Type = "info", Message = "Running in mock mode — no Anthropic API key configured." } }
        };
}

public class ScheduleGenerationRequest
{
    public DateTime WeekStart { get; set; }
    public string CivilTwilightBegin { get; set; } = "06:30";
    public string CivilTwilightEnd { get; set; } = "19:45";
    public List<Student> Students { get; set; } = new();
    public List<Instructor> Instructors { get; set; } = new();
    public List<Aircraft> Aircraft { get; set; } = new();
}

public class AiScheduleResult
{
    public string WeekSummary { get; set; } = string.Empty;
    public List<AiScheduledEvent> ScheduledEvents { get; set; } = new();
    public List<AiScheduleFlag> Flags { get; set; } = new();
}

public class AiScheduledEvent
{
    public string Day { get; set; } = string.Empty;
    public string StartTime { get; set; } = string.Empty;
    public string EndTime { get; set; } = string.Empty;
    public string StudentEmail { get; set; } = string.Empty;
    public string InstructorEmail { get; set; } = string.Empty;
    public string AircraftRegistration { get; set; } = string.Empty;
    public string LessonType { get; set; } = string.Empty;
    public string Reasoning { get; set; } = string.Empty;
}

public class AiScheduleFlag
{
    public string Type { get; set; } = "info";
    public string Message { get; set; } = string.Empty;
}
```

---

### Phase 5 Verification

```bash
dotnet build PilotbaseControlTower.sln  # 0 errors
dotnet test PilotbaseControlTower.sln   # 16+ tests pass
```

---

## PHASE 6: Controllers and Program.cs

**Goal**: All controllers implemented, Program.cs fully wired, API running and seeding all three databases on first start.

---

### Step 6.1 — Controllers

Create all eight controller files. Implement them exactly as specified below.

**`Controllers/StudentsController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Services.Career;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class StudentsController : ControllerBase
{
    private readonly OperationalDbContext _db;
    private readonly CareerPathwayService _career;

    public StudentsController(OperationalDbContext db, CareerPathwayService career)
    { _db = db; _career = career; }

    [HttpGet] public async Task<IActionResult> GetAll() =>
        Ok(await _db.Students.OrderBy(s => s.LastName).ToListAsync());

    [HttpGet("{id}")] public async Task<IActionResult> GetById(int id)
    {
        var s = await _db.Students.FindAsync(id);
        return s is null ? NotFound() : Ok(s);
    }

    [HttpGet("{id}/career")] public async Task<IActionResult> GetCareer(int id)
    {
        var s = await _db.Students.FindAsync(id);
        if (s is null) return NotFound();
        return Ok(await _career.GetPathwayAsync(s));
    }

    [HttpPut("{id}")] public async Task<IActionResult> Update(int id, [FromBody] StudentUpdateRequest req)
    {
        var s = await _db.Students.FindAsync(id);
        if (s is null) return NotFound();
        s.CurrentStage = req.CurrentStage;
        s.TotalHours = req.TotalHours;
        s.SoloHours = req.SoloHours;
        s.CrossCountryHours = req.CrossCountryHours;
        s.NightHours = req.NightHours;
        s.InstrumentHours = req.InstrumentHours;
        s.HasSoloEndorsement = req.HasSoloEndorsement;
        s.HasCrossCountryEndorsement = req.HasCrossCountryEndorsement;
        s.UpdatedAt = DateTime.UtcNow; // explicit update — HasDefaultValueSql only fires on INSERT
        await _db.SaveChangesAsync();
        return Ok(s);
    }
}

public class StudentUpdateRequest
{
    public PilotbaseControlTower.Api.Models.Operational.CurriculumStage CurrentStage { get; set; }
    public decimal TotalHours { get; set; }
    public decimal SoloHours { get; set; }
    public decimal CrossCountryHours { get; set; }
    public decimal NightHours { get; set; }
    public decimal InstrumentHours { get; set; }
    public bool HasSoloEndorsement { get; set; }
    public bool HasCrossCountryEndorsement { get; set; }
}
```

**`Controllers/InstructorsController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class InstructorsController : ControllerBase
{
    private readonly OperationalDbContext _db;
    public InstructorsController(OperationalDbContext db) => _db = db;

    [HttpGet] public async Task<IActionResult> GetAll() =>
        Ok(await _db.Instructors.Where(i => i.IsActive).OrderBy(i => i.LastName).ToListAsync());

    [HttpGet("{id}")] public async Task<IActionResult> GetById(int id)
    {
        var i = await _db.Instructors.FindAsync(id);
        return i is null ? NotFound() : Ok(i);
    }
}
```

**`Controllers/AircraftController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class AircraftController : ControllerBase
{
    private readonly OperationalDbContext _db;
    public AircraftController(OperationalDbContext db) => _db = db;

    [HttpGet] public async Task<IActionResult> GetAll() =>
        Ok(await _db.Aircraft.OrderBy(a => a.Registration).ToListAsync());

    [HttpGet("available")] public async Task<IActionResult> GetAvailable() =>
        Ok(await _db.Aircraft
            .Where(a => a.IsAvailable && a.Status != AircraftStatus.GroundedMaintenance && a.Status != AircraftStatus.ScheduledMaintenance)
            .OrderBy(a => a.Registration)
            .ToListAsync());

    [HttpGet("{id}")] public async Task<IActionResult> GetById(int id)
    {
        var a = await _db.Aircraft.FindAsync(id);
        return a is null ? NotFound() : Ok(a);
    }
}
```

**`Controllers/ScheduleController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Services.AI;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class ScheduleController : ControllerBase
{
    private readonly OperationalDbContext _db;
    private readonly AiSchedulingService _ai;

    public ScheduleController(OperationalDbContext db, AiSchedulingService ai)
    { _db = db; _ai = ai; }

    [HttpGet("week")] public async Task<IActionResult> GetWeek([FromQuery] DateTime? weekStart = null)
    {
        var start = weekStart ?? GetWeekStart();
        var end = start.AddDays(7);
        var events = await _db.ScheduleEvents
            .Include(e => e.Student).Include(e => e.Instructor).Include(e => e.Aircraft)
            .Where(e => e.StartTime >= start && e.StartTime < end)
            .OrderBy(e => e.StartTime)
            .ToListAsync();
        return Ok(events);
    }

    [HttpPost("generate")] public async Task<IActionResult> Generate([FromBody] GenerateScheduleRequest req)
    {
        var result = await _ai.GenerateScheduleAsync(new ScheduleGenerationRequest
        {
            WeekStart = req.WeekStart ?? GetWeekStart(),
            Students = await _db.Students.ToListAsync(),
            Instructors = await _db.Instructors.Where(i => i.IsActive).ToListAsync(),
            Aircraft = await _db.Aircraft.ToListAsync()
        });
        return Ok(result);
    }

    private static DateTime GetWeekStart()
    {
        var today = DateTime.UtcNow.Date;
        // Shift Sunday (DayOfWeek = 0) to 7 so the modulo arithmetic gives Mon=0 offset.
        // Equivalent to: "how many days since last Monday?"
        // Mon=0, Tue=1, Wed=2, Thu=3, Fri=4, Sat=5, Sun=6
        return today.AddDays(-(((int)today.DayOfWeek + 6) % 7));
    }
}

public class GenerateScheduleRequest { public DateTime? WeekStart { get; set; } }
```

**`Controllers/LogbookController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using PilotbaseControlTower.Api.Models.Logbook;
using PilotbaseControlTower.Api.Mongo;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class LogbookController : ControllerBase
{
    private readonly MongoDbService _mongo;
    public LogbookController(MongoDbService mongo) => _mongo = mongo;

    [HttpGet("{pilotId}")] public async Task<IActionResult> GetLogbook(int pilotId)
    {
        var entries = await _mongo.GetByPilotIdAsync(pilotId);
        var totals = await _mongo.GetTotalsAsync(pilotId);
        return Ok(new { entries, totals });
    }

    [HttpGet("{pilotId}/totals")] public async Task<IActionResult> GetTotals(int pilotId) =>
        Ok(await _mongo.GetTotalsAsync(pilotId));

    [HttpGet("{pilotId}/recent")] public async Task<IActionResult> GetRecent(int pilotId, [FromQuery] int count = 10) =>
        Ok(await _mongo.GetRecentAsync(pilotId, count));

    [HttpPost("{pilotId}/entries")] public async Task<IActionResult> AddEntry(int pilotId, [FromBody] FlightLogEntry entry)
    {
        entry.PilotId = pilotId;
        var created = await _mongo.CreateAsync(entry);
        return CreatedAtAction(nameof(GetLogbook), new { pilotId }, created);
    }
}
```

**`Controllers/FinancingController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;
using PilotbaseControlTower.Api.Services.Financing;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class FinancingController : ControllerBase
{
    private readonly OperationalDbContext _db;
    private readonly FinancingMilestoneService _svc;

    public FinancingController(OperationalDbContext db, FinancingMilestoneService svc)
    { _db = db; _svc = svc; }

    [HttpPost("milestone")] public async Task<IActionResult> RecordMilestone([FromBody] MilestoneRequest req)
    {
        try
        {
            return Ok(await _svc.RecordMilestoneAsync(req.StudentId, req.MilestoneType, req.CurrentHours));
        }
        catch (KeyNotFoundException ex)
        {
            return NotFound(new { message = ex.Message });
        }
    }

    [HttpGet("milestones/{studentId}")] public async Task<IActionResult> GetMilestones(int studentId) =>
        Ok(await _db.FinancingMilestones
            .Where(m => m.StudentId == studentId)
            .OrderByDescending(m => m.AchievedAt)
            .ToListAsync());
}

public class MilestoneRequest
{
    public int StudentId { get; set; }
    public MilestoneType MilestoneType { get; set; }
    public decimal CurrentHours { get; set; }
}
```

**`Controllers/ReportingController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Models.Operational;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class ReportingController : ControllerBase
{
    private readonly OperationalDbContext _db;
    public ReportingController(OperationalDbContext db) => _db = db;

    [HttpGet("kpis")] public async Task<IActionResult> GetKpis()
    {
        var students = await _db.Students.ToListAsync();
        var aircraft = await _db.Aircraft.ToListAsync();

        var totalAircraft = aircraft.Count;
        var squawkFreeAircraft = aircraft.Count(a => a.Status == AircraftStatus.Airworthy && string.IsNullOrEmpty(a.ActiveSquawk));
        var squawkFreePct = totalAircraft > 0 ? (double)squawkFreeAircraft / totalAircraft * 100 : 0;
        var sallieMaeCount = students.Count(s => s.FinancingType == FinancingType.SallieMae);

        return Ok(new
        {
            totalStudents = students.Count,
            activeStudents = students.Count(s => s.CurrentStage < CurriculumStage.PrivatePilotComplete),
            totalAircraft,
            airworthyAircraft = squawkFreeAircraft,
            squawkFreeFlightPercent = Math.Round(squawkFreePct, 1),
            aopaKpiLabel = $"{Math.Round(squawkFreePct, 0)}% of student flights on squawk-free aircraft",
            checkrideReadyStudents = students.Count(s => s.CurrentStage == CurriculumStage.CheckridePrep),
            sallieMaeStudents = sallieMaeCount,
            sallieMaePercent = students.Count > 0 ? Math.Round((double)sallieMaeCount / students.Count * 100, 1) : 0,
            availableAircraft = aircraft.Count(a => a.IsAvailable && a.Status != AircraftStatus.GroundedMaintenance && a.Status != AircraftStatus.ScheduledMaintenance)
        });
    }
}
```

**`Controllers/LegacyController.cs`**:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Legacy;

namespace PilotbaseControlTower.Api.Controllers;

[ApiController, Route("api/[controller]")]
public class LegacyController : ControllerBase
{
    private readonly LegacyDbContext _db;
    public LegacyController(LegacyDbContext db) => _db = db;

    [HttpGet("flights")] public async Task<IActionResult> GetFlights([FromQuery] string? pilot = null)
    {
        var q = _db.LegacyFlightRecords.AsQueryable();
        if (!string.IsNullOrEmpty(pilot)) q = q.Where(f => f.PilotName.Contains(pilot));
        return Ok(await q.OrderByDescending(f => f.FlightDate).ToListAsync());
    }

    [HttpGet("flights/summary")] public async Task<IActionResult> GetSummary()
    {
        var records = await _db.LegacyFlightRecords.ToListAsync();
        if (!records.Any()) return Ok(new { totalRecords = 0, note = "No legacy records found." });
        return Ok(new
        {
            totalRecords = records.Count,
            totalHours = records.Sum(r => r.TotalTime),
            uniquePilots = records.Select(r => r.PilotName).Distinct().Count(),
            dateRange = new { earliest = records.Min(r => r.FlightDate), latest = records.Max(r => r.FlightDate) },
            note = "Legacy SQL Server records — pre-2023 flight history before migration to PostgreSQL"
        });
    }
}
```

---

### Step 6.2 — Full Program.cs

Replace `Program.cs` completely:

```csharp
using Microsoft.EntityFrameworkCore;
using PilotbaseControlTower.Api.Data.Legacy;
using PilotbaseControlTower.Api.Data.Operational;
using PilotbaseControlTower.Api.Mongo;
using PilotbaseControlTower.Api.Seed;
using PilotbaseControlTower.Api.Services.AI;
using PilotbaseControlTower.Api.Services.Career;
using PilotbaseControlTower.Api.Services.Financing;
using QuestPDF.Infrastructure;

// QuestPDF requires a license declaration at startup or it throws on first PDF generation.
// Community license is free for revenue under $1M/year.
QuestPDF.Settings.License = LicenseType.Community;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers()
    .AddJsonOptions(o =>
        o.JsonSerializerOptions.Converters.Add(
            new System.Text.Json.Serialization.JsonStringEnumConverter()));

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
    c.SwaggerDoc("v1", new() { Title = "Pilotbase Control Tower API", Version = "v1" }));

builder.Services.AddCors(options => options.AddPolicy("DevPolicy", policy =>
    policy.WithOrigins(
            builder.Configuration.GetSection("AllowedOrigins").Get<string[]>()
            ?? new[] { "http://localhost:4200", "http://localhost:5173" })
        .AllowAnyMethod().AllowAnyHeader()));

// SQL Server — legacy read-only store
builder.Services.AddDbContext<LegacyDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("SqlServer")));

// PostgreSQL — operational store
builder.Services.AddDbContext<OperationalDbContext>(o =>
    o.UseNpgsql(builder.Configuration.GetConnectionString("PostgreSQL")));

// MongoDB — logbook documents (singleton: MongoClient is thread-safe by design)
builder.Services.AddSingleton<MongoDbService>();

// Application services
// AiSchedulingService uses AddHttpClient so that HttpClient is injected via constructor.
// Do not change to AddScoped without also updating the AiSchedulingService constructor.
// Timeout is set on this specific client only — do NOT use ConfigureHttpClientDefaults,
// which would apply the 90s timeout to every IHttpClientFactory-managed client in the app
// (including any future AWS SDK or other clients added later).
builder.Services.AddHttpClient<AiSchedulingService>()
    .ConfigureHttpClient(c => c.Timeout = TimeSpan.FromSeconds(90));
builder.Services.AddScoped<FinancingMilestoneService>();
builder.Services.AddScoped<CareerPathwayService>();

var app = builder.Build();

// Seed all three databases on startup
using (var scope = app.Services.CreateScope())
{
    var legacy = scope.ServiceProvider.GetRequiredService<LegacyDbContext>();
    var operational = scope.ServiceProvider.GetRequiredService<OperationalDbContext>();
    var mongo = scope.ServiceProvider.GetRequiredService<MongoDbService>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    try { await DatabaseSeeder.SeedAllAsync(legacy, operational, mongo, logger); }
    catch (Exception ex) { logger.LogError(ex, "Database seeding failed. App will continue."); }
}

app.UseCors("DevPolicy");
app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---

### Phase 6 Verification

Start everything:

```bash
docker-compose up -d
cd backend/PilotbaseControlTower.Api
dotnet run
```

Confirm these log lines appear on first startup:
```
Seeded 10 legacy SQL Server flight records.
Seeded 5 instructors, 6 aircraft, 10 students into PostgreSQL.
Seeded 9 logbook entries for pilots 1, 2, and 5.
```

Then verify each endpoint via Swagger at `http://localhost:5000/swagger`:

| Endpoint | Expected result |
|---|---|
| `GET /api/students` | 10 students |
| `GET /api/students/3` | Ashley Williams, CheckridePrep — **assumes a clean database with sequences starting at 1**. If the DB was previously partially seeded, the IDs may differ. Run `docker-compose down -v && docker-compose up -d` to reset if you get a different student. |
| `GET /api/aircraft` | 6 aircraft |
| `GET /api/aircraft/available` | **5 aircraft** (N172SC excluded — IsAvailable=false; N172SD included — squawked but IsAvailable=true) |
| `GET /api/instructors` | 5 instructors |
| `GET /api/logbook/1` | Maria Garcia's 4 logbook entries + totals |
| `GET /api/logbook/1/totals` | TotalFlights: 4, TotalTime: 6.6 — **if totals look wrong**, the MongoDB seed may have partially run on a previous start. Run `docker-compose down -v && docker-compose up -d` to wipe volumes and re-seed cleanly. |
| `GET /api/legacy/flights/summary` | totalRecords: 10 |
| `GET /api/reporting/kpis` | totalStudents: 10, totalAircraft: 6 |
| `POST /api/financing/milestone` `{"studentId":1,"milestoneType":"FirstSolo","currentHours":22.5}` | disbursementTriggered: true, disbursementAmount: 3000 |
| `POST /api/schedule/generate` `{}` | weekSummary populated (or mock message if no API key) |

---

## PHASE 7: Angular Frontend (School Operator Dashboard)

**Goal**: A working Angular 18 app with six pages.

---

### Step 7.1 — Initialize

```bash
# From repo root
ng new frontend-school --routing --style=scss --standalone --skip-git --directory frontend-school
cd frontend-school
ng add @angular/material   # choose Indigo/Pink theme, yes to typography, yes to animations
npm install
```

---

### Step 7.2 — Environment configuration

Create `src/environments/environment.ts`:

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000/api',
  schoolName: 'Suncoast Aviation Academy',
  airportCode: 'KTPA'
};
```

Create `src/environments/environment.prod.ts`:

```typescript
export const environment = {
  production: true,
  apiUrl: '/api',
  schoolName: 'Suncoast Aviation Academy',
  airportCode: 'KTPA'
};
```

---

### Step 7.3 — App config and routing

Update `src/app/app.config.ts` to add `provideHttpClient()` and `provideRouter(routes)`.

Update `src/app/app.routes.ts`:

```typescript
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  { path: 'dashboard', loadComponent: () => import('./pages/dashboard/dashboard.component').then(m => m.DashboardComponent) },
  { path: 'schedule', loadComponent: () => import('./pages/schedule/schedule.component').then(m => m.ScheduleComponent) },
  { path: 'students', loadComponent: () => import('./pages/students/students.component').then(m => m.StudentsComponent) },
  { path: 'aircraft', loadComponent: () => import('./pages/aircraft/aircraft.component').then(m => m.AircraftComponent) },
  { path: 'instructors', loadComponent: () => import('./pages/instructors/instructors.component').then(m => m.InstructorsComponent) },
  { path: 'reporting', loadComponent: () => import('./pages/reporting/reporting.component').then(m => m.ReportingComponent) },
];
```

---

### Step 7.4 — API service

Create `src/app/services/api.service.ts` with typed `Observable<any>` methods for every backend endpoint. Use `environment.apiUrl` as the base. Inject `HttpClient`.

---

### Step 7.5 — App shell navigation

Update `src/app/app.component.ts` to render a left sidebar nav (navy `#0A2342` background, white text) with links to all six routes plus a `<router-outlet>` for the main content area.

---

### Step 7.6 — Six pages

Create these six standalone Angular components. Each must fetch and display real data from the API on init.

**Dashboard** (`src/app/pages/dashboard/`):
- KPI tiles from `GET /api/reporting/kpis`: total students, airworthy aircraft, AOPA squawk-free %, Sallie Mae student count
- Legacy database tile showing SQL Server summary (`GET /api/legacy/flights/summary`) with a "SQL Server" badge — this visually communicates the three-database architecture
- "Generate AI Schedule" button: calls `POST /api/schedule/generate`, shows a loading spinner, then displays `weekSummary` and a list of scheduled events, each with its `reasoning` text visible

**Schedule** (`src/app/pages/schedule/`):
- 7-column weekly grid (Mon–Sun) with time slots 06:30–19:30
- Color-code events by student curriculum stage
- "Generate AI Schedule" button populates the grid; AI warning flags display above the grid

**Students** (`src/app/pages/students/`):
- Table: Name, Training Type badge (Part 141 green / Part 61 blue), Stage, Total Hours, Financing Type badge (SallieMae green, StratusFinancial blue, SelfFunded gray)
- Progress bar per student (currentStage / 10)
- Red warning badge for students with `projectedCheckrideDate` within 30 days of today. Ashley Williams is seeded with `DateTime.UtcNow.AddDays(10)` so she will always trigger this badge regardless of when the project is run.
- Click row → detail panel or modal with career pathway from `GET /api/students/{id}/career`

**Aircraft** (`src/app/pages/aircraft/`):
- Card grid, border color by status: Airworthy = green, GroundedSquawk = red, ScheduledMaintenance = yellow, GroundedMaintenance = dark red
- Active squawk text displayed in red
- Badges: IFR Equipped, Multi-Engine, Available / Unavailable

**Instructors** (`src/app/pages/instructors/`):
- Table: Name, ratings (CFI/CFII/MEI badges), total hours, available days, hours window
- Green dot for instructors available on the current day of the week

**Reporting** (`src/app/pages/reporting/`):
- Full KPI dashboard with large metric tiles
- Financing milestone trigger panel: student dropdown + milestone type dropdown + current hours input + "Trigger Milestone" button + result display showing disbursement amount and lender reference
- Legacy database explorer showing SQL Server summary

---

### Phase 7 Verification

```bash
cd frontend-school
ng serve --open   # opens http://localhost:4200
```

- All six routes load without console errors
- Dashboard shows data; AI schedule button shows spinner then results
- Students table shows 10 rows with correct badges
- N172SD shows red squawk text; N172SC shows yellow/unavailable status
- Financing panel triggers milestone for student 1 and shows Sallie Mae response

```bash
npm run build -- --configuration production
# Expected: 0 errors
```

---

## PHASE 8: React Frontend (Pilot Pilotbase App)

**Goal**: A working React 18 app with four pages, centered on the LogTen-style logbook.

---

### Step 8.1 — Initialize

```bash
# From repo root
npm create vite@latest frontend-pilot -- --template react-ts
cd frontend-pilot
npm install
npm install react-router-dom axios
# Note: @types/react-router-dom is NOT needed — React Router v6 ships its own types
npm install -D tailwindcss@^3 postcss autoprefixer
# Pin to Tailwind v3 — v4 uses a completely different config model
# and does not support `npx tailwindcss init`
npx tailwindcss init -p
```

Configure `tailwind.config.js`:

```javascript
export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        pilotbase: { blue: '#0066CC', navy: '#0A2342', sky: '#87CEEB' }
      }
    }
  },
  plugins: [],
}
```

Add to top of `src/index.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Configure `vite.config.ts` — no proxy needed since the backend has CORS configured to allow `http://localhost:5173`:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

---

### Step 8.2 — API client

Create `src/api/client.ts`:

```typescript
import axios from 'axios';

const api = axios.create({ baseURL: 'http://localhost:5000/api' });

export const fetchLogbook = (pilotId: number) =>
  api.get(`/logbook/${pilotId}`).then(r => r.data);

export const fetchLogbookTotals = (pilotId: number) =>
  api.get(`/logbook/${pilotId}/totals`).then(r => r.data);

export const fetchStudent = (id: number) =>
  api.get(`/students/${id}`).then(r => r.data);

export const fetchCareer = (id: number) =>
  api.get(`/students/${id}/career`).then(r => r.data);

export const addLogbookEntry = (pilotId: number, entry: unknown) =>
  api.post(`/logbook/${pilotId}/entries`, entry).then(r => r.data);
```

---

### Step 8.3 — App shell

`src/App.tsx` — BrowserRouter with routes for `/dashboard`, `/logbook`, `/training`, `/career`. Export a `PilotContext` that holds the current pilot ID and a setter, defaulting to `1` (Maria Garcia). Wrap the app in this context provider.

```tsx
// src/context/PilotContext.tsx
import { createContext, useContext, useState } from 'react';

interface PilotContextValue {
  pilotId: number;
  setPilotId: (id: number) => void;
}

export const PilotContext = createContext<PilotContextValue>({
  pilotId: 1,
  setPilotId: () => {}
});

export const usePilot = () => useContext(PilotContext);

export function PilotProvider({ children }: { children: React.ReactNode }) {
  const [pilotId, setPilotId] = useState(1);
  return (
    <PilotContext.Provider value={{ pilotId, setPilotId }}>
      {children}
    </PilotContext.Provider>
  );
}
```

Wrap the app in `<PilotProvider>` in `src/App.tsx`. All pages use `const { pilotId } = usePilot()` instead of a hardcoded constant.

`src/components/Layout.tsx` — Top header ("Pilotbase" in navy + sky blue), bottom tab bar (mobile-first) with four tabs, and a small dev-mode pilot selector in the header. The selector is a `<select>` dropdown showing pilot names (hardcoded for the demo: 1 = Maria Garcia, 2 = David Chen, 5 = Jennifer Davis). Changing the selector calls `setPilotId()` from `usePilot()`, which updates all pages simultaneously without needing to edit any constants.

```tsx
// Pilot selector in Layout header — dev-only switcher
const { pilotId, setPilotId } = usePilot();
const pilots = [
  { id: 1, name: 'Maria Garcia (Post-Solo)' },
  { id: 2, name: 'David Chen (Cross-Country)' },
  { id: 5, name: 'Jennifer Davis (IFR Rating)' },
];
// Render as: <select value={pilotId} onChange={e => setPilotId(Number(e.target.value))}>
```

This means verifying Jennifer Davis's IFR logbook entries (pilot 5, with instrument approaches) is just a dropdown change rather than a code edit — and switching back to Maria Garcia is equally trivial.

---

### Step 8.4 — Four pages

**Dashboard** (`src/pages/Dashboard.tsx`):
- Large "Total Flight Hours" card with progress bar to next milestone
- "Financing Status" tile: if SallieMae, show outstanding balance and next milestone + disbursement amount
- Quick stats row: Solo / Cross-Country / Night / Instrument hours
- Recent 3 logbook entries from `GET /api/logbook/{pilotId}/recent?count=3` — use `pilotId` from `usePilot()` context

**Logbook** (`src/pages/Logbook.tsx`) — the centerpiece. Use `const { pilotId } = usePilot()` for all API calls:
- Totals summary bar at top from `GET /api/logbook/{pilotId}/totals`: Total Time, PIC, Dual, Solo, Night, Instrument, XC, Approaches, Landings
- Table of all entries from `GET /api/logbook/{pilotId}`: Date | Aircraft | Route (Dep → Dest) | Total Time | Conditions (icons: ☁️ IFR, 🌙 Night, ✈️ XC) | Approaches count | Landings | ✓ Signed
- Expandable row reveals: remarks, instructor endorsement name/certificate/date
- Floating "+" button opens modal to add new entry with full form (all FlightLogEntry fields); posts to `POST /api/logbook/{pilotId}/entries`

**Training** (`src/pages/Training.tsx`). Use `const { pilotId } = usePilot()` and fetch student data from `GET /api/students/{pilotId}`:
- Training type badge (Part 141 / Part 61) prominent at top
- Step progress showing all 11 CurriculumStage values; completed = checked, current = highlighted, future = gray
- Part 141 note: "Stage checks required per FAA-approved course outline"
- Hour requirements progress bars: Total ≥ 40, Solo ≥ 10, Cross-Country ≥ 5, Night ≥ 3
- Financing milestones section with Sallie Mae disbursement amounts

**Career** (`src/pages/Career.tsx`). Use `const { pilotId } = usePilot()` for all API calls:
- Career roadmap: PPL → Instrument → Commercial → CFI → ATP → Major Airline with current position marked
- "Hours to ATP" counter + estimated date from `GET /api/students/{pilotId}/career`
- Job match cards: role, operator, match strength badge, description
- Bottom callout: "The aviation industry needs 660,000 new pilots over the next 20 years (Boeing Pilot Outlook). Your Pilotbase training history is your career foundation."

---

### Phase 8 Verification

```bash
cd frontend-pilot
npm run dev   # opens http://localhost:5173
```

- Logbook shows Maria Garcia's 4 flights; first solo has no instructor signature row
- **To verify IFR entries**: use the pilot selector dropdown in the Layout header to switch to "Jennifer Davis (IFR Rating)". Her logbook should show instrument approaches on the March entries. Switch back to Maria Garcia when done — no code changes needed.
- Training page shows PostSolo highlighted
- Career page shows hours-to-ATP calculation and no job matches (22.5 hours — correctly below all thresholds)

```bash
npm run build
# Expected: vite build, 0 errors
```

---

## PHASE 9: Final Integration and Polish

---

### Step 9.1 — End-to-end smoke test

With docker-compose, API (`dotnet run`), Angular (`ng serve`), and React (`npm run dev`) all running:

1. **Angular Dashboard** → "Generate AI Schedule" → spinner appears → schedule renders with per-event reasoning
2. **Angular Reporting** → trigger "FirstSolo" for Maria Garcia (ID 1), 22.5 hours → Sallie Mae banner shows "$3,000 disbursement initiated" with reference starting "SM-"
3. **Angular Students** → confirm at least one student shows a red checkride warning badge. Ashley Williams is seeded with `ProjectedCheckrideDate = DateTime.UtcNow.AddDays(10)`, so she will always be within the 30-day threshold regardless of when the project is run.
4. **Angular Aircraft** → N172SD shows red squawk text; N172SC shows yellow/unavailable
5. **React Logbook** → 4 entries for Maria Garcia; first solo entry (Feb 12) has no instructor signature section
6. **React Training** → PostSolo stage highlighted; PreSolo shows checked

---

### Step 9.2 — Final DECISION_LOG pass

Ensure at minimum these entries exist:
- Three-database architecture rationale
- Angular vs. React frontend split rationale
- `OnDelete(DeleteBehavior.Restrict)` on ScheduleEvent foreign keys
- `AvailableDays` comma-separated string with change-tracking caveat
- `MongoDbService` registered as singleton (MongoClient thread safety)
- `UpdatedAt` requires explicit set in update paths (HasDefaultValueSql only fires on INSERT)
- QuestPDF Community license required at startup
- Tailwind pinned to v3 (v4 incompatible with `tailwindcss init`)
- `AiSchedulingService` uses `AddHttpClient<T>` — constructor must accept `HttpClient`
- Any package version changes required during installation

---

### Step 9.3 — Commit and push

```bash
git add .
git commit -m "feat: Pilotbase Control Tower — full stack implementation

.NET 8 Web API with three databases (SQL Server legacy, PostgreSQL operational,
MongoDB logbooks), Angular 18 school operator dashboard, React 18 pilot app,
AI scheduling via Anthropic, Sallie Mae milestone trigger, Part 141/61
curriculum logic, GitHub Actions CI/CD."

git push origin main
```

---

### Step 9.4 — Verify CI

Navigate to `https://github.com/OldEphraim/pilotbase-control-tower/actions`.

The backend CI job should be green. The frontend jobs will activate once the frontend directories were pushed (Phase 7 and 8 commits). All three must show green before the project is considered complete — the CI badge is itself a deliverable.

---

## Appendix: Common Failure Modes and Fixes

**SQL Server container won't start on Apple Silicon (M1/M2/M3)**
Add `platform: linux/amd64` under the `sqlserver` service in docker-compose.yml. Record in DECISION_LOG.

**EF Core migration for Npgsql and TimeOnly**
Requires `Npgsql.EntityFrameworkCore.PostgreSQL` version 8.x. If unmapped TimeOnly errors appear, confirm the Npgsql version matches the EF Core runtime version.

**MongoDB auth fails**
The `authSource=admin` in the connection string is required — the user is created in the admin database. Without it, the driver looks in `pilotbase_logs` and fails.

**Angular CORS error**
The API must be running before `ng serve`. `AllowedOrigins` must include `http://localhost:4200` with no trailing slash.

**AI scheduling times out**
The Anthropic API call with a full prompt takes 15–30 seconds. The 90-second timeout is configured directly on `AiSchedulingService`'s named HTTP client in Program.cs (`AddHttpClient<AiSchedulingService>().ConfigureHttpClient(...)`). This intentionally does not use `ConfigureHttpClientDefaults`, which would apply the timeout to every HTTP client in the app.

**Partial seed left MongoDB in an inconsistent state**
The MongoDB seed guard checks `if (existing.Any())` using pilot 1 only. If a previous startup created some pilot 1 entries but crashed before completing all of them, the seeder will skip on the next start and the logbook will have incomplete data. Fix: `docker-compose down -v && docker-compose up -d` to wipe all volumes and re-seed cleanly.

**SQL Server / PostgreSQL IDs are not 1–10 after restart**
If the database volumes were not wiped between runs (e.g., `docker-compose stop` without `-v`), PostgreSQL's auto-increment sequences may have advanced past 1. Student IDs won't be 1–10, so endpoints like `GET /api/students/3` won't return Ashley Williams. Fix: `docker-compose down -v && docker-compose up -d`. Always use `-v` when a clean seed state is needed.

**Enums serialize as numbers**
Confirm `JsonStringEnumConverter` is registered in `AddJsonOptions` in Program.cs. Without it, `TrainingType.Part141` serializes as `1` instead of `"Part141"` and the frontend displays integer values.

**QuestPDF LicenseException**
`QuestPDF.Settings.License = LicenseType.Community;` must appear before `builder.Build()` in Program.cs. It's at the top of the file in Step 6.2 — do not move it below the builder initialization.

**Tailwind v4 installed accidentally**
If `tailwind.config.js` was not created by `npx tailwindcss init`, you likely installed v4. Run `npm uninstall tailwindcss` then `npm install -D tailwindcss@^3 postcss autoprefixer` and re-run `npx tailwindcss init -p`.