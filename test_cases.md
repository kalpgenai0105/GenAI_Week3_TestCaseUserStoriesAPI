# OSCS Test Cases
**System:** Outpatient Patient Scheduling & Care System (OSCS)
**Owner:** ABC Health Care Company
**PRD Version:** 1.0 (December 20, 2025)

Test case IDs use format TC-NNN. Each case cites its linked user story (US-NNN) and PRD requirement.

Legend:  
- **Positive:** Happy-path scenario (expected to succeed).  
- **Negative:** Invalid input or constraint violation (expected to fail gracefully).  
- **Boundary:** Edge/limit values.  
- **Security/Privacy:** Authentication, authorization, PHI handling, audit logging.  
- **NFR:** Non-functional requirement (performance, availability).

---

## A. Patient Registration and Authentication

### TC-001 — Successful Registration with Email
| Field | Value |
|-------|-------|
| **ID** | TC-001 |
| **Type** | Positive |
| **Story / PRD** | US-001 / FR-001 |
| **Preconditions** | Email address not already registered in OSCS |
| **Steps** | 1. POST /auth/register with valid email and password. 2. Retrieve OTP from email. 3. POST /auth/verify-otp with the OTP. |
| **Test Data** | email: `tc001@testpatient.com`, password: `T3stP@ss!` |
| **Expected Result** | 201 Created; account is activated; authentication event logged with timestamp, actor, and channel=email |

---

### TC-002 — Registration with Duplicate Email
| Field | Value |
|-------|-------|
| **ID** | TC-002 |
| **Type** | Negative |
| **Story / PRD** | US-001 / FR-001 |
| **Preconditions** | Email `tc001@testpatient.com` already registered |
| **Steps** | 1. POST /auth/register with same email. |
| **Test Data** | email: `tc001@testpatient.com` |
| **Expected Result** | 409 Conflict; error body does NOT expose whether the email is registered (generic message to prevent enumeration); no new account created; event logged |

---

### TC-003 — OTP Expiry
| Field | Value |
|-------|-------|
| **ID** | TC-003 |
| **Type** | Boundary / Negative |
| **Story / PRD** | US-001 / FR-001 |
| **Preconditions** | Registration submitted; OTP issued |
| **Steps** | 1. Wait for OTP TTL to expire (assumption: 10 min). 2. POST /auth/verify-otp with the expired OTP. |
| **Test Data** | Expired OTP token |
| **Expected Result** | 400 Bad Request with error code `OTP_EXPIRED`; account remains unactivated |

---

### TC-004 — Login Rate Limiting
| Field | Value |
|-------|-------|
| **ID** | TC-004 |
| **Type** | Security |
| **Story / PRD** | US-002 / FR-001 |
| **Preconditions** | Active patient account exists |
| **Steps** | 1. POST /auth/login with wrong password 5 times consecutively. |
| **Test Data** | Correct email, wrong password repeated |
| **Expected Result** | After 5th failure: 429 Too Many Requests; account locked temporarily; lockout notification sent; all 5 failed attempts recorded in authentication audit log |

---

### TC-005 — Successful Login
| Field | Value |
|-------|-------|
| **ID** | TC-005 |
| **Type** | Positive |
| **Story / PRD** | US-002 / FR-001 |
| **Preconditions** | Active patient account; account not locked |
| **Steps** | 1. POST /auth/login with valid email and password. |
| **Test Data** | Valid credentials |
| **Expected Result** | 200 OK; JWT (or session token) returned; successful authentication event written to audit log |

---

### TC-006 — Session Expiry After Idle Timeout
| Field | Value |
|-------|-------|
| **ID** | TC-006 |
| **Type** | Security / Boundary |
| **Story / PRD** | US-002 / FR-001, NFR Security |
| **Preconditions** | Patient logged in; session active |
| **Steps** | 1. Do not interact with the API for the configured idle timeout duration. 2. Attempt any authenticated request. |
| **Test Data** | Valid but expired session token |
| **Expected Result** | 401 Unauthorized with `SESSION_EXPIRED`; patient must re-authenticate |

---

## B. Patient Profile Management

### TC-007 — Update Patient Demographics
| Field | Value |
|-------|-------|
| **ID** | TC-007 |
| **Type** | Positive |
| **Story / PRD** | US-003 / FR-002 |
| **Preconditions** | Patient authenticated |
| **Steps** | 1. PATCH /patients/{id}/profile with updated address and phone. |
| **Test Data** | `{"address": "123 Main St", "phone": "+15551234567"}` |
| **Expected Result** | 200 OK; audit log entry created with before/after values for address and phone fields |

---

### TC-008 — Patient Cannot Update Another Patient's Profile
| Field | Value |
|-------|-------|
| **ID** | TC-008 |
| **Type** | Security |
| **Story / PRD** | US-003 / FR-002, NFR RBAC |
| **Preconditions** | Two patient accounts: Patient A (authenticated), Patient B |
| **Steps** | 1. Patient A attempts PATCH /patients/{Patient B id}/profile. |
| **Test Data** | Patient A's token, Patient B's patient ID |
| **Expected Result** | 403 Forbidden; no data changed; authorization failure logged |

---

### TC-009 — Preferred Language Update
| Field | Value |
|-------|-------|
| **ID** | TC-009 |
| **Type** | Positive |
| **Story / PRD** | US-003 / FR-002 |
| **Preconditions** | Patient authenticated |
| **Steps** | 1. PATCH /patients/{id}/profile with `{"preferredLanguage": "es"}`. |
| **Test Data** | Language code: `es` (Spanish) |
| **Expected Result** | 200 OK; preferred language stored; subsequent portal responses (where supported) use Spanish |

---

## C. Appointment Self-Scheduling

### TC-010 — Search Available Slots (Happy Path)
| Field | Value |
|-------|-------|
| **ID** | TC-010 |
| **Type** | Positive |
| **Story / PRD** | US-005 / FR-003 |
| **Preconditions** | Patient authenticated; at least one provider has available slots matching the search criteria |
| **Steps** | 1. GET /availability?visitTypeId=&locationId=&startDate=&endDate= |
| **Test Data** | visitType=`general-checkup`, location=`clinic-north`, dateRange=next 7 days |
| **Expected Result** | 200 OK; response body contains array of available slots with provider, location, datetime; response time ≤ 2 seconds (NFR) |

---

### TC-011 — Staff-Only Visit Type Hidden from Patient Self-Scheduling
| Field | Value |
|-------|-------|
| **ID** | TC-011 |
| **Type** | Security / Negative |
| **Story / PRD** | US-005, US-013 / FR-003, FR-103 |
| **Preconditions** | Visit type `surgical-consult` configured as staff-only |
| **Steps** | 1. Patient calls GET /availability?visitTypeId=surgical-consult. |
| **Test Data** | Staff-only visit type ID |
| **Expected Result** | 404 Not Found or 403 Forbidden; no slots returned; no server-side error logged (expected behavior) |

---

### TC-012 — Concurrent Booking Race Condition
| Field | Value |
|-------|-------|
| **ID** | TC-012 |
| **Type** | Boundary / Negative |
| **Story / PRD** | US-005 / FR-003 |
| **Preconditions** | One available slot; two patients attempt to book simultaneously |
| **Steps** | 1. Patient A and Patient B each POST /appointments with the same slot simultaneously. |
| **Test Data** | Same slotId for two concurrent requests |
| **Expected Result** | Exactly one 201 Created; the other receives 409 Conflict with `SLOT_UNAVAILABLE`; no double-booking in database |

---

### TC-013 — Confirmation Notification Sent After Booking
| Field | Value |
|-------|-------|
| **ID** | TC-013 |
| **Type** | Positive |
| **Story / PRD** | US-005, US-029 / FR-003, FR-401 |
| **Preconditions** | Patient has SMS and email opted in |
| **Steps** | 1. POST /appointments to book a valid slot. |
| **Test Data** | Valid slot, patient with both notification channels opted in |
| **Expected Result** | Appointment confirmed; SMS and email notifications dispatched within 2 minutes; delivery status records created |

---

### TC-014 — Booking Blocked When Constraint Fails (New Patient / Established-Only Visit)
| Field | Value |
|-------|-------|
| **ID** | TC-014 |
| **Type** | Negative |
| **Story / PRD** | US-018 / FR-107 |
| **Preconditions** | Patient has no prior appointments (new patient); visit type requires established patient |
| **Steps** | 1. POST /appointments with visit type marked `established-only`. |
| **Test Data** | New patient ID, established-only visit type |
| **Expected Result** | 422 Unprocessable Entity; error code `CONSTRAINT_NEW_PATIENT`; constraint failure reason returned in response body |

---

### TC-015 — Appointment Request-and-Approve Mode
| Field | Value |
|-------|-------|
| **ID** | TC-015 |
| **Type** | Positive |
| **Story / PRD** | US-006 / FR-003 |
| **Preconditions** | Visit type configured for request-and-approve mode |
| **Steps** | 1. POST /appointment-requests with preferred date range and visit type. 2. Staff approves via PATCH /appointment-requests/{id}/approve. |
| **Test Data** | Specialty visit type requiring approval |
| **Expected Result** | Step 1: 201 Created, state=Requested, patient notification sent. Step 2: Appointment created in Scheduled state; patient notified of approval |

---

## D. Reschedule and Cancellation

### TC-016 — Successful Patient Reschedule
| Field | Value |
|-------|-------|
| **ID** | TC-016 |
| **Type** | Positive |
| **Story / PRD** | US-007 / FR-004 |
| **Preconditions** | Appointment in Scheduled/Confirmed state; within policy reschedule window |
| **Steps** | 1. POST /appointments/{id}/reschedule with new slot ID. |
| **Test Data** | Appointment 48h in the future; policy allows reschedule up to 24h before |
| **Expected Result** | 200 OK; original slot released; new slot confirmed; reschedule notification sent; audit log entry created |

---

### TC-017 — Reschedule Outside Policy Window Blocked
| Field | Value |
|-------|-------|
| **ID** | TC-017 |
| **Type** | Negative / Boundary |
| **Story / PRD** | US-007 / FR-004 |
| **Preconditions** | Appointment in Scheduled state; policy requires 24h notice; appointment is 2h away |
| **Steps** | 1. POST /appointments/{id}/reschedule within the blocked window. |
| **Test Data** | Appointment 2h in the future, 24h policy |
| **Expected Result** | 422 Unprocessable Entity; error code `RESCHEDULE_WINDOW_EXPIRED`; no change to appointment |

---

### TC-018 — Cancellation with Reason Captured
| Field | Value |
|-------|-------|
| **ID** | TC-018 |
| **Type** | Positive |
| **Story / PRD** | US-008 / FR-004 |
| **Preconditions** | Appointment in Scheduled state; within cancellation window |
| **Steps** | 1. POST /appointments/{id}/cancel with `{"reasonCode": "SCHEDULING_CONFLICT", "reasonText": "Work meeting"}`. |
| **Test Data** | Valid reason code from structured list |
| **Expected Result** | 200 OK; appointment state=Cancelled; reason persisted; waitlist workflow triggered if enabled; audit log entry includes reason |

---

### TC-019 — Cancellation Triggers Waitlist Offer
| Field | Value |
|-------|-------|
| **ID** | TC-019 |
| **Type** | Positive |
| **Story / PRD** | US-008, US-016 / FR-004, FR-105 |
| **Preconditions** | Waitlist enabled; at least one patient on waitlist for same slot criteria |
| **Steps** | 1. Cancel an appointment. 2. Observe waitlist notification dispatch. |
| **Test Data** | Cancelled slot matching a waitlist entry's preferences |
| **Expected Result** | Waitlist offer notification sent to highest-priority waitlisted patient within configured time; offer expiry timer started |

---

### TC-020 — Waitlist Offer Expiry and Slot Re-release
| Field | Value |
|-------|-------|
| **ID** | TC-020 |
| **Type** | Boundary |
| **Story / PRD** | US-016 / FR-105 |
| **Preconditions** | Waitlist offer sent; offer expiry time configured to 30 minutes |
| **Steps** | 1. Do not respond to offer. 2. Wait for offer expiry. |
| **Test Data** | Offer sent; no acceptance |
| **Expected Result** | After 30 min: offer expires; slot released back to general availability; offer expiry event logged; next waitlisted patient offered (if applicable) |

---

## E. Intake Forms and Consent

### TC-021 — Patient Completes Intake Form
| Field | Value |
|-------|-------|
| **ID** | TC-021 |
| **Type** | Positive |
| **Story / PRD** | US-009 / FR-005 |
| **Preconditions** | Appointment booked; intake form assigned to appointment |
| **Steps** | 1. GET /appointments/{id}/forms. 2. POST /forms/{formId}/responses with all required fields. |
| **Test Data** | Complete intake form responses |
| **Expected Result** | 201 Created; form status=Completed; staff can see completion status; form response encrypted at rest |

---

### TC-022 — Submitted Form is Immutable
| Field | Value |
|-------|-------|
| **ID** | TC-022 |
| **Type** | Negative / Security |
| **Story / PRD** | US-009 / FR-005 |
| **Preconditions** | Patient has already submitted an intake form |
| **Steps** | 1. Attempt PATCH /forms/{formId}/responses/{responseId} to modify a submitted response. |
| **Test Data** | Already-submitted response ID |
| **Expected Result** | 409 Conflict or 403 Forbidden; error code `FORM_ALREADY_SUBMITTED`; no data altered; original versioned submission intact |

---

### TC-023 — E-Signature on Consent Form
| Field | Value |
|-------|-------|
| **ID** | TC-023 |
| **Type** | Positive |
| **Story / PRD** | US-010 / FR-005 |
| **Preconditions** | Consent form assigned to appointment; patient authenticated |
| **Steps** | 1. POST /consents/{consentId}/sign with signature data and timestamp. |
| **Test Data** | Base64-encoded signature image |
| **Expected Result** | 201 Created; signed consent document versioned and immutable; signature timestamp and patient ID stored; staff can view signed status |

---

## F. Provider Calendar and Resource Management

### TC-024 — Provider Calendar Template Update With Future Effective Date
| Field | Value |
|-------|-------|
| **ID** | TC-024 |
| **Type** | Positive |
| **Story / PRD** | US-011 / FR-101 |
| **Preconditions** | Provider has existing schedule; admin authenticated |
| **Steps** | 1. PUT /providers/{id}/schedule-templates with new template and effectiveDate=30 days in future. |
| **Test Data** | New Monday hours: 9am–3pm; effectiveDate: +30 days |
| **Expected Result** | 200 OK; existing appointments in the next 30 days unaffected; new template applies from effective date; change audit logged |

---

### TC-025 — Resource Conflict Prevention
| Field | Value |
|-------|-------|
| **ID** | TC-025 |
| **Type** | Negative |
| **Story / PRD** | US-014 / FR-104 |
| **Preconditions** | Room A is already reserved for appointment at 10:00am; visit type for new booking requires Room A |
| **Steps** | 1. POST /appointments for same room, same time. |
| **Test Data** | Room A, 10:00am slot already booked |
| **Expected Result** | 409 Conflict; error code `RESOURCE_CONFLICT`; conflicting resource identified in error body; no double-booking |

---

## G. Overbooking

### TC-026 — Successful Overbooking with Justification
| Field | Value |
|-------|-------|
| **ID** | TC-026 |
| **Type** | Positive |
| **Story / PRD** | US-017 / FR-106 |
| **Preconditions** | Overbooking policy enabled for provider; staff authenticated |
| **Steps** | 1. POST /appointments with `{"overbook": true, "overbookJustification": "Urgent patient need"}`. |
| **Test Data** | A slot that is already at capacity; staff role |
| **Expected Result** | 201 Created; appointment flagged as overbooked (visual flag); overbooking action logged with actor, justification, timestamp |

---

### TC-027 — Overbooking Without Justification Blocked
| Field | Value |
|-------|-------|
| **ID** | TC-027 |
| **Type** | Negative |
| **Story / PRD** | US-017 / FR-106 |
| **Preconditions** | Overbooking policy enabled; staff authenticated |
| **Steps** | 1. POST /appointments with `{"overbook": true}` (no justification field). |
| **Test Data** | Missing justification |
| **Expected Result** | 422 Unprocessable Entity; error code `OVERBOOK_JUSTIFICATION_REQUIRED`; appointment not created |

---

### TC-028 — Patient Cannot Trigger Overbooking
| Field | Value |
|-------|-------|
| **ID** | TC-028 |
| **Type** | Security |
| **Story / PRD** | US-017, US-038 / FR-106, NFR RBAC |
| **Preconditions** | Patient authenticated; patient role only |
| **Steps** | 1. POST /appointments with `{"overbook": true}`. |
| **Test Data** | Patient JWT |
| **Expected Result** | 403 Forbidden; overbook flag ignored or rejected; authorization failure logged |

---

## H. Appointment Lifecycle

### TC-029 — Valid Appointment State Transition
| Field | Value |
|-------|-------|
| **ID** | TC-029 |
| **Type** | Positive |
| **Story / PRD** | US-019 / FR-108 |
| **Preconditions** | Appointment in Scheduled state |
| **Steps** | 1. PATCH /appointments/{id}/status with `{"status": "Confirmed"}`. |
| **Test Data** | Valid transition: Scheduled → Confirmed |
| **Expected Result** | 200 OK; state updated; transition event audit logged |

---

### TC-030 — Invalid Appointment State Transition
| Field | Value |
|-------|-------|
| **ID** | TC-030 |
| **Type** | Negative |
| **Story / PRD** | US-019 / FR-108 |
| **Preconditions** | Appointment in Scheduled state |
| **Steps** | 1. PATCH /appointments/{id}/status with `{"status": "Completed"}` (skipping intermediate states). |
| **Test Data** | Invalid transition: Scheduled → Completed (skipping Checked-in/In-room) |
| **Expected Result** | 422 Unprocessable Entity; error code `INVALID_STATE_TRANSITION`; current state unchanged |

---

### TC-031 — No-Show State Transition
| Field | Value |
|-------|-------|
| **ID** | TC-031 |
| **Type** | Positive |
| **Story / PRD** | US-019 / FR-108, Section 3 Exceptions |
| **Preconditions** | Appointment in Confirmed state; past appointment time without check-in |
| **Steps** | 1. PATCH /appointments/{id}/status with `{"status": "No-show"}`. |
| **Test Data** | Staff role marking a past appointment as no-show |
| **Expected Result** | 200 OK; state=No-show; no-show event audit logged; waitlist not triggered (slot is in the past) |

---

## I. Check-in

### TC-032 — Front Desk Check-in
| Field | Value |
|-------|-------|
| **ID** | TC-032 |
| **Type** | Positive |
| **Story / PRD** | US-022 / FR-202 |
| **Preconditions** | Appointment in Confirmed state; front desk staff authenticated |
| **Steps** | 1. POST /appointments/{id}/check-in via staff interface. |
| **Test Data** | Valid appointment ID; staff JWT |
| **Expected Result** | 200 OK; state=Checked-in; clinic status board updated within 5 seconds; check-in event audit logged |

---

### TC-033 — Mobile Self Check-in Outside Time Window
| Field | Value |
|-------|-------|
| **ID** | TC-033 |
| **Type** | Negative / Boundary |
| **Story / PRD** | US-023 / FR-202 |
| **Preconditions** | Appointment is 2 hours away; self check-in window is 30 minutes before |
| **Steps** | 1. Patient attempts POST /appointments/{id}/self-check-in. |
| **Test Data** | 2h prior to appointment; self check-in window=30 min before |
| **Expected Result** | 422 Unprocessable Entity; error code `CHECKIN_WINDOW_NOT_OPEN`; state unchanged |

---

### TC-034 — Clinic Status Board Real-Time Update
| Field | Value |
|-------|-------|
| **ID** | TC-034 |
| **Type** | NFR (Performance) |
| **Story / PRD** | US-024 / FR-203, Section 7 NFR Performance |
| **Preconditions** | Clinic status board visible to staff; patient appointment in Confirmed state |
| **Steps** | 1. Staff marks patient as Checked-in. 2. Measure time until status board reflects the change. |
| **Test Data** | Real-time update latency measurement |
| **Expected Result** | Status board reflects Checked-in status within 5 seconds under normal load |

---

## J. Care Management

### TC-035 — Care Plan Creation by Clinician
| Field | Value |
|-------|-------|
| **ID** | TC-035 |
| **Type** | Positive |
| **Story / PRD** | US-025 / FR-301 |
| **Preconditions** | Encounter exists in Completed state; clinician authenticated |
| **Steps** | 1. POST /encounters/{encounterId}/care-plans with goals, interventions, tasks. |
| **Test Data** | `{"goals": ["Reduce HbA1c"], "tasks": [{"title": "Follow-up labs", "dueDate": "2026-01-15"}]}` |
| **Expected Result** | 201 Created; care plan linked to encounter; creation audit logged; patient can view plan in portal |

---

### TC-036 — Non-Clinical Role Cannot Create Care Plan
| Field | Value |
|-------|-------|
| **ID** | TC-036 |
| **Type** | Security |
| **Story / PRD** | US-025, US-038 / FR-301, NFR RBAC |
| **Preconditions** | Front desk staff authenticated (non-clinical role) |
| **Steps** | 1. POST /encounters/{encounterId}/care-plans. |
| **Test Data** | Front desk staff JWT |
| **Expected Result** | 403 Forbidden; authorization failure logged |

---

### TC-037 — Overdue Task Escalation
| Field | Value |
|-------|-------|
| **ID** | TC-037 |
| **Type** | Positive |
| **Story / PRD** | US-026 / FR-302 |
| **Preconditions** | Task with due date yesterday; escalation policy configured |
| **Steps** | 1. Scheduled job runs overdue-task check. 2. Verify escalation notification sent. |
| **Test Data** | Task with dueDate=yesterday; status=open |
| **Expected Result** | Escalation notification sent to task owner and/or configured escalation recipient; task marked as overdue in system |

---

## K. Communications and Notifications

### TC-038 — Appointment Reminder Sent at Configured Interval
| Field | Value |
|-------|-------|
| **ID** | TC-038 |
| **Type** | Positive |
| **Story / PRD** | US-029 / FR-401 |
| **Preconditions** | Appointment in Scheduled state; reminder configured for 24h before; patient opted in to SMS |
| **Steps** | 1. System sends reminders 24h before appointment time. 2. Check notification delivery log. |
| **Test Data** | SMS opt-in patient; appointment tomorrow |
| **Expected Result** | SMS reminder sent; delivery_status=`delivered`; delivery record created with timestamp |

---

### TC-039 — No Notification to Opted-Out Patient
| Field | Value |
|-------|-------|
| **ID** | TC-039 |
| **Type** | Negative / Privacy |
| **Story / PRD** | US-029 / FR-401 |
| **Preconditions** | Patient has opted out of SMS notifications |
| **Steps** | 1. Appointment booked for SMS-opted-out patient. 2. Check notification dispatch. |
| **Test Data** | Patient with `smsOptIn=false` |
| **Expected Result** | No SMS sent; notification preference respected and enforced at dispatch time |

---

### TC-040 — Keyword "CANCEL" via Two-Way SMS
| Field | Value |
|-------|-------|
| **ID** | TC-040 |
| **Type** | Positive |
| **Story / PRD** | US-030 / FR-402 |
| **Preconditions** | Patient has an upcoming appointment; appointment is within cancellation window |
| **Steps** | 1. Patient replies "CANCEL" to appointment reminder SMS. |
| **Test Data** | SMS reply: `CANCEL` |
| **Expected Result** | Appointment cancelled; cancellation confirmation SMS sent; cancellation event audit logged; waitlist workflow triggered if enabled |

---

### TC-041 — PHI in Messages Stored Securely
| Field | Value |
|-------|-------|
| **ID** | TC-041 |
| **Type** | Security / Privacy |
| **Story / PRD** | US-030 / FR-402, NFR Security |
| **Preconditions** | Patient sends a portal message containing their DOB |
| **Steps** | 1. POST /messages with message body containing PHI. 2. Verify storage encryption. |
| **Test Data** | Message body: "My DOB is 01/01/1980..." |
| **Expected Result** | Message stored encrypted at rest; accessible only to authorized roles; message stored in audit-compliant message log |

---

## L. Reporting

### TC-042 — Operational Dashboard Data Accuracy
| Field | Value |
|-------|-------|
| **ID** | TC-042 |
| **Type** | Positive |
| **Story / PRD** | US-031 / FR-501 |
| **Preconditions** | 10 appointments scheduled; 2 no-shows; 1 waitlist conversion |
| **Steps** | 1. GET /reports/operational-dashboard?startDate=&endDate= |
| **Test Data** | Known appointment, no-show, and waitlist data |
| **Expected Result** | No-show rate=20% (2/10); waitlist conversion=1; data matches source records |

---

### TC-043 — CSV Export Audit Logged
| Field | Value |
|-------|-------|
| **ID** | TC-043 |
| **Type** | Security / Positive |
| **Story / PRD** | US-033 / FR-503, FR-603 |
| **Preconditions** | Clinic manager authenticated with export permission |
| **Steps** | 1. GET /reports/export?format=csv&scope=no-show-rate. |
| **Test Data** | Manager role JWT |
| **Expected Result** | CSV file returned; export action written to audit log (actor, timestamp, data scope) |

---

### TC-044 — Unauthorized User Cannot Export Data
| Field | Value |
|-------|-------|
| **ID** | TC-044 |
| **Type** | Security |
| **Story / PRD** | US-033, US-038 / FR-503, NFR RBAC |
| **Preconditions** | Front desk staff authenticated (no export permission) |
| **Steps** | 1. GET /reports/export?format=csv. |
| **Test Data** | Front desk staff JWT |
| **Expected Result** | 403 Forbidden; no data exported; authorization failure logged |

---

## M. Administration

### TC-045 — Audit Log Immutability
| Field | Value |
|-------|-------|
| **ID** | TC-045 |
| **Type** | Security |
| **Story / PRD** | US-036 / FR-603, NFR Compliance |
| **Preconditions** | Existing audit log entry |
| **Steps** | 1. Attempt DELETE /audit-logs/{id}. 2. Attempt PUT /audit-logs/{id} to modify entry. |
| **Test Data** | Existing audit log entry; system admin JWT |
| **Expected Result** | Both requests return 403 Forbidden or 405 Method Not Allowed; audit log entry unchanged |

---

### TC-046 — Template Versioning Preserves Old Appointments
| Field | Value |
|-------|-------|
| **ID** | TC-046 |
| **Type** | Positive |
| **Story / PRD** | US-035 / FR-602 |
| **Preconditions** | Intake form template v1 in use; appointment booked with v1 form |
| **Steps** | 1. Admin publishes form template v2 with new questions. 2. Check the pre-existing appointment's form assignment. |
| **Test Data** | Form template v1 ID; existing appointment |
| **Expected Result** | Existing appointment still references form template v1; new appointments use v2; v1 responses readable against v1 schema |

---

## N. Security and Privacy

### TC-047 — RBAC: Patient Cannot Access Staff Endpoints
| Field | Value |
|-------|-------|
| **ID** | TC-047 |
| **Type** | Security |
| **Story / PRD** | US-038 / Section 2, NFR RBAC |
| **Preconditions** | Patient authenticated |
| **Steps** | 1. GET /admin/locations (admin-only endpoint). 2. GET /clinic-status-board (staff-only endpoint). |
| **Test Data** | Patient role JWT |
| **Expected Result** | 403 Forbidden for both endpoints; access attempts logged |

---

### TC-048 — TLS Enforcement (Encryption in Transit)
| Field | Value |
|-------|-------|
| **ID** | TC-048 |
| **Type** | Security / NFR |
| **Story / PRD** | NFR Security (Section 7) |
| **Preconditions** | OSCS API deployed |
| **Steps** | 1. Attempt HTTP (non-TLS) connection to API. 2. Connect via TLS 1.0 (deprecated). 3. Connect via TLS 1.2. |
| **Test Data** | Plain HTTP, TLS 1.0, TLS 1.2 connection attempts |
| **Expected Result** | HTTP request rejected or redirected to HTTPS; TLS 1.0 handshake rejected; TLS 1.2 connection succeeds |

---

### TC-049 — MFA Enforcement for Staff
| Field | Value |
|-------|-------|
| **ID** | TC-049 |
| **Type** | Security / Positive |
| **Story / PRD** | US-039 / NFR Security |
| **Preconditions** | MFA enabled for clinician role; clinician has registered TOTP device |
| **Steps** | 1. POST /auth/login with clinician credentials. 2. Submit TOTP code from authenticator app. |
| **Test Data** | Valid clinician credentials; valid TOTP code |
| **Expected Result** | Login proceeds only after valid TOTP submitted; MFA event logged; without TOTP, login rejected at step 1 with `MFA_REQUIRED` |

---

### TC-050 — PHI Minimum Necessary in Notification Content
| Field | Value |
|-------|-------|
| **ID** | TC-050 |
| **Type** | Privacy |
| **Story / PRD** | US-029 / FR-401, NFR Privacy |
| **Preconditions** | Appointment reminder configured and dispatched |
| **Steps** | 1. Inspect SMS reminder content sent to patient. |
| **Test Data** | Appointment reminder SMS text |
| **Expected Result** | SMS contains only: provider first name (no last name — assumption), appointment date/time, clinic name, and a portal link; does NOT contain diagnosis, medications, or full DOB |

---

## O. Performance

### TC-051 — Availability Search Under 2 Seconds
| Field | Value |
|-------|-------|
| **ID** | TC-051 |
| **Type** | NFR (Performance) |
| **Story / PRD** | US-040 / Section 7 NFR Performance |
| **Preconditions** | System at typical clinic data volume (assumption: 50 providers, 500 slots/day) |
| **Steps** | 1. Execute 100 concurrent GET /availability requests. 2. Measure p95 response time. |
| **Test Data** | Load test with 100 concurrent users |
| **Expected Result** | p95 response time ≤ 2,000ms; no errors or 5xx responses |

---

### TC-052 — Integration Failure Retry Queue
| Field | Value |
|-------|-------|
| **ID** | TC-052 |
| **Type** | NFR (Reliability) |
| **Story / PRD** | FR-701 / Section 7 NFR Reliability |
| **Preconditions** | EHR integration endpoint down |
| **Steps** | 1. Trigger appointment sync event to EHR. 2. Simulate EHR endpoint failure. 3. Verify retry queue behavior. |
| **Test Data** | Mocked EHR endpoint returning 503 |
| **Expected Result** | Sync message queued for retry; operational alert generated; after EHR recovers, message retried and sync completes |

---

## Traceability Matrix

| PRD Requirement | User Story | Test Cases |
|---|---|---|
| FR-001 | US-001, US-002 | TC-001, TC-002, TC-003, TC-004, TC-005, TC-006 |
| FR-002 | US-003, US-004 | TC-007, TC-008, TC-009 |
| FR-003 | US-005, US-006 | TC-010, TC-011, TC-012, TC-013, TC-015 |
| FR-004 | US-007, US-008 | TC-016, TC-017, TC-018, TC-019 |
| FR-005 | US-009, US-010 | TC-021, TC-022, TC-023 |
| FR-101 | US-011 | TC-024 |
| FR-102 | US-012 | TC-010 (cross-location filter) |
| FR-103 | US-013 | TC-011 |
| FR-104 | US-014 | TC-025 |
| FR-105 | US-015, US-016 | TC-019, TC-020 |
| FR-106 | US-017 | TC-026, TC-027, TC-028 |
| FR-107 | US-018 | TC-014 |
| FR-108 | US-019 | TC-029, TC-030, TC-031 |
| FR-201 | US-020, US-021 | TC-038 (intake reminder path) |
| FR-202 | US-022, US-023 | TC-032, TC-033 |
| FR-203 | US-024 | TC-034 |
| FR-301 | US-025 | TC-035, TC-036 |
| FR-302 | US-026 | TC-037 |
| FR-303 | US-027 | (Covered by care plan + scheduling integration) |
| FR-304 | US-028 | TC-041 (PHI in messages) |
| FR-401 | US-029 | TC-038, TC-039, TC-050 |
| FR-402 | US-030 | TC-040, TC-041 |
| FR-501 | US-031 | TC-042 |
| FR-502 | US-032 | TC-036 (role restriction path) |
| FR-503 | US-033 | TC-043, TC-044 |
| FR-601 | US-034 | TC-047 (admin access control) |
| FR-602 | US-035 | TC-046 |
| FR-603 | US-036 | TC-043, TC-045 |
| FR-701 | US-037 | TC-052 |
| FR-702 | US-029 | TC-038, TC-039 |
| FR-703 | (Phase 2) | No MVP test cases |
| FR-704 | (Phase 2) | No MVP test cases |
| NFR Security | US-038, US-039 | TC-006, TC-008, TC-028, TC-036, TC-044, TC-047, TC-048, TC-049 |
| NFR Performance | US-040 | TC-034, TC-051 |
| NFR Reliability | — | TC-052 |
| NFR Privacy/PHI | US-003, US-009, US-030 | TC-008, TC-041, TC-050 |
