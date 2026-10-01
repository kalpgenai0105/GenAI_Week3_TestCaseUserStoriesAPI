# OSCS User Stories
**System:** Outpatient Patient Scheduling & Care System (OSCS)
**Owner:** ABC Health Care Company
**PRD Version:** 1.0 (December 20, 2025)

All story IDs use the format US-NNN. Each story cites its PRD requirement.

---

## A. Patient Portal and Access

### US-001 — Patient Account Registration
**PRD Ref:** FR-001  
**As a** patient,  
**I want** to create an OSCS account using my email address or phone number,  
**so that** I can access the patient portal to schedule and manage my appointments.

**Acceptance Criteria:**
- AC1: Patient can register with a valid email address OR a valid mobile phone number (or both).
- AC2: After submitting registration, an OTP is sent to the chosen contact method within 60 seconds.
- AC3: The OTP expires after a configurable time window (assumption: 10 minutes).
- AC4: Account is not activated until OTP is successfully verified.
- AC5: A successful registration event is written to the audit log (actor, timestamp, contact method used — no raw OTP logged).
- AC6: Failed registration attempts (e.g., duplicate email) return a descriptive but non-leaking error.

---

### US-002 — Patient Login and Rate Limiting
**PRD Ref:** FR-001  
**As a** patient,  
**I want** to log in to my portal account securely,  
**so that** only I can access my personal health information.

**Acceptance Criteria:**
- AC1: Patient can log in with registered email/phone and password (or OTP if configured).
- AC2: After N consecutive failed login attempts (configurable; assumption: 5), the account is temporarily locked and the patient is notified.
- AC3: Every login attempt (success or failure) is written to the authentication audit log with actor, IP, timestamp, and outcome.
- AC4: Sessions expire after the configured idle timeout.

---

### US-003 — Patient Profile Management
**PRD Ref:** FR-002  
**As a** patient,  
**I want** to update my demographics, contact information, preferred language, and communication preferences in my profile,  
**so that** the clinic has accurate information and contacts me in my preferred way.

**Acceptance Criteria:**
- AC1: Patient can update: first name, last name, date of birth, address, phone, email, preferred language, and notification preferences (SMS opt-in, email opt-in).
- AC2: Changes to demographics are audit logged (before/after values).
- AC3: PHI fields (DOB, address) are only editable by the authenticated patient or authorized staff.
- AC4: Preferred language selection applies to portal UI language where supported.

---

### US-004 — Emergency Contact Management
**PRD Ref:** FR-002  
**As a** patient,  
**I want** to add and update my emergency contact information,  
**so that** the clinic can reach a trusted person if needed during my care.

**Acceptance Criteria:**
- AC1: Patient can add up to one primary emergency contact (name, relationship, phone number).
- AC2: Emergency contact information is stored as part of the Patient record.
- AC3: Emergency contact data is not used for marketing or non-emergency communications.
- AC4: Changes to emergency contact are audit logged.

---

### US-005 — Patient Self-Scheduling (Instant Booking)
**PRD Ref:** FR-003  
**As a** patient,  
**I want** to search for available appointment slots by visit type, location, provider preference, and time window, and instantly book an eligible slot,  
**so that** I can secure my appointment without calling the clinic.

**Acceptance Criteria:**
- AC1: Search results display only eligible, available slots (no already-booked or blocked times shown).
- AC2: Patient can filter by: visit type, preferred provider, preferred location, and date/time window.
- AC3: Booking a slot is atomic — two simultaneous bookings of the same slot result in exactly one success and one conflict error.
- AC4: A confirmation notification (SMS and/or email per preference) is sent within 2 minutes of booking.
- AC5: The booking action is audit logged.
- AC6: Visit types configured as staff-only are not displayed to patients in self-scheduling.

---

### US-006 — Request-and-Approve Scheduling
**PRD Ref:** FR-003  
**As a** patient,  
**I want** to submit an appointment request for visit types that require staff approval,  
**so that** I can request care even when instant booking is not available.

**Acceptance Criteria:**
- AC1: For visit types requiring approval, patient submits a request with preferred date/time range, visit type, provider/location preference, and reason.
- AC2: The request enters "Requested" state and is routed to the appropriate scheduling staff queue.
- AC3: Patient receives a notification that their request is under review.
- AC4: Staff can approve (converting to a Scheduled appointment) or deny with a reason.
- AC5: Patient is notified of the decision.

---

### US-007 — Appointment Reschedule (Patient)
**PRD Ref:** FR-004  
**As a** patient,  
**I want** to reschedule my appointment within the policy window,  
**so that** I can adjust my visit time without losing my appointment entirely.

**Acceptance Criteria:**
- AC1: Patient can reschedule if the current appointment is more than the configured minimum lead time in the future.
- AC2: The original slot is released and a new slot is atomically reserved.
- AC3: A reschedule notification is sent to the patient.
- AC4: If the waitlist feature is enabled, the released slot triggers waitlist offer logic.
- AC5: The reschedule event is audit logged.

---

### US-008 — Appointment Cancellation (Patient)
**PRD Ref:** FR-004  
**As a** patient,  
**I want** to cancel my appointment within the policy window and provide a reason,  
**so that** the clinic can reallocate the slot and I am not penalized unfairly.

**Acceptance Criteria:**
- AC1: Patient can cancel within the configured cancellation policy window.
- AC2: Patient is required to select a cancellation reason from a structured list; an optional free-text field is available.
- AC3: Appointment state transitions to "Cancelled."
- AC4: A cancellation confirmation is sent to the patient.
- AC5: If waitlist is enabled, waitlist offer workflow is triggered.
- AC6: The cancellation event (including reason) is audit logged.

---

### US-009 — Digital Intake Forms Completion
**PRD Ref:** FR-005  
**As a** patient,  
**I want** to complete my pre-visit intake forms, questionnaires, and consent documents digitally through the portal,  
**so that** I do not have to fill out paper forms at the clinic.

**Acceptance Criteria:**
- AC1: Patient sees a list of pending intake forms for each upcoming appointment.
- AC2: Patient can complete and submit forms online; partially completed forms are saved as drafts.
- AC3: Required consent forms require e-signature before submission is accepted.
- AC4: Once submitted, forms are immutable — the patient cannot edit a submitted form (a versioned copy is retained).
- AC5: Completion status is visible to clinic staff per appointment.
- AC6: Form responses containing PHI are encrypted at rest and in transit.

---

### US-010 — E-Signature for Consent
**PRD Ref:** FR-005  
**As a** patient,  
**I want** to electronically sign required consent documents,  
**so that** I can provide legally valid consent without a physical signature.

**Acceptance Criteria:**
- AC1: Patient can draw, type, or upload a signature image as configured.
- AC2: Signed documents are versioned and immutable; the signature timestamp and patient identity are stored.
- AC3: Staff can view signed document status (signed/unsigned) per appointment.
- AC4: Signed consent documents are retained per data retention policy.

---

## B. Scheduling and Calendar Management

### US-011 — Provider Calendar Configuration
**PRD Ref:** FR-101  
**As a** clinic manager or system admin,  
**I want** to define working hours, schedule templates, breaks, and buffers for each provider,  
**so that** the scheduling system accurately reflects each provider's availability.

**Acceptance Criteria:**
- AC1: Admin can set recurring weekly working hours per provider.
- AC2: Admin can define schedule templates (e.g., "Mon/Wed General Medicine") and apply effective dates.
- AC3: Template changes with a future effective date do not alter already-published slots.
- AC4: Admin can add breaks and buffer times between appointments.
- AC5: Changes to provider schedules are audit logged.

---

### US-012 — Multi-Location Availability Search
**PRD Ref:** FR-102  
**As a** patient or front desk staff,  
**I want** to search for appointment availability across multiple clinic locations,  
**so that** I can find the most convenient time and location.

**Acceptance Criteria:**
- AC1: Search can be scoped to a specific location or expanded to all accessible locations.
- AC2: Results clearly show provider name, location name, address, and available slot times.
- AC3: Each location's distinct hours and resources are respected in availability calculation.

---

### US-013 — Visit Type Configuration
**PRD Ref:** FR-103  
**As a** clinic manager or system admin,  
**I want** to configure visit types with their default duration, required resources, prerequisites, and allowed booking channels,  
**so that** the scheduling rules reflect clinical needs.

**Acceptance Criteria:**
- AC1: Admin can create/edit visit types with: name, default duration (minutes), required resources, prerequisites (e.g., prior lab), and booking channels (patient self-scheduling, staff-only, or both).
- AC2: Visit types marked staff-only are not visible to patients in the self-scheduling flow.
- AC3: Changes to visit types are audit logged.

---

### US-014 — Resource Reservation on Booking
**PRD Ref:** FR-104  
**As a** front desk staff member,  
**I want** the system to automatically reserve the required rooms and equipment when an appointment is booked,  
**so that** resource conflicts are prevented.

**Acceptance Criteria:**
- AC1: When an appointment is booked, all resources required by the visit type are reserved for the appointment's duration.
- AC2: If a required resource is unavailable (conflict with another appointment), the slot is not offered.
- AC3: Conflicts are detected across provider, room, equipment, and staff resources simultaneously.
- AC4: Resource reservations are released when an appointment is cancelled or rescheduled.

---

### US-015 — Waitlist Enrollment
**PRD Ref:** FR-105  
**As a** patient,  
**I want** to join a waitlist for an appointment with preferred time windows and provider/location preferences,  
**so that** I can be notified if an earlier slot becomes available.

**Acceptance Criteria:**
- AC1: Patient can opt into the waitlist for a specific visit type, with optional provider/location preferences and acceptable time windows.
- AC2: Patient receives confirmation that they are on the waitlist.
- AC3: Waitlist entries are ordered by priority (assumption: chronological enrollment order unless clinic configures otherwise).

---

### US-016 — Waitlist Slot Offer
**PRD Ref:** FR-105  
**As a** patient on the waitlist,  
**I want** to receive a time-limited offer when a slot matching my preferences opens,  
**so that** I can take an earlier appointment.

**Acceptance Criteria:**
- AC1: When a slot opens, the system sends offers to waitlisted patients in priority order.
- AC2: Each offer expires after a configurable time window; if not accepted, the next patient is offered.
- AC3: If the offer expires with no acceptance, the slot is released back to general availability.
- AC4: Accepting the offer books the appointment and moves the patient off the waitlist.
- AC5: Offer send events and acceptance/expiry events are audit logged.

---

### US-017 — Overbooking with Justification
**PRD Ref:** FR-106  
**As a** front desk staff member or clinic manager,  
**I want** to overbook an appointment slot when clinically necessary, with a required justification,  
**so that** urgent patient needs can be accommodated within audit-compliant guardrails.

**Acceptance Criteria:**
- AC1: Overbooking is only available when the overbooking policy allows it (configured by provider, visit type, time-of-day).
- AC2: Staff must enter a free-text justification before the overbooked appointment is saved.
- AC3: Overbooked appointments are visually flagged in the scheduling view and clinic status board.
- AC4: All overbooking actions (including justification and actor) are written to the audit log.

---

### US-018 — Scheduling Constraints Enforcement
**PRD Ref:** FR-107  
**As a** front desk staff member or scheduling system,  
**I want** scheduling constraints (new vs. established patient, age limits, insurance acceptance, referral requirements) to be automatically enforced,  
**so that** only eligible patients can book appointments for applicable visit types.

**Acceptance Criteria:**
- AC1: When a patient attempts to book, applicable constraints are evaluated automatically.
- AC2: If a constraint fails (e.g., patient is "new" but visit type requires "established"), the system blocks the booking and displays a clear reason.
- AC3: For some constraint failures, the system routes the patient to the request-and-review workflow rather than hard-blocking.
- AC4: Constraint rules are configurable per visit type by admin.

---

### US-019 — Appointment Lifecycle State Management
**PRD Ref:** FR-108  
**As a** front desk staff member or clinician,  
**I want** appointment state (Requested, Scheduled, Confirmed, Checked-in, In-room, Completed, Cancelled, No-show) to be accurately tracked and transitioned,  
**so that** all stakeholders have real-time visibility into appointment status.

**Acceptance Criteria:**
- AC1: Only valid state transitions are permitted (e.g., Scheduled → Confirmed → Checked-in → In-room → Completed).
- AC2: Invalid state transitions return a validation error.
- AC3: Every state transition is written to the audit log (previous state, new state, actor, timestamp).
- AC4: The current state is always visible to authorized staff on the appointment record and clinic status board.

---

## C. Intake, Check-in, and Clinic Flow

### US-020 — Pre-visit Intake Reminders
**PRD Ref:** FR-201  
**As a** patient,  
**I want** to receive automated reminders about incomplete pre-visit intake items,  
**so that** I complete forms before my appointment and avoid delays at check-in.

**Acceptance Criteria:**
- AC1: If a patient has incomplete intake items, automated reminders are sent at configured intervals (e.g., 3 days before, 1 day before).
- AC2: Reminders are sent via the patient's preferred channel (SMS and/or email).
- AC3: Reminders stop once all intake items are complete.

---

### US-021 — Staff View of Incomplete Intake Items
**PRD Ref:** FR-201  
**As a** front desk staff member,  
**I want** to see which patients have incomplete intake items for their upcoming appointments,  
**so that** I can follow up proactively or handle them at check-in.

**Acceptance Criteria:**
- AC1: Staff can view a list of appointments with pending intake forms per patient.
- AC2: The view shows which specific items are outstanding (form name, type, required/optional).
- AC3: Access to patient intake data requires the appropriate RBAC role.

---

### US-022 — Front Desk Check-in
**PRD Ref:** FR-202  
**As a** front desk staff member,  
**I want** to check in an arrived patient through the staff interface,  
**so that** the appointment moves to "Checked-in" state and the clinic status board is updated.

**Acceptance Criteria:**
- AC1: Staff can locate a patient's appointment by name, DOB, or appointment ID and mark them as checked in.
- AC2: Appointment state transitions to "Checked-in."
- AC3: Optional: staff can capture insurance card and ID images at check-in.
- AC4: Check-in event is audit logged.

---

### US-023 — Mobile Self Check-in
**PRD Ref:** FR-202  
**As a** patient,  
**I want** to check in for my appointment from my mobile device within the allowed time window,  
**so that** I can skip the front desk queue.

**Acceptance Criteria:**
- AC1: Self check-in is only available within the policy-defined time window (e.g., 30 minutes before to 10 minutes after appointment time).
- AC2: Optional geo-fence enforcement: if configured, self check-in is only permitted when the patient's device is within a configurable radius of the clinic.
- AC3: Successful self check-in transitions the appointment to "Checked-in."
- AC4: Self check-in event is audit logged.

---

### US-024 — Clinic Status Board
**PRD Ref:** FR-203  
**As a** front desk staff member or clinic manager,  
**I want** to view a real-time clinic status board showing all patients and their current flow state,  
**so that** I can manage patient flow and identify bottlenecks.

**Acceptance Criteria:**
- AC1: Status board shows each patient's current state: arrived, waiting, roomed, clinician assigned, completed.
- AC2: Board updates within 5 seconds of any state change under normal load.
- AC3: Staff can filter the board by provider, location, and visit type.
- AC4: Access to the status board is restricted by RBAC role.

---

## D. Care Management

### US-025 — Care Plan Creation
**PRD Ref:** FR-301  
**As a** clinician or care coordinator,  
**I want** to create a care plan tied to a patient's encounter,  
**so that** the patient's goals, interventions, tasks, and follow-up items are documented and trackable.

**Acceptance Criteria:**
- AC1: Care plans can be created by clinicians and authorized care team members tied to a specific encounter.
- AC2: Care plan includes: goals, interventions, tasks, and due dates.
- AC3: Care plan is linked to the patient record and encounter.
- AC4: Creation and modification of care plans are audit logged.
- AC5: Patient (or their authorized representative) can view their own care plan in the portal.

---

### US-026 — Task Assignment and Tracking
**PRD Ref:** FR-302  
**As a** care coordinator or clinician,  
**I want** to assign tasks to patients or staff with due dates, priority, and status tracking,  
**so that** follow-up actions are not lost and accountability is clear.

**Acceptance Criteria:**
- AC1: Tasks have: title, description, assigned owner (patient or staff), due date, priority (high/medium/low), and status (open/in-progress/completed/cancelled).
- AC2: Overdue tasks are automatically escalated per policy (e.g., notification to the task owner and/or their supervisor).
- AC3: Reminders are sent to task owners per configured schedule.
- AC4: Task status changes are audit logged.

---

### US-027 — Follow-up Scheduling from Care Plan
**PRD Ref:** FR-303  
**As a** clinician or care coordinator,  
**I want** to generate follow-up appointment suggestions from care plan tasks and initiate a "schedule next" workflow directly from visit documentation,  
**so that** follow-up visits are scheduled before the patient leaves.

**Acceptance Criteria:**
- AC1: From an encounter/care plan view, clinician can trigger a "schedule next" workflow.
- AC2: The system suggests follow-up appointment types based on care plan tasks (if configured).
- AC3: The scheduling flow pre-fills visit type and preferred timeframe from the care plan.

---

### US-028 — Care Team Messaging
**PRD Ref:** FR-304  
**As a** clinician or care coordinator,  
**I want** to send internal messages to care team members with @mentions,  
**so that** I can coordinate care without leaving the system.

**Acceptance Criteria:**
- AC1: Internal messages can be sent between authorized care team members.
- AC2: Messages support @mention of other team members.
- AC3: Internal messages are separate from patient-facing communications.
- AC4: Messages containing PHI are stored securely and access-controlled by RBAC.

---

## E. Communication and Notifications

### US-029 — Appointment Notification Engine
**PRD Ref:** FR-401  
**As a** patient,  
**I want** to receive SMS and/or email notifications for appointment confirmations, reminders, intake pending alerts, cancellation/reschedule notices, waitlist offers, and post-visit instructions,  
**so that** I am always informed about my healthcare appointments.

**Acceptance Criteria:**
- AC1: Notifications are sent via SMS and/or email based on the patient's opt-in preferences.
- AC2: Reminder timing is configurable (e.g., 72 hours, 24 hours before appointment).
- AC3: Patients who have opted out of a notification type do not receive that type.
- AC4: Delivery status (sent, delivered, failed) is tracked for each notification.
- AC5: Notification content does not include more PHI than necessary (minimum necessary standard).

---

### US-030 — Two-Way Patient Messaging
**PRD Ref:** FR-402  
**As a** patient,  
**I want** to reply to appointment SMS messages or send messages through the portal,  
**so that** I can communicate with my clinic without calling.

**Acceptance Criteria:**
- AC1: Patient replies to SMS are received by the system and routed to the appropriate staff inbox.
- AC2: Keyword routing handles common responses: "CANCEL" triggers cancellation workflow; "CONFIRM" confirms the appointment.
- AC3: Patient messages submitted via the portal are stored securely.
- AC4: Messages containing PHI are encrypted at rest and in transit, and are included in audit logs.
- AC5: Staff can view and respond to patient messages from the staff inbox.

---

## F. Reporting and Analytics

### US-031 — Operational Dashboards
**PRD Ref:** FR-501  
**As a** clinic manager,  
**I want** to view operational dashboards showing no-show rate, late cancel rate, utilization, lead time, fill rate, and waitlist conversion,  
**so that** I can identify trends and take action to improve clinic efficiency.

**Acceptance Criteria:**
- AC1: Dashboard displays all specified metrics at clinic, provider, and visit-type level.
- AC2: Date range filters are available.
- AC3: Access to specific provider-level data is restricted by RBAC role.
- AC4: Dashboard data does not expose individual patient PHI in aggregate views.

---

### US-032 — Clinical Follow-through Reporting
**PRD Ref:** FR-502  
**As a** clinic manager or clinical lead,  
**I want** to view task completion rates, follow-up adherence, and referral completion metrics,  
**so that** I can ensure patients are receiving the follow-up care they need.

**Acceptance Criteria:**
- AC1: Reports show task completion rates at clinic and provider level.
- AC2: Role restrictions prevent front-desk staff from viewing clinical follow-through data.
- AC3: Data is filterable by date range, provider, and visit type.

---

### US-033 — Data Export
**PRD Ref:** FR-503  
**As a** clinic manager or authorized staff,  
**I want** to export report data to CSV,  
**so that** I can perform additional analysis in external tools.

**Acceptance Criteria:**
- AC1: CSV export is available only to roles with the export permission.
- AC2: Every export action is audit logged (actor, timestamp, data scope exported).
- AC3: Exported data is scoped by the user's RBAC permissions — a user cannot export data beyond their access level.

---

## G. Administration and Configuration

### US-034 — Clinic Setup and Administration
**PRD Ref:** FR-601  
**As a** system admin or clinic manager,  
**I want** to manage locations, operating hours, holidays, rooms, equipment, providers, roles, specialties, visit types, intake requirements, and policy rules,  
**so that** the system accurately reflects the clinic's operational structure.

**Acceptance Criteria:**
- AC1: Admin can CRUD (create, read, update, deactivate) all listed entities.
- AC2: Deactivating a location or provider does not delete historical appointment records.
- AC3: All admin configuration changes are audit logged.
- AC4: Only users with the system admin or clinic manager role can access administration screens.

---

### US-035 — Template Management
**PRD Ref:** FR-602  
**As a** system admin or clinic manager,  
**I want** to create and manage appointment schedule templates and form templates with versioning and effective dates,  
**so that** changes take effect on the correct date without disrupting existing appointments.

**Acceptance Criteria:**
- AC1: Templates have a version number and an effective date.
- AC2: A new version of a template does not affect appointments already scheduled under a prior version.
- AC3: Form templates are versioned; form responses reference the version of the form that was presented.

---

### US-036 — Audit Log Access
**PRD Ref:** FR-603, Section 7 (Compliance)  
**As a** system admin or compliance officer,  
**I want** to query and export audit logs for all PHI-relevant actions,  
**so that** ABC Health Care Company can demonstrate compliance and investigate incidents.

**Acceptance Criteria:**
- AC1: Audit logs capture: entity type, entity ID, action (create/update/delete), actor (user ID and role), timestamp, before/after values.
- AC2: Audit logs are immutable — no user, including system admins, can edit or delete audit log entries.
- AC3: Access to audit logs is restricted to authorized roles.
- AC4: Audit log export is itself audit logged.
- AC5: Audit logs are retained per ABC Health Care Company's data retention policy.

---

## H. Integrations

### US-037 — EHR/EMR Integration
**PRD Ref:** FR-701  
**As a** system admin,  
**I want** the OSCS to exchange patient and appointment data with the existing EHR/EMR using HL7 v2 and/or FHIR,  
**so that** the clinical record remains the source of truth for patient data.

**Acceptance Criteria:**
- AC1: Patient demographics synced from EHR to OSCS (or vice versa, per source-of-truth configuration).
- AC2: Appointment data is synchronized between OSCS and EHR per the defined sync strategy.
- AC3: Integration failures are queued for retry and generate operational alerts.
- AC4: Data exchanged with the EHR is encrypted in transit (TLS 1.2+).

---

### US-038 — Role-Based Access Control (RBAC)
**PRD Ref:** Section 2 (Access model), Section 7 (NFR Security)  
**As a** system admin,  
**I want** to assign roles to users with least-privilege permissions,  
**so that** each user can only access data and functions appropriate to their role.

**Acceptance Criteria:**
- AC1: Roles defined: Patient, Front Desk/Scheduling Staff, Clinician, Nurse/Care Coordinator, Clinic Manager, System Admin, Billing Staff.
- AC2: Each role has a defined permission set; users cannot self-escalate permissions.
- AC3: RBAC is enforced at the API layer, not only in the UI.
- AC4: Role assignment and changes are audit logged.
- AC5: Separation of duties is enforced where the PRD specifies (e.g., overbooking requires staff role, not patient role).

---

### US-039 — MFA for Staff
**PRD Ref:** Section 7 (NFR Security)  
**As a** system admin,  
**I want** to configure mandatory MFA for staff accounts,  
**so that** the risk of unauthorized PHI access from compromised credentials is reduced.

**Acceptance Criteria:**
- AC1: When MFA is enabled for a role, staff cannot complete login without the second factor.
- AC2: Supported second factors: TOTP authenticator app (assumption; PRD says "optional MFA").
- AC3: MFA enrollment and bypass events are audit logged.
- AC4: Admin can enable/disable MFA enforcement per role.

---

### US-040 — Performance: Availability Search
**PRD Ref:** Section 7 (NFR Performance)  
**As a** patient or front desk staff,  
**I want** availability search results to return within 2 seconds,  
**so that** the scheduling experience is fast and does not cause frustration.

**Acceptance Criteria:**
- AC1: Availability search API response time is ≤ 2 seconds at the 95th percentile under typical clinic data volumes.
- AC2: Clinic status board updates reflect changes within 5 seconds under normal load.
