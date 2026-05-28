# Practice Management System - Development Plan

> Project: Practice Management System (Candidate #102)
> Created: 2026-05-25
> Status: Specification Complete - Ready for Development

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB (Data Model Suggestion 3)

**Rationale:** The Hybrid Relational + JSONB model (Suggestion 3) is selected as the primary data model for the following reasons:

1. **Multi-specialty flexibility.** The PMS must serve behavioral health, medical, and allied health practices across US and international markets. JSONB columns absorb specialty-specific and jurisdiction-specific field variation (intake forms, note structures, payer rules, provincial billing codes) without schema migrations. This is the same pattern used by Jane App and Cliniko, two successful multi-specialty PMS products.

2. **Faster MVP velocity.** 20 tables versus 37 (normalized) or 19 (event-sourced) keeps the initial schema approachable. New practice types can be onboarded by adding JSON Schema configurations rather than database migrations.

3. **Pragmatic audit trail.** A partitioned `audit_log` table provides HIPAA-compliant audit logging without the full complexity of event sourcing. If event sourcing is needed for revenue cycle analytics later, it can be added as a secondary projection layer (as described in Suggestion 2) on top of the operational tables.

4. **FHIR alignment.** Core entities (Patient, Appointment, Encounter, Claim) map to FHIR R4 resources with relational columns for FHIR-mandated fields and JSONB for FHIR extension data, which is the natural mapping for FHIR's "80/20 rule" (80% of use cases covered by base resources, 20% by extensions).

5. **Graph deferral.** The graph-relational model (Suggestion 4) adds referral network and care coordination value, but these are backlog features. The `graph_node` and `graph_edge` tables can be added in a later phase without modifying existing operational tables.

**Key database features used:**
- Row-Level Security (RLS) for multi-tenant isolation via `practice_id`
- Table partitioning on `audit_log` by month (HIPAA requires 6-year retention)
- GIN indexes on JSONB columns for containment queries
- Full-text search indexes on reference code descriptions
- UUID primary keys for distributed ID generation and enumeration prevention

### Backend: Node.js (TypeScript) with NestJS

**Rationale:**
- NestJS provides a modular, testable architecture with built-in support for OpenAPI documentation, guards/interceptors for RBAC, and dependency injection.
- TypeScript provides type safety across the API layer, JSON Schema validation (with `ajv`), and FHIR resource typing (via `@types/fhir`).
- Node.js ecosystem has mature libraries for X12 EDI parsing (`x12-parser`), HL7 v2 messaging, and FHIR client/server operations (`@medplum/fhirtypes`).
- Async/event-driven architecture is well-suited for AI pipeline orchestration (no-show prediction, claim scrubbing, ambient documentation).

### Frontend: Next.js 15 (React) with Tailwind CSS

**Rationale:**
- Server-side rendering for HIPAA-compliant session management (no PHI in client-side state).
- App Router architecture for role-based layouts (admin, billing, clinical, front-desk).
- React ecosystem has scheduling/calendar components (e.g., `@schedule-x/react`) critical for the core scheduling UI.
- Tailwind CSS enables rapid UI development with consistent design tokens.

### AI/ML: Python microservices with FastAPI

**Rationale:**
- Python is the standard for ML model development (scikit-learn, XGBoost for no-show prediction; transformer models for ambient documentation).
- FastAPI provides OpenAPI-documented endpoints that the NestJS backend calls as internal services.
- Separation of AI services from the main backend allows independent scaling and model deployment.

### Message Queue: Redis Streams (initial) / NATS (scale)

**Rationale:**
- Async processing for: appointment reminders, eligibility verification batch jobs, claim submission pipelines, AI prediction requests, ERA posting.
- Redis Streams for MVP simplicity; NATS for production multi-node deployment.

### Infrastructure: Docker Compose (dev) / Kubernetes (prod)

**Rationale:**
- Self-hosted deployment is a stated differentiator versus proprietary SaaS incumbents.
- Docker Compose for local development and small practice self-hosting.
- Kubernetes with Helm charts for cloud deployment (AWS/GCP/Azure).
- HIPAA compliance requires encrypted storage at rest (EBS encryption / managed disk encryption), TLS 1.2+ in transit, and VPC network isolation.

### External Integrations

| Integration | Technology | Phase |
|-------------|-----------|-------|
| Clearinghouse (eligibility, claims) | X12 EDI via Availity/Office Ally REST API | Phase 3-4 |
| E-Prescribing | Surescripts network (NCPDP SCRIPT) | Backlog |
| FHIR R4 API | HAPI FHIR or custom NestJS FHIR server | Phase 6 |
| SMS/Email Reminders | Twilio (SMS) / SendGrid (email) | Phase 2 |
| Payment Processing | Stripe | Phase 4 |
| Telehealth | Daily.co or Twilio Video (WebRTC) | Phase 7 |

---

## Project Structure

```
practice-management-system/
|-- apps/
|   |-- api/                          # NestJS backend
|   |   |-- src/
|   |   |   |-- modules/
|   |   |   |   |-- auth/             # Authentication, OAuth 2.0, SMART on FHIR
|   |   |   |   |-- practice/         # Practice/tenant management
|   |   |   |   |-- patient/          # Patient demographics, identifiers
|   |   |   |   |-- scheduling/       # Appointments, slots, waitlist
|   |   |   |   |-- clinical/         # Encounters, notes, diagnoses
|   |   |   |   |-- billing/          # Charges, claims, remittances
|   |   |   |   |-- coverage/         # Insurance, eligibility, prior auth
|   |   |   |   |-- messaging/        # Patient communication
|   |   |   |   |-- forms/            # Intake forms, templates
|   |   |   |   |-- audit/            # Audit logging
|   |   |   |   |-- ai/               # AI prediction orchestration
|   |   |   |   |-- fhir/             # FHIR R4 API endpoints
|   |   |   |-- common/
|   |   |   |   |-- guards/           # RBAC guards, tenant guards
|   |   |   |   |-- interceptors/     # Audit logging interceptor
|   |   |   |   |-- filters/          # Error handling
|   |   |   |   |-- pipes/            # Validation pipes (JSON Schema)
|   |   |   |   |-- decorators/       # Custom decorators (@CurrentPractice, @Roles)
|   |   |   |-- database/
|   |   |   |   |-- migrations/       # Knex/TypeORM migrations
|   |   |   |   |-- seeds/            # Reference data (ICD-10, CPT, POS codes)
|   |   |   |   |-- rls/              # Row-Level Security policies
|   |   |-- test/
|   |
|   |-- web/                          # Next.js frontend
|   |   |-- src/
|   |   |   |-- app/
|   |   |   |   |-- (auth)/           # Login, MFA, password reset
|   |   |   |   |-- (dashboard)/      # Role-based dashboards
|   |   |   |   |-- scheduling/       # Calendar, booking, waitlist
|   |   |   |   |-- patients/         # Patient list, demographics, chart
|   |   |   |   |-- billing/          # Claims, charges, remittances
|   |   |   |   |-- reports/          # Financial, operational analytics
|   |   |   |   |-- settings/         # Practice, user, payer configuration
|   |   |   |-- components/
|   |   |   |-- hooks/
|   |   |   |-- lib/
|   |
|   |-- patient-portal/               # Separate Next.js app for patient-facing portal
|   |
|-- services/
|   |-- ai-prediction/                # Python FastAPI - no-show, denial prediction
|   |-- claim-scrubber/               # Python FastAPI - AI claim scrubbing
|   |-- ambient-doc/                  # Python FastAPI - ambient documentation
|   |-- edi-gateway/                  # Node.js - X12 EDI transaction processing
|   |-- notification/                 # Node.js - SMS/email delivery
|
|-- packages/
|   |-- shared-types/                 # TypeScript type definitions shared across apps
|   |-- fhir-types/                   # FHIR R4 resource type definitions
|   |-- edi-parser/                   # X12 EDI parsing/generation library
|   |-- json-schema-forms/            # JSON Schema form rendering library
|
|-- infrastructure/
|   |-- docker/                       # Docker Compose for local dev
|   |-- k8s/                          # Kubernetes manifests / Helm charts
|   |-- terraform/                    # Cloud infrastructure provisioning
|
|-- docs/
|   |-- api/                          # OpenAPI specs (auto-generated)
|   |-- architecture/                 # Architecture Decision Records (ADRs)
|   |-- compliance/                   # HIPAA compliance documentation
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Scheduling -------> Phase 3: Patient Management
    |                              |
    v                              v
Phase 4: Billing & Claims <------ |
    |                              |
    v                              v
Phase 5: Clinical Documentation   |
    |                              |
    v                              v
Phase 6: FHIR & Interoperability  |
    |                              |
    v                              v
Phase 7: AI Features (No-Show, Claim Scrub)
    |
    v
Phase 8: Patient Portal & Engagement
    |
    v
Phase 9: Revenue Cycle Analytics
    |
    v
Phase 10: Advanced AI (Ambient Docs, Conversational Intake)
    |
    v
Phase 11: Compliance Hardening & Certification
    |
    v
Phase 12: Multi-Jurisdiction & Internationalization
```

**Legend:**
- Arrows indicate "must complete before"
- Phases 2 and 3 can be developed in parallel after Phase 1
- Phases 5 and 6 can overlap
- Phases 7-10 each depend on the preceding phases but represent incremental feature additions

---

## Phase 1: Foundation & Infrastructure

**Goal:** Establish the database schema, authentication, multi-tenancy, and deployment infrastructure. All subsequent phases build on this foundation.

**Duration estimate:** 4-5 weeks

### Task 1.1: Database Schema & Migrations

**What:** Create the PostgreSQL database schema implementing the Hybrid Relational + JSONB data model (Suggestion 3). Set up migration tooling, seed reference data, and configure Row-Level Security policies.

**Design:**

```typescript
// database/migrations/001_create_practice_and_location.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('practice', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.string('name', 255).notNullable();
    table.string('npi', 10);
    table.string('tax_id', 20);
    table.string('specialty_type', 50);
    table.string('country_code', 2).defaultTo('US');
    table.string('timezone', 50).notNullable().defaultTo('America/New_York');
    table.string('status', 20).notNullable().defaultTo('active');
    table.jsonb('settings').notNullable().defaultTo('{}');
    table.jsonb('billing_address');
    table.timestamps(true, true);
  });

  await knex.schema.createTable('location', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.uuid('practice_id').notNullable().references('id').inTable('practice');
    table.string('name', 255).notNullable();
    table.jsonb('address').notNullable();
    table.string('phone', 20);
    table.string('timezone', 50);
    table.boolean('is_billing_location').notNullable().defaultTo(false);
    table.string('status', 20).notNullable().defaultTo('active');
    table.jsonb('metadata').notNullable().defaultTo('{}');
    table.timestamps(true, true);
    table.index('practice_id');
  });
}
```

```sql
-- database/rls/practice_isolation.sql
ALTER TABLE patient ENABLE ROW LEVEL SECURITY;
CREATE POLICY practice_isolation ON patient
  USING (practice_id = current_setting('app.current_practice_id')::uuid);

ALTER TABLE appointment ENABLE ROW LEVEL SECURITY;
CREATE POLICY practice_isolation ON appointment
  USING (practice_id = current_setting('app.current_practice_id')::uuid);

-- Repeat for all tables with practice_id
```

```typescript
// database/seeds/001_reference_codes.ts
// Seed ICD-10-CM, Place of Service codes
// CPT codes require AMA licence - seed structure only, load from licensed data file
export async function seed(knex: Knex): Promise<void> {
  await knex('ref_code').insert([
    { code_system: 'place_of_service', code: '11', display: 'Office', is_active: true },
    { code_system: 'place_of_service', code: '02', display: 'Telehealth - Patient Location', is_active: true },
    // ... 50+ POS codes
  ]);
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Migrations run cleanly on empty database | Integration | Run all migrations from scratch against a fresh PostgreSQL instance; verify all 20 tables created |
| Migrations are reversible | Integration | Run all up migrations, then all down migrations; verify database is empty |
| RLS blocks cross-tenant access | Integration | Insert records for practice A and B; set `app.current_practice_id` to A; verify SELECT on practice B records returns empty |
| RLS allows same-tenant access | Integration | Set `app.current_practice_id` to A; verify SELECT returns practice A records |
| Reference data seeds load | Integration | Run seed files; verify `ref_code` contains ICD-10 codes (at least 70,000 entries), POS codes, and placeholder CPT structure |
| UUID generation works | Unit | Insert 10,000 records; verify no UUID collisions |
| JSONB columns accept valid data | Integration | Insert practice with nested `settings` JSON; retrieve and verify structure preserved |
| JSONB columns reject non-object values | Integration | Attempt to insert `settings = 'not json'`; verify database rejects |
| Indexes exist and are used | Integration | Run `EXPLAIN ANALYZE` on key queries (patient lookup by name, appointment lookup by date range); verify index scans |

### Task 1.2: Authentication & Authorization

**What:** Implement email/password authentication with MFA support, JWT session management, role-based access control (RBAC), and the tenant context middleware that sets `app.current_practice_id` for RLS.

**Design:**

```typescript
// modules/auth/auth.service.ts
@Injectable()
export class AuthService {
  async login(email: string, password: string): Promise<AuthResponse> {
    const user = await this.userRepository.findByEmail(email);
    if (!user || !await bcrypt.compare(password, user.password_hash)) {
      throw new UnauthorizedException('Invalid credentials');
    }
    if (user.mfa_enabled) {
      return { requiresMfa: true, mfaToken: this.generateMfaToken(user) };
    }
    return this.issueTokens(user);
  }

  private issueTokens(user: AppUser): AuthResponse {
    const payload: JwtPayload = {
      sub: user.id,
      practice_id: user.practice_id,
      roles: user.roles,
      email: user.email,
    };
    return {
      accessToken: this.jwtService.sign(payload, { expiresIn: '15m' }),
      refreshToken: this.jwtService.sign({ sub: user.id }, { expiresIn: '7d' }),
    };
  }
}

// common/guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return requiredRoles.some((role) => user.roles.includes(role));
  }
}

// common/interceptors/tenant.interceptor.ts
@Injectable()
export class TenantInterceptor implements NestInterceptor {
  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const practiceId = request.user.practice_id;
    // Set PostgreSQL session variable for RLS
    await this.dataSource.query(`SET app.current_practice_id = '${practiceId}'`);
    return next.handle();
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Login with valid credentials returns JWT | Integration | POST `/auth/login` with valid email/password; verify 200 response with `accessToken` and `refreshToken` |
| Login with invalid password returns 401 | Integration | POST `/auth/login` with wrong password; verify 401 response |
| Login with MFA returns MFA challenge | Integration | POST `/auth/login` for MFA-enabled user; verify response includes `requiresMfa: true` |
| MFA verification with valid TOTP succeeds | Integration | POST `/auth/mfa/verify` with valid TOTP code; verify JWT issued |
| MFA verification with invalid TOTP returns 401 | Integration | POST `/auth/mfa/verify` with wrong TOTP; verify 401 |
| Expired JWT returns 401 | Integration | Create JWT with `expiresIn: '0s'`; use in request; verify 401 |
| Refresh token issues new access token | Integration | POST `/auth/refresh` with valid refresh token; verify new access token issued |
| RolesGuard blocks unauthorized role | Unit | Request endpoint requiring `admin` role with `front_desk` user; verify 403 |
| RolesGuard allows authorized role | Unit | Request endpoint requiring `billing` role with `billing` user; verify 200 |
| TenantInterceptor sets practice context | Integration | Authenticate as user of practice A; make request; verify `app.current_practice_id` is set to practice A UUID |
| Password hashing uses bcrypt with cost >= 12 | Unit | Verify `bcrypt.hash` called with rounds >= 12 |
| Brute force protection after 5 failed attempts | Integration | Attempt 5 failed logins; verify 6th attempt returns 429 |

### Task 1.3: Audit Logging Infrastructure

**What:** Implement the HIPAA-compliant audit logging system using the partitioned `audit_log` table, automatic partition creation, and an interceptor that logs all PHI access.

**Design:**

```typescript
// common/interceptors/audit.interceptor.ts
@Injectable()
export class AuditInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const startTime = Date.now();

    return next.handle().pipe(
      tap(async (responseBody) => {
        const resourceType = this.extractResourceType(request.path);
        const resourceId = request.params.id;
        const action = this.httpMethodToAction(request.method);
        const phiAccessed = PHI_RESOURCE_TYPES.includes(resourceType);

        await this.auditService.log({
          practice_id: request.user?.practice_id,
          user_id: request.user?.sub,
          action,
          resource_type: resourceType,
          resource_id: resourceId,
          ip_address: request.ip,
          phi_accessed: phiAccessed,
          changes: action === 'update' ? { old: request.previousState, new: request.body } : null,
          context: {
            user_agent: request.headers['user-agent'],
            request_id: request.id,
            duration_ms: Date.now() - startTime,
          },
        });
      }),
    );
  }
}

// modules/audit/partition-manager.service.ts
@Injectable()
export class PartitionManagerService {
  @Cron('0 0 1 * *') // Run on first of each month
  async createNextMonthPartition(): Promise<void> {
    const nextMonth = dayjs().add(1, 'month');
    const partitionName = `audit_log_${nextMonth.format('YYYY_MM')}`;
    const startDate = nextMonth.startOf('month').format('YYYY-MM-DD');
    const endDate = nextMonth.add(1, 'month').startOf('month').format('YYYY-MM-DD');

    await this.dataSource.query(`
      CREATE TABLE IF NOT EXISTS ${partitionName}
      PARTITION OF audit_log
      FOR VALUES FROM ('${startDate}') TO ('${endDate}')
    `);
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| API call creates audit log entry | Integration | GET `/patients/:id`; verify row inserted in `audit_log` with correct `resource_type`, `action`, `user_id` |
| PHI access is flagged | Integration | Access patient endpoint; verify `phi_accessed = true` in audit log |
| Non-PHI access is not flagged | Integration | Access practice settings; verify `phi_accessed = false` |
| Audit log captures IP address | Integration | Make request; verify `ip_address` field populated |
| Update operations capture old/new values | Integration | PATCH patient record; verify `changes` JSONB contains `old` and `new` objects |
| Monthly partition creation | Integration | Trigger partition creation; verify new partition table exists with correct range bounds |
| Audit log queries across partitions | Integration | Insert records in two different monthly partitions; query spanning both months; verify all records returned |
| Audit log query by user and date range | Integration | Insert 100 audit records for various users; query for specific user in 30-day window; verify only matching records returned |
| Partition pruning works | Integration | Run `EXPLAIN` on date-filtered query; verify only relevant partition scanned |

### Task 1.4: Docker Compose Development Environment

**What:** Create a Docker Compose setup that runs PostgreSQL, Redis, the NestJS API, the Next.js frontend, and stub services for development. Include hot-reload for all application services.

**Design:**

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: pms_dev
      POSTGRES_USER: pms
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./infrastructure/docker/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pms"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile.dev
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgres://pms:dev_password@postgres:5432/pms_dev
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-secret-change-in-production
      NODE_ENV: development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./apps/api/src:/app/src
    command: npm run start:dev

  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile.dev
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:3001
    depends_on:
      - api
    volumes:
      - ./apps/web/src:/app/src
    command: npm run dev

volumes:
  pgdata:
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| `docker compose up` starts all services | Integration | Run `docker compose up`; verify all 4 services reach healthy state within 60 seconds |
| API connects to PostgreSQL | Integration | Hit `/health` endpoint; verify database connectivity check passes |
| API connects to Redis | Integration | Hit `/health` endpoint; verify Redis connectivity check passes |
| Hot-reload works for API | Integration | Modify a source file in `apps/api/src`; verify NestJS restarts within 5 seconds |
| Hot-reload works for frontend | Integration | Modify a source file in `apps/web/src`; verify Next.js recompiles |
| Database persists across restarts | Integration | Insert data; `docker compose down && docker compose up`; verify data persists |

### Task 1.5: CI/CD Pipeline

**What:** Set up GitHub Actions for lint, type-check, unit tests, integration tests (against PostgreSQL in a service container), and Docker image builds.

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck

  test-api:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: pms_test
          POSTGRES_USER: pms
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run db:migrate --workspace=apps/api
      - run: npm run db:seed --workspace=apps/api
      - run: npm run test --workspace=apps/api
      - run: npm run test:e2e --workspace=apps/api
    env:
      DATABASE_URL: postgres://pms:test@localhost:5432/pms_test
      REDIS_URL: redis://localhost:6379

  test-web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run test --workspace=apps/web
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| CI passes on clean main branch | Pipeline | Push to main; verify all jobs pass (lint, typecheck, test-api, test-web) |
| CI fails on lint error | Pipeline | Push code with ESLint violation; verify lint job fails |
| CI fails on type error | Pipeline | Push code with TypeScript error; verify typecheck job fails |
| Integration tests run against PostgreSQL service | Pipeline | Verify test-api job connects to service container PostgreSQL |

---

## Phase 2: Scheduling Engine

**Goal:** Implement the core scheduling system: practitioner schedules, appointment booking, automated reminders, and the waitlist. This is the most visible and most-used feature of any PMS.

**Duration estimate:** 5-6 weeks

**Dependencies:** Phase 1 (database, auth, audit)

### Task 2.1: Practitioner & Schedule Management

**What:** CRUD operations for practitioners and their schedule blocks (recurring weekly schedules and date-specific overrides). Includes multi-location support.

**Design:**

```typescript
// modules/scheduling/dto/create-schedule-block.dto.ts
export class CreateScheduleBlockDto {
  @IsUUID() practitioner_id: string;
  @IsUUID() location_id: string;
  @IsOptional() @IsInt() @Min(0) @Max(6) day_of_week?: number;
  @IsOptional() @IsDateString() specific_date?: string;
  @Matches(/^\d{2}:\d{2}$/) start_time: string;
  @Matches(/^\d{2}:\d{2}$/) end_time: string;
  @IsIn(['available', 'blocked', 'lunch', 'meeting']) block_type: string;
  @IsOptional() @IsInt() @Min(5) @Max(120) slot_duration_minutes?: number;
  @IsOptional() @IsDateString() effective_from?: string;
  @IsOptional() @IsDateString() effective_to?: string;
}

// modules/scheduling/scheduling.service.ts
@Injectable()
export class SchedulingService {
  async getAvailableSlots(
    practitionerId: string,
    locationId: string,
    dateRange: { start: Date; end: Date },
    slotDurationMinutes: number,
  ): Promise<TimeSlot[]> {
    // 1. Fetch recurring schedule blocks for the practitioner at this location
    // 2. Fetch date-specific overrides (blocks or exceptions)
    // 3. Fetch existing appointments in the date range
    // 4. Generate available time slots by subtracting booked appointments
    //    and blocked periods from the schedule template
    // 5. Return array of {start, end, location_id, practitioner_id}
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Create practitioner with NPI | Integration | POST practitioner with NPI; verify created with status `active` |
| Create weekly recurring schedule | Integration | Create schedule block for Mon 9:00-17:00; verify stored correctly |
| Date-specific override blocks recurring slot | Unit | Recurring Mon 9:00-17:00 exists; add specific_date block for upcoming Monday 9:00-12:00; verify `getAvailableSlots` returns only 12:00-17:00 |
| Available slots correctly exclude existing appointments | Unit | Schedule 9:00-17:00 with 30-min slots; book 10:00-10:30; verify 10:00 slot not in available slots |
| Multi-location schedules do not overlap validation | Unit | Practitioner scheduled at Location A Mon 9-12 and Location B Mon 13-17; attempt to add Location A Mon 11-14; verify validation error |
| Slot generation respects slot_duration | Unit | Schedule 9:00-10:00 with 15-min slots; verify 4 slots generated |
| Schedule block CRUD with audit logging | Integration | Create, update, delete schedule block; verify audit_log entries for each operation |
| Practitioner deactivation | Integration | Deactivate practitioner; verify future appointments flagged for reassignment |

### Task 2.2: Appointment Booking

**What:** Book, reschedule, cancel, and check-in appointments. Support appointment types with configurable durations and default CPT codes. Track appointment source (manual, online, phone, AI agent).

**Design:**

```typescript
// modules/scheduling/appointment.service.ts
@Injectable()
export class AppointmentService {
  async bookAppointment(dto: BookAppointmentDto): Promise<Appointment> {
    // 1. Validate slot is available (check schedule + existing appointments)
    // 2. Validate patient exists and is active
    // 3. Create appointment with status 'booked'
    // 4. Trigger eligibility check if auto-check enabled in practice settings
    // 5. Schedule appointment reminders (SMS 24h before, email 48h before)
    // 6. If waitlist backfill enabled, check if this fills a waitlist request
    // 7. Return appointment with confirmation details
  }

  async cancelAppointment(appointmentId: string, reason: string): Promise<Appointment> {
    // 1. Update appointment status to 'cancelled'
    // 2. Store cancellation reason in details JSONB
    // 3. Cancel pending reminders
    // 4. If waitlist_auto_fill enabled, trigger waitlist matching for freed slot
    // 5. Log cancellation in audit trail
  }

  async checkIn(appointmentId: string): Promise<Appointment> {
    // 1. Update status from 'booked' to 'arrived'
    // 2. Verify intake forms completed; flag if incomplete
    // 3. Trigger real-time eligibility re-verification
  }
}
```

```typescript
// modules/scheduling/dto/book-appointment.dto.ts
export class BookAppointmentDto {
  @IsUUID() patient_id: string;
  @IsUUID() practitioner_id: string;
  @IsUUID() location_id: string;
  @IsDateString() start_time: string;
  @IsDateString() end_time: string;
  @IsOptional() @IsString() appointment_type?: string;
  @IsOptional() @IsString() reason_text?: string;
  @IsOptional() @IsIn(['manual', 'online', 'phone', 'ai_agent']) source?: string;
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Book appointment in available slot succeeds | Integration | Book appointment during open schedule; verify status is `booked`, audit log entry created |
| Double-booking same slot returns 409 Conflict | Integration | Book slot; attempt to book same slot for different patient; verify 409 response |
| Book appointment for inactive patient returns 400 | Integration | Deactivate patient; attempt booking; verify 400 response |
| Cancel appointment updates status | Integration | Book then cancel; verify status is `cancelled` and cancellation_reason stored |
| Cancel appointment triggers waitlist matching | Integration | Add patient to waitlist for Mon morning; cancel Mon morning appointment; verify waitlist entry status changes to `offered` |
| Reschedule preserves original booking metadata | Integration | Book, then reschedule to new time; verify original source and reason preserved |
| Check-in updates status to arrived | Integration | Book appointment; call check-in; verify status `arrived` |
| Check-in flags incomplete intake forms | Integration | Book appointment with required intake form; check-in without completing form; verify warning in response |
| Appointment list filtered by date range | Integration | Create 10 appointments across 3 days; query for 1 day; verify only that day's appointments returned |
| Appointment list filtered by practitioner | Integration | Create appointments for 2 practitioners; filter by one; verify only their appointments returned |
| FHIR-aligned status transitions enforced | Unit | Attempt invalid transition `cancelled` -> `booked`; verify rejection |
| Concurrent booking race condition handled | Integration | Simultaneously POST two bookings for same slot; verify exactly one succeeds, one gets 409 |

### Task 2.3: Appointment Reminders

**What:** Automated SMS and email reminder scheduling and delivery. Configurable reminder windows (e.g., 48h email, 24h SMS). Patient confirmation/cancellation response handling.

**Design:**

```typescript
// modules/scheduling/reminder.service.ts
@Injectable()
export class ReminderService {
  async scheduleReminders(appointment: Appointment): Promise<void> {
    const practice = await this.practiceService.findById(appointment.practice_id);
    const reminderConfig = practice.settings?.scheduling?.reminders || DEFAULT_REMINDERS;
    // DEFAULT_REMINDERS = [
    //   { channel: 'email', hours_before: 48 },
    //   { channel: 'sms',   hours_before: 24 },
    // ]

    for (const config of reminderConfig) {
      const scheduledAt = dayjs(appointment.start_time)
        .subtract(config.hours_before, 'hours')
        .toDate();

      if (scheduledAt > new Date()) {
        await this.reminderQueue.add('send-reminder', {
          appointment_id: appointment.id,
          channel: config.channel,
          scheduled_at: scheduledAt,
        }, { delay: scheduledAt.getTime() - Date.now() });
      }
    }
  }

  async processPatientResponse(appointmentId: string, response: 'confirmed' | 'cancelled'): Promise<void> {
    if (response === 'cancelled') {
      await this.appointmentService.cancelAppointment(appointmentId, 'Patient cancelled via reminder response');
    }
    // Update reminder record with response
    await this.updateReminderResponse(appointmentId, response);
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Booking schedules correct number of reminders | Integration | Book appointment; verify 2 reminders enqueued (email 48h, SMS 24h) |
| Reminder not scheduled if appointment is less than window hours away | Unit | Book appointment 12h from now; verify no 48h email reminder scheduled; 24h SMS also skipped |
| Cancelled appointment cancels pending reminders | Integration | Book appointment; cancel; verify pending reminder jobs removed from queue |
| SMS reminder delivery | Integration (with mock) | Process reminder job; verify Twilio API called with correct phone number and appointment details |
| Email reminder delivery | Integration (with mock) | Process reminder job; verify SendGrid API called with correct email and appointment details |
| Patient confirmation response updates reminder record | Integration | Simulate patient replying "C" to SMS; verify reminder response set to `confirmed` |
| Patient cancellation response triggers cancellation | Integration | Simulate patient replying "CANCEL" to SMS; verify appointment cancelled and waitlist triggered |
| Practice-specific reminder configuration | Unit | Practice with custom config (72h email only); verify only 1 reminder scheduled at 72h |

### Task 2.4: Waitlist Management

**What:** Patients can be added to a waitlist with preferences (practitioner, day, time range). When a slot opens (via cancellation), the system matches waitlist entries and offers the slot.

**Design:**

```typescript
// modules/scheduling/waitlist.service.ts
@Injectable()
export class WaitlistService {
  async addToWaitlist(dto: CreateWaitlistEntryDto): Promise<WaitlistEntry> {
    return this.waitlistRepo.create({
      practice_id: dto.practice_id,
      patient_id: dto.patient_id,
      practitioner_id: dto.practitioner_id,
      appointment_type: dto.appointment_type,
      preferences: {
        preferred_days: dto.preferred_days,
        preferred_time_start: dto.preferred_time_start,
        preferred_time_end: dto.preferred_time_end,
        location_id: dto.location_id,
      },
      status: 'waiting',
    });
  }

  async matchCancelledSlot(slot: { practitioner_id: string; location_id: string; start_time: Date; end_time: Date }): Promise<WaitlistEntry | null> {
    const dayOfWeek = dayjs(slot.start_time).day();
    const timeOfDay = dayjs(slot.start_time).format('HH:mm');

    // Find matching waitlist entries ordered by priority, then created_at
    const matches = await this.waitlistRepo.findMatching({
      practice_id: slot.practitioner_id, // via practitioner -> practice
      practitioner_id: slot.practitioner_id,
      day_of_week: dayOfWeek,
      time_of_day: timeOfDay,
      status: 'waiting',
    });

    if (matches.length > 0) {
      const bestMatch = matches[0];
      await this.waitlistRepo.updateStatus(bestMatch.id, 'offered');
      await this.notifyPatientOfOpening(bestMatch, slot);
      return bestMatch;
    }
    return null;
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Add patient to waitlist | Integration | POST waitlist entry with preferences; verify created with status `waiting` |
| Cancel appointment triggers waitlist match | Integration | Add patient to waitlist for Mon 9-12; cancel Mon 10:00 appointment; verify waitlist entry matched |
| Waitlist match respects day preference | Unit | Patient wants Tue/Thu only; slot opens Mon; verify no match |
| Waitlist match respects time preference | Unit | Patient wants 9:00-12:00; slot opens at 15:00; verify no match |
| Waitlist match respects practitioner preference | Unit | Patient wants Dr. A; slot opens for Dr. B; verify no match |
| Multiple waitlist entries prioritized by priority then created_at | Unit | 3 waitlist entries matching same slot; verify highest priority, then earliest created_at wins |
| Offered patient who declines returns entry to waiting | Integration | Match slot to patient; patient declines; verify status returns to `waiting` |
| Offered patient who accepts creates appointment | Integration | Match slot; patient accepts; verify appointment created and waitlist entry status is `booked` |
| Expired waitlist entries cleaned up | Integration | Create waitlist entry with 30-day expiry; advance time 31 days; run cleanup; verify status `expired` |

### Task 2.5: Scheduling Calendar UI

**What:** Multi-provider, multi-location calendar view. Day, week, and month views. Drag-and-drop appointment management. Color-coded by appointment type and practitioner. Online booking widget.

**Design:**

```typescript
// apps/web/src/app/scheduling/page.tsx
'use client';

import { ScheduleCalendar } from '@/components/scheduling/ScheduleCalendar';
import { PractitionerFilter } from '@/components/scheduling/PractitionerFilter';
import { LocationFilter } from '@/components/scheduling/LocationFilter';

export default function SchedulingPage() {
  const [selectedPractitioners, setSelectedPractitioners] = useState<string[]>([]);
  const [selectedLocation, setSelectedLocation] = useState<string | null>(null);
  const [view, setView] = useState<'day' | 'week' | 'month'>('day');
  const [currentDate, setCurrentDate] = useState(new Date());

  return (
    <div className="flex h-screen">
      <aside className="w-64 border-r p-4">
        <PractitionerFilter
          selected={selectedPractitioners}
          onChange={setSelectedPractitioners}
        />
        <LocationFilter
          selected={selectedLocation}
          onChange={setSelectedLocation}
        />
      </aside>
      <main className="flex-1">
        <ScheduleCalendar
          view={view}
          date={currentDate}
          practitioners={selectedPractitioners}
          location={selectedLocation}
          onAppointmentClick={handleAppointmentClick}
          onSlotClick={handleSlotClick}
          onAppointmentDrop={handleReschedule}
        />
      </main>
    </div>
  );
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Calendar renders day view with correct time slots | E2E | Navigate to scheduling page; verify day view shows time slots from practice open to close |
| Calendar shows appointments for selected practitioner | E2E | Select one practitioner; verify only their appointments displayed |
| Calendar shows appointments across all practitioners | E2E | Select multiple practitioners; verify color-coded appointments for each |
| Click empty slot opens booking dialog | E2E | Click empty time slot; verify booking form appears with pre-filled time and practitioner |
| Click existing appointment opens detail panel | E2E | Click appointment; verify patient name, time, type, status displayed |
| Drag appointment to new time triggers reschedule | E2E | Drag appointment to new slot; verify confirmation dialog; confirm; verify appointment moved |
| Location filter updates calendar | E2E | Switch location; verify calendar refreshes with appointments for new location |
| View toggle (day/week/month) works | E2E | Toggle between views; verify correct layout for each |

---

## Phase 3: Patient Management

**Goal:** Patient demographics, insurance coverage, digital intake forms, and the patient search/list interface.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 1 (database, auth), Phase 2 (scheduling - for linking patients to appointments)

### Task 3.1: Patient Demographics CRUD

**What:** Create, search, update, and manage patient records. Includes demographics, contact information, identifiers, and patient tags. Supports specialty-specific clinical summary via JSONB.

**Design:**

```typescript
// modules/patient/patient.service.ts
@Injectable()
export class PatientService {
  async create(dto: CreatePatientDto): Promise<Patient> {
    const mrn = await this.generateMrn(dto.practice_id);
    return this.patientRepo.create({
      ...dto,
      mrn,
      address: dto.address || null,
      contacts: dto.contacts || {},
      identifiers: dto.identifiers || [],
      clinical_summary: {},
      tags: dto.tags || [],
    });
  }

  async search(practiceId: string, query: PatientSearchQuery): Promise<PaginatedResult<Patient>> {
    // Supports: last_name, first_name, DOB, MRN, phone, email, tags
    // Full-text search across name fields
    // Pagination with cursor-based pagination for large result sets
  }

  async merge(primaryId: string, duplicateId: string): Promise<Patient> {
    // 1. Move all appointments, encounters, claims from duplicate to primary
    // 2. Merge identifiers, contacts, tags (union)
    // 3. Mark duplicate as inactive with reference to primary
    // 4. Log merge in audit trail
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Create patient with required fields | Integration | POST patient with name, DOB; verify MRN auto-generated |
| Create patient with full demographics | Integration | POST patient with all fields including address, contacts, identifiers; verify all stored |
| Search by last name (partial match) | Integration | Create patients "Smith", "Smithson"; search "Smi"; verify both returned |
| Search by DOB | Integration | Create patient with DOB 1985-03-15; search by DOB; verify found |
| Search by MRN (exact match) | Integration | Search by MRN; verify exact match returned |
| Search by phone number | Integration | Search by phone; verify patient found |
| Patient tags: add and filter | Integration | Add tag "VIP" to patient; search by tag "VIP"; verify found |
| Patient merge | Integration | Create 2 patients with overlapping data; merge; verify primary has combined data, duplicate marked inactive |
| Update address (US format) | Integration | Update patient address with US format; verify stored correctly |
| Update address (Canadian format) | Integration | Update patient address with Canadian province/postal code; verify stored correctly |
| Patient list pagination | Integration | Create 50 patients; request page 1 (20); verify 20 returned with cursor for next page |
| Deactivate patient | Integration | Deactivate patient; verify status `inactive`; verify patient excluded from default searches |
| PHI access logged | Integration | Read patient record; verify audit log entry with `phi_accessed = true` |

### Task 3.2: Insurance Coverage Management

**What:** Manage patient insurance coverages (primary, secondary, tertiary). Store payer information, subscriber details, plan details, and eligibility verification results.

**Design:**

```typescript
// modules/coverage/coverage.service.ts
@Injectable()
export class CoverageService {
  async addCoverage(dto: AddCoverageDto): Promise<PatientCoverage> {
    // Validate payer exists
    // Set coverage_order based on existing coverages (auto-increment if not specified)
    // Store subscriber_info JSONB
    // Trigger initial eligibility check if auto-check enabled
  }

  async verifyCoverage(coverageId: string): Promise<EligibilityResult> {
    // 1. Build X12 270 eligibility inquiry
    // 2. Submit to clearinghouse API (Availity/Office Ally)
    // 3. Parse X12 271 response
    // 4. Update patient_coverage.last_eligibility JSONB with parsed result
    // 5. Return structured eligibility result
  }

  async batchVerify(practiceId: string, date: Date): Promise<BatchEligibilityResult> {
    // Find all appointments for the given date
    // For each patient, check if eligibility was verified in last 24 hours
    // If not, add to batch verification queue
    // Process batch via clearinghouse batch API
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Add primary coverage | Integration | Add coverage with order 1; verify stored with subscriber details |
| Add secondary coverage | Integration | Add coverage with order 2; verify both primary and secondary returned for patient |
| Coverage order auto-assignment | Integration | Add coverage without specifying order; verify assigned next available order number |
| Payer validation | Integration | Attempt to add coverage with non-existent payer ID; verify 400 error |
| Eligibility verification (mocked clearinghouse) | Integration | Call verify; mock 271 response with active status; verify `last_eligibility` updated |
| Eligibility verification inactive result | Integration | Mock 271 response with inactive coverage; verify status stored as `inactive` |
| Batch eligibility for day's appointments | Integration | Create 5 appointments for tomorrow; run batch verify; verify all 5 coverages checked |
| Coverage termination | Integration | Set termination_date to today; verify coverage status changes to `inactive` |
| Coverage history preserved | Integration | Update coverage subscriber_id; verify old value accessible in audit log |

### Task 3.3: Digital Intake Forms

**What:** Practice-configurable intake forms using JSON Schema for form definition and JSONB for response storage. Forms are sent to patients before appointments and auto-import demographics into the patient record.

**Design:**

```typescript
// modules/forms/form-template.service.ts
@Injectable()
export class FormTemplateService {
  async createTemplate(dto: CreateFormTemplateDto): Promise<FormTemplate> {
    // Validate JSON Schema is well-formed
    // Store template with ui_schema for rendering hints
    return this.templateRepo.create({
      practice_id: dto.practice_id,
      name: dto.name,
      category: dto.category, // intake, consent, assessment, custom
      schema: dto.schema,      // JSON Schema Draft 2020-12
      ui_schema: dto.ui_schema,
      version: 1,
    });
  }
}

// modules/forms/form-submission.service.ts
@Injectable()
export class FormSubmissionService {
  async submitForm(dto: SubmitFormDto): Promise<FormSubmission> {
    const template = await this.templateRepo.findById(dto.template_id);
    // Validate responses against template JSON Schema using ajv
    const ajv = new Ajv();
    const validate = ajv.compile(template.schema);
    if (!validate(dto.responses)) {
      throw new BadRequestException(validate.errors);
    }
    return this.submissionRepo.create(dto);
  }

  async importDemographics(submissionId: string): Promise<void> {
    // Read submission responses
    // Map form fields to patient record fields (configured in template ui_schema)
    // Update patient record with imported data
    // Mark submission as 'imported'
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Create form template with valid JSON Schema | Integration | POST template with JSON Schema; verify stored |
| Submit form with valid responses | Integration | Submit responses matching schema; verify stored with status `submitted` |
| Submit form with invalid responses returns 400 | Integration | Submit responses missing required field; verify 400 with validation errors |
| Import demographics updates patient record | Integration | Submit demographics form; import; verify patient address/phone updated |
| Form versioning | Integration | Update template; verify version incremented; old submissions retain reference to old version |
| Link form submission to appointment | Integration | Submit form with appointment_id; verify retrievable via appointment |
| List pending intake forms for patient | Integration | Assign 3 forms to upcoming appointment; verify all 3 returned as pending |
| Form template deactivation | Integration | Deactivate template; verify not included in new form assignments |

### Task 3.4: Patient List & Search UI

**What:** Searchable, filterable patient list with quick-access to demographics, coverage, appointments, and balances. Patient detail view with tabbed layout.

**Design:**

```typescript
// apps/web/src/app/patients/page.tsx
export default function PatientsPage() {
  return (
    <div>
      <PatientSearchBar onSearch={handleSearch} />
      <PatientFilters
        filters={filters}
        onChange={setFilters}
        // Filters: status, tags, payer, last_visit_range
      />
      <PatientTable
        patients={patients}
        columns={['mrn', 'name', 'dob', 'phone', 'payer', 'last_visit', 'balance']}
        onRowClick={navigateToPatientDetail}
      />
    </div>
  );
}

// apps/web/src/app/patients/[id]/page.tsx
export default function PatientDetailPage({ params }: { params: { id: string } }) {
  return (
    <div>
      <PatientHeader patient={patient} />
      <Tabs defaultValue="demographics">
        <TabsList>
          <TabsTrigger value="demographics">Demographics</TabsTrigger>
          <TabsTrigger value="coverage">Insurance</TabsTrigger>
          <TabsTrigger value="appointments">Appointments</TabsTrigger>
          <TabsTrigger value="encounters">Encounters</TabsTrigger>
          <TabsTrigger value="billing">Billing</TabsTrigger>
          <TabsTrigger value="forms">Forms</TabsTrigger>
          <TabsTrigger value="messages">Messages</TabsTrigger>
        </TabsList>
        {/* Tab panels */}
      </Tabs>
    </div>
  );
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Patient search by name displays results | E2E | Type "Smi" in search bar; verify matching patients appear within 500ms |
| Patient detail page loads all tabs | E2E | Navigate to patient detail; verify all tabs render without errors |
| Demographics tab shows editable form | E2E | Click demographics tab; verify form fields populated; edit and save; verify updated |
| Insurance tab shows coverage cards | E2E | Navigate to coverage tab; verify primary/secondary insurance displayed |
| Appointments tab shows past and future | E2E | Verify upcoming and past appointments listed with status badges |
| Empty state for new patient | E2E | View patient with no appointments/encounters; verify appropriate empty state messages |
| Tag management on patient detail | E2E | Add tag via UI; verify tag appears; remove tag; verify removed |

---

## Phase 4: Billing & Claims Engine

**Goal:** Charge capture, claim generation, X12 EDI 837 claim submission, 835 ERA posting, denial tracking, and payment management.

**Duration estimate:** 6-7 weeks

**Dependencies:** Phase 3 (patient, coverage), Phase 2 (scheduling - encounters originate from appointments)

### Task 4.1: Charge Capture & Superbill

**What:** Record charges (CPT codes with modifiers, diagnosis pointers, units, amounts) for encounters. Support superbill templates per appointment type. Validate NCCI edits and modifier rules.

**Design:**

```typescript
// modules/billing/charge.service.ts
@Injectable()
export class ChargeService {
  async captureCharges(encounterId: string, charges: ChargeDto[]): Promise<Encounter> {
    // 1. Validate each CPT code exists in ref_code
    // 2. Validate each ICD-10 pointer references a valid encounter diagnosis
    // 3. Run NCCI edit checks (cannot bill code A with code B)
    // 4. Store charges in encounter.charges JSONB array
    // 5. Return updated encounter
  }

  async generateSuperbill(appointmentTypeId: string): Promise<ChargeDto[]> {
    // Return default charges for this appointment type
    // Including default CPT, units, and fee schedule amount
  }

  async lookupFeeSchedule(practiceId: string, cptCode: string, payerId?: string): Promise<number> {
    // Return the charge amount for this CPT code
    // Priority: payer-specific contracted rate > practice fee schedule > Medicare RBRVS
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Capture single charge with valid CPT/ICD | Integration | Add charge to encounter; verify stored in charges JSONB |
| Capture multiple charges | Integration | Add 3 charges; verify all stored with correct line numbers |
| Invalid CPT code rejected | Integration | Submit charge with non-existent CPT; verify 400 error |
| NCCI edit violation detected | Unit | Submit CPT codes that violate NCCI bundling rules; verify warning returned |
| ICD pointer validation | Unit | Submit charge pointing to diagnosis rank 3 when only 2 diagnoses exist; verify error |
| Superbill template generation | Integration | Generate superbill for "New Patient Visit"; verify default CPT codes returned |
| Fee schedule lookup | Integration | Look up charge amount for CPT 99213; verify correct amount returned |
| Modifier validation | Unit | Add modifier "25" to E/M code; verify accepted. Add invalid modifier; verify rejected |

### Task 4.2: Claim Generation & Submission

**What:** Generate claims from encounter charges. Build X12 EDI 837P transactions. Submit to clearinghouse via API. Track claim status.

**Design:**

```typescript
// modules/billing/claim.service.ts
@Injectable()
export class ClaimService {
  async generateClaim(encounterId: string): Promise<Claim> {
    const encounter = await this.encounterRepo.findWithCharges(encounterId);
    const coverage = await this.coverageService.getPrimaryCoverage(encounter.patient_id);
    
    // Build claim from encounter charges
    const claim = await this.claimRepo.create({
      practice_id: encounter.practice_id,
      patient_id: encounter.patient_id,
      encounter_id: encounter.id,
      patient_coverage_id: coverage.id,
      payer_id: coverage.payer_id,
      claim_type: 'professional',
      total_charge: this.sumCharges(encounter.charges),
      status: 'draft',
      lines: encounter.charges.map((charge, i) => ({
        line_number: i + 1,
        cpt_code: charge.cpt_code,
        modifiers: charge.modifiers || [],
        icd_codes: this.resolveIcdPointers(charge.icd_pointers, encounter.diagnoses),
        units: charge.units,
        charge_amount: charge.charge_amount,
        paid_amount: 0,
        adjusted_amount: 0,
        status: 'pending',
      })),
      submission: {
        billing_provider_npi: encounter.practice_npi,
        rendering_provider_npi: encounter.practitioner_npi,
        place_of_service: encounter.charges[0]?.place_of_service || '11',
      },
    });

    return claim;
  }

  async submitClaim(claimId: string): Promise<Claim> {
    // 1. Run AI claim scrubber (Phase 7) - for now, basic validation
    // 2. Generate X12 837P transaction via edi-gateway service
    // 3. Submit to clearinghouse API
    // 4. Update claim status to 'submitted'
    // 5. Store clearinghouse tracking number in submission JSONB
  }
}

// services/edi-gateway/x12-837.builder.ts
export class X12837Builder {
  build(claim: Claim, patient: Patient, coverage: PatientCoverage, practice: Practice): string {
    // Generate X12 EDI 837P transaction segments:
    // ISA (Interchange Control Header)
    // GS (Functional Group Header)
    // ST (Transaction Set Header)
    // BHT (Beginning of Hierarchical Transaction)
    // Loop 2000A (Billing Provider)
    // Loop 2000B (Subscriber)
    // Loop 2300 (Claim Information)
    // Loop 2400 (Service Line)
    // SE, GE, IEA (Trailers)
    return x12String;
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Generate claim from encounter with charges | Integration | Create encounter with 2 charges; generate claim; verify total_charge and 2 service lines |
| Claim status transitions | Unit | Draft -> submitted -> accepted -> paid; verify each transition valid |
| Invalid status transition rejected | Unit | Attempt `cancelled` -> `submitted`; verify error |
| X12 837P generation with valid segments | Unit | Build 837P; verify ISA, GS, ST, BHT, 2000A, 2000B, 2300, 2400 loops present |
| X12 837P contains correct NPI | Unit | Verify billing and rendering provider NPIs in correct segments |
| X12 837P contains correct subscriber info | Unit | Verify subscriber ID, group number, DOB in Loop 2010BA |
| Claim submission to clearinghouse (mocked) | Integration | Submit claim; mock clearinghouse API response; verify status updated to `submitted` |
| Clearinghouse rejection handling | Integration | Mock clearinghouse 4xx response; verify claim status `rejected` with error details |
| Claim list filtered by status | Integration | Create claims in various statuses; filter by `denied`; verify only denied returned |
| Claim list filtered by payer | Integration | Filter by specific payer; verify only that payer's claims returned |
| Duplicate claim detection | Unit | Attempt to generate second claim for same encounter/payer; verify warning |
| Secondary claim generation | Integration | Primary claim paid; generate secondary claim for secondary coverage; verify correct payer and adjusted amounts |

### Task 4.3: ERA 835 Posting & Payment Reconciliation

**What:** Receive and parse X12 EDI 835 Electronic Remittance Advice transactions. Post payments to claim lines. Track adjustment codes (CARC/RARC). Reconcile check/EFT payments.

**Design:**

```typescript
// services/edi-gateway/x12-835.parser.ts
export class X12835Parser {
  parse(rawEra: string): ParsedRemittance {
    // Parse X12 835 segments:
    // BPR (Financial Information) - payment method, amount, trace number
    // TRN (Reassociation Trace Number)
    // Loop 2100 (Claim Payment Information) - per-claim payment details
    // CAS (Claim Adjustment Segment) - CARC and RARC codes
    // SVC (Service Payment Information) - per-line payment details
    return {
      checkNumber: string,
      checkDate: Date,
      totalPaid: number,
      paymentMethod: 'eft' | 'check' | 'virtual_card',
      eftTraceNumber: string,
      claims: ClaimPayment[],
    };
  }
}

// modules/billing/remittance.service.ts
@Injectable()
export class RemittanceService {
  async postEra(rawEra: string): Promise<PostingResult> {
    const parsed = this.parser.parse(rawEra);
    // 1. For each claim in the 835:
    //    a. Match to existing claim by payer_claim_number or claim_number
    //    b. Update claim_line paid_amount and adjusted_amount
    //    c. Store CARC/RARC codes in adjustment_codes
    //    d. Update claim total_paid, total_adjusted
    //    e. Calculate patient_responsibility
    //    f. Update claim status (paid, partially_paid, denied)
    // 2. Store raw 835 in claim.responses JSONB array
    // 3. Return posting summary with matched/unmatched counts
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Parse valid X12 835 | Unit | Parse sample 835; verify check number, date, total, and per-claim details extracted |
| Post ERA updates claim status to paid | Integration | Submit claim; post 835 with full payment; verify claim status `paid` |
| Post ERA with partial payment | Integration | Post 835 with 70% payment; verify status `partially_paid` and patient responsibility calculated |
| Post ERA with denial | Integration | Post 835 with $0 payment and denial CARC codes; verify status `denied` and codes stored |
| CARC/RARC codes stored on claim lines | Integration | Post 835 with adjustment codes; verify codes stored per line |
| Unmatched claim in ERA logged | Integration | Post 835 with claim not in system; verify logged as unmatched |
| EFT trace number stored | Integration | Post 835 with EFT; verify trace number in responses JSONB |
| Duplicate ERA posting prevented | Integration | Post same 835 twice; verify second posting rejected or idempotent |
| ERA posting summary accurate | Integration | Post 835 with 5 claims (3 paid, 1 partial, 1 denied); verify summary counts |

### Task 4.4: Denial Tracking Dashboard

**What:** Visual dashboard showing claim denial rates by payer, code, and time period. Drill-down to individual denied claims. Denial reason categorization.

**Design:**

```typescript
// modules/billing/denial.service.ts
@Injectable()
export class DenialService {
  async getDenialSummary(practiceId: string, dateRange: DateRange): Promise<DenialSummary> {
    return this.claimRepo.query(`
      SELECT
        p.name AS payer_name,
        COUNT(*) AS total_claims,
        COUNT(*) FILTER (WHERE c.status = 'denied') AS denied_count,
        ROUND(COUNT(*) FILTER (WHERE c.status = 'denied')::NUMERIC / NULLIF(COUNT(*), 0) * 100, 1) AS denial_rate,
        -- Top denial codes
        (SELECT jsonb_agg(jsonb_build_object('code', dc.code, 'count', dc.cnt))
         FROM (
           SELECT elem->>'denial_code' AS code, COUNT(*) AS cnt
           FROM claim c2, jsonb_array_elements(c2.responses) AS elem
           WHERE c2.payer_id = c.payer_id AND c2.status = 'denied'
           GROUP BY elem->>'denial_code' ORDER BY cnt DESC LIMIT 5
         ) dc
        ) AS top_denial_codes
      FROM claim c
      JOIN payer p ON p.id = c.payer_id
      WHERE c.practice_id = $1
        AND c.submitted_at BETWEEN $2 AND $3
      GROUP BY p.name, c.payer_id
      ORDER BY denial_rate DESC
    `, [practiceId, dateRange.start, dateRange.end]);
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Denial summary by payer | Integration | Create claims for 3 payers with varying denial rates; verify summary sorted by denial rate |
| Denial rate calculation | Unit | 10 claims, 3 denied; verify denial rate = 30.0% |
| Top denial codes aggregated | Integration | Create denied claims with CARC codes; verify top codes listed |
| Date range filter works | Integration | Claims across 3 months; filter for 1 month; verify only that month's data |
| Drill-down to denied claims | Integration | Click payer in summary; verify list of denied claims for that payer |
| Dashboard renders chart and table | E2E | Navigate to denial dashboard; verify bar chart and data table render |
| Empty state when no denials | E2E | Practice with all paid claims; verify "No denials" message |

### Task 4.5: Payment Processing (Stripe)

**What:** Patient payment collection via Stripe for copays, deductibles, and patient responsibility balances. Payment plans for larger balances.

**Design:**

```typescript
// modules/billing/payment.service.ts
@Injectable()
export class PaymentService {
  async collectPayment(dto: CollectPaymentDto): Promise<PaymentResult> {
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount: Math.round(dto.amount * 100), // Stripe uses cents
      currency: 'usd',
      customer: dto.stripe_customer_id,
      metadata: {
        practice_id: dto.practice_id,
        patient_id: dto.patient_id,
        claim_ids: dto.claim_ids?.join(','),
      },
    });
    // Store payment record and link to claims
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Collect copay payment | Integration (Stripe test mode) | Charge $25 copay; verify Stripe payment intent created |
| Payment links to claim | Integration | Pay on claim; verify claim.patient_responsibility reduced |
| Payment receipt generation | Integration | Complete payment; verify receipt details returned |
| Failed payment handling | Integration | Use Stripe test card that declines; verify error returned to UI |
| Refund processing | Integration | Refund a payment; verify Stripe refund created and claim amounts adjusted |

---

## Phase 5: Clinical Documentation

**Goal:** Encounter management, clinical note creation with structured templates, diagnosis coding, and supervisor co-signature workflows.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 3 (patient), Phase 2 (scheduling - encounters link to appointments)

### Task 5.1: Encounter Management

**What:** Create encounters from checked-in appointments. Track encounter status (planned, in-progress, completed, cancelled). Link diagnoses, charges, and notes to encounters.

**Design:**

```typescript
// modules/clinical/encounter.service.ts
@Injectable()
export class EncounterService {
  async createFromAppointment(appointmentId: string): Promise<Encounter> {
    const appointment = await this.appointmentService.findById(appointmentId);
    if (appointment.status !== 'arrived' && appointment.status !== 'checked-in') {
      throw new BadRequestException('Appointment must be checked-in to create encounter');
    }
    return this.encounterRepo.create({
      practice_id: appointment.practice_id,
      patient_id: appointment.patient_id,
      practitioner_id: appointment.practitioner_id,
      appointment_id: appointmentId,
      location_id: appointment.location_id,
      encounter_date: new Date(),
      status: 'in-progress',
      class_code: appointment.details?.telehealth_url ? 'VR' : 'AMB',
      diagnoses: [],
      reason_text: appointment.reason_text,
    });
  }

  async addDiagnosis(encounterId: string, dto: AddDiagnosisDto): Promise<Encounter> {
    // Validate ICD-10 code exists in ref_code
    // Add to encounter.diagnoses JSONB array with auto-assigned rank
    // Update encounter
  }

  async completeEncounter(encounterId: string): Promise<Encounter> {
    // Verify at least 1 diagnosis added
    // Verify clinical note is signed
    // Update status to 'completed'
    // Trigger charge capture prompt if no charges recorded
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Create encounter from checked-in appointment | Integration | Check in appointment; create encounter; verify linked and status `in-progress` |
| Encounter from non-checked-in appointment fails | Integration | Attempt to create encounter from `booked` appointment; verify 400 |
| Add ICD-10 diagnosis | Integration | Add diagnosis with valid ICD-10; verify stored with rank 1 |
| Add multiple diagnoses with auto-ranking | Integration | Add 3 diagnoses; verify ranks 1, 2, 3 |
| Remove diagnosis reorders ranks | Integration | Remove rank 2 diagnosis; verify remaining are renumbered |
| Complete encounter with all requirements met | Integration | Add diagnosis, sign note, capture charge; complete; verify status `completed` |
| Complete encounter without diagnosis fails | Integration | Attempt to complete without diagnosis; verify 400 |
| Telehealth encounter has class_code VR | Integration | Create encounter from telehealth appointment; verify `class_code = 'VR'` |

### Task 5.2: Clinical Note Templates & Editor

**What:** Structured clinical note editor supporting SOAP, behavioral health progress, and custom note templates. Notes stored as structured JSONB. Draft/final/amended status workflow. Supervisor co-signature.

**Design:**

```typescript
// modules/clinical/note.service.ts
@Injectable()
export class NoteService {
  async createNote(dto: CreateNoteDto): Promise<ClinicalNote> {
    const template = await this.templateService.getByType(dto.note_type);
    return this.noteRepo.create({
      encounter_id: dto.encounter_id,
      author_id: dto.author_id,
      note_type: dto.note_type,
      content: dto.content, // Must conform to template schema
      status: 'draft',
      template_id: template.id,
    });
  }

  async signNote(noteId: string, userId: string): Promise<ClinicalNote> {
    const note = await this.noteRepo.findById(noteId);
    if (note.status !== 'draft') throw new BadRequestException('Only draft notes can be signed');
    if (note.author_id !== userId) throw new ForbiddenException('Only the author can sign');
    return this.noteRepo.update(noteId, {
      status: 'final',
      signed_by: userId,
      signed_at: new Date(),
    });
  }

  async cosignNote(noteId: string, supervisorId: string): Promise<ClinicalNote> {
    const note = await this.noteRepo.findById(noteId);
    if (note.status !== 'final') throw new BadRequestException('Note must be signed before co-signature');
    return this.noteRepo.update(noteId, {
      cosigned_by: supervisorId,
      cosigned_at: new Date(),
    });
  }

  async amendNote(noteId: string, amendment: any): Promise<ClinicalNote> {
    // Create amendment entry preserving original content
    // Update status to 'amended'
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Create SOAP note with structured content | Integration | Create note with subjective/objective/assessment/plan; verify stored |
| Create behavioral health progress note | Integration | Create note with therapy-specific fields; verify stored |
| Note content validated against template schema | Integration | Submit note with missing required field; verify 400 |
| Sign note transitions to final | Integration | Create draft note; sign; verify status `final` and signed_at populated |
| Only author can sign | Integration | Attempt sign by non-author; verify 403 |
| Co-sign requires signed note | Integration | Attempt co-sign on draft; verify 400 |
| Co-signature workflow | Integration | Author signs; supervisor co-signs; verify both signatures recorded |
| Amend note preserves original | Integration | Sign note; amend; verify status `amended` and original content preserved |
| Note history tracked | Integration | Create, sign, amend; verify audit trail has all 3 actions |
| Note editor UI renders SOAP template | E2E | Open note editor; select SOAP template; verify sections rendered |
| ICD-10 search in note editor | E2E | Type "hypertension" in diagnosis field; verify ICD-10 suggestions appear |

### Task 5.3: ICD-10 / CPT Code Search

**What:** Full-text search across ICD-10-CM and CPT code descriptions. Typeahead suggestions. Frequently-used codes per provider.

**Design:**

```typescript
// modules/clinical/code-search.service.ts
@Injectable()
export class CodeSearchService {
  async searchIcd10(query: string, limit: number = 20): Promise<RefCode[]> {
    return this.refCodeRepo.query(`
      SELECT code, display, metadata
      FROM ref_code
      WHERE code_system = 'icd10'
        AND is_active = true
        AND (
          code ILIKE $1
          OR to_tsvector('english', display) @@ plainto_tsquery('english', $2)
        )
      ORDER BY
        CASE WHEN code ILIKE $1 THEN 0 ELSE 1 END,
        ts_rank(to_tsvector('english', display), plainto_tsquery('english', $2)) DESC
      LIMIT $3
    `, [`${query}%`, query, limit]);
  }

  async getFrequentCodes(practitionerId: string, codeSystem: string): Promise<RefCode[]> {
    // Query encounter diagnoses/charges for this practitioner
    // Return top 20 most-used codes
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Search ICD-10 by code prefix | Integration | Search "J06"; verify returns J06.x codes |
| Search ICD-10 by description | Integration | Search "diabetes"; verify relevant diabetes codes returned |
| Full-text search ranks relevance | Integration | Search "heart failure"; verify I50.x codes ranked above general cardiac codes |
| CPT code search | Integration | Search "99213"; verify office visit code returned |
| Frequent codes for practitioner | Integration | Provider has 50 encounters with CPT 99213; verify 99213 in top codes |
| Search with no results | Integration | Search "xyznonexistent"; verify empty array returned |
| Search performance under 100ms | Integration | Search against full ICD-10 dataset (70,000+ codes); verify response under 100ms |

---

## Phase 6: FHIR R4 API & Interoperability

**Goal:** Expose a FHIR R4 compliant API for Patient, Appointment, Schedule, Encounter, and Claim resources. Implement SMART on FHIR for third-party app authorization.

**Duration estimate:** 5-6 weeks

**Dependencies:** Phase 3 (patient), Phase 2 (scheduling), Phase 5 (clinical)

### Task 6.1: FHIR Resource Endpoints

**What:** RESTful FHIR R4 endpoints for core resources: Patient, Practitioner, Appointment, Schedule, Slot, Encounter, Condition, Claim. Support FHIR search parameters, `_include`, and `_revinclude`.

**Design:**

```typescript
// modules/fhir/patient.fhir.controller.ts
@Controller('fhir/r4/Patient')
export class PatientFhirController {
  @Get(':id')
  async read(@Param('id') id: string): Promise<FhirPatient> {
    const patient = await this.patientService.findById(id);
    return this.mapper.toFhirPatient(patient);
  }

  @Get()
  async search(@Query() params: FhirSearchParams): Promise<FhirBundle> {
    // Support search params: family, given, birthdate, identifier, phone, email
    // Return FHIR Bundle of type 'searchset'
  }

  @Post()
  async create(@Body() resource: FhirPatient): Promise<FhirPatient> {
    const dto = this.mapper.fromFhirPatient(resource);
    const patient = await this.patientService.create(dto);
    return this.mapper.toFhirPatient(patient);
  }
}

// modules/fhir/mappers/patient.mapper.ts
export class PatientFhirMapper {
  toFhirPatient(patient: Patient): FhirPatient {
    return {
      resourceType: 'Patient',
      id: patient.id,
      identifier: patient.identifiers?.map(id => ({
        system: id.system,
        value: id.value,
        type: { coding: [{ code: id.type }] },
      })),
      name: [{ family: patient.last_name, given: [patient.first_name] }],
      birthDate: patient.date_of_birth,
      gender: patient.gender as FhirGender,
      telecom: this.buildTelecom(patient),
      address: patient.address ? [this.mapAddress(patient.address)] : [],
    };
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| GET /fhir/r4/Patient/:id returns FHIR Patient | Integration | Read patient; verify response conforms to FHIR Patient resource structure |
| GET /fhir/r4/Patient?family=Smith returns Bundle | Integration | Search by family name; verify FHIR Bundle with matching entries |
| POST /fhir/r4/Patient creates patient | Integration | POST FHIR Patient resource; verify patient created and FHIR response returned |
| FHIR Appointment resource mapping | Integration | Read appointment; verify FHIR Appointment with participant, period, status |
| FHIR Encounter resource mapping | Integration | Read encounter; verify FHIR Encounter with class, diagnosis, participant |
| FHIR search with _include | Integration | Search Appointment?_include=Appointment:patient; verify Patient included in Bundle |
| FHIR search pagination | Integration | 50 patients; search with _count=10; verify Bundle with next link |
| FHIR resource validation | Integration | POST invalid FHIR resource (missing required field); verify OperationOutcome error |
| CapabilityStatement endpoint | Integration | GET /fhir/r4/metadata; verify CapabilityStatement listing supported resources |

### Task 6.2: SMART on FHIR Authorization

**What:** Implement SMART App Launch Framework v2.2.0 for third-party application authorization. Support EHR-launch and standalone-launch flows. Granular FHIR scopes.

**Design:**

```typescript
// modules/fhir/smart/smart-auth.controller.ts
@Controller('.well-known')
export class SmartConfigController {
  @Get('smart-configuration')
  getSmartConfiguration(): SmartConfiguration {
    return {
      authorization_endpoint: `${this.baseUrl}/oauth/authorize`,
      token_endpoint: `${this.baseUrl}/oauth/token`,
      capabilities: ['launch-ehr', 'launch-standalone', 'client-public', 'client-confidential-symmetric'],
      scopes_supported: ['openid', 'fhirUser', 'patient/*.read', 'patient/Appointment.write'],
      // ... per SMART v2.2.0 spec
    };
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| /.well-known/smart-configuration returns valid config | Integration | GET endpoint; verify all required SMART configuration fields present |
| OAuth authorize flow with PKCE | Integration | Initiate authorization with code_verifier; verify code returned |
| Token exchange with valid code | Integration | Exchange authorization code for access token; verify FHIR scopes in token |
| Access FHIR endpoint with valid SMART token | Integration | Use SMART token to GET /fhir/r4/Patient; verify access granted |
| Scope enforcement | Integration | Token with `patient/Appointment.read` scope; attempt `patient/Patient.read`; verify 403 |
| EHR-launch context | Integration | Launch with `launch` and `iss` parameters; verify launch context resolved |

### Task 6.3: Bulk Data Export

**What:** Implement FHIR Bulk Data Access ($export) for population-level data export. Required for ONC certification.

**Design:**

```typescript
// modules/fhir/bulk/bulk-export.controller.ts
@Controller('fhir/r4')
export class BulkExportController {
  @Post('$export')
  @Header('Prefer', 'respond-async')
  async initiateExport(@Body() params: ExportParams): Promise<void> {
    // 1. Validate export parameters (_type, _since, _typeFilter)
    // 2. Create async export job
    // 3. Return 202 Accepted with Content-Location header for status polling
  }

  @Get('$export/:jobId')
  async checkExportStatus(@Param('jobId') jobId: string): Promise<ExportStatus> {
    // Return 202 (in-progress) or 200 (complete) with NDJSON file URLs
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Initiate $export returns 202 | Integration | POST $export; verify 202 with Content-Location header |
| Export status polling returns 202 while processing | Integration | Check status immediately after initiation; verify 202 |
| Completed export returns NDJSON URLs | Integration | Wait for export; verify 200 with output array of NDJSON file URLs |
| Export filtered by _type | Integration | Export only Patient and Encounter; verify only those resource types in output |
| Export filtered by _since | Integration | Export resources modified after specific date; verify older resources excluded |
| NDJSON format validation | Integration | Download exported file; verify each line is valid FHIR JSON |

---

## Phase 7: AI Features - No-Show Prediction & Claim Scrubbing

**Goal:** Deploy the first AI features: per-appointment no-show prediction with waitlist backfill, and AI-powered claim scrubbing before submission.

**Duration estimate:** 5-6 weeks

**Dependencies:** Phase 2 (scheduling), Phase 4 (billing)

### Task 7.1: No-Show Prediction Model

**What:** Train and deploy an ML model that predicts no-show probability for each appointment. Features include: patient history (prior no-show rate), day of week, time of day, appointment type, weather, lead time. Update waitlist backfill to use predictions.

**Design:**

```python
# services/ai-prediction/models/noshow_model.py
import xgboost as xgb
from fastapi import FastAPI

app = FastAPI()

class NoShowPredictor:
    def __init__(self):
        self.model = xgb.XGBClassifier()

    def extract_features(self, appointment: dict, patient_history: dict) -> dict:
        return {
            'prior_noshow_rate': patient_history.get('noshow_rate', 0),
            'prior_cancel_rate': patient_history.get('cancel_rate', 0),
            'total_prior_appointments': patient_history.get('total_appointments', 0),
            'day_of_week': appointment['day_of_week'],
            'hour_of_day': appointment['hour_of_day'],
            'is_new_patient': 1 if appointment.get('is_new_patient') else 0,
            'appointment_type_encoded': appointment.get('type_encoding', 0),
            'lead_time_days': appointment.get('lead_time_days', 0),
            'is_telehealth': 1 if appointment.get('is_telehealth') else 0,
            'has_insurance': 1 if appointment.get('has_insurance') else 0,
        }

    def predict(self, features: dict) -> float:
        # Returns probability 0.0 - 1.0
        return float(self.model.predict_proba([list(features.values())])[0][1])

@app.post("/predict/noshow")
async def predict_noshow(request: NoShowRequest):
    features = predictor.extract_features(request.appointment, request.patient_history)
    probability = predictor.predict(features)
    return {"probability": probability, "features": features, "model_version": MODEL_VERSION}
```

```typescript
// modules/ai/noshow.service.ts (NestJS orchestration)
@Injectable()
export class NoShowService {
  @Cron('0 2 * * *') // Run nightly at 2 AM
  async predictNextDayNoShows(): Promise<void> {
    const tomorrow = dayjs().add(1, 'day');
    const appointments = await this.appointmentRepo.findByDate(tomorrow);

    for (const appointment of appointments) {
      const history = await this.getPatientHistory(appointment.patient_id);
      const prediction = await this.aiClient.predictNoShow(appointment, history);

      // Update appointment with prediction
      await this.appointmentRepo.updateDetails(appointment.id, {
        no_show_probability: prediction.probability,
      });

      // Store prediction for model feedback
      await this.aiPredictionRepo.create({
        practice_id: appointment.practice_id,
        prediction_type: 'no_show',
        entity_type: 'appointment',
        entity_id: appointment.id,
        probability: prediction.probability,
        result: { model_version: prediction.model_version, features: prediction.features },
      });

      // If high probability, trigger waitlist backfill
      if (prediction.probability > 0.3) {
        await this.waitlistService.preemptiveMatch(appointment);
      }
    }
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Feature extraction produces correct features | Unit | Provide appointment and history; verify all 10 features extracted correctly |
| Model prediction returns valid probability | Unit | Send features to model; verify probability between 0.0 and 1.0 |
| High no-show prediction triggers waitlist check | Integration | Set threshold 0.3; predict 0.45; verify waitlist matching triggered |
| Low no-show prediction does not trigger waitlist | Integration | Predict 0.08; verify no waitlist action |
| Prediction stored in ai_prediction table | Integration | Run prediction; verify ai_prediction record created with features and model_version |
| Nightly batch processes all next-day appointments | Integration | Create 10 appointments for tomorrow; run batch; verify all 10 have predictions |
| Model feedback loop | Integration | After appointment occurs, record actual outcome (noshow/attended); verify `outcome` field updated |
| Prediction API performance | Load | 100 concurrent prediction requests; verify p99 latency under 200ms |

### Task 7.2: AI Claim Scrubbing

**What:** Before claim submission, run AI-powered validation that checks payer-specific rules, NCCI edits, and historical denial patterns to flag likely rejections.

**Design:**

```python
# services/claim-scrubber/scrubber.py
class ClaimScrubber:
    def scrub(self, claim: dict, payer_rules: dict, denial_history: list) -> list[ScrubResult]:
        results = []
        # 1. NCCI edit checks (CCI edits - column 1/column 2 pairs)
        results.extend(self.check_ncci_edits(claim['lines']))
        # 2. Payer-specific rules (from payer.rules JSONB)
        results.extend(self.check_payer_rules(claim, payer_rules))
        # 3. AI denial prediction based on historical patterns
        results.extend(self.predict_denial_risk(claim, denial_history))
        # 4. Documentation sufficiency check
        results.extend(self.check_documentation(claim))
        return results

    def predict_denial_risk(self, claim: dict, denial_history: list) -> list[ScrubResult]:
        # Use trained model to predict denial probability per line
        # Based on: CPT+ICD combination, payer, provider, historical denial rate for this combo
        pass
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| NCCI edit violation detected | Unit | Submit CPT codes 99213 + 99214 (mutually exclusive); verify error returned |
| NCCI edit with valid modifier passes | Unit | Submit CPT codes with modifier 25 bypassing NCCI edit; verify no error |
| Payer-specific prior auth rule flagged | Unit | Claim for CPT requiring PA per payer rules; no PA on file; verify warning |
| AI denial risk above threshold flagged | Unit | Claim with CPT+ICD combo historically denied 40% by this payer; verify warning |
| Clean claim returns empty results | Unit | Well-formed claim with no issues; verify empty scrub results array |
| Scrub results stored on claim | Integration | Scrub claim; verify scrub_results JSONB populated |
| Scrub results block submission if severity = error | Integration | Claim with NCCI error; attempt submit; verify blocked with error details |
| Scrub results allow submission if severity = warning | Integration | Claim with AI warning only; submit; verify submission proceeds |

---

## Phase 8: Patient Portal & Engagement

**Goal:** Patient-facing portal for self-scheduling, intake form completion, balance viewing, and secure messaging.

**Duration estimate:** 4-5 weeks

**Dependencies:** Phase 2 (scheduling), Phase 3 (patient, forms), Phase 4 (billing)

### Task 8.1: Patient Portal Authentication

**What:** Separate authentication flow for patients (not practice users). Email/password with magic link option. Patient identity verification.

**Design:**

```typescript
// apps/patient-portal/src/modules/auth/patient-auth.service.ts
@Injectable()
export class PatientAuthService {
  async sendMagicLink(email: string): Promise<void> {
    const patient = await this.patientRepo.findByEmail(email);
    if (!patient) throw new NotFoundException('No account found');
    const token = this.generateMagicToken(patient.id);
    await this.emailService.sendMagicLink(email, token);
  }

  async verifyMagicLink(token: string): Promise<PatientAuthResponse> {
    const patientId = this.verifyMagicToken(token);
    return this.issuePatientTokens(patientId);
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Magic link sent to valid patient email | Integration | Request magic link; verify email sent with valid token |
| Magic link login | Integration | Click magic link; verify patient session created |
| Expired magic link rejected | Integration | Use magic link after 15 minutes; verify 401 |
| Patient cannot access other patients' data | Integration | Authenticate as patient A; attempt to access patient B data; verify 403 |
| Patient portal RLS isolation | Integration | Patient portal queries only return data for authenticated patient |

### Task 8.2: Self-Scheduling & Intake

**What:** Patient-facing appointment booking widget. Practitioner/location/type selection. Available slot display. Intake form completion before appointment.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Patient sees available slots | E2E | Open booking widget; select practitioner; verify available slots displayed |
| Patient books appointment | E2E | Select slot; confirm booking; verify appointment created with source `online` |
| Intake forms delivered after booking | E2E | Book appointment; verify intake forms appear in portal |
| Patient completes intake form | E2E | Fill out intake form; submit; verify submission stored |
| Double-booking prevented | E2E | Two patients simultaneously attempt same slot; verify one succeeds, one gets error |

### Task 8.3: Balance View & Payments

**What:** Patient can view outstanding balances, payment history, and make payments via Stripe.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Patient sees outstanding balance | E2E | Navigate to billing tab; verify correct balance displayed |
| Patient makes payment | E2E | Click pay; enter card; verify payment processed and balance updated |
| Payment history shows past payments | E2E | Verify list of past payments with dates and amounts |

### Task 8.4: Secure Messaging

**What:** HIPAA-compliant messaging between patient and practice staff. Message threads with read receipts.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Patient sends message to practice | Integration | POST message from patient portal; verify received in practice inbox |
| Practice staff replies to patient | Integration | Reply from practice UI; verify patient sees reply in portal |
| Unread message count | Integration | Send 3 messages; verify unread count = 3; read 1; verify count = 2 |
| Message notification (email) | Integration | Practice sends message; verify patient email notification sent |

---

## Phase 9: Revenue Cycle Analytics

**Goal:** Financial dashboards, reporting, and revenue cycle KPIs for practice administrators.

**Duration estimate:** 3-4 weeks

**Dependencies:** Phase 4 (billing)

### Task 9.1: Financial Dashboards

**What:** Dashboard showing: collections by period, denial rate by payer, days in A/R, clean claim rate, provider productivity, and undercoding alerts.

**Design:**

```typescript
// modules/reports/revenue-cycle.service.ts
@Injectable()
export class RevenueCycleService {
  async getDashboardKPIs(practiceId: string, period: DateRange): Promise<RevenueCycleDashboard> {
    return {
      totalCharges: await this.getTotalCharges(practiceId, period),
      totalCollections: await this.getTotalCollections(practiceId, period),
      collectionRate: await this.getCollectionRate(practiceId, period),
      denialRate: await this.getDenialRate(practiceId, period),
      averageDaysInAR: await this.getAvgDaysInAR(practiceId, period),
      cleanClaimRate: await this.getCleanClaimRate(practiceId, period),
      topDenialReasons: await this.getTopDenialReasons(practiceId, period),
      collectionsByPayer: await this.getCollectionsByPayer(practiceId, period),
      productivityByProvider: await this.getProductivityByProvider(practiceId, period),
    };
  }
}
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Collection rate calculated correctly | Unit | $100K charges, $85K collected; verify rate = 85.0% |
| Days in A/R calculated correctly | Unit | Claim submitted 30 days ago, paid 10 days ago; verify days in AR = 20 |
| Clean claim rate | Unit | 100 claims submitted, 90 accepted on first pass; verify clean claim rate = 90% |
| Dashboard renders charts | E2E | Navigate to revenue cycle dashboard; verify charts render with data |
| Date range filter updates all KPIs | E2E | Change date range; verify all metrics refresh |
| Export to CSV | E2E | Click export; verify CSV file downloaded with correct data |

### Task 9.2: Scheduled Reports

**What:** Automated report generation and email delivery (daily, weekly, monthly). Report types: collections summary, aging report, denial analysis.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Schedule weekly report | Integration | Schedule report; verify cron job created |
| Report generated and emailed | Integration | Trigger report; verify PDF/CSV generated and emailed to configured recipients |
| Report contains correct date range | Integration | Weekly report for last week; verify data matches date range |

---

## Phase 10: Advanced AI Features

**Goal:** Ambient clinical documentation, conversational patient intake, and revenue cycle anomaly detection.

**Duration estimate:** 6-8 weeks

**Dependencies:** Phase 5 (clinical documentation), Phase 7 (AI infrastructure)

### Task 10.1: AI Ambient Documentation

**What:** Process encounter audio to generate structured clinical note drafts with CPT/ICD-10 code suggestions. Provider reviews and edits before signing.

**Design:**

```python
# services/ambient-doc/ambient_service.py
class AmbientDocumentationService:
    async def process_encounter_audio(self, audio_url: str, note_type: str) -> AmbientResult:
        # 1. Transcribe audio using speech-to-text (Whisper or cloud STT)
        transcript = await self.transcribe(audio_url)
        # 2. Extract structured note from transcript using LLM
        structured_note = await self.extract_note(transcript, note_type)
        # 3. Suggest ICD-10 codes from assessment
        icd_suggestions = await self.suggest_icd_codes(structured_note['assessment'])
        # 4. Suggest CPT codes from services described
        cpt_suggestions = await self.suggest_cpt_codes(structured_note, transcript)
        return AmbientResult(
            note_content=structured_note,
            icd_suggestions=icd_suggestions,
            cpt_suggestions=cpt_suggestions,
            confidence=self.calculate_confidence(structured_note),
            ai_metadata={'model': MODEL_VERSION, 'transcript_length': len(transcript)},
        )
```

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Audio transcription | Integration | Process sample audio; verify transcript generated |
| SOAP note extraction from transcript | Integration | Process office visit transcript; verify S/O/A/P sections populated |
| ICD-10 suggestions relevant | Integration | Transcript mentions "hypertension"; verify I10 suggested |
| CPT suggestions relevant | Integration | Transcript describes established patient visit; verify 99213/99214 suggested |
| AI-generated note flagged | Integration | Verify note created with `ai_metadata.generated = true` |
| Provider can edit AI-generated note | Integration | Generate AI note; provider edits subjective section; save; verify edits persisted |
| Confidence score calculated | Unit | Note with all sections populated gets higher confidence than partial note |

### Task 10.2: Conversational Patient Intake

**What:** AI-powered voice or chat agent that handles appointment booking, intake form completion, and insurance capture without front-desk involvement.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Chat agent books appointment | Integration | Simulate patient conversation requesting appointment; verify appointment created |
| Chat agent collects insurance info | Integration | Simulate insurance capture conversation; verify coverage record created |
| Chat agent completes intake form | Integration | Agent guides patient through form questions; verify form submission created |
| Handoff to human when needed | Integration | Agent encounters unsupported request; verify handoff to front desk queue |
| Conversation logged for audit | Integration | Verify conversation transcript stored in patient_message table |

### Task 10.3: Revenue Cycle Anomaly Detection

**What:** AI monitoring that surfaces undercoding, missed charges, and unusual denial spikes without requiring a data analyst.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Undercoding detection | Integration | Provider consistently codes 99213 when 99214 documentation supports it; verify alert generated |
| Missed charge detection | Integration | Encounter has documented procedure without corresponding charge; verify alert |
| Denial spike alert | Integration | Payer denial rate jumps from 5% to 20% in one week; verify alert generated |
| Anomaly dashboard | E2E | Navigate to anomaly dashboard; verify alerts listed with severity and recommended action |

---

## Phase 11: Compliance Hardening & Certification

**Goal:** Security hardening, penetration testing, HIPAA compliance documentation, and ONC certification preparation.

**Duration estimate:** 4-6 weeks

**Dependencies:** All prior phases

### Task 11.1: HIPAA Security Hardening

**What:** Encryption at rest (database, file storage), encryption in transit (TLS 1.2+), access control review, backup and disaster recovery procedures, workforce training documentation.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| All API endpoints require TLS 1.2+ | Security | Attempt HTTP connection; verify rejected. Attempt TLS 1.1; verify rejected |
| Database encryption at rest enabled | Security | Verify PostgreSQL data directory on encrypted volume |
| PHI fields encrypted at application level | Security | SSN, DOB stored with application-level encryption; verify ciphertext in database |
| Backup encryption | Security | Verify database backups are encrypted |
| Session timeout after 15 minutes inactivity | Security | Leave session idle; verify auto-logout after 15 minutes |
| Account lockout after failed attempts | Security | 5 failed login attempts; verify account locked |
| Audit log tamper protection | Security | Verify audit log table has no DELETE/UPDATE permissions for application role |

### Task 11.2: Penetration Testing

**What:** Engage third-party security firm for HIPAA penetration testing. Remediate findings.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| SQL injection | Security | Attempt SQL injection on all input fields; verify no injection possible |
| XSS prevention | Security | Attempt XSS payloads in all text fields; verify sanitized |
| IDOR prevention | Security | Attempt to access resources belonging to other practices; verify blocked |
| CSRF protection | Security | Attempt CSRF attacks; verify tokens required and validated |
| Rate limiting | Security | Exceed rate limits on authentication endpoints; verify 429 returned |

### Task 11.3: ONC Certification Preparation

**What:** Align FHIR API, SMART on FHIR, Bulk Data, and USCDI data elements with ONC Health IT Certification requirements (21st Century Cures Act).

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| USCDI v3 data elements supported | Compliance | Verify all mandatory USCDI v3 data elements are represented in FHIR Patient/Encounter resources |
| FHIR Capability Statement accurate | Compliance | Verify CapabilityStatement reflects actual supported resources and operations |
| Information blocking prevention | Compliance | Verify no patient data access requests are blocked or delayed beyond specification |
| Bulk Data export test suite | Compliance | Run ONC test suite for Bulk Data; verify all test cases pass |
| SMART App Launch test suite | Compliance | Run SMART on FHIR test suite; verify authorization flows pass |

---

## Phase 12: Multi-Jurisdiction & Internationalization

**Goal:** Support for Canadian provincial billing (Ontario OHIP, BC MSP), Australian and UK payment models, and localization (date formats, currencies, languages).

**Duration estimate:** 4-6 weeks

**Dependencies:** Phase 4 (billing), Phase 3 (patient)

### Task 12.1: Canadian Provincial Billing

**What:** Support for provincial billing integration: OHIP (Ontario), MSP (British Columbia), AHCIP (Alberta). Province-specific claim formats and fee schedules.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| OHIP claim submission | Integration | Submit claim in OHIP format; verify correct fee code and diagnostic code format |
| MSP claim submission | Integration | Submit MSP claim; verify BC-specific fields populated |
| Provincial health number stored | Integration | Store OHIP number in patient identifiers JSONB; verify retrievable |
| Multi-province practice | Integration | Practice with locations in ON and BC; verify correct billing format per location |

### Task 12.2: Localization

**What:** Date format, currency, timezone, and language localization. Support for English, French, Spanish.

**Testing:**

| Test Case | Type | Description |
|-----------|------|-------------|
| Date format respects practice locale | E2E | Practice set to Canadian locale; verify DD/MM/YYYY format |
| Currency display | E2E | Canadian practice; verify amounts displayed with CAD symbol |
| Timezone handling | Integration | Appointment booked in EST practice; viewed by PST user; verify correct time displayed |
| French language UI | E2E | Switch language to French; verify all UI strings translated |

---

## Definition of Done

A phase is considered complete when all of the following criteria are met:

### Code Quality
- [ ] All tasks in the phase have been implemented
- [ ] Code passes lint (ESLint for TypeScript, Ruff for Python) with zero warnings
- [ ] TypeScript strict mode compilation passes with zero errors
- [ ] All functions/methods have JSDoc/docstring documentation
- [ ] No `any` types in TypeScript code (exceptions must be documented)

### Testing
- [ ] Unit test coverage >= 80% for business logic modules
- [ ] Integration tests pass against PostgreSQL and Redis service containers
- [ ] E2E tests pass for all UI workflows in the phase
- [ ] All test cases listed in the phase specification have been implemented and pass
- [ ] No known flaky tests

### Security & Compliance
- [ ] HIPAA audit logging verified for all PHI access endpoints
- [ ] Row-Level Security (RLS) tested with cross-tenant access attempts
- [ ] No secrets or credentials in code or configuration files
- [ ] Dependency vulnerability scan (npm audit, pip audit) shows no high/critical vulnerabilities
- [ ] Input validation on all API endpoints (request body, query parameters, path parameters)

### Documentation
- [ ] OpenAPI spec updated and matches implemented endpoints
- [ ] Architecture Decision Records (ADRs) written for significant technical decisions
- [ ] Database migration files are reversible (up and down)
- [ ] README updated with any new setup steps or environment variables

### Deployment
- [ ] Docker Compose setup works for local development
- [ ] CI pipeline passes (lint, typecheck, test, build)
- [ ] Database migrations run successfully against a fresh database
- [ ] Database migrations run successfully against the previous phase's database state
- [ ] No breaking changes to existing API endpoints (or version bump if breaking)

### Review
- [ ] Pull request reviewed and approved by at least one team member
- [ ] Demo conducted showing all phase features working end-to-end
- [ ] Performance benchmarks recorded for key operations (search < 100ms, page load < 2s)
