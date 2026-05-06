# Practice Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source practice management system for appointment scheduling, billing, insurance claims, and patient communication.

Practice Management System is an open-source platform for medical and allied-health practices that unifies scheduling, insurance eligibility, claim submission, and patient engagement in a single workflow. It targets independent and mid-size practices that today pay 4–7% of collections or hundreds of dollars per provider per month for proprietary SaaS, and it uses AI to attack the largest sources of administrative waste: no-shows, prior authorizations, and claim denials.

---

## Why Practice Management System?

- Incumbents like Athenahealth charge approximately 4–7% of collections, which becomes punitive for high-revenue specialty practices, while AdvancedMD lists at $429–$729 per provider per month — pricing that locks out solo and small practices.
- Existing products are siloed by specialty: SimplePractice and TherapyNotes serve behavioral health, Jane App serves Canadian allied-health clinics, and Office Ally Practice Mate is free but dated; no open-source alternative offers a modern, AI-native experience across these segments.
- AI capabilities are unevenly distributed across incumbents: TherapyNotes has no AI documentation, Cliniko has limited AI, and where AI exists (Tebra, DrChrono, Athenahealth Ambient Notes) it is bolted to proprietary platforms.
- OpenEMR, the leading open-source option, is GPL-2.0 and carries a dated UX; the market lacks a permissively licensed, modern PMS that practices can self-host or run as SaaS.
- Underserved opportunities — AI no-show prediction with waitlist backfill, automated prior authorization, real-time denial root-cause feedback, and conversational patient intake — are not addressed end-to-end by any single incumbent.

---

## Key Features

### Scheduling and Patient Access

- Multi-provider, multi-location appointment calendar with online self-booking
- Automated SMS and email reminders, recall management, and waitlist handling
- Digital intake forms delivered before the appointment and auto-populated into the patient record
- HIPAA-compliant patient portal for appointment management, balance view, and secure messaging

### Billing and Revenue Cycle

- Charge entry and X12 EDI 837 claim submission (Professional and Institutional)
- Real-time insurance eligibility verification via X12 270/271 at booking and day-before
- ERA 835 posting, denial tracking dashboard, and claim status inquiries (276/277)
- Revenue cycle analytics: denial rate by payer and code, collection rate by provider, undercoding flags

### Clinical and Documentation Workflow

- Structured note templates with role-based workflows for billing, clinical, and front-desk users
- AI ambient documentation: encounter audio to structured clinical note draft with CPT and ICD-10 code suggestions
- Supervisor co-signature workflow for supervised clinicians
- Customizable forms for specialty-specific intake and assessment

### Compliance and Access Control

- Role-based access control with audit logging that meets HIPAA Security Rule requirements
- HIPAA-compliant secure messaging between practice and patient
- Support for the HIPAA-mandated transaction set (837/835/270/271/276/277)
- Architecture aligned with ONC 21st Century Cures Act patient-access API requirements

### Patient Engagement (Backlog)

- Conversational AI patient intake agent for booking, insurance capture, and symptom collection
- Automated post-visit review requests and reputation management
- Multi-location inventory management for allied-health product dispensing
- Population health dashboard for care gap reports and chronic-disease registries

---

## AI-Native Advantage

Unlike incumbents where AI is added to legacy workflows, this project treats AI as core infrastructure: per-patient no-show prediction with automated waitlist backfill before cancellations occur, automated detection and initiation of prior authorizations at booking, real-time claim scrubbing that models payer-specific rules before submission, and revenue cycle anomaly detection that surfaces undercoding and missed charges without a data analyst. Ambient documentation converts encounter audio into structured notes with CPT and ICD-10 suggestions, and a conversational intake agent can absorb 40–60% of front-desk call volume. These capabilities target the largest sources of administrative waste in current practice management workflows.

---

## Tech Stack & Deployment

The project is designed to support both self-hosted and cloud deployment, with HIPAA-compliant operation as a baseline. Integration is built on HL7 FHIR R4 and SMART on FHIR for app interoperability, HL7 v2.x SIU/SCH for legacy hospital scheduling, X12 EDI 837/835/270/271/276/277 for payer transactions, NCPDP SCRIPT (typically via Surescripts) for e-prescribing, and IHE Scheduling FHIR profiles for cross-system appointments. CPT code processing requires an AMA licence — a recurring IP cost that any US-market PMS must budget for; ICD-10-CM is freely usable.

---

## Market Context

The global practice management system market was valued at approximately USD 14.45 billion in 2024 and is projected to reach USD 25.54 billion by 2030 at a CAGR of about 10.2% (Grand View Research); a separate estimate puts the 2025 market at USD 17.06 billion growing to USD 44.40 billion by 2035 (Towards Healthcare). Incumbent pricing ranges from free (Office Ally Practice Mate) and $29/month (SimplePractice solo) at the low end to $500–$1,200/provider/month or 4–7% of collections at the mid-to-enterprise end. Primary buyers are practice administrators at independent physician groups, revenue cycle managers at specialty practices, behavioral health practice owners, and IT directors at community health centers and FQHCs.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
