# Standards & API Reference

> Project: Practice Management System · Generated: 2026-05-06

## Industry Standards & Specifications

### HL7 Standards

**HL7 FHIR R4 (Release 4)**
- **URL:** https://www.hl7.org/fhir/R4/
- FHIR is the primary interoperability standard for modern practice management systems. FHIR R4 (v4.0.1) defines RESTful APIs and JSON/XML resource models for Patient, Appointment, Schedule, Slot, Practitioner, Organization, Coverage, Claim, and related resources. ONC mandates FHIR R4 support for certified health IT under the 21st Century Cures Act. All new PMS integrations should target FHIR R4 as the baseline exchange format.

**HL7 FHIR R4 — Schedule, Slot & Appointment Resources**
- **URL:** https://www.hl7.org/fhir/R4/schedule-examples.html
- The Schedule resource defines a time period and availability window for booking; Slot resources represent individual bookable time periods within a Schedule; and Appointment resources represent confirmed bookings. These three resources together define the FHIR scheduling data model used by products such as athenahealth, Oracle Health, and DrChrono.

**HL7 FHIR R5 (Release 5)**
- **URL:** https://hl7.org/fhir/R5/
- The current published release of FHIR (May 2023). Introduces improvements to the scheduling model, refined subscription mechanisms, and enhanced support for clinical documentation. PMS vendors are beginning R5 evaluation; R4 remains the regulatory baseline for the US market through at least 2028.

**HL7 v2.x — SIU / SRM Scheduling Messages**
- **URL:** https://www.hl7.eu/HL7v2x/v24/std24/ch10.htm
- Legacy HL7 v2 scheduling messages remain widespread in hospital and ambulatory settings. SIU (Scheduling Information Unsolicited) messages with 14 trigger events (S12–S26) communicate appointment bookings, rescheduling, cancellations, and modifications between PMS and hospital information systems. SRM (Schedule Request Message) is the request counterpart. Any PMS targeting enterprise hospital integration must implement HL7 v2.x SIU/SRM alongside FHIR.

**HL7 FHIR Bulk Data Access IG (v2.0.0)**
- **URL:** https://hl7.org/fhir/uv/bulkdata/
- Defines a standardized, asynchronous export operation (`$export`) for retrieving large patient populations from a FHIR server. Required under ONC 21st Century Cures Act rules for certified EHR/PMS to support population-level data export. Critical for analytics, value-based care reporting, and AI model training pipelines.

**SMART App Launch Framework (v2.2.0)**
- **URL:** https://hl7.org/fhir/smart-app-launch/
- Defines OAuth 2.0 and OpenID Connect patterns for launching third-party applications within an EHR/PMS context (EHR-launch and standalone-launch). Specifies scope syntax for fine-grained FHIR resource access (e.g., `patient/Appointment.read`). Required for ONC certification and is the de-facto standard for PMS app marketplaces. SMART v2.2.0 adds backend services (machine-to-machine) and granular scopes.

---

### X12 / HIPAA EDI Transaction Standards

**ASC X12 837P / 837I / 837D — Healthcare Claims**
- **URL:** https://x12.org/flow/health-care
- HIPAA-mandated transaction format for submitting professional (837P), institutional (837I), and dental (837D) claims to payers. Version 5010A1 is the adopted standard (45 CFR Part 162). All US practice management systems handling insurance billing must implement 837 claim submission. X12 Version 5010 is the current regulatory baseline; approximately 95% of provider claims are submitted electronically in this format.

**ASC X12 835 — Electronic Remittance Advice (ERA)**
- **URL:** https://x12.org/flow/health-care
- The 835 transaction communicates payer payment decisions, adjustments, and denial codes back to the provider. Enables automated payment posting (ERA posting) and denial identification in PMS billing workflows. HIPAA-mandated for payer payment reconciliation.

**ASC X12 270 / 271 — Eligibility Inquiry and Response**
- **URL:** https://intuitionlabs.ai/articles/x12-edi-transactions-guide
- The 270 transaction queries a health plan for a member's current eligibility, benefits, copays, deductibles, and coverage limitations. The 271 response returns this information in a structured format. Triggered at appointment booking and again the day before the appointment in modern PMS workflows. AI-driven eligibility agents can initiate and parse these transactions without human intervention.

**ASC X12 276 / 277 — Claim Status Inquiry and Response**
- **URL:** https://www.cms.gov/priorities/key-initiatives/burden-reduction/administrative-simplification/hipaa/adopted-standards-operating-rules
- Allows a PMS to query payer systems for the status of a submitted claim (276) and receive a structured status response (277). Reduces manual phone calls to payers for claim status follow-up.

**ASC X12 278 — Prior Authorization (Request and Response)**
- **URL:** https://intuitionlabs.ai/articles/x12-edi-transactions-guide
- The 278 transaction is used for submitting prior authorization (PA) requests to payers and receiving approval or denial decisions. Increasingly being supplemented by FHIR-based Da Vinci PAS workflows (see below) for real-time PA processing.

---

### ONC / CMS Interoperability Rules

**21st Century Cures Act — ONC Interoperability & Information Blocking Rule (2020)**
- **URL:** https://www.healthit.gov/topic/oncs-cures-act-final-rule
- Establishes information blocking prohibitions, mandates FHIR R4-based patient access APIs, and requires certified health IT to support SMART App Launch, FHIR Bulk Data, and US Core Implementation Guide. Any PMS seeking ONC certification for the US market must comply. USCDI v3 compliance becomes mandatory January 1, 2026.

**United States Core Data for Interoperability (USCDI)**
- **URL:** https://isp.healthit.gov/united-states-core-data-interoperability-uscdi
- Defines the minimum standardized clinical data elements required for interoperable health information exchange. USCDI v1 (2020) established 53 data elements; v3 (mandatory from January 1, 2026) expands to address health equity; v4 adds 20 elements; v6 (July 2025) adds Portable Medical Order, Family Health History, and Unique Device Identifier; Draft v7 (January 2026) includes 30 new elements. PMS systems must align their patient data models to the required USCDI version.

**CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F, 2024)**
- **URL:** https://www.cms.gov/newsroom/fact-sheets/cms-interoperability-prior-authorization-final-rule-cms-0057-f
- Requires Medicare Advantage, Medicaid, CHIP, and QHP payers to implement FHIR-based APIs for Patient Access, Provider Access, Payer-to-Payer exchange, and Prior Authorization by January 1, 2027. Directly impacts PMS prior authorization workflows — payers must expose FHIR Prior Authorization APIs that PMS systems can query.

**HL7 Da Vinci Prior Authorization Support (PAS) IG (v2.1.0)**
- **URL:** https://hl7.org/fhir/us/davinci-pas/
- FHIR-based implementation guide for submitting prior authorization requests directly to payers using FHIR R4, replacing or augmenting the legacy X12 278 workflow. Combined with the Coverage Requirements Discovery IG (CRD) and Documentation Templates and Rules IG (DTR), enables real-time AI-driven PA initiation at point-of-care.

---

### IHE Profiles

**IHE ITI FHIR Scheduling Profile (v1.0.0 / v1.0.1)**
- **URL:** https://profiles.ihe.net/ITI/Scheduling/index.html
- IHE Integrating the Healthcare Enterprise profile providing FHIR R4 APIs and guidance for cross-organizational appointment scheduling between patient-facing applications, PMS systems, and hospital scheduling systems. Defines the Find Potential Appointments, Hold Appointment, Book Appointment, and Find Existing Appointments transactions. Current version dated December 2024.

---

### Security & Compliance Standards

**HIPAA Privacy Rule (45 CFR Part 164, Subparts A and E)**
- **URL:** https://www.hhs.gov/hipaa/for-professionals/privacy/index.html
- Governs the permissible uses and disclosures of protected health information (PHI) by covered entities (healthcare providers, health plans, clearinghouses) and their business associates. Any PMS handling US patient data must comply with HIPAA Privacy Rule, execute BAAs with all sub-processors, and implement minimum necessary access controls.

**HIPAA Security Rule (45 CFR Part 164, Subpart C)**
- **URL:** https://www.hhs.gov/hipaa/for-professionals/security/index.html
- Mandates administrative, physical, and technical safeguards for electronic PHI (ePHI). Requires access controls, audit logging, encryption, and workforce training. Technical safeguards directly map to PMS API authentication, role-based access control, and audit trail requirements.

**NIST SP 800-66 Rev. 2 — Implementing the HIPAA Security Rule (2024)**
- **URL:** https://csrc.nist.gov/pubs/sp/800/66/r2/final
- NIST guidance translating the HIPAA Security Rule into practical cybersecurity controls, mapped to the NIST Cybersecurity Framework and NIST SP 800-53r5. Published 2024. Provides the implementation roadmap that most US healthcare software vendors follow to demonstrate HIPAA Security Rule compliance.

**ISO 27799:2016 — Health Informatics Information Security Management**
- **URL:** https://www.iso.org/standard/62777.html
- International standard providing healthcare-specific guidance for implementing ISO/IEC 27002 information security controls in health informatics environments. Applicable to PMS vendors targeting markets outside the US where HIPAA does not apply (EU, Australia, Canada) but equivalent patient data confidentiality obligations exist. Technology-neutral; specifies what is required rather than how.

**ISO 13606-1:2019 — Electronic Health Record Communication (Reference Model)**
- **URL:** https://www.iso.org/standard/67868.html
- ISO standard defining the information architecture for communicating EHR data between systems. Relevant for PMS vendors building interoperability with EHR systems in European and international markets where ISO 13606 archetypes are used. Part 1 defines the reference model; Parts 2–5 cover archetype interchange, reference archetypes, security, and interface specifications.

**OAuth 2.0 (RFC 6749) and OpenID Connect Core 1.0**
- **URL (OAuth 2.0):** https://datatracker.ietf.org/doc/html/rfc6749
- **URL (OIDC):** https://openid.net/specs/openid-connect-core-1_0.html
- OAuth 2.0 is the authorization framework used by SMART on FHIR; OIDC adds identity layer for user authentication. Required for secure PMS API access by third-party applications, patient-facing portals, and AI agents. PKCE (RFC 7636) is required for public clients (mobile apps, SPAs).

---

### E-Prescribing Standards

**NCPDP SCRIPT Standard (v2023011)**
- **URL:** https://surescripts.com/GetSCRIPT
- The National Council for Prescription Drug Programs SCRIPT Standard governs electronic prescribing transactions between PMS/EHR and pharmacy systems. v2023011 is the current version being rolled out by Surescripts through 2026, covering e-prescribing, medication history, electronic prior authorization, and real-time prescription benefit (RTPB). PMS vendors integrate via Surescripts network rather than directly with NCPDP (requires NCPDP membership or Surescripts partnership).

---

### Data Format Standards

**OpenAPI Specification (OAS) v3.1.1**
- **URL:** https://spec.openapis.org/oas/v3.1.1.html
- Vendor-neutral specification for describing RESTful APIs using JSON/YAML. PMS API documentation should conform to OAS 3.1 to enable code generation, mock servers, automated conformance testing, and interoperability testing tooling. FHIR Capability Statements serve an analogous role within the FHIR ecosystem.

**JSON Schema (Draft 2020-12)**
- **URL:** https://json-schema.org/specification
- Schema language for validating JSON data structures. Underpins OpenAPI 3.1 data model definitions and FHIR resource validation. Used in PMS API request/response payload validation and test harnesses.

---

## Similar Products — Developer Documentation & APIs

### athenahealth

- **Description:** Cloud-native PMS and EHR platform serving 160,000+ providers across ambulatory and specialty practices. Offers both a proprietary REST API (800+ endpoints) and a FHIR R4-compliant API layer for regulatory interoperability.
- **API Documentation:** https://docs.athenahealth.com/api
- **FHIR R4 API Reference:** https://mydata.athenahealth.com/fhirapidoc/r4
- **Developer Portal:** https://www.athenahealth.com/developer-portal
- **SDKs / Libraries:** No official SDK; REST and FHIR APIs are documented for direct HTTP integration.
- **Developer Guide:** https://docs.athenahealth.com/api/guides/overview
- **Standards:** FHIR R4, SMART on FHIR, ONC 21st Century Cures Act compliant, USCDI v1+
- **Authentication:** OAuth 2.0 (SMART on FHIR flows for FHIR API; proprietary OAuth for legacy REST API)

---

### DrChrono (EverHealth)

- **Description:** iPad-native EHR and PMS serving independent practices. Exposes a RESTful proprietary API and a separate FHIR R4 API. API access is available to practices and approved third-party developers.
- **API Documentation:** https://app.drchrono.com/api-docs/
- **FHIR API Documentation:** https://drchrono-fhirpresentation.everhealthsoftware.com/drchrono/498711/r4/Home/ApiDocumentation
- **SDKs / Libraries:** Python and C# sample code available in developer support resources; no official SDK.
- **Developer Guide:** https://support.drchrono.com/home/23607539700635-api
- **Standards:** FHIR R4, SMART on FHIR; REST/JSON for proprietary API
- **Authentication:** OAuth 2.0; authorization endpoint `https://drchrono.com/o/authorize/`, token endpoint `https://app.drchrono.com/o/token/`

---

### AdvancedMD

- **Description:** Integrated EHR, PMS, and billing platform for independent practices. Offers XMLRPC and RESTful proprietary APIs plus a FHIR Developer Portal. API access requires a Certified API Developer Agreement.
- **Developer Portal:** https://developer.advancedmd.com/
- **FHIR Developer Portal:** https://devportal.advancedmd.com/
- **API Documentation:** https://devportal.advancedmd.com/api-documentation
- **SDKs / Libraries:** None publicly listed; sandbox environment provided to approved developers.
- **Developer Guide:** https://developer.advancedmd.com/documentation
- **Standards:** FHIR R4 (FHIR APIs at no cost); proprietary XMLRPC and REST for legacy integrations
- **Authentication:** OAuth 2.0 for FHIR API; proprietary API key for legacy endpoints. NDA / Developer Agreement required for full documentation access.

---

### Tebra (formerly Kareo)

- **Description:** PMS combining EHR, scheduling, billing, and patient acquisition tools for small independent practices. Primary API is SOAP-based (legacy Kareo API); a Clinical Open API (REST/Swagger) is also available.
- **API Documentation (SOAP):** https://helpme.tebra.com/Tebra_PM/12_API_and_Integration
- **Clinical Open API (REST / Swagger):** https://api.kareo.com/clinical/v1/swagger/
- **Technical Guide (PDF):** https://kareocustomertraining.s3.amazonaws.com/Help%20Center/Guides/Kareo_IntegrationAPI_TechnicalGuide.pdf
- **SDKs / Libraries:** Community tools available (e.g., KareoTool on GitHub); no official SDK.
- **Standards:** SOAP (primary legacy API); REST/JSON for Clinical Open API; X12 EDI for claims
- **Authentication:** API key / credentials via Tebra developer account; OAuth 2.0 for newer endpoints.

---

### NextGen Healthcare

- **Description:** Ambulatory EHR and PMS for mid-to-large specialty practices, with full revenue cycle management. Offers FHIR-based Patient Access, Bulk FHIR, and SMART App Launch APIs for ONC certification compliance.
- **Developer Portal:** https://www.nextgen.com/api
- **Public API Documentation:** https://dev-cd.nextgen.com/api
- **APIs & FHIR Overview:** https://www.nextgen.com/solutions/interoperability/api-fhir
- **GitHub:** https://github.com/nextgenhealthcare
- **SDKs / Libraries:** Not publicly listed; sandbox environments provided on request.
- **Standards:** FHIR R4 and DSTU2; SMART App Launch; FHIR Bulk Data; USCDI v1+; ONC certified
- **Authentication:** OAuth 2.0 / SMART on FHIR

---

### OpenEMR

- **Description:** Open-source EHR and PMS (GPL-2.0). Provides a comprehensive FHIR R4 API compliant with US Core 8.0 and SMART on FHIR v2.2.0, documented via Swagger/OpenAPI.
- **API Documentation (GitHub):** https://github.com/openemr/openemr/blob/master/API_README.md
- **FHIR API Documentation:** https://github.com/openemr/openemr/blob/master/Documentation/api/FHIR_API.md
- **FHIR Wiki:** https://www.open-emr.org/wiki/index.php/OpenEMR_7.0.2_API
- **SDKs / Libraries:** None; uses standard FHIR HTTP client pattern. Swagger UI available at `/swagger/` endpoint.
- **Standards:** FHIR R4, US Core 8.0, SMART on FHIR v2.2.0, OAuth 2.0 / OIDC
- **Authentication:** OAuth 2.0 with OIDC; API key for backward-compatible legacy endpoints

---

### Availity (Clearinghouse / Payer Connectivity)

- **Description:** Major US healthcare clearinghouse processing 13+ billion transactions annually. Provides REST APIs for eligibility (270/271), claim status (276/277), and claims submission. Connects to 5,000+ payers. Primary integration point for PMS eligibility and claim status workflows.
- **Developer Portal:** https://developer.availity.com
- **API Guide:** https://developer.availity.com/blog/2025/3/25/availity-api-guide
- **EDI Clearinghouse:** https://www.availity.com/edi-clearinghouse/
- **API Marketplace:** https://www.availity.com/api-marketplace/
- **SDKs / Libraries:** REST client; code examples provided in documentation.
- **Standards:** X12 EDI v5010 (270/271, 276/277, 835, 837); REST/JSON for newer APIs
- **Authentication:** OAuth 2.0 (application-only flow); HTTPS required for all API calls.

---

### Office Ally (Clearinghouse / PMS)

- **Description:** US healthcare clearinghouse and PMS provider. Supports EDI transactions to 5,000+ payers via SFTP, SOAP/MIME, and REST APIs. Offers a free basic PMS (Practice Mate) alongside enterprise clearinghouse services. Widely used by small practices.
- **EDI Clearinghouse:** https://cms.officeally.com/clearinghouse
- **Payer Gateway:** https://cms.officeally.com/payer-gateway
- **API Integration:** https://docs.supergood.ai/office-ally-api-2/ (community-documented)
- **SDKs / Libraries:** REST API Companion Guide and Realtime EDI Companion Guide (available via registration).
- **Standards:** X12 EDI v5010 (270/271, 276/277, 835, 837, claim attachments); JSON API; SOAP/MIME; SFTP
- **Authentication:** Credentials-based; SFTP key authentication for file-based EDI.

---

### Jane App

- **Description:** Multi-disciplinary PMS for allied health clinics (physiotherapy, chiropractic, massage, psychology). Operates a vetted partner API program (Jane Developer Platform). OAuth 2.0 PKCE is required. The API uses date-based URL versioning. Not a public open API — partner approval required.
- **Developer Platform:** https://developers.jane.app/
- **Integrations Program:** https://jane.app/integrations
- **SDKs / Libraries:** None; REST/JSON with date-versioned endpoints.
- **Standards:** REST/JSON; OAuth 2.0 PKCE; HIPAA compliant (US); PIPEDA compliant (Canada)
- **Authentication:** OAuth 2.0 PKCE; practitioner-scoped authorization per clinic.

---

### Surescripts (E-Prescribing Network)

- **Description:** The dominant US e-prescribing network connecting PMS/EHR systems to pharmacies, pharmacy benefit managers, and payers. Implements the NCPDP SCRIPT standard for prescription routing, medication history, and electronic prior authorization. PMS vendors integrate via Surescripts certification rather than direct pharmacy connections.
- **Overview:** https://surescripts.com
- **NCPDP SCRIPT Upgrade Guide:** https://surescripts.com/GetSCRIPT
- **Developer / Certification:** https://surescripts.com (requires partnership agreement and NCPDP certification)
- **Standards:** NCPDP SCRIPT v2023011; Real-Time Prescription Benefit v13; Formulary & Benefit v60
- **Authentication:** Surescripts network credentials; TLS mutual authentication.

---

## Notes

**Regulatory timeline:** USCDI v3 compliance becomes mandatory for ONC-certified health IT on January 1, 2026. The CMS-0057-F Prior Authorization API mandates take effect January 1, 2027. Any PMS roadmap targeting the US market should plan around these deadlines.

**Da Vinci IG suite:** The HL7 Da Vinci project has produced a family of related FHIR IGs (CRD, DTR, PAS, PDex) that together describe the end-to-end prior authorization and payer data exchange workflow. These are emerging as the FHIR-native replacement for X12 278 prior authorization transactions and should be tracked closely for the PA automation feature.

**Clearinghouse choice:** US PMS vendors typically integrate with one or more clearinghouses (Availity, Office Ally, Change Healthcare/Optum, Waystar) rather than establishing direct payer connections. Clearinghouse APIs abstract the X12 EDI complexity and provide connectivity to thousands of payers via a single integration point. Change Healthcare experienced a major ransomware outage in 2024 that disrupted claims processing industry-wide, highlighting the risk of single-clearinghouse dependency.

**TherapyNotes API:** As of May 2026, TherapyNotes does not expose a public developer API. Third-party integrations (e.g., Mentalyc AI scribe) integrate via file export or screen-level automation rather than a programmatic API. This is a significant gap for behavioral health PMS integrators.

**SimplePractice API:** SimplePractice's public API is limited to the SimplePractice Enterprise tier for EAP/MCO network scheduling. Standard practice accounts have no programmatic API access. Third-party tools integrate via OAuth-based read access or export workflows only.
