# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Practice Management System · Created: 2026-05-19

## Philosophy

This model treats every state change in the system as an immutable event stored in a single append-only event store. The event store is the sole source of truth; all queryable state (patient records, appointment calendars, claim statuses, financial summaries) is derived by replaying events into materialized read models (projections). This is the CQRS (Command Query Responsibility Segregation) pattern applied to healthcare.

The key insight for a practice management system is that HIPAA requires comprehensive audit trails for all PHI access and modifications, and healthcare billing requires the ability to answer temporal questions ("what was the patient's coverage on the date of service?"). In a traditional system, audit logging is bolted on as a secondary concern. In an event-sourced system, the audit trail IS the data — every change is recorded with who did it, when, and why. Temporal queries become trivial: replay events up to any point in time to reconstruct the state at that moment.

This pattern is used in financial systems (banking ledgers are inherently event-sourced), healthcare systems described in InfoQ's "Healthy Architectures" series, and is particularly well-suited for revenue cycle management where claim state transitions (submitted, accepted, denied, appealed, paid) form a natural event stream. The trade-off is increased complexity: developers must think in terms of events rather than CRUD, and the read models require careful design and maintenance.

**Best for:** Teams prioritizing HIPAA audit compliance, temporal queries, AI analytics on change patterns, and systems where the complete history of every entity matters.

**Trade-offs:**
- (+) Complete, immutable audit trail for HIPAA compliance — audit IS the data, not an afterthought
- (+) Temporal queries are trivial: reconstruct any entity's state at any point in time
- (+) AI analytics can process the event stream for pattern detection (denial patterns, no-show patterns, billing anomalies)
- (+) Event replay enables schema evolution without data migration — add new projections from existing events
- (+) Natural fit for revenue cycle management where claim status transitions are the core workflow
- (-) Higher implementation complexity: developers must design events, commands, and projections
- (-) Eventual consistency between write (event store) and read (projections) can confuse users expecting immediate reads
- (-) Projection rebuilds can be slow for long-lived entities with many events
- (-) Debugging requires understanding the event stream, not just the current state
- (-) Storage grows continuously (events are never deleted); requires archival strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HIPAA Security Rule | The event store IS the audit trail — every PHI access/modification is an immutable event with user, timestamp, and IP |
| HL7 FHIR R4 | Read model projections are shaped to match FHIR resources for API responses; events carry FHIR resource type metadata |
| X12 EDI 837/835 | Claim submission and remittance events capture the full X12 lifecycle as discrete events |
| X12 EDI 270/271 | Eligibility verification events record request and response as separate events in the stream |
| OCSF (Open Cybersecurity Schema Framework) | Event metadata structure (actor, action, object, timestamp) aligned with OCSF audit event patterns |
| ICD-10-CM / CPT | Diagnosis and procedure codes embedded in clinical and billing events |
| ISO 8601 | All event timestamps in UTC with ISO 8601 format |

---

## Event Store (Core Infrastructure)

```sql
-- The event store is the single source of truth. All state is derived from events.
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,              -- Aggregate root ID (patient, appointment, claim, etc.)
    stream_type     VARCHAR(50) NOT NULL,        -- Aggregate type: 'Patient', 'Appointment', 'Claim', etc.
    event_type      VARCHAR(100) NOT NULL,       -- e.g., 'PatientRegistered', 'AppointmentBooked', 'ClaimSubmitted'
    event_version   INTEGER NOT NULL,            -- Monotonically increasing per stream (optimistic concurrency)
    practice_id     UUID NOT NULL,               -- Tenant isolation
    actor_id        UUID,                        -- User or system that caused the event
    actor_type      VARCHAR(20) NOT NULL DEFAULT 'user',  -- user, system, ai_agent, patient
    actor_ip        INET,
    payload         JSONB NOT NULL,              -- Event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- Correlation ID, causation ID, FHIR resource type, etc.
    phi_involved    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)             -- Optimistic concurrency control
) PARTITION BY RANGE (created_at);

-- Monthly partitions for performance and archival
CREATE TABLE event_store_2026_05 PARTITION OF event_store
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, created_at);
CREATE INDEX idx_event_practice ON event_store(practice_id, created_at);
CREATE INDEX idx_event_actor ON event_store(actor_id, created_at);
CREATE INDEX idx_event_phi ON event_store(practice_id, created_at) WHERE phi_involved = true;

-- Event type registry for documentation and validation
CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT,
    payload_schema  JSONB,              -- JSON Schema for payload validation
    phi_default     BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Events

```json
-- PatientRegistered event payload
{
    "first_name": "Jane",
    "last_name": "Doe",
    "date_of_birth": "1985-03-15",
    "gender": "female",
    "email": "jane.doe@example.com",
    "phone_mobile": "+12025551234",
    "address": {
        "line1": "123 Main St",
        "city": "Springfield",
        "state": "IL",
        "postal_code": "62701",
        "country": "US"
    },
    "mrn": "MRN-2026-0042"
}

-- AppointmentBooked event payload
{
    "patient_id": "a1b2c3d4-...",
    "practitioner_id": "e5f6g7h8-...",
    "location_id": "i9j0k1l2-...",
    "appointment_type": "follow_up",
    "start_time": "2026-05-20T09:00:00-05:00",
    "end_time": "2026-05-20T09:30:00-05:00",
    "reason_text": "Follow-up for hypertension management",
    "source": "online_booking"
}

-- ClaimDenied event payload
{
    "claim_id": "m3n4o5p6-...",
    "payer_claim_number": "CLM-2026-78901",
    "denial_code": "CO-4",
    "denial_reason": "The procedure code is inconsistent with the modifier used",
    "denied_lines": [1, 3],
    "total_denied": 285.00,
    "payer_id": "q7r8s9t0-..."
}
```

## Command Processing

```sql
-- Commands represent intentions. They are validated, then produce events.
-- Commands are stored for debugging and replay analysis.
CREATE TABLE command_log (
    command_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type    VARCHAR(100) NOT NULL,   -- 'RegisterPatient', 'BookAppointment', 'SubmitClaim'
    practice_id     UUID NOT NULL,
    actor_id        UUID,
    payload         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, accepted, rejected, failed
    rejection_reason TEXT,
    events_produced UUID[],                 -- References to event_store.event_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);
CREATE INDEX idx_command_practice ON command_log(practice_id, created_at);
CREATE INDEX idx_command_status ON command_log(status) WHERE status IN ('pending', 'failed');
```

## Read Model Projections

Read models are derived entirely from events. They can be rebuilt at any time by replaying the event stream.

### Patient Projection

```sql
CREATE TABLE rm_patient (
    id                  UUID PRIMARY KEY,       -- Same as stream_id from event_store
    practice_id         UUID NOT NULL,
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    middle_name         VARCHAR(100),
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),
    email               VARCHAR(255),
    phone_mobile        VARCHAR(20),
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state_code          VARCHAR(10),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'US',
    preferred_language  VARCHAR(10) DEFAULT 'en',
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    last_event_version  INTEGER NOT NULL,       -- Last applied event version for idempotency
    last_event_at       TIMESTAMPTZ NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_patient_practice ON rm_patient(practice_id);
CREATE INDEX idx_rm_patient_name ON rm_patient(practice_id, last_name, first_name);
```

### Appointment Projection

```sql
CREATE TABLE rm_appointment (
    id                  UUID PRIMARY KEY,
    practice_id         UUID NOT NULL,
    patient_id          UUID NOT NULL,
    practitioner_id     UUID NOT NULL,
    location_id         UUID NOT NULL,
    appointment_type    VARCHAR(100),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'booked',
    reason_text         TEXT,
    telehealth_url      VARCHAR(500),
    no_show_probability NUMERIC(5,4),
    source              VARCHAR(20),
    cancellation_reason VARCHAR(255),
    status_history      JSONB NOT NULL DEFAULT '[]',
        -- [{status: "booked", at: "...", by: "..."}, {status: "arrived", at: "...", by: "..."}]
    last_event_version  INTEGER NOT NULL,
    last_event_at       TIMESTAMPTZ NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_appt_practice ON rm_appointment(practice_id);
CREATE INDEX idx_rm_appt_patient ON rm_appointment(patient_id);
CREATE INDEX idx_rm_appt_practitioner_time ON rm_appointment(practitioner_id, start_time);
CREATE INDEX idx_rm_appt_date ON rm_appointment(practice_id, start_time);
CREATE INDEX idx_rm_appt_status ON rm_appointment(practice_id, status);
```

### Claim Lifecycle Projection

```sql
CREATE TABLE rm_claim (
    id                      UUID PRIMARY KEY,
    practice_id             UUID NOT NULL,
    patient_id              UUID NOT NULL,
    payer_id                UUID NOT NULL,
    claim_number            VARCHAR(50),
    payer_claim_number      VARCHAR(50),
    claim_type              VARCHAR(20) NOT NULL,
    total_charge            NUMERIC(10,2) NOT NULL,
    total_paid              NUMERIC(10,2) DEFAULT 0,
    total_adjusted          NUMERIC(10,2) DEFAULT 0,
    patient_responsibility  NUMERIC(10,2) DEFAULT 0,
    status                  VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at            TIMESTAMPTZ,
    first_response_at       TIMESTAMPTZ,
    days_to_payment         INTEGER,               -- Calculated: paid_at - submitted_at
    denial_codes            VARCHAR(10)[],
    appeal_count            INTEGER DEFAULT 0,
    current_payer_response  JSONB,                 -- Latest 835/277 response summary
    status_history          JSONB NOT NULL DEFAULT '[]',
        -- Full claim lifecycle as array of {status, at, by, details}
    last_event_version      INTEGER NOT NULL,
    last_event_at           TIMESTAMPTZ NOT NULL,
    projection_updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_claim_practice ON rm_claim(practice_id);
CREATE INDEX idx_rm_claim_status ON rm_claim(practice_id, status);
CREATE INDEX idx_rm_claim_patient ON rm_claim(patient_id);
CREATE INDEX idx_rm_claim_payer ON rm_claim(payer_id, status);

CREATE TABLE rm_claim_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES rm_claim(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    cpt_code        VARCHAR(10) NOT NULL,
    modifiers       VARCHAR(5)[],
    icd_codes       VARCHAR(10)[],
    units           NUMERIC(5,2) NOT NULL,
    charge_amount   NUMERIC(10,2) NOT NULL,
    paid_amount     NUMERIC(10,2) DEFAULT 0,
    adjusted_amount NUMERIC(10,2) DEFAULT 0,
    adjustment_codes VARCHAR(10)[],
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_claim_line_claim ON rm_claim_line(claim_id);
```

### Coverage Projection

```sql
CREATE TABLE rm_patient_coverage (
    id                  UUID PRIMARY KEY,
    patient_id          UUID NOT NULL,
    practice_id         UUID NOT NULL,
    payer_id            UUID NOT NULL,
    payer_name          VARCHAR(255),
    coverage_order      SMALLINT NOT NULL,
    subscriber_id       VARCHAR(100),
    group_number        VARCHAR(50),
    plan_name           VARCHAR(255),
    effective_date      DATE,
    termination_date    DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    last_eligibility_check JSONB,          -- Latest 271 response summary
    last_checked_at     TIMESTAMPTZ,
    last_event_version  INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_coverage_patient ON rm_patient_coverage(patient_id);
```

### Encounter & Clinical Projection

```sql
CREATE TABLE rm_encounter (
    id                  UUID PRIMARY KEY,
    practice_id         UUID NOT NULL,
    patient_id          UUID NOT NULL,
    practitioner_id     UUID NOT NULL,
    appointment_id      UUID,
    location_id         UUID NOT NULL,
    encounter_date      DATE NOT NULL,
    status              VARCHAR(20) NOT NULL,
    class_code          VARCHAR(20),
    reason_text         TEXT,
    diagnoses           JSONB NOT NULL DEFAULT '[]',
        -- [{rank: 1, code: "J06.9", description: "Acute upper respiratory infection"}, ...]
    charges             JSONB NOT NULL DEFAULT '[]',
        -- [{cpt: "99213", units: 1, amount: 150.00, status: "submitted"}, ...]
    notes_count         INTEGER DEFAULT 0,
    has_unsigned_notes  BOOLEAN DEFAULT false,
    last_event_version  INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_encounter_practice ON rm_encounter(practice_id);
CREATE INDEX idx_rm_encounter_patient ON rm_encounter(patient_id);
CREATE INDEX idx_rm_encounter_date ON rm_encounter(practice_id, encounter_date);
```

### Revenue Cycle Analytics Projection

```sql
-- Denormalized projection optimized for revenue cycle dashboards and AI analytics
CREATE TABLE rm_revenue_cycle (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL,
    claim_id            UUID NOT NULL,
    patient_id          UUID NOT NULL,
    payer_id            UUID NOT NULL,
    practitioner_id     UUID NOT NULL,
    service_date        DATE NOT NULL,
    cpt_code            VARCHAR(10) NOT NULL,
    icd_primary         VARCHAR(10),
    charge_amount       NUMERIC(10,2),
    paid_amount         NUMERIC(10,2),
    adjusted_amount     NUMERIC(10,2),
    days_in_ar          INTEGER,               -- Days in accounts receivable
    denial_code         VARCHAR(10),
    denial_category     VARCHAR(50),            -- Categorized: coding, auth, eligibility, timely_filing, etc.
    was_denied          BOOLEAN DEFAULT false,
    was_appealed        BOOLEAN DEFAULT false,
    appeal_successful   BOOLEAN,
    submission_month    DATE,                   -- Truncated to month for aggregation
    payment_month       DATE,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_revcycle_practice ON rm_revenue_cycle(practice_id, service_date);
CREATE INDEX idx_rm_revcycle_payer ON rm_revenue_cycle(payer_id, denial_code);
CREATE INDEX idx_rm_revcycle_denial ON rm_revenue_cycle(practice_id, was_denied, denial_category);
```

## Projection Management

```sql
-- Tracks projection rebuild status
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,                   -- Last processed event_store.event_id
    last_event_at   TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, rebuilding, failed
    error_message   TEXT,
    events_processed BIGINT DEFAULT 0,
    rebuild_started_at TIMESTAMPTZ,
    rebuild_completed_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Subscription registry for event-driven projection updates
CREATE TABLE event_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_name VARCHAR(100) NOT NULL UNIQUE,  -- 'rm_patient_projector', 'rm_claim_projector', etc.
    event_types     VARCHAR(100)[] NOT NULL,         -- Which event types this subscriber handles
    last_position   BIGINT DEFAULT 0,               -- Position in the global event stream
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Multi-Tenancy & Access Control (Read Models)

```sql
CREATE TABLE rm_practice (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10),
    status          VARCHAR(20) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    last_event_version INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_practitioner (
    id              UUID PRIMARY KEY,
    practice_id     UUID NOT NULL,
    user_id         UUID,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),
    npi             VARCHAR(10),
    specialty_code  VARCHAR(20),
    specialty_name  VARCHAR(255),
    status          VARCHAR(20) NOT NULL,
    last_event_version INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_practitioner_practice ON rm_practitioner(practice_id);

CREATE TABLE rm_location (
    id              UUID PRIMARY KEY,
    practice_id     UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(10),
    status          VARCHAR(20) NOT NULL,
    last_event_version INTEGER NOT NULL,
    projection_updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Temporal Query Examples

```sql
-- Reconstruct a patient's state as of a specific date
-- "What was this patient's address on 2026-01-15?"
SELECT payload
FROM event_store
WHERE stream_id = 'patient-uuid-here'
  AND stream_type = 'Patient'
  AND event_type IN ('PatientRegistered', 'PatientAddressUpdated')
  AND created_at <= '2026-01-15T23:59:59Z'
ORDER BY event_version ASC;

-- Reconstruct a claim's full lifecycle
SELECT event_type, payload, actor_id, created_at
FROM event_store
WHERE stream_id = 'claim-uuid-here'
  AND stream_type = 'Claim'
ORDER BY event_version ASC;

-- Find all denial events for a specific payer in the last 90 days
-- (used by AI denial prediction model)
SELECT payload->>'denial_code' AS denial_code,
       payload->>'cpt_code' AS cpt_code,
       COUNT(*) AS denial_count
FROM event_store
WHERE practice_id = 'practice-uuid'
  AND event_type = 'ClaimDenied'
  AND created_at >= now() - INTERVAL '90 days'
GROUP BY payload->>'denial_code', payload->>'cpt_code'
ORDER BY denial_count DESC;

-- Audit query: all PHI access by a specific user in the last 30 days
SELECT event_type, stream_type, stream_id, created_at, actor_ip
FROM event_store
WHERE actor_id = 'user-uuid'
  AND phi_involved = true
  AND created_at >= now() - INTERVAL '30 days'
ORDER BY created_at DESC;
```

## Reference Data (Shared, Not Event-Sourced)

```sql
-- Reference data is NOT event-sourced. It is loaded and updated via migrations.
CREATE TABLE ref_icd10_code (
    code            VARCHAR(10) PRIMARY KEY,
    description     TEXT NOT NULL,
    is_billable     BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE ref_cpt_code (
    code            VARCHAR(10) PRIMARY KEY,
    description     TEXT NOT NULL,
    rvu_work        NUMERIC(6,2)
);

CREATE TABLE ref_payer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id        VARCHAR(50),
    electronic_id   VARCHAR(50),
    type            VARCHAR(50),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
);

CREATE TABLE ref_place_of_service (
    code            VARCHAR(5) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 2 | event_store (partitioned), event_type_registry |
| Command Processing | 1 | command_log |
| Patient Projection | 1 | rm_patient |
| Appointment Projection | 1 | rm_appointment |
| Claim Projections | 2 | rm_claim, rm_claim_line |
| Coverage Projection | 1 | rm_patient_coverage |
| Clinical Projection | 1 | rm_encounter |
| Revenue Analytics Projection | 1 | rm_revenue_cycle |
| Infrastructure Projections | 3 | rm_practice, rm_practitioner, rm_location |
| Projection Management | 2 | projection_checkpoint, event_subscription |
| Reference Data | 4 | ref_icd10_code, ref_cpt_code, ref_payer, ref_place_of_service |
| **Total** | **19** | Plus event_store partitions |

---

## Key Design Decisions

1. **Single event store table with JSONB payloads.** All events across all aggregate types live in one partitioned table. The `stream_type` column distinguishes Patient events from Claim events. This simplifies infrastructure (one table to backup, replicate, and monitor) while JSONB payloads provide type-specific flexibility.

2. **Optimistic concurrency via `(stream_id, event_version)` unique constraint.** Two concurrent commands targeting the same aggregate will conflict at the database level, preventing lost updates without distributed locks. The application retries the losing command.

3. **Read models are prefixed `rm_` and are disposable.** Any `rm_*` table can be dropped and rebuilt from the event store. This makes schema evolution for read models trivial — add a new projection, replay events, done.

4. **`status_history` JSONB arrays on projections.** Rather than a separate state transition table, appointment and claim projections embed the full status history as a JSONB array. This gives the API layer the complete lifecycle in a single row read.

5. **Revenue cycle analytics as a dedicated projection.** The `rm_revenue_cycle` table is a denormalized, analytics-optimized projection designed for dashboard queries and AI model training. It flattens claim/line/payer/practitioner data into one row per service line — no joins needed for denial rate by payer or collection rate by provider.

6. **PHI flag on events.** The `phi_involved` boolean on each event enables efficient HIPAA audit queries ("show me all PHI access by user X") without scanning every event's payload.

7. **Event partitioning by month.** The event store is partitioned by `created_at` to maintain write and query performance. Old partitions can be moved to cold storage while remaining queryable. HIPAA's 6-year retention maps to 72 monthly partitions.

8. **Commands are logged but not the source of truth.** The `command_log` table records every attempted command for debugging, but the event store is authoritative. This enables diagnosis of rejected commands and failed validations.

9. **Projection checkpointing for exactly-once processing.** Each projection tracks its last processed event ID, enabling crash recovery without reprocessing the entire stream. Projections can be rebuilt on demand by resetting the checkpoint.

10. **Fewer tables overall (~19 vs ~37 in normalized model).** Event sourcing consolidates write-side complexity into the event store and shifts complexity to projection logic in application code. The database schema is simpler; the application code is more complex.
