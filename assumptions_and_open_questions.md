# OSCS — Assumptions and Open Questions
**System:** Outpatient Patient Scheduling & Care System (OSCS)
**Owner:** ABC Health Care Company
**PRD Version:** 1.0 (December 20, 2025)

This document consolidates all assumptions made where the PRD is silent or ambiguous,
and all open questions that require stakeholder confirmation before implementation.

---

## Assumptions

These were applied as reasonable defaults in the user stories, test cases, and API spec.
Each requires explicit confirmation or override.

### Authentication and Session Management

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-001 | OTP expiry | OTP TTL is 10 minutes. | PRD says OTPs are used but does not specify expiry. |
| A-002 | Login lockout | Account locks after 5 consecutive failed login attempts. | PRD says "rate-limited" but does not give a threshold. |
| A-003 | MFA method | TOTP (authenticator app) is the supported second factor for staff. | PRD says "optional MFA for staff" but does not specify method. |
| A-004 | Session timeout | Sessions expire on idle; specific timeout is configurable by admin. | PRD says "configurable session timeouts" — no default given. |

### RBAC and Permissions

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-005 | Compliance officer | Compliance officer access is mapped to the System Admin role for audit log queries. | PRD lists System Admin/IT but does not define a Compliance Officer role. |
| A-006 | Patient self-scheduling scope | Patients can only view/modify their own records through the portal. | PRD implies this but does not explicitly state cross-patient isolation. |
| A-007 | Billing staff access | Billing staff (optional for MVP) have read-only access to appointment and financial data. | PRD marks this role "optional" with no permission definition. |

### Scheduling and Calendar

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-008 | Waitlist priority | Waitlist offers are sent in chronological enrollment order (first-in, first-offered) unless the clinic configures an alternative priority algorithm. | FR-105 says "priority order" without defining it. |
| A-009 | Waitlist offer TTL | Default waitlist offer expiry is 30 minutes. Configurable per clinic. | FR-105 says "expire after a configurable time" — no default stated. |
| A-010 | Availability search window | Maximum date range for a single availability search call is 90 days. | PRD mentions search but does not cap the date range. |
| A-011 | Reschedule window | Reschedule and cancellation policy windows are expressed in hours lead time (e.g., 24 hours). | PRD says "within configured policy windows" — no unit or example given. |
| A-012 | Double-booking prevention | The system uses optimistic locking / atomic reservation at the database level to prevent double-booking. | PRD says "double-booking is prevented" — implementation mechanism not specified. |

### Intake and Forms

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-013 | Draft form saving | Partially completed forms are saved as drafts; only submitted forms are immutable. | PRD says submitted forms are immutable and versioned but does not address drafts. |
| A-014 | Signature methods | Patients may draw, type, or upload a signature image for e-consent. | FR-005 requires e-signature but does not specify the accepted signature methods. |

### Notifications

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-015 | PHI minimum necessary in SMS | SMS content includes only: provider first name (no last name), appointment date/time, clinic name, and a portal link. Diagnosis, medications, and full DOB are not included. | FR-401 and NFR require HIPAA minimum-necessary but do not define the content boundary. |
| A-016 | Default reminder timing | Default reminder intervals are 72 hours and 24 hours before appointment. Both configurable. | FR-401 says "configurable timing" — no default given. |

### Performance and Infrastructure

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-017 | Typical clinic data volume | For the performance test baseline, "typical clinic data volumes" is defined as: up to 50 providers, up to 500 appointment slots per day, up to 10,000 active patients. | PRD references "typical clinic data volumes" for the 2-second SLA without defining scale. |
| A-018 | Availability SLA method | The 2-second response time target is measured at the p95 (95th percentile) under the baseline load. | PRD states "under 2 seconds" but does not specify percentile or load conditions. |

### Data Retention

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-019 | Audit log retention | Audit logs are retained for a minimum of 7 years (consistent with common HIPAA implementation guidance). | PRD says "retained per ABC Health Care Company policy" — no duration stated. |
| A-020 | PHI retention | Patient records and appointment history are retained per the clinic's data retention policy; deactivated accounts are not hard-deleted until the retention period expires. | PRD does not specify PHI retention duration. |

### Integrations

| ID | Area | Assumption Applied | PRD Gap |
|----|------|--------------------|---------|
| A-021 | EHR source of truth | EHR/EMR is the source of truth for patient demographics; OSCS syncs from EHR on initial load and on EHR change events. OSCS changes to demographics are written back to EHR. | FR-701 says to "define source-of-truth" — no direction stated in PRD. |
| A-022 | Telehealth link generation | For tele-visit types, a meeting link is generated at booking time and attached to the appointment notification. The specific telehealth platform (Zoom, Teams, etc.) is not assumed. | FR-703 specifies link generation but not the platform or timing. |

---

## Open Questions

These require explicit decisions from ABC Health Care Company stakeholders before the corresponding features are built.

### High Priority (Blocks Implementation)

| ID | Question | PRD Section | Impacted Features |
|----|----------|-------------|-------------------|
| OQ-001 | What is the EHR/EMR system being integrated? The integration mechanism (HL7 v2 vs. FHIR), message broker, and conflict resolution strategy depend on this. | FR-701 | EHR integration (FR-701), patient demographic sync, appointment sync |
| OQ-002 | Who is the source of truth for patient demographics — OSCS or the EHR? If the EHR is master, what is the latency requirement for syncing changes to OSCS? | FR-701 | Patient profile (FR-002), scheduling constraints (FR-107) |
| OQ-003 | What SMS gateway and email provider will be used? This determines API keys, character limits, delivery receipt format, and opt-out handling mechanics. | FR-702 | All notification flows (FR-401, FR-402) |
| OQ-004 | Is MFA mandatory or optional per role? Which specific roles require it at launch? | Section 7 NFR | Authentication (FR-001), staff login (US-002, US-039) |
| OQ-005 | What is the cancellation/reschedule policy window (in hours) for each visit type? Should this be configurable per visit type, per provider, or per clinic? | FR-004 | Reschedule (US-007), cancellation (US-008) |
| OQ-006 | What are the specific waitlist priority rules beyond chronological order? Are there clinical criteria (e.g., urgency, diagnosis code) that affect priority? | FR-105 | Waitlist (US-015, US-016) |

### Medium Priority (Needed for Design Finalization)

| ID | Question | PRD Section | Impacted Features |
|----|----------|-------------|-------------------|
| OQ-007 | What payer/insurance integrations are in scope for MVP vs. Phase 2? FR-704 marks eligibility checks as optional — is this confirmed out of scope for MVP? | FR-704 | Insurance eligibility, scheduling constraints (FR-107) |
| OQ-008 | Is the co-pay collection integration (FR-202) in scope for MVP? If so, what payment processor? | FR-202 | Check-in workflow |
| OQ-009 | What is the overbooking cap per slot? Can a slot be overbooked by more than one appointment? | FR-106 | Overbooking (US-017) |
| OQ-010 | Which telehealth platform(s) will be integrated for tele-visit link generation? Is this MVP or Phase 2? | FR-703 | Telehealth visit booking |
| OQ-011 | For the BI read-only API (FR-503), is this a separate API key/token type, or a scoped JWT? What data entities are exposed? | FR-503 | Reporting and analytics |
| OQ-012 | What is the geo-fence radius default for mobile self check-in? Who configures it — per location or globally? | FR-202 | Mobile self check-in (US-023) |

### Low Priority (Can Defer to Phase 2 or Post-MVP)

| ID | Question | PRD Section | Impacted Features |
|----|----------|-------------|-------------------|
| OQ-013 | Will IVR (Interactive Voice Response) notifications ever be added? If yes, what IVR provider? | FR-401 | Notification engine |
| OQ-014 | Does the care team messaging feature (FR-304) require a ticketing system integration (e.g., ServiceNow, Zendesk)? | FR-304 | Care team messaging |
| OQ-015 | What is the required RPO (Recovery Point Objective) and RTO (Recovery Time Objective) for OSCS data backups? | Section 7 NFR Reliability | Disaster recovery planning |
| OQ-016 | Should the patient portal support proxy access (e.g., a parent managing a minor's appointments)? This is common in outpatient settings but not mentioned in the PRD. | FR-001, FR-002 | Patient registration, profile, scheduling |
| OQ-017 | Are there specific HIPAA Business Associate Agreements (BAAs) in place with the SMS gateway and email provider? This determines what PHI can appear in notifications. | FR-401, FR-402, NFR | All communication flows |
