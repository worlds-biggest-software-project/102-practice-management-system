# Practice Management System — Feature & Functionality Survey

> Candidate #102 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Athenahealth (athenaOne PMS) | Cloud ambulatory PMS + EHR + RCM | Proprietary SaaS | https://www.athenahealth.com |
| AdvancedMD | Integrated EHR + PMS + billing (independent practices) | Proprietary SaaS | https://www.advancedmd.com |
| Tebra (Kareo + PatientPop) | PMS + EHR + reputation management (small practices) | Proprietary SaaS | https://www.tebra.com |
| DrChrono | iPad-native EHR + PMS + billing | Proprietary SaaS | https://www.drchrono.com |
| SimplePractice | EHR + PMS for behavioral health / therapy | Proprietary SaaS | https://www.simplepractice.com |
| TherapyNotes | Behavioral health EHR + PMS + billing | Proprietary SaaS | https://www.therapynotes.com |
| Jane App | Multi-disciplinary clinic PMS + scheduling + billing | Proprietary SaaS | https://jane.app |
| Cliniko | Clinic management system (allied health, multi-location) | Proprietary SaaS | https://www.cliniko.com |
| Office Ally Practice Mate | Basic web PMS + billing (small practices) | Proprietary SaaS (free tier) | https://www.officeally.com |
| OpenEMR | Open-source EHR + PMS | Open Source — GPL-2.0 | https://www.open-emr.org |

## Feature Analysis by Solution

### Athenahealth (athenaOne PMS)

**Core features**
- Cloud-native scheduling: patient appointment booking, recall management, and waitlist management
- Automated insurance eligibility verification (real-time X12 270/271 queries before appointments)
- Medical billing: X12 837 claim submission, ERA posting (835), denial management
- AthenaNet shared network: payer-specific denial rules and billing intelligence pooled across 160,000+ providers
- Ambient Notes: AI-assisted clinical documentation from provider-patient conversation audio
- Patient engagement: portal access, automated appointment reminders, online intake forms
- Population health: care gap reports, value-based care dashboards, chronic disease registries
- Revenue cycle managed service option: athenahealth staff actively work denials

**Differentiating features**
- Network intelligence: denial patterns from the entire athenahealth customer base improve claim scrubbing for every practice automatically
- % of collections pricing aligns vendor incentives with practice revenue — vendor is motivated to maximize clean claim rate
- Fully cloud-native with no on-premise option; no client-managed upgrades
- Athenahealth actively pursues appeals and denial recovery on behalf of customers in managed service tier

**UX patterns**
- Task-based inbox driving daily billing and clinical workflows for administrators and providers
- Payer-specific claim edits suggested automatically before submission
- Patient engagement triggered at workflow events (booking confirmation, day-before reminder, post-visit survey)

**Integration points**
- HL7 FHIR R4 and SMART on FHIR for third-party app connectivity
- X12 EDI 837/835/270/271/276/277 for all payer transactions
- Surescripts for e-prescribing
- NCPDP EPCS for controlled substance prescribing
- Third-party app marketplace

**Known gaps**
- % of collections pricing can become expensive for high-revenue specialty practices
- Less configurable than enterprise systems for complex multi-specialty workflows
- Behavioral health and therapy-specific workflows not as deep as SimplePractice or TherapyNotes
- No inpatient capability

**Licence / IP notes**
- Proprietary SaaS; Athenahealth Inc. (held by Veritas Capital / Evergreen Coast Capital since 2022, $17B)
- HIPAA BAA provided; SOC 2 and HITRUST certified

---

### AdvancedMD

**Core features**
- Integrated scheduling with waitlist management, automated reminders, and multi-location calendar
- Centralized insurance eligibility verification with batch and real-time options
- Medical billing: claim scrubbing, EDI submission, ERA posting, denial management
- Reporting dashboards: financial performance, collections, denial rates, provider productivity
- EHR integration: clinical documentation, e-prescribing, patient portal
- Tiered subscription plans with à la carte add-ons

**Differentiating features**
- AI-powered analytics for revenue cycle performance and operational benchmarking
- Integrated reputation management and patient satisfaction surveying
- Single platform covering scheduling, clinical, billing, and analytics without third-party RCM required
- Multi-specialty template library for clinical documentation

**UX patterns**
- Dashboard-first interface showing financial KPIs, appointment volumes, and denial rates
- Roles-based access: separate views for billing staff, clinical staff, and administrators
- Automated workflow rules triggering reminders, eligibility checks, and claim scrubbing at configurable events

**Integration points**
- EDI clearinghouse integrations (X12 837/835/270/271)
- Surescripts e-prescribing
- Lab integrations via HL7 v2.x
- Third-party telehealth integrations

**Known gaps**
- Pricing at $429–$729/provider/month positions it above solo practitioners' budgets
- Implementation can be complex; onboarding reported as lengthy for multi-provider practices
- Customer support reviews mixed in independent surveys
- Less known outside the US market

**Licence / IP notes**
- Proprietary SaaS; Francisco Partners portfolio company (divested by Global Payments in 2023)
- HIPAA compliant; SOC 2 certified

---

### Tebra (Kareo + PatientPop)

**Core features**
- All-in-one platform: clinical documentation, scheduling, billing, patient engagement, and online reputation management
- AI-based note-taking: ambient documentation and structured clinical notes
- Claims processing: automated claim scrubbing, submission, and denial management
- Digital intake forms and online appointment booking
- Patient marketing tools: SEO, review management, and website builder for independent practices
- Telehealth: integrated video visits

**Differentiating features**
- Unique combination of clinical PMS with patient acquisition tools (SEO, online reputation management inherited from PatientPop)
- AI-assisted charting and review response time reduction
- Targets small/independent practices with an "all revenue operations" bundled approach
- Transparent per-provider pricing ($350+ per month) vs. percentage-of-collections models

**UX patterns**
- Unified dashboard combining clinical workflow with patient acquisition and reputation metrics
- Automated review requests sent post-visit via SMS or email
- Intake forms completed digitally before the appointment; auto-populated into the chart

**Integration points**
- X12 EDI 837/835 for claims
- Surescripts e-prescribing
- Lab integrations
- Google Business Profile integration for reputation management
- Telehealth via embedded video (Zoom-based)

**Known gaps**
- Post-merger product consolidation (Kareo + PatientPop) has generated UX inconsistencies reported by users
- Less suitable for behavioral health specialties compared to SimplePractice or TherapyNotes
- Reputation management and SEO tools of limited value for practices in referral-only specialties
- Enterprise / multi-location practices better served by AdvancedMD or athenahealth

**Licence / IP notes**
- Proprietary SaaS; Tebra Inc. (raised ~$165M total)
- HIPAA compliant; SOC 2 certified

---

### DrChrono

**Core features**
- iPad-native EHR and PMS with scheduling, charting, billing, and payments
- AI-powered documentation: ambient note generation and ICD/CPT code suggestions
- Customizable clinical forms and workflow templates for specialty practices
- Integrated medical billing with claim submission and payment tracking
- Patient portal: online scheduling, digital intake forms, test result access
- Built-in telehealth with video visits

**Differentiating features**
- iPad and iPhone-native interface designed for mobile-first clinical workflows
- Voice-to-text dictation integrated throughout the clinical experience
- AI documentation at a lower price point (~$199/provider/month) vs. comparable features in larger platforms
- Concierge medicine and direct primary care (DPC) pricing models supported

**UX patterns**
- Touch-optimized calendar and chart interface
- Split-screen support on iPad for simultaneous charting and appointment view
- Customizable form builder for specialty-specific intake and assessment workflows

**Integration points**
- FHIR R4 API for interoperability
- Surescripts e-prescribing (EPCS capable)
- Lab interface via HL7 v2.x
- Clearinghouse for EDI claim submission

**Known gaps**
- Not suitable for large group practices or health systems
- Billing capabilities less sophisticated than athenahealth's managed service RCM model
- Mixed customer support reviews since EverCommerce acquisition ($225M, 2022)
- Limited population health or value-based care tools

**Licence / IP notes**
- Proprietary SaaS; EverCommerce subsidiary
- HIPAA compliant; HITRUST certified

---

### SimplePractice

**Core features**
- Appointment scheduling with automated reminders (SMS, email)
- Digital intake forms and consent documents delivered via client portal
- Clinical documentation: therapy-specific note templates (progress notes, treatment plans, psychotherapy notes)
- Billing: superbill generation, insurance claim submission, and payment collection via Stripe integration
- Telehealth: built-in HIPAA-compliant video visits (no download required)
- AI Note Taker: optional add-on for session documentation support

**Differentiating features**
- Purpose-built for behavioral health, therapy, and wellness practitioners
- Lowest-cost entry point in this comparison (~$29/month for solo practitioner)
- Client portal designed for consumer experience: self-scheduling, intake, billing, and video visit in one link
- Wiley Practice Planners integration for treatment planning tools

**UX patterns**
- Clinician-first scheduling calendar with session note workflow integrated
- Automated insurance eligibility verification before each session
- Client portal: single link for scheduling, intake, telehealth, and payments

**Integration points**
- Insurance clearinghouse (Office Ally, Availity) for claim submission
- Stripe for payment processing
- Wiley Practice Planners for evidence-based treatment planning content
- Telehealth built on WebRTC (browser-based, no download)

**Known gaps**
- Not suitable for medical specialties outside behavioral health; lacks medication management, lab orders, e-prescribing
- Limited multi-provider and multi-location features; primarily solo and small group focus
- No advanced analytics or population health tools
- AI Note Taker is a paid add-on, not native to base subscription

**Licence / IP notes**
- Proprietary SaaS; SimplePractice LLC
- HIPAA compliant; SOC 2 certified

---

### TherapyNotes

**Core features**
- Behavioral health-specific EHR with structured note templates: initial assessments, progress notes, discharge summaries, treatment plans
- Appointment scheduling with recurring appointment support
- Insurance billing: batch claim submission, ERA posting, and denial tracking
- Client portal: online scheduling, intake forms, and homework assignment
- Secure HIPAA-compliant messaging between therapist and client
- Built-in telehealth with browser-based video sessions

**Differentiating features**
- Clinical-grade documentation system that meets requirements for insurance billing of behavioral health services
- Wide template library for psychology, social work, counseling, and psychiatry disciplines
- Insurance billing managed within the platform with real-time eligibility and ERA posting
- Strong reputation for reliability and customer support among behavioral health practitioners

**UX patterns**
- Structured note-writing workflow that enforces required fields for insurance-compliant documentation
- Calendar-integrated note workflow: click appointment, open note, complete and sign
- Supervisor co-signature workflow for supervised clinicians (important for intern/trainee settings)

**Integration points**
- Insurance clearinghouse for X12 EDI 837 submissions
- Telehealth built on WebRTC
- Limited third-party integrations; primarily standalone

**Known gaps**
- Limited to behavioral health; not suitable for medical practices
- No AI documentation features as of 2026 (significantly behind Tebra and SimplePractice on this dimension)
- No patient-facing mobile app
- Limited analytics and reporting depth beyond billing reports

**Licence / IP notes**
- Proprietary SaaS; TherapyNotes LLC
- HIPAA compliant

---

### Jane App

**Core features**
- Scheduling: online booking, multi-practitioner calendar, multi-location support, group appointment scheduling
- Clinical charting: SOAP notes, customizable intake forms, telehealth notes
- Billing: insurance billing (Canada and US), batch claim submission, and payment processing
- Patient communication: automated SMS / email reminders, digital intake forms, secure messaging
- Telehealth: integrated browser-based video visits
- AI Scribe: clinical note documentation support (released 2025)

**Differentiating features**
- Purpose-built for multi-disciplinary allied health clinics (physiotherapy, chiropractic, massage, occupational therapy, psychology)
- Strong in Canadian market with provincial billing integration (BC MSP, OHIP, AHCIP, RAMQ)
- Multi-location support with inventory management for products and supplies
- Group class and workshop scheduling for wellness and fitness providers
- 2026 Best Practice Management Software recognition in several independent surveys

**UX patterns**
- Color-coded multi-practitioner scheduling grid designed for front-desk staff
- Patient self-booking with practitioner and location selection
- Automated intake form delivery triggered by booking confirmation

**Integration points**
- Insurance clearinghouses (Canadian provincial billing systems; US clearinghouses)
- Stripe for payment processing
- Mailchimp for patient communication campaigns
- FHIR API (limited; expanding)

**Known gaps**
- Less suited for US medical billing complexity (ICD-10-PCS, CMS compliance, complex denial management) compared to athenahealth or AdvancedMD
- AI documentation features (AI Scribe) newer and less proven than established competitors
- No ONC certification; not suitable for US Meaningful Use programs
- Limited advanced analytics and revenue cycle intelligence

**Licence / IP notes**
- Proprietary SaaS; Jane App Inc. (Canadian company)
- HIPAA and PIPEDA compliant

---

### Cliniko

**Core features**
- Multi-location appointment scheduling with practitioner and room management
- Clinical notes: customizable SOAP note templates per discipline
- Automated patient reminder notifications (SMS, email)
- Patient record management: demographics, history, insurance details
- Online booking portal with practitioner and location selection
- Practice analytics: appointment volume, revenue, cancellation rates by practitioner and location

**Differentiating features**
- Designed for high-volume allied health clinics (physiotherapy, chiropractic, osteopathy)
- Strong multi-location management: single admin view across all sites
- Simple, modern UX highly rated for ease of adoption by non-technical clinic staff
- Popular in Australia, UK, and New Zealand markets

**UX patterns**
- Calendar-centric interface optimized for front-desk staff managing multi-room, multi-practitioner schedules
- Waitlist management with automated patient notification when slots open
- Per-practitioner analytics dashboards

**Integration points**
- Stripe and Square for payment processing
- Mailchimp and patient communication tools
- Limited EHR interoperability; primarily standalone
- Xero and MYOB for accounting

**Known gaps**
- No US insurance billing / EDI capability; not suitable for US practices billing private insurance or Medicare
- No e-prescribing
- No ONC certification
- Limited AI features
- No built-in telehealth as of 2026

**Licence / IP notes**
- Proprietary SaaS; Cliniko Pty Ltd (Australian company)
- HIPAA compliant for US customers; Australian Privacy Act compliant

---

### Office Ally Practice Mate

**Core features**
- Appointment scheduling with multi-provider calendar
- Patient demographic management
- Insurance billing: X12 EDI 837 claim submission via Office Ally clearinghouse (free)
- ERA posting and payment management
- Claim status inquiry and tracking
- Basic reporting: outstanding claims, appointment volumes

**Differentiating features**
- Free basic tier: the only major PMS in this comparison with a fully functional free plan
- Built-in clearinghouse integration with Office Ally (major US clearinghouse) eliminates a separate vendor
- Widely used by very small practices and solo practitioners who need US insurance billing without per-provider monthly fees

**UX patterns**
- Traditional web forms interface; functional but not modern
- Clearinghouse-integrated claim submission without additional setup

**Integration points**
- Office Ally clearinghouse (native; free EDI submission to 5,000+ payers)
- Practice Mate integrates with Office Ally EHR for clinical documentation

**Known gaps**
- No AI features
- No telehealth
- No patient portal with modern patient engagement (reminders, online booking)
- UX is dated compared to modern SaaS competitors
- Limited reporting depth

**Licence / IP notes**
- Proprietary SaaS (free tier); Office Ally LLC
- HIPAA compliant

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Appointment scheduling with automated patient reminders (SMS, email)
- Patient demographics management and searchable patient database
- Insurance eligibility verification (real-time and batch via X12 270/271)
- Medical billing: X12 EDI 837 claim submission and ERA 835 posting
- Patient portal with online scheduling and digital intake forms
- HIPAA-compliant secure messaging between practice and patient
- Basic practice analytics: collections, appointment volumes, denial rates
- Role-based access control (admin, billing, clinical, front-desk)

### Differentiating Features
- Network-intelligence claim scrubbing using cross-customer denial data (Athenahealth — unique)
- AI ambient documentation integrated in the PMS workflow (Tebra, DrChrono, Athenahealth Ambient Notes)
- Patient acquisition tools: SEO, online reputation management, review automation (Tebra — unique among PMS platforms)
- % of collections pricing with managed RCM services (Athenahealth — unique model)
- Provincial billing integration for Canadian markets (Jane App — unique)
- AI-powered no-show prediction and waitlist backfill (emerging; Mend leads but not in this set)
- Therapy-specific structured documentation with supervisor co-signature (TherapyNotes — unique)
- Multi-location inventory and class scheduling for allied health / wellness (Jane App)

### Underserved Areas / Opportunities
- Intelligent scheduling optimization: AI prediction of no-show probability per patient to dynamically fill cancellation slots in real time
- Automated prior authorization: AI agents that detect PA-required procedures and initiate the authorization before the appointment
- Conversational patient intake: AI-powered voice or chat agent handling appointment booking, intake form completion, and insurance capture with no front-desk involvement
- Real-time denial root-cause feedback to clinical staff at the point of documentation (not after claim rejection)
- Revenue cycle anomaly detection: AI surfacing undercoding, missed charges, and unusual denial spikes without requiring a data analyst
- Unified PMS that serves both US insurance-billing and international private-pay markets in one product

### AI-Augmentation Candidates
- No-show and cancellation prediction per patient: fill cancellation slots before they happen
- Automated insurance eligibility and prior authorization initiation at booking time
- Real-time claim scrubbing with AI payer-rule modeling before submission
- Ambient clinical documentation from encounter audio → structured note + billing code suggestion
- Conversational patient intake and triage via voice or chat agent
- Revenue leakage detection: AI audit of charge capture completeness across providers

## Legal & IP Summary

- **GPL-2.0 (OpenEMR)**: Strong copyleft — see EHR features.md for full analysis. Safe approach is API-only integration.
- All other platforms in this comparison are **proprietary SaaS** with no open-source components exposed.
- **X12 EDI transaction sets** (837, 835, 270/271, etc.): X12 is a standards body. Using X12 transactions for HIPAA-mandated transactions is required and legal; reproducing the X12 specification documents without a licence is not. Implementation against the standard is permitted.
- **CPT codes (AMA)**: Required for US medical billing. An AMA licence is needed to display or process CPT codes in software. This is a recurring IP cost any US-market PMS must budget for.
- **ICD-10-CM**: US government-published; free to use.
- **NCPDP SCRIPT** (e-prescribing): Requires NCPDP membership or a licence for implementation; most PMS vendors handle this via Surescripts integration which abstracts the standard.
- No GPL or AGPL tools in this comparison present embedding concerns beyond OpenEMR.

## Recommended Feature Scope

**Must-have (MVP)**:
- Appointment scheduling with multi-provider calendar, online booking, and automated SMS/email reminders
- Real-time insurance eligibility verification (X12 270/271) triggered at booking and day-before appointment
- Medical billing pipeline: charge entry, X12 837 claim submission, 835 ERA posting, and denial tracking dashboard
- Patient demographics and intake: digital forms completed before arrival, auto-populated into the record
- HIPAA-compliant patient messaging and basic patient portal (appointment management, balance view)
- Role-based access control with audit logging meeting HIPAA Security Rule requirements

**Should-have (v1.1)**:
- AI-powered no-show prediction with automated waitlist backfill when cancellation detected
- Automated prior authorization detection and initiation for common procedure types
- AI ambient documentation: encounter audio → structured clinical note draft + CPT/ICD-10 code suggestions
- Real-time AI claim scrubbing with payer-specific rule modeling before submission
- Revenue cycle analytics: denial rate by payer and code, collection rate by provider, undercoding flags

**Nice-to-have (backlog)**:
- Conversational AI patient intake agent (voice or chat) handling booking, insurance capture, and symptom collection
- Patient acquisition tools: online reputation management and automated post-visit review requests
- Multi-location inventory management for allied health product dispensing
- Provincial billing integration for Canadian markets (British Columbia MSP, Ontario OHIP, Alberta AHCIP)
- Population health dashboard: care gap reports and chronic disease registry for value-based care contracts
