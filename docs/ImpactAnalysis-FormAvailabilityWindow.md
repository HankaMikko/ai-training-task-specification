# Impact Analysis — Form Availability Window (Start/End Date)

## 1. Request

> "Hey, we have some forms that should only be open for a limited time but right now there's no way to control that — people keep submitting to old forms. Would it be possible to add some kind of start/end date thing to forms?"

### 1.1 Interpretation (working assumption)

An Admin can optionally set a **start** and/or **end** moment on a form definition. Outside that window the form is **not open for new submissions**. Forms without dates behave exactly as today (always open). Enforcement happens on the **backend**; the frontend reflects the state.

The underlying problem is *"people keep submitting to old forms"*, so the minimum viable solution is an **end date** enforced at submission time. The start date and the other details below need the customer to confirm them (see section 4).

### 1.2 Current state (relevant facts from the specs)

| Fact | Source | Why it matters |
|------|--------|----------------|
| Any authenticated user can submit any form; there is no form status concept | Req 3.3, UC-04 | The whole feature is new; there's nothing to extend |
| Users can edit their own submissions with no time limit | Req 3.3, UC-07 | Decide whether a closed form also blocks edits |
| Only option today to stop submissions is **deleting** the form | UC-11 | Deleting loses the form definition, and with it the read-only rendering of old submissions |
| Backend does not validate submission data against the form definition | Req 6.8, Tech 13.7 | Date checking must be added to the backend submit path explicitly; it won't come "for free" |
| `form` table uses `TIMESTAMP` (no time zone); API timestamps have no offset (`2024-01-15T10:30:00`) | Tech 4.1, 5.4 | Time-zone handling must be decided |
| Schema managed by Hibernate `ddl-auto: update`, no migration tool | Tech 8.3 | Adding nullable columns works automatically; anything more needs care |
| Initial forms loaded from JSON files at startup | Req 3.2, Tech 3.3 | JSON format and `DatabaseInitializer` must accept the new fields |
| `GET /api/forms` is **public** in Tech 5.3/6.2, but UC-03 says it **requires authentication** | Tech vs UC | An existing inconsistency. It matters because this endpoint will now filter on dates, and a public list could expose scheduled forms |
| Backend has no i18n; error messages are English only | Tech 9.1 | A "form closed" error needs a machine-readable code so the frontend can translate it |

## 2. Impact by Area

Rough size: **S** = small/isolated, **M** = moderate, **L** = significant.

### 2.1 Database — S

- Add to `form`:
  - `available_from TIMESTAMP WITH TIME ZONE NULL`
  - `available_until TIMESTAMP WITH TIME ZONE NULL`
- `NULL` = no limit, so existing rows keep today's behaviour and no data migration is needed.
- Optional check constraint: `available_until > available_from` when both are set.
- If a manual open/close switch is wanted (Q5), also add `status VARCHAR` or `is_closed BOOLEAN`.
- Note: `ddl-auto: update` will add the columns but won't add check constraints. Consider introducing Flyway/Liquibase now, or document a manual DDL step for production.
- `form_data` is unchanged; `submitted_at` already exists. It's also `TIMESTAMP` without a zone, so comparing it with the window must use consistent zones.

### 2.2 Backend — M

| Component | Change |
|-----------|--------|
| `entity/Form` | New `availableFrom`, `availableUntil` fields (`Instant`/`OffsetDateTime`) |
| `dto/` (form DTOs, list DTO) | Expose the new fields. Optionally add a computed `status` (`SCHEDULED` / `OPEN` / `CLOSED`) so clients don't compute it with their own clock |
| `mapper/` (MapStruct) | Map the new fields and compute the derived status |
| Form validation (service) | `until > from`. Decide whether past dates are allowed on create/update (Q9) |
| `service` — submit (`POST /api/form-data/{key}`) | **Core change:** reject when `now` is outside the window. Throw a dedicated exception (e.g. `FormClosedException`) or `IllegalStateException` |
| `service` — update submission (`PUT /api/form-data/submission/{id}`) | Apply the same rule if edits are blocked after close (Q3) |
| `service` — list forms (`GET /api/forms`, `/api/forms/all`) | Filter or annotate by availability for Users; Admins see everything |
| `service` — get form (`GET /api/forms/{key}`) | Must keep returning closed forms, because the submission detail/edit views render from the definition. Return a status flag instead of a 404 |
| `@RestControllerAdvice` | Map the new exception to **409 Conflict** (or 403/422; agree on one) with an error code such as `FORM_NOT_OPEN` / `FORM_CLOSED` |
| Clock | Inject `java.time.Clock` into services so time-based logic is testable |
| `DatabaseInitializer` + `resources/forms/*.json` | Accept optional date fields. Decide whether a restart overwrites dates an Admin changed in the UI |
| `SecurityConfig` | No rule change, but resolve the public vs authenticated `GET /api/forms` inconsistency |

### 2.3 API contract — S/M

- **Form DTO** (create/update/get/all): add `availableFrom` and `availableUntil` as ISO-8601 strings **with offset** (e.g. `2026-11-01T00:00:00+02:00`), both optional. Optionally add a read-only `status`.
- **`GET /api/forms`**: decide whether non-open forms are omitted or returned with a status (Q6).
- **`POST /api/form-data/{key}`**: new error response, e.g.
  ```json
  { "code": "FORM_CLOSED", "message": "Form is not accepting submissions", "availableUntil": "..." }
  ```
- **`PUT /api/form-data/submission/{id}`**: same error, if edits are restricted.
- Backward compatible: new fields are optional and existing clients keep working.
- Update the API section of `TechnicalSpecification.md`.

### 2.4 Frontend — M

| File / area | Change |
|-------------|--------|
| `types/Form.ts` | Add `availableFrom?`, `availableUntil?`, `status?` |
| `pages/EditForm.tsx` | Date/time inputs for start and end, clearable. Zod rule `end > start`. Show the time zone being used |
| `pages/Forms.tsx` | Show status badges ("Opens 1 Nov", "Closes in 3 days", "Closed"). Hide or disable non-open forms for Users. Admin view shows all |
| `pages/FormSubmission.tsx` (new submission) | If the form is not open, show a "this form is closed / opens on …" message instead of the form. This covers direct URLs and bookmarks |
| `pages/FormSubmission.tsx` (edit mode) | Disable or keep editing depending on Q3 |
| `pages/FormSubmissions.tsx` | Hide the **Edit** action for submissions on closed forms, if edits are blocked |
| `services/formClient.ts` / `http.ts` | Handle the new 409 / `FORM_CLOSED` error and show a translated message. Users who had the form open when it closed must not lose their input silently |
| React Query | The `['forms']` and `['form', key]` caches can be stale around the cut-off. Rely on the backend and consider refetching on focus |
| `public/locales/*/translation.json` | New keys for labels, badges, the closed message, and the error. Dates formatted per locale |

### 2.5 Tests — M

- **Backend unit** (service, fixed `Clock`):
  - no dates
  - before start, at start, inside the window, exactly at end, after end
  - only start set; only end set
  - `until < from` rejected
  - update-submission behaviour after close
  - Admin bypass (if any)
- **Backend integration** (Testcontainers/PostgreSQL and H2): persistence of zoned timestamps, the 409 response body, list filtering per role, and JSON initial-load parsing.
- **Frontend (vitest/RTL)**: editor date validation, badges, closed-form view, error handling for the 409 response.
- **Selenium/E2E**: Admin sets an end date in the past → User can't submit; scheduled form isn't visible or submittable before start.
- Make sure the > 70 % coverage target still holds.

### 2.6 Documentation — S

- **RequirementsSpecification.md**: new requirement under 3.2 (Admin sets a window) and 3.3 (submissions only inside the window); maybe a note under Known Limitations.
- **UseCases.md**:
  - UC-03: list filtering and status
  - UC-04: new alternative flow "Form not open"
  - UC-07: edit after close
  - UC-09/UC-10: new editor fields
  - Authorization matrix: does Admin bypass the window?
  - Optionally a new UC: "Close a form immediately"
- **TechnicalSpecification.md**: schema, DTOs, endpoints, error codes, time-zone policy, and the `GET /api/forms` auth fix.

### 2.7 Operations / non-functional

- **Time source:** use server time (UTC internally) and never the client clock. Make sure server/DB clocks are NTP-synced.
- **Existing forms:** nothing changes on deploy. Admins must edit old forms to close them. Consider a quick "Close now" action (Q5).
- **Logging:** log rejected submissions at INFO/DEBUG without submission data (Req 4.3).
- **Release:** backward compatible and additive, so it doesn't need a feature flag.

### 2.8 Out of scope unless the customer asks (confirm in Q11)

- Email notifications on open/close (email isn't supported today)
- Maximum number of submissions
- Recurring windows
- Per-user or per-group availability
- Auto-delete or archive after close

## 3. Ambiguities and Edge Cases

### 3.1 Ambiguities in the request

1. **What "open" means.** Does it block only new submissions, or also viewing the form and editing existing submissions?
2. **Start date needed?** The pain point is old forms, so an end date alone may solve it. Is scheduling a future start actually needed?
3. **Granularity.** Date only ("until 31 Oct") or date + time ("until 31 Oct 16:00")? If date only, is the end date inclusive (open all day on 31 Oct)?
4. **Time zone.** Is everyone in one time zone (e.g. Europe/Helsinki)? The app is single-tenant, but users may travel or be remote.
5. **Visibility.** Should closed and not-yet-open forms be hidden from the list, or shown as "Closed" / "Opens on …"?
6. **Admin privileges.** Can Admins still submit to or edit submissions on a closed form, e.g. to enter late paper forms?
7. **Manual control.** Is a manual "close now / reopen" switch also wanted, or are dates enough?
8. **Old forms.** Should the feature be applied to existing forms automatically, or will Admins set the dates themselves?

### 3.2 Edge cases

| # | Scenario | Proposed handling (to confirm) |
|---|----------|-------------------------------|
| E1 | User opens the form before the deadline and clicks Submit after it | Backend rejects. Frontend keeps the entered data and shows a clear message. Possibly a short grace period (Q4) |
| E2 | Submission arrives exactly at `availableUntil` | Define the window as `from ≤ now < until` (end exclusive) and document it |
| E3 | Admin sets an end date in the past on create/update | Allowed (closes immediately); the editor shows a warning |
| E4 | `availableUntil` ≤ `availableFrom` | Validation error on client and server |
| E5 | Admin extends or reopens a closed form | Allowed; the form becomes open again immediately |
| E6 | Admin removes both dates | Form is open indefinitely, as today |
| E7 | User edits an existing submission after close | Depends on Q3 |
| E8 | User opens a bookmarked or direct URL to a closed form | Show the closed message instead of the form |
| E9 | Form list cached in the browser shows "open" after the cut-off | Backend is authoritative; UI refetches and handles the 409 |
| E10 | Client clock is wrong | Status is computed on the server; the client only displays it |
| E11 | Daylight-saving transition at the boundary (e.g. last Sunday of Oct/Mar) | Store as UTC instants and convert in the UI using an explicit zone |
| E12 | Initial JSON form on restart has different dates than those an Admin set in the UI | Decide whether the initializer only inserts missing forms or overwrites |
| E13 | Form is deleted while open | Unchanged behaviour (UC-11) |
| E14 | Viewing old submissions of a closed form | Must keep working: `GET /api/forms/{key}` still returns closed forms |
| E15 | Unauthenticated access to `GET /api/forms` | Don't leak scheduled (not yet public) forms. Resolve the existing auth inconsistency |
| E16 | Form definition changed while open | Same as today (no versioning). Not affected by this feature |

## 4. Clarifying Questions for the Customer

*(Written to be sent as-is.)*

Thanks for the request. Adding opening and closing dates to forms is definitely doable. Before we estimate and build it, we'd like to check a few details so it works the way you expect:

1. **Start, end, or both?** Do you need forms to open automatically at a future date, or is an automatic closing date enough to solve the problem with old forms?
2. **Date or exact time?** Is a date enough (e.g. "open until 31 October"), or do you need an exact time (e.g. "closes 31 October at 16:00")? If date only, should the form stay open until the end of that day?
3. **Editing after closing.** Once a form has closed, should people still be able to view and **edit** submissions they already sent, or should those be locked too?
4. **Last-minute submissions.** If someone starts filling in a form before the deadline but presses Submit just after it, should we reject the submission or allow a short grace period (e.g. a few minutes)?
5. **Manual control.** Besides dates, would you like a simple "Close now" / "Reopen" button for Admins, so a form can be closed immediately without picking a date?
6. **What users see.** Should closed forms and forms that haven't opened yet be **hidden** from the forms list, or shown with a label like "Closed" or "Opens 1 November"?
7. **Admin exceptions.** Should Admins be able to submit or edit entries on a closed form (for example, to add late or paper submissions)?
8. **Time zone.** Are all your users in the same time zone? Which one should the dates follow (e.g. Finnish time)?
9. **Existing forms.** Should we close any of your current forms as part of the rollout, or will your Admins set dates on them themselves?
10. **Message to users.** Is a standard message such as "This form is no longer accepting responses" fine, or would you like to write a custom message per form?
11. **Anything related?** Do you also need any of the following now, or can they wait for later?
    - email notifications when a form opens or closes
    - a maximum number of submissions
    - forms that reopen on a recurring schedule

## 5. Recommended Next Steps

1. Send the questions in section 4 and agree on the answers.
2. Update the three specification documents with the agreed behaviour (section 2.6).
3. Implement in this order: database and backend enforcement → API → frontend → tests.
4. Resolve the `GET /api/forms` public vs authenticated inconsistency as part of this change.
