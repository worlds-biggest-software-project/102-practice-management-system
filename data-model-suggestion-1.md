# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Practice Management System · Created: 2026-05-19

## Philosophy

This model follows a traditional normalized relational design where every domain concept receives its own table with explicit foreign key relationships. The schema is directly inspired by the HL7 FHIR R4 resource model — tables like `appointment`, `schedule`, `slot`, `patient`, `practitioner`, `claim`, and `coverage` map closely to their FHIR counterparts. This alignment simplifies FHIR API implementation because the internal data model and the external API model share the same conceptual structure.

Normalization to 3NF (Third Normal Form) ensures data integrity: a patient's address is stored once, a payer's rules are stored once, and updates propagate without anomalies. Junction tables handle many-to-many relationships (e.g., a patient can have multiple insurance coverages; a practitioner can belong to multiple locations). Reference data tables for ICD-10-CM codes, CPT codes, payer identifiers, and jurisdiction codes enforce consistency through foreign keys.

This approach is proven in healthcare — OpenEMR uses a similar normalized structure with over 300 tables. The trade-off is query complexity: retrieving a complete patient encounter with appointment, notes, charges, and claim status requires multi-table joins. However, modern PostgreSQL handles this well with proper indexing.

**Best for:** Teams prioritizing data integrity, regulatory compliance, and long-term maintainability over rapid prototyping speed.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned records and data inconsistencies
- (+) Direct mapping to FHIR R4 resources simplifies API layer implementation
- (+) Well-understood by database engineers; extensive tooling support
- (+) Schema changes are explicit and trackable via migrations
- (-) Higher table count (~60-80 tables) increases migration complexity
- (-) Multi-table joins for common read operations can be slower without careful indexing
- (-) Adding jurisdiction-specific or specialty-specific fields requires schema migrations
- (-) Rigid structure makes rapid prototyping slower than document or JSONB approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Table structure mirrors FHIR resources: Patient, Practitioner, Appointment, Schedule, Slot, Claim, Coverage, ExplanationOfBenefit, Encounter |
| X12 EDI 837/835 | Claim and remittance tables store fields needed to generate/parse X12 transactions (subscriber ID, service lines, adjustment reason codes) |
| X12 EDI 270/271 | Coverage and eligibility tables capture verification results with payer response codes |
| ICD-10-CM | Diagnosis code reference table with WHO/CMS code set |
| CPT (AMA) | Procedure code reference table (requires AMA licence) |
| HIPAA Security Rule | Audit log table with user, action, resource, timestamp, IP for all PHI access |
| ISO 3166-1/2 | Jurisdiction codes for multi-region support |
| SMART on FHIR | OAuth scope definitions stored in app_registration and oauth_scope tables |

---

## Core Identity & Multi-Tenancy

```sql
-- Each practice is a tenant. Row-level security filters all queries by practice_id.
CREATE TABLE practice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10),          -- National Provider Identifier (10-digit)
    tax_id          VARCHAR(20),          -- EIN or SSN for billing
    billing_address JSONB,                -- {street, city, state, zip, country}
    phone           VARCHAR(20),
    email           VARCHAR(255),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, suspended, closed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_code      VARCHAR(10),          -- ISO 3166-2 subdivision code
    postal_code     VARCHAR(20),
    country_code    CHAR(2) DEFAULT 'US', -- ISO 3166-1 alpha-2
    phone           VARCHAR(20),
    timezone        VARCHAR(50),
    is_billing_location BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_location_practice ON location(practice_id);
```

## User & Access Control

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, locked, deactivated
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(practice_id, email)
);
CREATE INDEX idx_app_user_practice ON app_user(practice_id);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(100) NOT NULL,  -- admin, billing, clinical, front_desk, provider
    description     TEXT,
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- e.g., 'patient.read', 'claim.submit'
    description     TEXT,
    resource_type   VARCHAR(50) NOT NULL            -- maps to FHIR resource or internal resource
);

CREATE TABLE role_permission (
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permission(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES role(id) ON DELETE CASCADE,
    location_id     UUID REFERENCES location(id),  -- NULL = all locations
    PRIMARY KEY (user_id, role_id, location_id)
);
```

## Patient Management

```sql
CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    mrn                 VARCHAR(50),              -- Medical Record Number (practice-assigned)
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    middle_name         VARCHAR(100),
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),              -- FHIR AdministrativeGender: male, female, other, unknown
    sex_at_birth        VARCHAR(20),
    email               VARCHAR(255),
    phone_home          VARCHAR(20),
    phone_mobile        VARCHAR(20),
    phone_work          VARCHAR(20),
    preferred_contact   VARCHAR(20) DEFAULT 'phone_mobile',
    preferred_language  VARCHAR(10) DEFAULT 'en', -- BCP 47 language tag
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state_code          VARCHAR(10),
    postal_code         VARCHAR(20),
    country_code        CHAR(2) DEFAULT 'US',
    ssn_last4           VARCHAR(4),               -- Last 4 digits only; encrypted at rest
    emergency_contact_name  VARCHAR(200),
    emergency_contact_phone VARCHAR(20),
    status              VARCHAR(20) NOT NULL DEFAULT 'active', -- active, inactive, deceased
    deceased_date       DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_patient_practice ON patient(practice_id);
CREATE INDEX idx_patient_name ON patient(practice_id, last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(practice_id, date_of_birth);
CREATE INDEX idx_patient_mrn ON patient(practice_id, mrn);

CREATE TABLE patient_identifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(id) ON DELETE CASCADE,
    system          VARCHAR(255) NOT NULL,  -- e.g., 'http://hl7.org/fhir/sid/us-ssn', practice MRN URI
    value           VARCHAR(255) NOT NULL,
    type_code       VARCHAR(50),            -- MR, SS, DL, etc. (FHIR IdentifierType)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_patient_identifier_patient ON patient_identifier(patient_id);
```

## Insurance & Coverage

```sql
CREATE TABLE payer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id        VARCHAR(50),           -- Clearinghouse payer ID
    type            VARCHAR(50),           -- commercial, medicare, medicaid, tricare, workers_comp
    address         JSONB,
    phone           VARCHAR(20),
    electronic_id   VARCHAR(50),           -- X12 payer identifier
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patient_coverage (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    coverage_order      SMALLINT NOT NULL DEFAULT 1,  -- 1=primary, 2=secondary, 3=tertiary
    subscriber_id       VARCHAR(100) NOT NULL,         -- Member/subscriber ID on the insurance card
    group_number        VARCHAR(50),
    plan_name           VARCHAR(255),
    relationship_to_subscriber VARCHAR(20),            -- self, spouse, child, other
    subscriber_first_name      VARCHAR(100),
    subscriber_last_name       VARCHAR(100),
    subscriber_dob             DATE,
    effective_date      DATE NOT NULL,
    termination_date    DATE,
    copay_amount        NUMERIC(10,2),
    deductible_amount   NUMERIC(10,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_coverage_patient ON patient_coverage(patient_id);
CREATE INDEX idx_coverage_payer ON patient_coverage(payer_id);

CREATE TABLE eligibility_check (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_coverage_id UUID NOT NULL REFERENCES patient_coverage(id),
    checked_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    checked_by          UUID REFERENCES app_user(id),
    request_type        VARCHAR(20) NOT NULL,   -- realtime, batch
    status              VARCHAR(20) NOT NULL,    -- active, inactive, unknown, error
    response_code       VARCHAR(10),             -- X12 271 response code
    benefits_summary    JSONB,                   -- Parsed 271 benefit details
    raw_response        TEXT,                    -- Raw X12 271 for audit
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_eligibility_coverage ON eligibility_check(patient_coverage_id);
```

## Scheduling

```sql
CREATE TABLE practitioner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    user_id         UUID REFERENCES app_user(id),
    npi             VARCHAR(10),            -- Individual NPI
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),            -- MD, DO, NP, PA, LCSW, etc.
    specialty_code  VARCHAR(20),            -- NUCC taxonomy code
    specialty_name  VARCHAR(255),
    license_number  VARCHAR(50),
    license_state   VARCHAR(10),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_practitioner_practice ON practitioner(practice_id);

CREATE TABLE schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practitioner_id UUID NOT NULL REFERENCES practitioner(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    day_of_week     SMALLINT NOT NULL,      -- 0=Sunday, 6=Saturday (ISO 8601: 1=Monday)
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    slot_duration   INTERVAL NOT NULL DEFAULT '30 minutes',
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_practitioner ON schedule(practitioner_id);

CREATE TABLE slot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES schedule(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'free',  -- free, busy, busy-unavailable, busy-tentative (FHIR SlotStatus)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_slot_schedule ON slot(schedule_id);
CREATE INDEX idx_slot_start ON slot(start_time);
CREATE INDEX idx_slot_status ON slot(status) WHERE status = 'free';

CREATE TABLE appointment_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(100) NOT NULL,  -- 'New Patient', 'Follow-Up', 'Annual Physical', 'Telehealth'
    duration         INTERVAL NOT NULL DEFAULT '30 minutes',
    color           VARCHAR(7),             -- Hex color for calendar display
    default_cpt     VARCHAR(10),            -- Default CPT code for this type
    is_telehealth   BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID NOT NULL REFERENCES practitioner(id),
    location_id         UUID NOT NULL REFERENCES location(id),
    appointment_type_id UUID REFERENCES appointment_type(id),
    slot_id             UUID REFERENCES slot(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'booked',  -- proposed, booked, arrived, checked-in, fulfilled, cancelled, noshow (FHIR)
    cancellation_reason VARCHAR(255),
    reason_code         VARCHAR(20),        -- ICD-10-CM chief complaint code
    reason_text         TEXT,               -- Free-text reason for visit
    priority            SMALLINT DEFAULT 0,
    instructions        TEXT,               -- Pre-appointment instructions for patient
    telehealth_url      VARCHAR(500),
    no_show_probability NUMERIC(5,4),       -- AI-predicted no-show probability (0.0000–1.0000)
    source              VARCHAR(20) DEFAULT 'manual',  -- manual, online, phone, ai_agent
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appointment_practice ON appointment(practice_id);
CREATE INDEX idx_appointment_patient ON appointment(patient_id);
CREATE INDEX idx_appointment_practitioner_time ON appointment(practitioner_id, start_time);
CREATE INDEX idx_appointment_status ON appointment(practice_id, status);
CREATE INDEX idx_appointment_date ON appointment(practice_id, start_time);

CREATE TABLE waitlist_entry (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    appointment_type_id UUID REFERENCES appointment_type(id),
    preferred_days      SMALLINT[],         -- Array of preferred days of week
    preferred_time_start TIME,
    preferred_time_end   TIME,
    priority            SMALLINT DEFAULT 0,
    status              VARCHAR(20) NOT NULL DEFAULT 'waiting',  -- waiting, offered, booked, expired, cancelled
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_waitlist_practice ON waitlist_entry(practice_id, status);

CREATE TABLE appointment_reminder (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    appointment_id  UUID NOT NULL REFERENCES appointment(id) ON DELETE CASCADE,
    channel         VARCHAR(20) NOT NULL,   -- sms, email, voice
    scheduled_at    TIMESTAMPTZ NOT NULL,
    sent_at         TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, sent, delivered, failed
    response        VARCHAR(20),            -- confirmed, cancelled, no_response
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_reminder_appointment ON appointment_reminder(appointment_id);
```

## Clinical Documentation

```sql
CREATE TABLE encounter (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID NOT NULL REFERENCES practitioner(id),
    appointment_id      UUID REFERENCES appointment(id),
    location_id         UUID NOT NULL REFERENCES location(id),
    encounter_date      DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'in-progress', -- planned, in-progress, completed, cancelled (FHIR)
    class_code          VARCHAR(20) NOT NULL DEFAULT 'AMB',  -- AMB (ambulatory), VR (virtual), HH (home health)
    reason_code         VARCHAR(20),
    reason_text         TEXT,
    discharge_disposition VARCHAR(20),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_encounter_practice ON encounter(practice_id);
CREATE INDEX idx_encounter_patient ON encounter(patient_id);
CREATE INDEX idx_encounter_date ON encounter(practice_id, encounter_date);

CREATE TABLE clinical_note (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounter(id),
    author_id       UUID NOT NULL REFERENCES app_user(id),
    note_type       VARCHAR(50) NOT NULL,   -- progress_note, soap_note, initial_assessment, treatment_plan, discharge_summary
    content         TEXT NOT NULL,           -- Structured or free-text note content
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, final, amended, entered-in-error
    signed_by       UUID REFERENCES app_user(id),
    signed_at       TIMESTAMPTZ,
    cosigned_by     UUID REFERENCES app_user(id),  -- Supervisor co-signature
    cosigned_at     TIMESTAMPTZ,
    ai_generated    BOOLEAN NOT NULL DEFAULT false,  -- Flag for AI ambient documentation
    ai_confidence   NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_note_encounter ON clinical_note(encounter_id);

CREATE TABLE encounter_diagnosis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounter(id) ON DELETE CASCADE,
    icd10_code      VARCHAR(10) NOT NULL,   -- ICD-10-CM code (e.g., 'J06.9')
    description     TEXT,
    rank            SMALLINT NOT NULL DEFAULT 1,  -- 1=primary, 2+=secondary
    type            VARCHAR(20) DEFAULT 'encounter-diagnosis',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_enc_diag_encounter ON encounter_diagnosis(encounter_id);
```

## Billing & Claims

```sql
CREATE TABLE charge (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id        UUID NOT NULL REFERENCES encounter(id),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID NOT NULL REFERENCES practitioner(id),
    cpt_code            VARCHAR(10) NOT NULL,     -- CPT/HCPCS code
    modifier1           VARCHAR(5),               -- CPT modifier (e.g., '25', '59')
    modifier2           VARCHAR(5),
    modifier3           VARCHAR(5),
    modifier4           VARCHAR(5),
    icd_pointer         SMALLINT[] NOT NULL,       -- References encounter_diagnosis.rank
    units               NUMERIC(5,2) NOT NULL DEFAULT 1,
    charge_amount       NUMERIC(10,2) NOT NULL,
    place_of_service    VARCHAR(5) NOT NULL DEFAULT '11',  -- CMS Place of Service code
    service_date        DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, submitted, paid, denied, adjusted
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_charge_encounter ON charge(encounter_id);
CREATE INDEX idx_charge_practice_date ON charge(practice_id, service_date);

CREATE TABLE claim (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    patient_coverage_id UUID NOT NULL REFERENCES patient_coverage(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    claim_number        VARCHAR(50),               -- Practice-assigned claim number
    payer_claim_number  VARCHAR(50),               -- Payer-assigned claim number (from 277)
    claim_type          VARCHAR(20) NOT NULL,       -- professional (837P), institutional (837I)
    total_charge        NUMERIC(10,2) NOT NULL,
    total_paid          NUMERIC(10,2) DEFAULT 0,
    total_adjusted      NUMERIC(10,2) DEFAULT 0,
    patient_responsibility NUMERIC(10,2) DEFAULT 0,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, ready, submitted, accepted, rejected, denied, partially_paid, paid, appealed
    submitted_at        TIMESTAMPTZ,
    submitted_via       VARCHAR(20),               -- clearinghouse, direct, paper
    clearinghouse_id    VARCHAR(50),               -- Clearinghouse tracking number
    billing_provider_npi VARCHAR(10),
    rendering_provider_npi VARCHAR(10),
    place_of_service    VARCHAR(5),
    frequency_code      VARCHAR(2) DEFAULT '1',    -- 1=original, 7=replacement, 8=void
    prior_auth_number   VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_claim_practice ON claim(practice_id);
CREATE INDEX idx_claim_patient ON claim(patient_id);
CREATE INDEX idx_claim_status ON claim(practice_id, status);
CREATE INDEX idx_claim_submitted ON claim(practice_id, submitted_at);

CREATE TABLE claim_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim(id) ON DELETE CASCADE,
    charge_id       UUID NOT NULL REFERENCES charge(id),
    line_number     SMALLINT NOT NULL,
    cpt_code        VARCHAR(10) NOT NULL,
    modifier1       VARCHAR(5),
    modifier2       VARCHAR(5),
    icd_codes       VARCHAR(10)[] NOT NULL,  -- Array of ICD-10 codes for this line
    units           NUMERIC(5,2) NOT NULL,
    charge_amount   NUMERIC(10,2) NOT NULL,
    paid_amount     NUMERIC(10,2) DEFAULT 0,
    adjusted_amount NUMERIC(10,2) DEFAULT 0,
    adjustment_reason_codes VARCHAR(10)[],    -- CARC codes from 835
    remark_codes    VARCHAR(10)[],            -- RARC codes from 835
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_claim_line_claim ON claim_line(claim_id);

CREATE TABLE remittance (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    check_number        VARCHAR(50),
    check_date          DATE,
    total_paid          NUMERIC(10,2) NOT NULL,
    payment_method      VARCHAR(20),               -- eft, check, virtual_card
    eft_trace_number    VARCHAR(50),
    raw_835             TEXT,                       -- Raw X12 835 for audit
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    posted_at           TIMESTAMPTZ,
    posted_by           UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_remittance_practice ON remittance(practice_id);

CREATE TABLE remittance_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    remittance_id   UUID NOT NULL REFERENCES remittance(id) ON DELETE CASCADE,
    claim_line_id   UUID REFERENCES claim_line(id),
    claim_id        UUID REFERENCES claim(id),
    paid_amount     NUMERIC(10,2) NOT NULL,
    adjusted_amount NUMERIC(10,2) DEFAULT 0,
    adjustment_reason_codes VARCHAR(10)[],
    remark_codes    VARCHAR(10)[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Patient Communication & Intake

```sql
CREATE TABLE intake_form_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(255) NOT NULL,
    form_type       VARCHAR(50) NOT NULL,   -- demographics, medical_history, consent, custom
    schema          JSONB NOT NULL,          -- JSON Schema defining form fields
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE intake_form_submission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES intake_form_template(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    appointment_id  UUID REFERENCES appointment(id),
    responses       JSONB NOT NULL,          -- Patient responses matching template schema
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted',  -- submitted, reviewed, imported
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ
);
CREATE INDEX idx_intake_patient ON intake_form_submission(patient_id);

CREATE TABLE patient_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    sender_type     VARCHAR(20) NOT NULL,   -- patient, staff, system, ai_agent
    sender_id       UUID,                   -- app_user.id or NULL for system/AI
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    channel         VARCHAR(20) NOT NULL,   -- portal, sms, email
    direction       VARCHAR(10) NOT NULL,   -- inbound, outbound
    is_read         BOOLEAN NOT NULL DEFAULT false,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_message_patient ON patient_message(patient_id);
CREATE INDEX idx_message_practice_unread ON patient_message(practice_id) WHERE NOT is_read;
```

## Prior Authorization

```sql
CREATE TABLE prior_authorization (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    patient_coverage_id UUID NOT NULL REFERENCES patient_coverage(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    practitioner_id     UUID REFERENCES practitioner(id),
    auth_number         VARCHAR(50),             -- Payer-assigned authorization number
    cpt_codes           VARCHAR(10)[] NOT NULL,  -- Procedures requiring authorization
    icd10_codes         VARCHAR(10)[],           -- Supporting diagnoses
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
        -- pending, submitted, approved, denied, partially_approved, expired, cancelled
    requested_units     INTEGER,
    approved_units      INTEGER,
    effective_date      DATE,
    expiration_date     DATE,
    submission_method   VARCHAR(20),             -- x12_278, fhir_pas, phone, fax, portal
    ai_initiated        BOOLEAN NOT NULL DEFAULT false,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prior_auth_patient ON prior_authorization(patient_id);
CREATE INDEX idx_prior_auth_status ON prior_authorization(practice_id, status);
```

## AI & Analytics

```sql
CREATE TABLE ai_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    prediction_type VARCHAR(50) NOT NULL,    -- no_show, denial_risk, undercoding, pa_required
    entity_type     VARCHAR(50) NOT NULL,    -- appointment, claim, encounter, charge
    entity_id       UUID NOT NULL,
    probability     NUMERIC(5,4),            -- 0.0000–1.0000
    model_version   VARCHAR(50),
    features        JSONB,                   -- Input features used for prediction
    outcome         VARCHAR(20),             -- actual outcome for model feedback (correct, incorrect)
    outcome_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_prediction_entity ON ai_prediction(entity_type, entity_id);
CREATE INDEX idx_ai_prediction_type ON ai_prediction(practice_id, prediction_type);

CREATE TABLE claim_scrub_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    claim_id        UUID NOT NULL REFERENCES claim(id),
    rule_source     VARCHAR(50) NOT NULL,    -- ncci_edit, payer_specific, ai_model, lcd_ncd
    rule_code       VARCHAR(50),
    severity        VARCHAR(20) NOT NULL,    -- error, warning, info
    message         TEXT NOT NULL,
    auto_resolved   BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES app_user(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scrub_claim ON claim_scrub_result(claim_id);
```

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL,
    user_id         UUID,
    user_email      VARCHAR(255),
    action          VARCHAR(20) NOT NULL,    -- create, read, update, delete, login, logout, export
    resource_type   VARCHAR(50) NOT NULL,    -- patient, appointment, claim, clinical_note, etc.
    resource_id     UUID,
    ip_address      INET,
    user_agent      VARCHAR(500),
    old_values      JSONB,                   -- Previous state (for update/delete)
    new_values      JSONB,                   -- New state (for create/update)
    phi_accessed    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (example for one month; automate in production)
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_practice ON audit_log(practice_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_phi ON audit_log(practice_id, created_at) WHERE phi_accessed = true;
```

## Reference Data

```sql
CREATE TABLE icd10_code (
    code            VARCHAR(10) PRIMARY KEY,
    description     TEXT NOT NULL,
    category        VARCHAR(100),
    is_billable     BOOLEAN NOT NULL DEFAULT true,
    effective_date  DATE,
    termination_date DATE
);
CREATE INDEX idx_icd10_description ON icd10_code USING gin(to_tsvector('english', description));

CREATE TABLE cpt_code (
    code            VARCHAR(10) PRIMARY KEY,
    description     TEXT NOT NULL,
    category        VARCHAR(100),
    rvu_work        NUMERIC(6,2),           -- Work Relative Value Unit
    rvu_pe          NUMERIC(6,2),           -- Practice Expense RVU
    rvu_mp          NUMERIC(6,2),           -- Malpractice RVU
    effective_date  DATE,
    termination_date DATE
);
CREATE INDEX idx_cpt_description ON cpt_code USING gin(to_tsvector('english', description));

CREATE TABLE place_of_service (
    code            VARCHAR(5) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    description     TEXT
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | practice, location |
| User & Access Control | 5 | app_user, role, permission, role_permission, user_role |
| Patient Management | 2 | patient, patient_identifier |
| Insurance & Coverage | 3 | payer, patient_coverage, eligibility_check |
| Scheduling | 6 | practitioner, schedule, slot, appointment_type, appointment, waitlist_entry |
| Appointment Communication | 1 | appointment_reminder |
| Clinical Documentation | 3 | encounter, clinical_note, encounter_diagnosis |
| Billing & Claims | 5 | charge, claim, claim_line, remittance, remittance_line |
| Patient Communication & Intake | 3 | intake_form_template, intake_form_submission, patient_message |
| Prior Authorization | 1 | prior_authorization |
| AI & Analytics | 2 | ai_prediction, claim_scrub_result |
| Audit & Compliance | 1 | audit_log (partitioned) |
| Reference Data | 3 | icd10_code, cpt_code, place_of_service |
| **Total** | **37** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere.** Supports distributed ID generation, prevents enumeration attacks on PHI, and aligns with FHIR resource identifiers.

2. **FHIR-aligned status enumerations.** Appointment status values (proposed, booked, arrived, fulfilled, cancelled, noshow) and encounter class codes (AMB, VR) directly match FHIR value sets, minimizing mapping logic in the API layer.

3. **Separate `charge` and `claim_line` tables.** A charge captures what happened clinically; a claim line captures what was billed. This separation allows rebilling, secondary claim generation, and claim resubmission without modifying the original clinical charge.

4. **Partitioned audit log.** The `audit_log` table is partitioned by month to maintain query performance as the table grows (HIPAA requires 6-year retention). Monthly partitions enable efficient purging and archival.

5. **Row-level security via `practice_id`.** Every clinical and financial table includes `practice_id`. PostgreSQL Row-Level Security policies will filter all queries by the authenticated practice context, preventing cross-tenant data leakage.

6. **AI predictions as a separate entity.** Rather than embedding ML predictions inline in appointment or claim tables, a dedicated `ai_prediction` table stores all predictions with model version, input features, and outcome tracking — enabling model performance monitoring and A/B testing.

7. **Raw X12 storage for audit.** The `eligibility_check.raw_response` and `remittance.raw_835` fields store the original X12 transaction text. This enables dispute resolution and regulatory audit without requiring clearinghouse log retrieval.

8. **Intake forms use JSON Schema.** The `intake_form_template.schema` column stores a JSON Schema definition, and `intake_form_submission.responses` stores patient answers. This allows practices to create custom forms without schema migrations while maintaining validation.

9. **Practitioner modeled separately from app_user.** A practitioner may not have a system login (e.g., a referring provider), and a user may not be a practitioner (e.g., billing staff). The optional `user_id` FK links them when both apply.

10. **No-show probability stored on the appointment.** The `appointment.no_show_probability` field is updated by the AI prediction pipeline and is directly queryable for waitlist backfill logic without joining to the `ai_prediction` table.
