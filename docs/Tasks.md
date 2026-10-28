# Tasks — Form Availability

Checklist for [ImplementationPlan.md](ImplementationPlan.md) (section 7). Section numbers in parentheses refer to that plan.

**How to use**
- Tick a task (`- [x]`) when its PR is merged. Add the PR number at the end of the line, e.g. `(#42)`.
- Tracks inside a phase are **independent**: separate developers can work on them at the same time. A track marked ⛓ **join** waits for the tracks listed in its *Depends on* line.
- Before starting a track, check that everything in its *Depends on* line is ticked.
- Task IDs (`1A.3`) are for PR titles and branch names, e.g. `feat/1A.3-grace-period`.

---

## Phase gates

- [ ] **Phase 0** done: foundations & contract merged
- [ ] **Phase 1** done: **Release 1** deployed (closing date, grace, manual close, edit lock, labels)
- [ ] **Phase 2** done: limit, recurrence, and notifications merged
- [ ] **Phase 3** done: **Release 2** deployed and accepted by the customer

## Customer follow-ups (section 9)

These don't block Phases 0–1. Each one is needed before the Phase 2 track listed. If no answer arrives, the track proceeds on the documented assumption.

- [ ] Q1 Notification recipients → **2C**
- [ ] Q2 Email language → **2C**
- [ ] Q3 Limit total vs per person; do Admin submissions count? → **2A**
- [ ] Q4 Recurrence patterns; manual close scope → **2B**
- [ ] Q5 Opening date on normal forms → **2B**
- [ ] Q6 No grace when the form is full → **2A**

---

## Phase 0 — Foundations & contract

**Entry**
- [ ] Implementation plan approved
- [ ] Follow-up questions (section 9) sent to the customer

> 0A ∥ 0B ∥ 0C ∥ 0D: all independent, start on day 1.

### 0A Backend foundations · BE-2 · 1 d
*Depends on: –*
- [ ] **0A.1** `config/ClockConfig.java`: `Clock` bean (`Clock.systemUTC()`)
- [ ] **0A.2** `config/FormAvailabilityProperties.java` (`app.forms.time-zone`, `grace-period`, `notification-poll`) + defaults in `application.yaml`
- [ ] **0A.3** Set `spring.jpa.properties.hibernate.jdbc.time_zone=UTC`
- [ ] **0A.4** Standard error body `{code, message, details}` and the error codes from 4.5 in `@RestControllerAdvice`
- [ ] **0A.5** `MutableClock` `@Primary` bean + test-only endpoint to move it (`test` profile only)
- [ ] **0A.6** Tests: config binding, error mapping, the test clock is not active in prod profiles

### 0B Contract & schema · BE-1 (FE-2 reviews) · 2 d
*Depends on: –*
- [ ] **0B.1** `Form` entity: all new fields from 3.1 + `RecurrenceFrequency` enum
- [ ] **0B.2** `FormNotificationLog` entity + repository, unique `(form_key, event_type, event_key)`
- [ ] **0B.3** DTOs: `FormDto` fields, `FormAvailabilityDto`, `availability` on `FormSummaryDto`, `editable` on `FormDataDto`
- [ ] **0B.4** `FormMapper` maps the new fields (no logic yet)
- [ ] **0B.5** "Not supported yet" validation guards for limit, recurrence, `opensOn`, and notification fields
- [ ] **0B.6** TS types in `types/Form.ts` (`FormAvailability`, `AvailabilityStatus`, `Recurrence`, new fields, `editable`)
- [ ] **0B.7** Error-code enum shared by backend and frontend (`FORM_CLOSED`, `FORM_NOT_YET_OPEN`, `FORM_FULL`, `SUBMISSION_LOCKED`, `VALIDATION_ERROR`)
- [ ] **0B.8** Draft of the TechnicalSpecification API section (5.x)
- [ ] **0B.9** Contract PR approved by 1 backend + 1 frontend developer
- [ ] **0B.10** Check: existing tests pass; columns are `NULL` for existing forms

### 0C Frontend foundations · FE-1 · 1 d
*Depends on: –*
- [ ] **0C.1** `services/http.ts` keeps error bodies; typed `ApiError` with `code` and `details`
- [ ] **0C.2** `utils/dates.ts`: `Intl.DateTimeFormat` with `timeZone: 'Europe/Helsinki'`
- [ ] **0C.3** `AvailabilitySection` shell in `EditForm.tsx` with empty slots for the sub-components
- [ ] **0C.4** i18n namespaces: `forms.status.*`, `form.closed.*`, `editForm.availability.{closing,limit,recurrence,notifications}.*`, `errors.*`
- [ ] **0C.5** Mocked `formClient` for component tests
- [ ] **0C.6** i18n key-parity test (all locales have the same keys)

### 0D Requirements & use-case docs · QA / analyst · 2 d
*Depends on: –*
- [ ] **0D.1** RequirementsSpecification 3.2–3.5 + Known Limitations draft (section 10)
- [ ] **0D.2** UseCases: authorization matrix rows
- [ ] **0D.3** UseCases: UC-03/04/07/09/10 alternative flows
- [ ] **0D.4** UseCases: new UC-13 Close/Reopen, UC-14 Notifications (draft)
- [ ] **0D.5** Drafts reviewed

**Exit**
- [ ] 0A, 0B, and 0C merged; CI green (build, tests, lint)
- [ ] Contract frozen (changes only via review from now on)
- [ ] Existing behaviour unchanged
- [ ] Doc drafts reviewed

---

## Phase 1 — Core availability (Release 1)

**Entry**
- [ ] Phase 0 exit met

> 1A ∥ 1B ∥ 1C ∥ 1D ∥ 1E: all independent.
> 1F ⛓ waits for 1A + 1B. 1G ⛓ waits for everything.
> 💡 BE-2 can start **2C** (Phase 2) after finishing 1B.

### 1A Availability rules engine · BE-1 · 2 d · 🔴 critical path
*Depends on: 0A, 0B*
- [ ] **1A.1** `FormAvailabilityService` as an ordered rule pipeline (D15); `currentPeriod()` returns a single period
- [ ] **1A.2** `deadline(closesOn)` = next day 00:00 Europe/Helsinki
- [ ] **1A.3** Grace period: accept while `now < close + grace`
- [ ] **1A.4** Manual close rule (`closed_at` + grace)
- [ ] **1A.5** Admin bypass
- [ ] **1A.6** `canEditSubmission` (edit lock)
- [ ] **1A.7** `evaluate()` → `Availability` (status, closeReason, closesAt)
- [ ] **1A.8** Parameterized unit tests: Phase 1 rows of 8.1, including DST and grace boundaries
- [ ] **1A.9** 100 % branch coverage on the service

### 1B Form admin backend · BE-2 · 2 d · 🔴 critical path
*Depends on: 0A, 0B*
- [ ] **1B.1** Remove the `closesOn` guard; validation (`closes_on ≥ opens_on`; a past date is allowed)
- [ ] **1B.2** `DatabaseInitializer`: parse the new JSON fields; insert-only (never overwrite)
- [ ] **1B.3** `FormService.close()` / `reopen()` (`closed_at`, `closed_by`; closing again changes nothing)
- [ ] **1B.4** `POST /api/forms/{key}/close` and `/reopen` endpoints
- [ ] **1B.5** `SecurityConfig` + `@PreAuthorize` `ROLE_ADMIN` for the new endpoints
- [ ] **1B.6** Hide `notificationEmails` from non-Admins
- [ ] **1B.7** Tests: validation (8.2), close/reopen, 403 for User, 404, initializer doesn't overwrite

### 1C Admin UI · FE-1 · 2 d
*Depends on: 0B, 0C*
- [ ] **1C.1** `ClosingDateFields`: date input (clearable) + help text "Open until 23:59 Finnish time"
- [ ] **1C.2** Zod rules + past-date warning
- [ ] **1C.3** `closeForm` / `reopenForm` in `formClient.ts`
- [ ] **1C.4** Close now / Reopen buttons with a confirm dialog (editor + Admin forms list)
- [ ] **1C.5** Reopen when the date has passed: hint to change the closing date
- [ ] **1C.6** Invalidate React Query caches after save, close, or reopen
- [ ] **1C.7** i18n keys `editForm.availability.closing.*`, `common.closeNow`, `common.reopen`
- [ ] **1C.8** Vitest: validation, warning, confirm + API call

### 1D User UI · FE-2 · 3 d
*Depends on: 0B, 0C*
- [ ] **1D.1** `FormAvailabilityBadge` ("Closes 31.10.", "Closed")
- [ ] **1D.2** `Forms.tsx`: badges; non-open forms not clickable for Users; Admins can click every form
- [ ] **1D.3** `FormSubmission.tsx`: standard closed message for Users
- [ ] **1D.4** `FormSubmission.tsx`: Admin banner on non-open forms
- [ ] **1D.5** `FormSubmission.tsx` edit mode: read-only when `editable=false`
- [ ] **1D.6** Handle 409 errors: keep entered data, show a translated alert
- [ ] **1D.7** Hide Edit in `FormSubmissions.tsx` and `SubmissionDetail.tsx`
- [ ] **1D.8** `refetchOnWindowFocus` for form pages
- [ ] **1D.9** i18n keys `forms.status.*`, `form.closed.*`, `errors.*`
- [ ] **1D.10** Vitest, including `TZ=America/New_York` badge tests

### 1E E2E authoring · QA · 2 d
*Depends on: 0A, 0B*
- [ ] **1E.1** Test fixtures: User + Admin accounts, sample forms
- [ ] **1E.2** Selenium helper to control the test clock
- [ ] **1E.3** Scenario 1: past closing date → User blocked; Admin can submit and edit
- [ ] **1E.4** Scenario 2: Close now → submit within grace succeeds; after grace it's rejected and the data is kept
- [ ] **1E.5** Scenario 3: Reopen → User can submit again

### 1F Enforcement & wiring · BE-1 · 2 d · ⛓ join · 🔴 critical path
*Depends on: 1A, 1B*
- [ ] **1F.1** `FormDataService.submit`: availability check → `FormNotAcceptingSubmissionsException`
- [ ] **1F.2** `FormDataService.update`: edit-lock check → `SubmissionLockedException`
- [ ] **1F.3** `editable` filled in on submission responses
- [ ] **1F.4** `availability` filled in on form, list, and all-forms responses
- [ ] **1F.5** Integration tests (8.3, Phase 1): 409 codes, Admin bypass, `editable`, time travel with `MutableClock`
- [ ] **1F.6** H2 + PostgreSQL (Testcontainers) persistence round-trip for `DATE` / `TIMESTAMPTZ`

### 1G Integrate & release · All · 1 d · ⛓ join · 🔴 critical path
*Depends on: 1C, 1D, 1E, 1F*
- [ ] **1G.1** Frontend switched from mocks to the real API; smoke test
- [ ] **1G.2** E2E scenarios 1–3 green
- [ ] **1G.3** TechnicalSpecification updated for Phase 1 (schema, endpoints, error codes, config)
- [ ] **1G.4** RequirementsSpecification and UseCases drafts finalized for Phase 1 scope
- [ ] **1G.5** Release 1 deployed
- [ ] **1G.6** Admins told how to set closing dates on existing forms

**Exit**
- [ ] Phase 1 rows of 8.1 + Phase 1 parts of 8.2–8.4 green; E2E 1–3 pass
- [ ] Coverage ≥ 70 %; 100 % branch coverage on `FormAvailabilityService`
- [ ] Guards still active for limit, recurrence, and notification fields
- [ ] Release 1 in production

---

## Phase 2 — Extensions

**Entry (per track)**
- [ ] 1F merged (needed for 2A and 2B)
- [ ] 0A + 0B merged (needed for 2C, which may start early during Phase 1)

> 2A ∥ 2B ∥ 2C: all independent. Inside each track, **BE ∥ FE**.
> ⚠ Merge **2A before 2B** (both change `FormAvailabilityService`).
> Each track can ship on its own when its exit boxes are ticked.

### 2A Submission limit · BE-2 / FE-2 · BE 2 d · FE 1 d
*Depends on: 1F · Customer: Q3, Q6*

**Backend**
- [ ] **2A.1** Remove the `maxSubmissions` guard; validation `≥ 1`
- [ ] **2A.2** `FormRepository.findByFormKeyForUpdate` (`PESSIMISTIC_WRITE`) used in submit
- [ ] **2A.3** Limit rule in the pipeline: count submissions in `currentPeriod()` → `FULL` (no grace; Admin bypass)
- [ ] **2A.4** Index `form_data (form_key, submitted_at)`
- [ ] **2A.5** `FORM_FULL` error mapping
- [ ] **2A.6** Unit tests (8.1 2A rows) + concurrency test on PostgreSQL (10 parallel submits → 1 succeeds)

**Frontend**
- [ ] **2A.7** `LimitField` in `AvailabilitySection` + Zod
- [ ] **2A.8** "Full" badge + `FORM_FULL` message
- [ ] **2A.9** i18n `editForm.availability.limit.*` + vitest

**Exit**
- [ ] Guard removed; tests green; can ship

### 2B Recurring forms · BE-1 / FE-1 · BE 4 d · FE 2 d · 🔴 critical path
*Depends on: 1F (merge after 2A) · Customer: Q4, Q5*

**Backend**
- [ ] **2B.1** Remove the `opensOn` and recurrence guards
- [ ] **2B.2** Validation: recurrence needs both dates, `interval ≥ 1`, period shorter than the interval
- [ ] **2B.3** Recurrence period provider (`plusWeeks/Months/Years`, month-end clamp) replaces the single period
- [ ] **2B.4** `SCHEDULED` status + `opensAt` (next opening)
- [ ] **2B.5** Period-scoped edit lock (a submission from a previous period can't be edited)
- [ ] **2B.6** Manual close overrides all future periods until reopened
- [ ] **2B.7** Unit tests (8.1 2B rows): month-end, leap year, between periods, period-scoped edit
- [ ] **2B.8** Validation tests (8.2)

**Frontend**
- [ ] **2B.9** `RecurrenceFields` (none/weekly/monthly/yearly, every n) + opening date field
- [ ] **2B.10** Zod: opening date required when repeating
- [ ] **2B.11** "Opens …" badge + "This form opens on …" message
- [ ] **2B.12** i18n `editForm.availability.recurrence.*` + vitest

**Exit**
- [ ] Guards removed; tests green; can ship

### 2C Email notifications · BE-2 (early start) / FE-1 · BE 3 d · FE 1 d
*Depends on: 0A, 0B (event detection: 1F) · Customer: Q1, Q2*

**Backend**
- [ ] **2C.1** `spring-boot-starter-mail` + `greenmail-junit5` in `pom.xml`
- [ ] **2C.2** `spring.mail.*` config via env vars; Mailpit in `application-dev.yaml`
- [ ] **2C.3** Remove the `notificationEmails` guard; validation (valid addresses, at most 20)
- [ ] **2C.4** `NotificationService`: plain-text English template, send, upsert log row (`SENT` / `FAILED`, attempts)
- [ ] **2C.5** `@EnableScheduling` + `FormNotificationScheduler` (poll `app.forms.notification-poll`)
- [ ] **2C.6** CLOSED events: deadline + grace passed, manual close + grace passed
- [ ] **2C.7** Idempotency (skip `SENT`), retry up to 5 attempts, no backlog burst on startup (only the last 24 h)
- [ ] **2C.8** GreenMail tests: one email per event, no duplicates across ticks or restarts, failure → retry → give up
- [ ] **2C.9** SMTP env vars in the TechnicalSpecification deployment section (8.3)

**Frontend**
- [ ] **2C.10** `NotificationRecipientsField` (add/remove email list) + Zod
- [ ] **2C.11** i18n `editForm.availability.notifications.*` + vitest

**Exit**
- [ ] Guard removed; tests green
- [ ] SMTP account available in the target environment; can ship

---

## Phase 3 — Integration & hardening (Release 2)

**Entry**
- [ ] 2A, 2B, and 2C merged

> 3A → 3B in sequence. **3C ∥ 3D** run alongside (3C authoring can start in Phase 2).
> 3E ⛓ waits for everything.

### 3A Cross-feature wiring · BE-1 · 1 d · 🔴 critical path
*Depends on: 2A, 2B, 2C*
- [ ] **3A.1** OPENED notification when a recurrence period starts (2B × 2C)
- [ ] **3A.2** CLOSED notification when a form becomes FULL (2A × 2C)
- [ ] **3A.3** Limit resets each recurrence period (2A × 2B)

### 3B Cross-feature tests · BE-2 · 1 d · 🔴 critical path
*Depends on: 3A*
- [ ] **3B.1** Combined unit cases: recurring + limit + manual close
- [ ] **3B.2** Concurrency test on PostgreSQL for a recurring form with a limit
- [ ] **3B.3** Notifications through a full recurrence cycle (open → full/close → next open)

### 3C E2E scenarios 4–5 + notifications · QA · 2 d
*Depends on: 1G for authoring; 3A to run*
- [ ] **3C.1** Scenario 4: limit of 1 reached → second User sees "Full"
- [ ] **3C.2** Scenario 5: recurring form shows "Opens …" between periods
- [ ] **3C.3** Notification check via the test SMTP (Mailpit/GreenMail)
- [ ] **3C.4** All E2E scenarios 1–5 green

### 3D Final documentation · FE-2 / analyst · 1 d
*Depends on: 2A, 2B, 2C*
- [ ] **3D.1** TechnicalSpecification: availability rules, time-zone policy, scheduler, `app.forms.*` config
- [ ] **3D.2** UseCases: UC-13 and UC-14 finalized
- [ ] **3D.3** RequirementsSpecification: add "Notification emails in English only" to Known Limitations
- [ ] **3D.4** All items in plan section 10 done

### 3E Regression & release · All · 1 d · ⛓ join · 🔴 critical path
*Depends on: 3B, 3C, 3D*
- [ ] **3E.1** Full regression (backend, frontend, E2E)
- [ ] **3E.2** Coverage check (≥ 70 %; 100 % branch coverage on `FormAvailabilityService`)
- [ ] **3E.3** Release 2 deployed
- [ ] **3E.4** Customer acceptance demo of all features from answers 1–11

**Exit**
- [ ] All tests in plan section 8 green; all guards removed
- [ ] Documentation complete
- [ ] Customer acceptance received

---

## Follow-ups outside this feature

- [ ] Fix the `GET /api/forms` public vs authenticated inconsistency (Tech 5.3/6.2 vs UC-03)
- [ ] Evaluate Flyway instead of `ddl-auto: update`
