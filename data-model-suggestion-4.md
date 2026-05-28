# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Practice Management System · Created: 2026-05-19

## Philosophy

This model uses a standard relational schema for operational CRUD (scheduling, billing, clinical documentation) but adds a property graph layer for relationship-intensive queries: provider referral networks, patient coverage chains, care team composition, payer-provider contractual relationships, and conflict-of-interest detection. The graph layer is implemented using PostgreSQL `graph_node` and `graph_edge` tables with recursive CTEs, avoiding the need for a separate graph database while enabling relationship traversal queries that would be expensive or impractical in a purely relational model.

The insight driving this approach is that a practice management system is fundamentally a relationship management system. Patients are connected to providers, providers to locations, locations to practices, practices to payer networks, payers to coverage plans, and plans to prior authorization requirements. These relationships form a graph, and many of the AI features (referral pattern analysis, coverage optimization, denial pattern correlation, care coordination) are inherently graph traversal problems.

Property graphs are used in healthcare by Epic's Care Everywhere network for provider relationship mapping, by payer platforms for network adequacy analysis, and by clinical trial matching systems. The relational+graph hybrid avoids vendor lock-in to graph databases (Neo4j, Amazon Neptune) while keeping the operational data model in PostgreSQL where it benefits from mature tooling, ACID transactions, and row-level security.

**Best for:** Teams building referral management, care coordination, multi-payer network analysis, or AI-powered relationship insights as core differentiators.

**Trade-offs:**
- (+) Efficient multi-hop relationship queries: "find all providers within 2 referral hops of Dr. Smith who accept Aetna"
- (+) Natural model for care team composition and provider network analysis
- (+) Enables AI features like referral pattern analysis and coverage optimization
- (+) No external graph database dependency — all in PostgreSQL with recursive CTEs
- (+) Graph layer is additive — operational tables work independently if graph queries are not needed
- (-) Dual write: changes to relational tables must be reflected in graph nodes/edges
- (-) Recursive CTEs can be slow for deep traversals (>5 hops) on large datasets
- (-) More complex application code to maintain graph consistency
- (-) Developers must understand both relational and graph query patterns
- (-) Graph indexes (GiST/GIN) consume additional storage

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HL7 FHIR R4 | Operational tables mirror FHIR resources; graph edges model FHIR References between resources |
| FHIR CareTeam | Care team relationships modeled as graph edges between Patient, Practitioner, and Organization nodes |
| FHIR Provenance | Graph edges carry provenance metadata (who created the relationship, when, why) |
| X12 EDI 837/835 | Billing tables are relational; payer-provider contractual relationships are graph edges |
| NUCC Taxonomy | Provider specialty codes stored on practitioner graph nodes for network adequacy queries |
| NPI Registry | Provider NPI stored on practitioner nodes; NPI-to-NPI referral patterns modeled as edges |
| HIPAA Security Rule | Graph access queries logged in audit table; PHI nodes tagged for access control |
| ISO 3166-1/2 | Location nodes carry jurisdiction codes for geographic network analysis |

---

## Graph Layer (Relationship Infrastructure)

```sql
-- Generic property graph implemented in PostgreSQL
-- Every entity that participates in relationships gets a graph_node entry

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY,       -- Same UUID as the entity in its operational table
    practice_id     UUID NOT NULL,          -- Tenant isolation
    node_type       VARCHAR(50) NOT NULL,   -- Patient, Practitioner, Location, Practice, Payer, CoveragePlan
    label           VARCHAR(255) NOT NULL,  -- Human-readable label: "Dr. Jane Smith, MD" or "Aetna PPO"
    properties      JSONB NOT NULL DEFAULT '{}',
    /*  Practitioner node properties:
    {
        "npi": "1234567890",
        "specialty_code": "207Q00000X",
        "specialty_name": "Family Medicine",
        "credentials": "MD",
        "accepting_new_patients": true
    }

    Payer node properties:
    {
        "payer_id": "62308",
        "type": "commercial",
        "network_name": "Aetna PPO",
        "state_licensed": ["TX", "CA", "NY"]
    }

    Patient node properties:
    {
        "age_group": "35-44",
        "zip_code": "78701",
        "primary_language": "en",
        "risk_score": 2.1
    }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gnode_practice ON graph_node(practice_id);
CREATE INDEX idx_gnode_type ON graph_node(practice_id, node_type);
CREATE INDEX idx_gnode_properties ON graph_node USING gin(properties);
CREATE INDEX idx_gnode_label ON graph_node USING gin(to_tsvector('english', label));

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,
    /*  Edge types:
        REFERRED_BY         — Practitioner -> Practitioner (referral relationship)
        TREATS              — Practitioner -> Patient
        MEMBER_OF           — Practitioner -> Location (works at)
        COVERED_BY          — Patient -> CoveragePlan
        PROVIDES_PLAN       — Payer -> CoveragePlan
        IN_NETWORK          — Practitioner -> Payer (contracted)
        CARE_TEAM           — Patient -> Practitioner (care team membership)
        SUPERVISES          — Practitioner -> Practitioner
        LOCATED_IN          — Location -> Jurisdiction
        AFFILIATED_WITH     — Practice -> Practice (multi-practice groups)
    */
    properties      JSONB NOT NULL DEFAULT '{}',
    /*  REFERRED_BY edge:
    {
        "referral_count": 15,
        "first_referral_date": "2024-03-01",
        "last_referral_date": "2026-05-10",
        "avg_referral_value": 350.00,
        "specialties_referred": ["cardiology", "endocrinology"]
    }

    IN_NETWORK edge:
    {
        "contract_effective": "2025-01-01",
        "contract_termination": "2026-12-31",
        "fee_schedule_id": "FS-2025-AETNA-PPO",
        "credentialed": true,
        "credentialing_date": "2024-11-15"
    }

    TREATS edge:
    {
        "relationship": "primary_care",
        "first_visit": "2024-06-15",
        "last_visit": "2026-05-01",
        "visit_count": 8
    }
    */
    weight          NUMERIC(10,4) DEFAULT 1.0,  -- Edge weight for scoring/ranking
    effective_from  DATE,
    effective_to    DATE,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gedge_source ON graph_edge(source_id, edge_type);
CREATE INDEX idx_gedge_target ON graph_edge(target_id, edge_type);
CREATE INDEX idx_gedge_practice ON graph_edge(practice_id, edge_type);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_properties ON graph_edge USING gin(properties);
CREATE INDEX idx_gedge_active ON graph_edge(practice_id, edge_type)
    WHERE effective_to IS NULL OR effective_to >= CURRENT_DATE;
```

## Operational Tables (Relational)

### Practice & Location

```sql
CREATE TABLE practice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    npi             VARCHAR(10),
    tax_id          VARCHAR(20),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    billing_address JSONB,
    settings        JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    name            VARCHAR(255) NOT NULL,
    address         JSONB NOT NULL,
    phone           VARCHAR(20),
    timezone        VARCHAR(50),
    is_billing_location BOOLEAN NOT NULL DEFAULT false,
    geo_point       POINT,                  -- Latitude/longitude for geographic queries
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_location_practice ON location(practice_id);
CREATE INDEX idx_location_geo ON location USING gist(geo_point);
```

### Users & Access Control

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    roles           VARCHAR(50)[] NOT NULL DEFAULT '{front_desk}',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(practice_id, email)
);
CREATE INDEX idx_user_practice ON app_user(practice_id);
```

### Patient

```sql
CREATE TABLE patient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id     UUID NOT NULL REFERENCES practice(id),
    mrn             VARCHAR(50),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          VARCHAR(20),
    email           VARCHAR(255),
    phone_mobile    VARCHAR(20),
    address         JSONB,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_patient_practice ON patient(practice_id);
CREATE INDEX idx_patient_name ON patient(practice_id, last_name, first_name);
CREATE INDEX idx_patient_dob ON patient(practice_id, date_of_birth);
```

### Practitioner

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
    specialty_name  VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_practitioner_practice ON practitioner(practice_id);
CREATE INDEX idx_practitioner_npi ON practitioner(npi) WHERE npi IS NOT NULL;
```

### Insurance & Coverage

```sql
CREATE TABLE payer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    payer_id        VARCHAR(50),
    electronic_id   VARCHAR(50),
    type            VARCHAR(50),
    rules           JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE coverage_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payer_id        UUID NOT NULL REFERENCES payer(id),
    plan_name       VARCHAR(255) NOT NULL,
    plan_type       VARCHAR(50),            -- PPO, HMO, EPO, POS, Medicare Advantage, Medicaid
    network_tier    VARCHAR(20),            -- in_network, out_of_network, tier1, tier2
    state_codes     VARCHAR(10)[],          -- States where plan is available
    details         JSONB NOT NULL DEFAULT '{}',
    /*  {
        "copay_office_visit": 25.00,
        "copay_specialist": 50.00,
        "deductible_individual": 1500.00,
        "coinsurance": 0.20,
        "pa_required_procedures": ["99214", "90837"],
        "timely_filing_days": 90,
        "fee_schedule": "standard_2026"
    }  */
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_plan_payer ON coverage_plan(payer_id);

CREATE TABLE patient_coverage (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    coverage_plan_id    UUID NOT NULL REFERENCES coverage_plan(id),
    coverage_order      SMALLINT NOT NULL DEFAULT 1,
    subscriber_id       VARCHAR(100) NOT NULL,
    group_number        VARCHAR(50),
    relationship        VARCHAR(20),
    subscriber_info     JSONB,
    effective_date      DATE NOT NULL,
    termination_date    DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    last_eligibility    JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_coverage_patient ON patient_coverage(patient_id);
CREATE INDEX idx_coverage_plan ON patient_coverage(coverage_plan_id);
```

### Scheduling

```sql
CREATE TABLE appointment (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    practitioner_id     UUID NOT NULL REFERENCES practitioner(id),
    location_id         UUID NOT NULL REFERENCES location(id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'booked',
    appointment_type    VARCHAR(100),
    reason_text         TEXT,
    source              VARCHAR(20) DEFAULT 'manual',
    referring_provider_id UUID REFERENCES practitioner(id),  -- Links to graph referral analysis
    no_show_probability NUMERIC(5,4),
    details             JSONB NOT NULL DEFAULT '{}',
    created_by          UUID REFERENCES app_user(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_appt_practice ON appointment(practice_id);
CREATE INDEX idx_appt_patient ON appointment(patient_id);
CREATE INDEX idx_appt_practitioner_time ON appointment(practitioner_id, start_time);
CREATE INDEX idx_appt_referrer ON appointment(referring_provider_id) WHERE referring_provider_id IS NOT NULL;
```

### Clinical & Billing

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
    diagnoses           JSONB NOT NULL DEFAULT '[]',
    reason_text         TEXT,
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
    note_type       VARCHAR(50) NOT NULL,
    content         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    signed_by       UUID REFERENCES app_user(id),
    signed_at       TIMESTAMPTZ,
    cosigned_by     UUID REFERENCES app_user(id),
    cosigned_at     TIMESTAMPTZ,
    ai_metadata     JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_note_encounter ON clinical_note(encounter_id);

CREATE TABLE claim (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    practice_id         UUID NOT NULL REFERENCES practice(id),
    patient_id          UUID NOT NULL REFERENCES patient(id),
    encounter_id        UUID REFERENCES encounter(id),
    patient_coverage_id UUID NOT NULL REFERENCES patient_coverage(id),
    payer_id            UUID NOT NULL REFERENCES payer(id),
    claim_number        VARCHAR(50),
    payer_claim_number  VARCHAR(50),
    claim_type          VARCHAR(20) NOT NULL,
    total_charge        NUMERIC(10,2) NOT NULL,
    total_paid          NUMERIC(10,2) DEFAULT 0,
    total_adjusted      NUMERIC(10,2) DEFAULT 0,
    patient_responsibility NUMERIC(10,2) DEFAULT 0,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    submitted_at        TIMESTAMPTZ,
    lines               JSONB NOT NULL DEFAULT '[]',
    submission          JSONB NOT NULL DEFAULT '{}',
    responses           JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_claim_practice ON claim(practice_id);
CREATE INDEX idx_claim_patient ON claim(patient_id);
CREATE INDEX idx_claim_status ON claim(practice_id, status);
CREATE INDEX idx_claim_payer ON claim(payer_id, status);
```

### Prior Authorization

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
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pa_patient ON prior_authorization(patient_id);
CREATE INDEX idx_pa_status ON prior_authorization(practice_id, status);
```

## Audit

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
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_practice ON audit_log(practice_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

## Reference Data

```sql
CREATE TABLE ref_code (
    code_system     VARCHAR(50) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    display         TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB,
    PRIMARY KEY (code_system, code)
);
CREATE INDEX idx_ref_code_search ON ref_code USING gin(to_tsvector('english', display));
```

---

## Graph Query Examples

```sql
-- 1. Find all providers a practitioner has referred patients to (1 hop)
SELECT gn.label AS referred_to,
       ge.properties->>'referral_count' AS referral_count,
       ge.properties->>'last_referral_date' AS last_referral
FROM graph_edge ge
JOIN graph_node gn ON gn.id = ge.target_id
WHERE ge.source_id = 'practitioner-uuid'
  AND ge.edge_type = 'REFERRED_BY'
  AND ge.practice_id = 'practice-uuid'
ORDER BY (ge.properties->>'referral_count')::INTEGER DESC;

-- 2. Find all providers reachable within 2 referral hops who accept a specific payer
WITH RECURSIVE referral_chain AS (
    -- Base: direct referral partners
    SELECT ge.target_id AS provider_id,
           gn.label AS provider_name,
           1 AS depth,
           ARRAY[ge.source_id, ge.target_id] AS path
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.target_id
    WHERE ge.source_id = 'practitioner-uuid'
      AND ge.edge_type = 'REFERRED_BY'
      AND ge.practice_id = 'practice-uuid'

    UNION ALL

    -- Recursive: referral partners of referral partners
    SELECT ge.target_id,
           gn.label,
           rc.depth + 1,
           rc.path || ge.target_id
    FROM referral_chain rc
    JOIN graph_edge ge ON ge.source_id = rc.provider_id
        AND ge.edge_type = 'REFERRED_BY'
    JOIN graph_node gn ON gn.id = ge.target_id
    WHERE rc.depth < 2
      AND NOT ge.target_id = ANY(rc.path)  -- Prevent cycles
)
SELECT DISTINCT rc.provider_name, rc.depth
FROM referral_chain rc
-- Filter to those in-network with Aetna
WHERE EXISTS (
    SELECT 1 FROM graph_edge net
    WHERE net.source_id = rc.provider_id
      AND net.edge_type = 'IN_NETWORK'
      AND net.target_id = 'aetna-payer-uuid'
      AND (net.effective_to IS NULL OR net.effective_to >= CURRENT_DATE)
)
ORDER BY rc.depth, rc.provider_name;

-- 3. Build a patient's complete care team
SELECT gn.label AS team_member,
       gn.node_type,
       ge.properties->>'relationship' AS relationship,
       ge.properties->>'first_visit' AS since
FROM graph_edge ge
JOIN graph_node gn ON gn.id = ge.target_id
WHERE ge.source_id = 'patient-uuid'
  AND ge.edge_type = 'CARE_TEAM'
  AND (ge.effective_to IS NULL OR ge.effective_to >= CURRENT_DATE)
ORDER BY ge.properties->>'relationship';

-- 4. Network adequacy: count in-network providers by specialty within a geographic radius
SELECT gn.properties->>'specialty_name' AS specialty,
       COUNT(*) AS provider_count
FROM graph_edge ge
JOIN graph_node gn ON gn.id = ge.source_id
JOIN practitioner p ON p.id = gn.id
JOIN location l ON l.practice_id = p.practice_id
WHERE ge.edge_type = 'IN_NETWORK'
  AND ge.target_id = 'payer-uuid'
  AND (ge.effective_to IS NULL OR ge.effective_to >= CURRENT_DATE)
  AND l.geo_point <@ circle '((30.2672, -97.7431), 0.5)'  -- ~30 mile radius from Austin
GROUP BY gn.properties->>'specialty_name'
ORDER BY provider_count DESC;

-- 5. Referral pattern analysis for AI: which referral paths lead to the highest claim denial rates?
SELECT source_prov.label AS referring_provider,
       target_prov.label AS receiving_provider,
       ge.properties->>'referral_count' AS referrals,
       COUNT(c.id) AS total_claims,
       COUNT(c.id) FILTER (WHERE c.status = 'denied') AS denied_claims,
       ROUND(COUNT(c.id) FILTER (WHERE c.status = 'denied')::NUMERIC / NULLIF(COUNT(c.id), 0) * 100, 1) AS denial_rate
FROM graph_edge ge
JOIN graph_node source_prov ON source_prov.id = ge.source_id
JOIN graph_node target_prov ON target_prov.id = ge.target_id
JOIN appointment a ON a.referring_provider_id = ge.source_id AND a.practitioner_id = ge.target_id
JOIN encounter e ON e.appointment_id = a.id
JOIN claim c ON c.encounter_id = e.id
WHERE ge.edge_type = 'REFERRED_BY'
  AND ge.practice_id = 'practice-uuid'
GROUP BY source_prov.label, target_prov.label, ge.properties->>'referral_count'
HAVING COUNT(c.id) > 5
ORDER BY denial_rate DESC;

-- 6. Detect potential coverage gaps: patients whose coverage plan's provider is not in-network
SELECT p.first_name, p.last_name,
       cp.plan_name,
       pract.first_name || ' ' || pract.last_name AS provider_name
FROM patient_coverage pc
JOIN coverage_plan cp ON cp.id = pc.coverage_plan_id
JOIN patient p ON p.id = pc.patient_id
JOIN appointment a ON a.patient_id = p.id AND a.start_time > now()
JOIN practitioner pract ON pract.id = a.practitioner_id
WHERE pc.practice_id = 'practice-uuid'
  AND pc.status = 'active'
  AND NOT EXISTS (
      SELECT 1 FROM graph_edge ge
      WHERE ge.source_id = a.practitioner_id
        AND ge.target_id = cp.payer_id
        AND ge.edge_type = 'IN_NETWORK'
        AND (ge.effective_to IS NULL OR ge.effective_to >= CURRENT_DATE)
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Core Identity | 2 | practice, location (with geo_point) |
| User & Access | 1 | app_user |
| Patient | 1 | patient |
| Practitioner | 1 | practitioner |
| Insurance & Coverage | 3 | payer, coverage_plan, patient_coverage |
| Scheduling | 1 | appointment |
| Clinical | 2 | encounter, clinical_note |
| Billing | 1 | claim |
| Prior Authorization | 1 | prior_authorization |
| Audit | 1 | audit_log (partitioned) |
| Reference Data | 1 | ref_code |
| **Total** | **17** | Plus graph_node/graph_edge which model all relationships |

---

## Key Design Decisions

1. **Graph layer as a separate concern from operational tables.** The `graph_node` and `graph_edge` tables do not replace relational tables — they supplement them. A `practitioner` row stores the provider's operational data; a `graph_node` with the same UUID stores the provider's relationship properties. This means the graph layer can be added incrementally and is not required for basic CRUD operations.

2. **Edge types are strings, not tables.** Rather than a separate table for each relationship type (referral, coverage, care_team), all relationships live in `graph_edge` with an `edge_type` discriminator. This makes adding new relationship types trivial (insert a new edge type) and enables cross-type queries (find all relationships for a node regardless of type).

3. **Coverage plans as first-class entities.** Unlike the other models where coverage is a direct patient-to-payer relationship, this model introduces `coverage_plan` as a separate entity. This enables graph queries like "find all patients on the Aetna PPO Gold plan" and "which providers are in-network for this specific plan" — queries that require the plan to be a node in the graph.

4. **Geographic indexing on locations.** The `geo_point` column with a GiST index enables geographic proximity queries for network adequacy analysis: "how many dermatologists within 30 miles accept this payer?" This is a regulatory requirement for payer network adequacy reporting.

5. **Temporal edges with effective_from/effective_to.** Graph edges are temporal: a provider's in-network status with a payer has a start and end date. This enables both current-state queries (filter by `effective_to IS NULL OR effective_to >= CURRENT_DATE`) and historical queries ("was Dr. Smith in-network with Aetna on the date of service?").

6. **Referral tracking on appointments.** The `appointment.referring_provider_id` column links each referred appointment to the referring provider, enabling the graph layer to build referral pattern analytics. When an appointment is created with a referral, a REFERRED_BY edge is created or its `referral_count` is incremented.

7. **Edge weights for scoring.** The `graph_edge.weight` column enables weighted graph algorithms: rank referral partners by referral volume, score care team members by visit frequency, or prioritize in-network providers by contract tier.

8. **Fewer operational tables than normalized model (~17 vs ~37).** The graph layer absorbs relationship complexity that would otherwise require junction tables and complex joins. The `coverage_plan` table is the only addition over the JSONB hybrid model; the graph tables handle all other relationship modeling.

9. **Properties on both nodes and edges.** Following the property graph model, both nodes and edges carry JSONB properties. Node properties store static attributes (specialty, NPI); edge properties store relationship-specific data (referral count, contract dates). This eliminates the need for relationship-specific attribute tables.

10. **Recursive CTE depth limit of 2-3 hops.** For healthcare graph queries, 2-3 hops covers the vast majority of use cases (referral chains, care team composition, coverage chains). The recursive CTEs include cycle detection (`NOT target_id = ANY(path)`) and explicit depth limits to prevent runaway queries.
