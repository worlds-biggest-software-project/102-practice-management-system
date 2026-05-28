# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Practice Management System · Created: 2026-05-19

## Philosophy

This model uses traditional relational tables for core entities that are well-defined and stable (patients, appointments, claims, users), but leverages PostgreSQL JSONB columns extensively for data that varies by specialty, jurisdiction, payer, or practice configuration. The guiding principle is: if a field exists on every record of that type across all practices and specialties, it gets a dedicated column with type constraints and indexes; if a field varies by context, it goes in a JSONB column.

This approach directly addresses one of the biggest challenges in practice management: a behavioral health practice in New York, a physiotherapy clinic in Ontario, and a multi-specialty group in Texas all need different intake forms, different documentation fields, different billing codes, and different regulatory data. A fully normalized schema would require either nullable columns for every possible field (leading to sparse, confusing tables) or a proliferation of specialty-specific tables. JSONB columns absorb this variability without schema migrations.

Jane App and Cliniko, two successful multi-specialty PMS products, use similar patterns — core relational structure with configurable fields stored as structured JSON. The trade-off is weaker referential integrity on JSONB data and the need for application-level validation using JSON Schema.

**Best for:** Teams targeting multi-specialty or multi-jurisdiction markets where data requirements vary significantly between practices, and where rapid iteration and MVP speed are priorities.

**Trade-offs:**
- (+) Accommodates specialty-specific and jurisdiction-specific fields without schema migrations
- (+) Faster MVP development: new practice types can be onboarded by adding JSON Schema configurations, not database migrations
- (+) Fewer tables than normalized model (~25 vs ~37), simpler to understand and maintain
- (+) JSONB GIN indexes enable efficient querying of variable fields
- (+) Practices can customize intake forms, note templates, and billing rules via configuration
- (-) JSONB data lacks foreign key constraints — referential integrity must be enforced in application code
- (-) Complex JSONB queries can be slower than equivalent relational queries without careful indexing
- (-) Data validation shifts from database constraints to application-layer JSON Schema validation
- (-) Reporting and analytics tools may not handle JSONB columns as well as flat relational columns
- (-) Type safety is weaker: a JSONB field can hold anything unless validated

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Core entity structure aligns with FHIR resources; FHIR extensions map naturally to JSONB `extensions` columns |
| JSON Schema (Draft 2020-12) | Used for validating JSONB columns at the application layer; form schemas, note templates, and payer rules stored as JSON Schema |
| X12 EDI 837/835 | Core claim fields are relational; payer-specific claim fields and raw EDI stored in JSONB |
| X12 EDI 270/271 | Eligibility responses vary significantly by payer; parsed response stored as JSONB |
| ICD-10-CM / CPT | Diagnosis and procedure codes in relational columns; code-specific metadata and modifiers in JSONB |
| HIPAA Security Rule | Audit trail with relational structure for queryable fields, JSONB for variable change details |
| ISO 3166-1/2 | Jurisdiction codes in relational columns; jurisdiction-specific regulatory fields in JSONB |
| SMART on FHIR | OAuth configuration stored in JSONB for flexible scope and launch context |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE practice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10),
    tax_id          VARCHAR(20),
    specialty_type  VARCHAR(50),            -- medical, behavioral_health, allied_health, dental, multi_specialty
    country_code    CHAR(2) DEFAULT 'US',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB for practice-specific configuration
    settings        JSONB NOT NULL DEFAULT '{}',
    /*  settings example:
    {
        "billing": {
            "default_clearinghouse": "availity",
            "auto_eligibility_check": true,
            "claim_scrub_enabled": true,
            "collection_percentage_pricing": false
        },
        "scheduling": {
            "default_slot_duration_minutes": 30,
            "online_booking_enabled": true,
            "no_show_prediction_enabled": true,
            "waitlist_auto_fill": true
        },
        "clinical": {
            "ambient_documentation": true,
            "supervisor_cosign_required": false,
            "default_note_template": "soap"
        },
        "locale": {
            "date_format": "MM/DD/YYYY",
            "currency": "USD",
            "provincial_billing": null
        },
        "branding": {
            "logo_url": "https://...",
            "primary_color": "#2563EB"
        }
    }
    */
    billing_address JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(255) NOT NULL,
    address         JSONB NOT NULL,
    /*  address example:
    {
        "line1": "456 Health Ave",
        "line2": "Suite 200",
        "city": "Austin",
        "state": "TX",
        "postal_code": "78701",
        "country": "US"
    }
    */
    phone           VARCHAR(20),
    timezone        VARCHAR(50),
    is_billing_location BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    metadata        JSONB NOT NULL DEFAULT '{}',  -- Location-specific fields (room count, equipment, etc.)
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
    roles           VARCHAR(50)[] NOT NULL DEFAULT '{front_desk}',
        -- Array of roles: admin, billing, clinical, front_desk, provider, supervisor
    permissions     JSONB NOT NULL DEFAULT '{}',
    /*  permissions example (overrides role defaults):
    {
        "patient.read": true,
        "patient.write": true,
        "claim.submit": false,
        "report.financial": true,
        "locations": ["loc-uuid-1", "loc-uuid-2"]  -- restrict to specific locations
    }
    */
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    preferences     JSONB NOT NULL DEFAULT '{}',  -- UI preferences, notification settings
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(practice_id, email)
);
CREATE INDEX idx_user_practice ON app_user(practice_id);
CREATE INDEX idx_user_roles ON app_user USING gin(roles);
```

## Patient Management

```sql
CREATE TABLE patient (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    mrn                 VARCHAR(50),
    -- Universal fields (every patient in every specialty has these)
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    gender              VARCHAR(20),
    email               VARCHAR(255),
    phone_mobile        VARCHAR(20),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Address as JSONB (format varies by country)
    address             JSONB,
    /*  US address:
    {"line1": "123 Main St", "city": "Springfield", "state": "IL", "postal_code": "62701", "country": "US"}

    Canadian address:
    {"line1": "456 Rue Principale", "city": "Montreal", "province": "QC", "postal_code": "H2X 1Y4", "country": "CA"}
    */
    -- Contact preferences
    contacts            JSONB NOT NULL DEFAULT '{}',
    /*  {
        "phone_home": "+12025551111",
        "phone_work": "+12025552222",
        "preferred_contact": "phone_mobile",
        "preferred_language": "en",
        "emergency": {"name": "John Doe", "phone": "+12025553333", "relationship": "spouse"}
    }  */
    -- Identifiers (SSN, driver's license, provincial health number, etc.)
    identifiers         JSONB NOT NULL DEFAULT '[]',
    /*  [
        {"system": "ssn", "value": "***-**-1234", "type": "SS"},
        {"system": "ohip", "value": "1234-567-890-AB", "type": "provincial_health"}
    ]  */
    -- Specialty-specific clinical data
    clinical_summary    JSONB NOT NULL DEFAULT '{}',
    /*  Behavioral health example:
    {
        "primary_therapist_id": "uuid",
        "therapy_modality": "CBT",
        "session_frequency": "weekly",
        "diagnosis_history": [{"code": "F33.1", "onset": "2024-01", "status": "active"}],
        "medications": [{"name": "Sertraline", "dose": "100mg", "frequency": "daily"}]
    }

    Allied health example:
    {
        "referring_physician": "Dr. Smith",
        "referral_number": "REF-2026-001",
        "treatment_area": "lower_back",
        "sessions_authorized": 12,
        "sessions_completed": 4
    }
    */
    tags                VARCHAR(50)[],   -- Practice-defined tags: ['VIP', 'spanish-speaking', 'worker-comp']
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_patient_practice ON patient(practice_id);
CREATE INDEX idx_patient_name ON patient(practice_id, last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(practice_id, date_of_birth);
CREATE INDEX idx_patient_mrn ON patient(practice_id, mrn);
CREATE INDEX idx_patient_tags ON patient USING gin(tags);
CREATE INDEX idx_patient_email ON patient(practice_id, email) WHERE email IS NOT NULL;
```

## Insurance & Coverage

```sql
CREATE TABLE payer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id        VARCHAR(50),
    electronic_id   VARCHAR(50),
    type            VARCHAR(50),
    country_code    CHAR(2) DEFAULT 'US',
    -- Payer-specific rules and configuration (varies wildly between payers)
    rules           JSONB NOT NULL DEFAULT '{}',
    /*  {
        "timely_filing_days": 90,
        "requires_prior_auth": ["99213", "99214"],
        "modifier_rules": {"25": "cannot combine with E/M on same DOS"},
        "submission_format": "x12_837p",
        "era_format": "x12_835",
        "portal_url": "https://provider.aetna.com"
    }  */
    contact         JSONB,  -- {phone, fax, address, website}
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE patient_coverage (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    coverage_order      SMALLINT NOT NULL DEFAULT 1,
    subscriber_id       VARCHAR(100) NOT NULL,
    group_number        VARCHAR(50),
    plan_name           VARCHAR(255),
    effective_date      DATE NOT NULL,
    termination_date    DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Subscriber details and plan specifics
    subscriber_info     JSONB NOT NULL DEFAULT '{}',
    /*  {
        "relationship": "self",
        "first_name": "Jane",
        "last_name": "Doe",
        "dob": "1985-03-15",
        "copay": {"office_visit": 25.00, "specialist": 50.00},
        "deductible": {"individual": 1500.00, "family": 3000.00, "met": 750.00},
        "coinsurance": 0.20,
        "out_of_pocket_max": 6000.00
    }  */
    -- Latest eligibility verification result
    last_eligibility    JSONB,
    /*  {
        "checked_at": "2026-05-19T08:30:00Z",
        "status": "active",
        "response_code": "AAA",
        "benefits": {...parsed 271 data...}
    }  */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_coverage_patient ON patient_coverage(patient_id);
CREATE INDEX idx_coverage_payer ON patient_coverage(payer_id);
```

## Scheduling

```sql
CREATE TABLE practitioner (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    user_id         UUID REFERENCES app_user(id),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    credentials     VARCHAR(50),
    npi             VARCHAR(10),
    specialty_code  VARCHAR(20),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Specialty/license details vary by jurisdiction
    professional    JSONB NOT NULL DEFAULT '{}',
    /*  {
        "specialty_name": "Family Medicine",
        "license_number": "MD-12345",
        "license_state": "TX",
        "license_expiry": "2027-12-31",
        "dea_number": "AB1234567",
        "taxonomy_code": "207Q00000X",
        "supervises": ["practitioner-uuid-1", "practitioner-uuid-2"]
    }  */
    -- Schedule preferences
    schedule_config JSONB NOT NULL DEFAULT '{}',
    /*  {
        "default_slot_minutes": 30,
        "new_patient_slot_minutes": 60,
        "max_daily_patients": 25,
        "buffer_minutes": 5,
        "telehealth_enabled": true,
        "color": "#4F46E5"
    }  */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_practitioner_practice ON practitioner(practice_id);

CREATE TABLE schedule_block (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practitioner_id UUID NOT NULL REFERENCES practitioner(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    day_of_week     SMALLINT,               -- 0-6; NULL for specific-date overrides
    specific_date   DATE,                   -- Non-null for date-specific schedule overrides
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    block_type      VARCHAR(20) NOT NULL DEFAULT 'available',  -- available, blocked, lunch, meeting
    slot_duration   INTERVAL NOT NULL DEFAULT '30 minutes',
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    recurrence      JSONB,                  -- For complex recurrence: {every: 2, unit: "weeks"}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_practitioner ON schedule_block(practitioner_id);
CREATE INDEX idx_schedule_date ON schedule_block(specific_date) WHERE specific_date IS NOT NULL;

CREATE TABLE appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID NOT NULL REFERENCES practitioner(id),
    location_id         UUID NOT NULL REFERENCES location(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'booked',
    appointment_type    VARCHAR(100),           -- Practice-defined type name
    reason_text         TEXT,
    source              VARCHAR(20) DEFAULT 'manual',
    -- Variable appointment data
    details             JSONB NOT NULL DEFAULT '{}',
    /*  {
        "telehealth_url": "https://...",
        "room": "Exam Room 3",
        "equipment_needed": ["ultrasound"],
        "pre_visit_instructions": "Fast for 12 hours",
        "no_show_probability": 0.12,
        "ai_prediction_id": "uuid",
        "intake_forms_sent": true,
        "intake_forms_completed": false,
        "cancellation_reason": null,
        "recurring": {"frequency": "weekly", "series_id": "uuid"}
    }  */
    -- Reminder tracking embedded
    reminders           JSONB NOT NULL DEFAULT '[]',
    /*  [
        {"channel": "sms", "scheduled_at": "...", "sent_at": "...", "status": "delivered", "response": "confirmed"},
        {"channel": "email", "scheduled_at": "...", "sent_at": "...", "status": "delivered"}
    ]  */
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appt_practice ON appointment(practice_id);
CREATE INDEX idx_appt_patient ON appointment(patient_id);
CREATE INDEX idx_appt_practitioner_time ON appointment(practitioner_id, start_time);
CREATE INDEX idx_appt_date ON appointment(practice_id, start_time);
CREATE INDEX idx_appt_status ON appointment(practice_id, status);

CREATE TABLE waitlist_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    practitioner_id UUID REFERENCES practitioner(id),
    appointment_type VARCHAR(100),
    preferences     JSONB NOT NULL DEFAULT '{}',
    /*  {
        "preferred_days": [1, 3, 5],
        "preferred_time_start": "09:00",
        "preferred_time_end": "14:00",
        "location_id": "uuid",
        "urgency": "routine"
    }  */
    status          VARCHAR(20) NOT NULL DEFAULT 'waiting',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_waitlist_practice ON waitlist_entry(practice_id, status);
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
    status              VARCHAR(20) NOT NULL DEFAULT 'in-progress',
    class_code          VARCHAR(20) NOT NULL DEFAULT 'AMB',
    -- Diagnoses embedded as JSONB array
    diagnoses           JSONB NOT NULL DEFAULT '[]',
    /*  [
        {"rank": 1, "code": "J06.9", "description": "Acute upper respiratory infection, unspecified", "type": "encounter-diagnosis"},
        {"rank": 2, "code": "R05.9", "description": "Cough, unspecified"}
    ]  */
    -- Charges/procedures embedded
    charges             JSONB NOT NULL DEFAULT '[]',
    /*  [
        {
            "charge_id": "uuid",
            "cpt_code": "99213",
            "modifiers": ["25"],
            "icd_pointers": [1, 2],
            "units": 1,
            "charge_amount": 150.00,
            "place_of_service": "11",
            "service_date": "2026-05-19",
            "status": "pending"
        }
    ]  */
    reason_text         TEXT,
    metadata            JSONB NOT NULL DEFAULT '{}',
    /*  {
        "discharge_disposition": null,
        "supervising_provider_id": "uuid",
        "referral_number": "REF-001",
        "workers_comp_case": false
    }  */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_encounter_practice ON encounter(practice_id);
CREATE INDEX idx_encounter_patient ON encounter(patient_id);
CREATE INDEX idx_encounter_date ON encounter(practice_id, encounter_date);
CREATE INDEX idx_encounter_diagnoses ON encounter USING gin(diagnoses);

CREATE TABLE clinical_note (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    encounter_id    UUID NOT NULL REFERENCES encounter(id),
    author_id       UUID NOT NULL REFERENCES app_user(id),
    note_type       VARCHAR(50) NOT NULL,
    -- Note content as structured JSONB (template-driven)
    content         JSONB NOT NULL,
    /*  SOAP note example:
    {
        "subjective": "Patient reports persistent cough for 5 days...",
        "objective": {
            "vitals": {"bp": "120/80", "hr": 72, "temp": 98.6, "spo2": 98},
            "exam": "Lungs: bilateral wheezes..."
        },
        "assessment": "Acute bronchitis",
        "plan": "Prescribed albuterol inhaler PRN..."
    }

    Behavioral health progress note example:
    {
        "presenting_issues": "Anxiety about work deadlines...",
        "interventions": ["cognitive restructuring", "mindfulness exercise"],
        "client_response": "Engaged and receptive...",
        "risk_assessment": {"suicidal_ideation": "none", "homicidal_ideation": "none", "self_harm": "none"},
        "treatment_plan_progress": "Good progress on goal 1...",
        "next_session_focus": "Continue exposure hierarchy"
    }
    */
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    signed_by       UUID REFERENCES app_user(id),
    signed_at       TIMESTAMPTZ,
    cosigned_by     UUID REFERENCES app_user(id),
    cosigned_at     TIMESTAMPTZ,
    ai_metadata     JSONB,   -- {generated: true, confidence: 0.92, model: "ambient-v3", reviewed: true}
    template_id     VARCHAR(100),            -- Reference to note template used
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_note_encounter ON clinical_note(encounter_id);
```

## Billing & Claims

```sql
CREATE TABLE claim (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    patient_coverage_id UUID NOT NULL REFERENCES patient_coverage(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    -- Core relational fields
    claim_number        VARCHAR(50),
    payer_claim_number  VARCHAR(50),
    claim_type          VARCHAR(20) NOT NULL,
    total_charge        NUMERIC(10,2) NOT NULL,
    total_paid          NUMERIC(10,2) DEFAULT 0,
    total_adjusted      NUMERIC(10,2) DEFAULT 0,
    patient_responsibility NUMERIC(10,2) DEFAULT 0,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at        TIMESTAMPTZ,
    -- Service lines as JSONB array (avoids separate claim_line table)
    lines               JSONB NOT NULL DEFAULT '[]',
    /*  [
        {
            "line_number": 1,
            "cpt_code": "99213",
            "modifiers": ["25"],
            "icd_codes": ["J06.9", "R05.9"],
            "units": 1,
            "charge_amount": 150.00,
            "paid_amount": 0,
            "adjusted_amount": 0,
            "adjustment_codes": [],
            "remark_codes": [],
            "status": "pending"
        }
    ]  */
    -- Submission and processing details
    submission          JSONB NOT NULL DEFAULT '{}',
    /*  {
        "method": "clearinghouse",
        "clearinghouse": "availity",
        "clearinghouse_id": "CLH-2026-12345",
        "billing_provider_npi": "1234567890",
        "rendering_provider_npi": "0987654321",
        "place_of_service": "11",
        "frequency_code": "1",
        "prior_auth_number": "PA-2026-001"
    }  */
    -- Scrub results
    scrub_results       JSONB NOT NULL DEFAULT '[]',
    /*  [
        {"rule": "ncci_edit", "severity": "error", "message": "...", "resolved": true},
        {"rule": "ai_model", "severity": "warning", "message": "...", "resolved": false}
    ]  */
    -- Payer response history
    responses           JSONB NOT NULL DEFAULT '[]',
    /*  [
        {"type": "277", "date": "2026-05-20", "status": "accepted", "raw": "..."},
        {"type": "835", "date": "2026-06-01", "check_number": "12345", "paid": 120.00, "adjusted": 30.00}
    ]  */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_claim_practice ON claim(practice_id);
CREATE INDEX idx_claim_patient ON claim(patient_id);
CREATE INDEX idx_claim_status ON claim(practice_id, status);
CREATE INDEX idx_claim_payer ON claim(payer_id, status);
CREATE INDEX idx_claim_submitted ON claim(practice_id, submitted_at);
-- GIN index for querying within service lines
CREATE INDEX idx_claim_lines ON claim USING gin(lines);
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
    auth_number         VARCHAR(50),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending',
    -- Authorization details vary by payer and procedure
    details             JSONB NOT NULL DEFAULT '{}',
    /*  {
        "cpt_codes": ["99214", "90837"],
        "icd10_codes": ["F33.1"],
        "requested_units": 12,
        "approved_units": 8,
        "effective_date": "2026-05-20",
        "expiration_date": "2026-11-20",
        "submission_method": "fhir_pas",
        "ai_initiated": true,
        "payer_reference": "PA-AETNA-2026-4567",
        "clinical_notes": "Patient requires ongoing therapy..."
    }  */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prior_auth_patient ON prior_authorization(patient_id);
CREATE INDEX idx_prior_auth_status ON prior_authorization(practice_id, status);
```

## Patient Communication

```sql
CREATE TABLE patient_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    sender_type     VARCHAR(20) NOT NULL,
    sender_id       UUID,
    channel         VARCHAR(20) NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    is_read         BOOLEAN NOT NULL DEFAULT false,
    metadata        JSONB NOT NULL DEFAULT '{}',
    /*  {
        "sms_sid": "SM...",
        "email_message_id": "...",
        "template_used": "appointment_reminder",
        "related_appointment_id": "uuid"
    }  */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_message_patient ON patient_message(patient_id);
CREATE INDEX idx_message_unread ON patient_message(practice_id) WHERE NOT is_read;
```

## Configurable Forms & Templates

```sql
-- Form and note templates stored as JSON Schema definitions
CREATE TABLE form_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL,   -- intake, consent, assessment, note, custom
    schema          JSONB NOT NULL,          -- JSON Schema defining the form structure
    ui_schema       JSONB,                  -- UI rendering hints (field order, widgets, conditionals)
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_template_practice ON form_template(practice_id, category);

CREATE TABLE form_submission (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES form_template(id),
    patient_id      UUID NOT NULL REFERENCES patient(id),
    encounter_id    UUID REFERENCES encounter(id),
    appointment_id  UUID REFERENCES appointment(id),
    responses       JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'submitted',
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    reviewed_by     UUID REFERENCES app_user(id),
    reviewed_at     TIMESTAMPTZ
);
CREATE INDEX idx_submission_patient ON form_submission(patient_id);
```

## AI & Analytics

```sql
CREATE TABLE ai_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    prediction_type VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    probability     NUMERIC(5,4),
    result          JSONB NOT NULL DEFAULT '{}',
    /*  {
        "model_version": "noshow-v2.1",
        "features": {"prior_noshow_rate": 0.15, "day_of_week": 3, "time_slot": "morning"},
        "confidence_interval": [0.08, 0.16],
        "outcome": null,
        "outcome_at": null
    }  */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_entity ON ai_prediction(entity_type, entity_id);
CREATE INDEX idx_ai_type ON ai_prediction(practice_id, prediction_type);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(20) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    ip_address      INET,
    phi_accessed    BOOLEAN NOT NULL DEFAULT false,
    changes         JSONB,                  -- {old: {...}, new: {...}} for mutations
    context         JSONB,                  -- {user_agent, session_id, request_id}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_practice ON audit_log(practice_id, created_at);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

## Reference Data

```sql
CREATE TABLE ref_code (
    code_system     VARCHAR(50) NOT NULL,   -- 'icd10', 'cpt', 'place_of_service', 'nucc_taxonomy'
    code            VARCHAR(20) NOT NULL,
    display         TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB,                  -- System-specific extra fields (RVU values for CPT, etc.)
    PRIMARY KEY (code_system, code)
);
CREATE INDEX idx_ref_code_search ON ref_code USING gin(to_tsvector('english', display));
```

---

## JSONB Query Examples

```sql
-- Find all patients tagged as 'VIP' in a practice
SELECT id, first_name, last_name
FROM patient
WHERE practice_id = 'practice-uuid'
  AND 'VIP' = ANY(tags);

-- Find all appointments with telehealth URLs
SELECT id, start_time, details->>'telehealth_url' AS url
FROM appointment
WHERE practice_id = 'practice-uuid'
  AND details ? 'telehealth_url'
  AND details->>'telehealth_url' IS NOT NULL
  AND start_time >= now();

-- Denial rate by payer using JSONB claim responses
SELECT p.name AS payer_name,
       COUNT(*) AS total_claims,
       COUNT(*) FILTER (WHERE c.status = 'denied') AS denied_claims,
       ROUND(COUNT(*) FILTER (WHERE c.status = 'denied')::NUMERIC / COUNT(*) * 100, 1) AS denial_rate
FROM claim c
JOIN payer p ON p.id = c.payer_id
WHERE c.practice_id = 'practice-uuid'
  AND c.submitted_at >= now() - INTERVAL '90 days'
GROUP BY p.name
ORDER BY denial_rate DESC;

-- Find claims with specific CPT code in service lines
SELECT id, claim_number, total_charge, status
FROM claim
WHERE practice_id = 'practice-uuid'
  AND lines @> '[{"cpt_code": "99213"}]'::jsonb;

-- Behavioral health patients with specific diagnosis
SELECT id, first_name, last_name
FROM patient
WHERE practice_id = 'practice-uuid'
  AND clinical_summary->'diagnosis_history' @> '[{"code": "F33.1", "status": "active"}]'::jsonb;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 2 | practice, location |
| User & Access Control | 1 | app_user (roles as array, permissions as JSONB) |
| Patient Management | 1 | patient (identifiers, contacts, clinical summary as JSONB) |
| Insurance & Coverage | 2 | payer, patient_coverage |
| Scheduling | 4 | practitioner, schedule_block, appointment, waitlist_entry |
| Clinical Documentation | 2 | encounter (diagnoses/charges as JSONB), clinical_note (content as JSONB) |
| Billing & Claims | 1 | claim (lines, scrub results, responses as JSONB) |
| Prior Authorization | 1 | prior_authorization |
| Patient Communication | 1 | patient_message |
| Forms & Templates | 2 | form_template, form_submission |
| AI & Analytics | 1 | ai_prediction |
| Audit & Compliance | 1 | audit_log (partitioned) |
| Reference Data | 1 | ref_code (unified reference table) |
| **Total** | **20** | |

---

## Key Design Decisions

1. **JSONB for variable data, columns for queryable/indexed data.** The rule: if you WHERE, ORDER BY, or JOIN on it, it is a column. If it varies by specialty/jurisdiction or is rarely queried independently, it is JSONB. Patient name and DOB are columns; clinical summary is JSONB.

2. **Claim service lines embedded in JSONB rather than a separate table.** A claim typically has 1-10 service lines. Embedding them avoids the claim/claim_line join for every claim read. The GIN index on `lines` enables containment queries (`@>`) for finding claims with specific CPT codes.

3. **Unified `ref_code` table replaces multiple reference tables.** ICD-10, CPT, Place of Service, and NUCC taxonomy codes all live in one table keyed by `(code_system, code)`. This simplifies code lookups and reduces the number of tables by consolidating four into one.

4. **Roles stored as PostgreSQL array, permissions as JSONB.** This eliminates the `role`, `permission`, `role_permission`, and `user_role` junction tables from the normalized model (4 tables reduced to 0). Role-based defaults are resolved in application code; per-user overrides are stored in the `permissions` JSONB column.

5. **Payer rules stored in JSONB.** Different payers have radically different timely filing limits, prior auth requirements, modifier rules, and submission preferences. JSONB absorbs this variation without schema migrations. AI claim scrubbing reads payer rules from this column.

6. **Appointment reminders embedded in the appointment row.** Reminders are always read in the context of their appointment and typically number 1-3 per appointment. Embedding them as a JSONB array eliminates a join and a separate table.

7. **Clinical note content is structured JSONB, not free text.** Different note types (SOAP, behavioral health progress, assessment) have different structures. Storing content as JSONB with JSON Schema validation means the same `clinical_note` table serves all specialties with different note formats.

8. **Address modeled as JSONB.** US addresses have state and zip; Canadian addresses have province and postal code; Australian addresses have state and postcode. JSONB absorbs these differences without country-specific columns.

9. **20 tables vs. 37 in the normalized model.** JSONB consolidation reduces table count by nearly half. This simplifies the codebase, reduces migration complexity, and makes the schema more approachable for new developers. The cost is shifted to JSON Schema validation in the application layer.

10. **Encounter embeds diagnoses and charges.** Rather than separate `encounter_diagnosis` and `charge` tables, diagnoses and charges are JSONB arrays on the encounter. For a PMS where encounters typically have 1-5 diagnoses and 1-5 charges, this avoids two joins on every encounter read. The claim table captures the billing-specific view of charges separately.
