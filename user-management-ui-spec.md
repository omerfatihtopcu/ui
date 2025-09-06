
# User Management UI — Specification (Developer‑Facing)

**Document Owner:** Frontend Team  
**Target Audience:** Frontend and Backend Engineers, QA  
**Format:** Markdown (can live as `docs/user-management-ui-spec.md` in the repo)

---

## 1) Purpose & Scope

Build a **User Management** screen that allows authorized staff to **view, create, and update** users. Deletion is **not** in scope; users are **soft‑disabled** via the `Enabled` flag. The UI must match the behavior depicted in the provided wireframe while following accessibility and performance requirements below.

Out of scope: password resets, MFA setup pages, audit log viewer (can be linked later).

---

## 2) Roles & Access Control

- **SuperAdmin / Admin**  
  - Can read the list; can create and update any user; can toggle **Enabled**.  
- **Guest**  
  - **No access** to this screen (route is protected).

> The backend must still enforce permissions; client-side gating is not sufficient.

Route protection: `/admin/users` (and nested routes). Unauthorized users are redirected to `/login` or a 403 page.

---

## 3) Data Model (API / UI contract)

**User** (canonical shape used by the UI):

```ts
type Role = 'Guest' | 'Admin' | 'SuperAdmin'

interface User {
  id: number
  username: string           // unique; 3–32; [a-z0-9._-]
  displayName: string        // 1–64 printable characters
  phone?: string | null      // E.164; up to 15 digits; optional
  email: string              // RFC 5322 basic format; unique
  roles: Role[]              // multi-select
  enabled: boolean
  createdAt: string          // ISO 8601 (read-only)
  updatedAt: string          // ISO 8601 (read-only)
}
```

**Validation rules (client + server):**
- `username`: required, 3–32 chars, regex `^[a-z0-9._-]{3,32}$` (lowercase).
- `displayName`: required, 1–64 chars, trim leading/trailing whitespace.
- `email`: required, must look like `local@domain`; normalize to lowercase.
- `phone`: optional; if provided, must satisfy `^\+?[1-9]\d{1,14}$` (E.164).
- `roles`: at least one role required.
- `enabled`: boolean.

**Uniqueness constraints:** `username`, `email` must be unique (server authoritative).

---

## 4) Layout Overview

Two‑pane layout:

- **Left pane: Grid (List View)** — paginated table of users.
- **Right pane: Form (Detail View)** — create/edit panel titled **“New User”** or **“Edit User”**.

Top actions (global to the screen):
- Primary button: **+ New User**
- Checkbox: **Hide Disabled Users** (default: **checked**)
- Primary action (right aligned): **Save User** (enabled only when form is dirty & valid).

---

## 5) Components & Behaviors

### 5.1 Command Bar
- **+ New User**: Clears the form, switches the detail pane to **New User** mode, sets `enabled = true` by default, focuses **Username**.
- **Hide Disabled Users (checkbox)**:
  - When checked: the grid lists only `enabled = true` users.
  - When unchecked: the grid lists all users.
  - State is sticky per user (persist in `localStorage` under `ui.userList.hideDisabled`).

- **Save User (primary button)**:
  - **Disabled** until: form is **dirty** *and* passes client validations.
  - On click:
    - **New mode** → `POST /api/users` with form payload.
    - **Edit mode** → `PUT /api/users/{id}` with changed fields (or full payload).
  - On success:
    - Show toast: “User saved”.
    - Refresh the grid (preserve pagination, sorting, filters).
    - Keep the detail pane bound to the just-saved user in **Edit** mode.
  - On server validation error (409/422):
    - Map known constraint errors to fields (e.g., “Email already in use”).

### 5.2 User Grid (Left Pane)
Columns: **ID**, **User Name**, **Email**, **Enabled**.

Capabilities:
- **Sorting**: Click column header toggles asc/desc; multi-sort not required.
- **Filtering**:
  - **Text filter** for `User Name`, `Email` (contains; case-insensitive).
  - **Boolean filter** for `Enabled`.
- **Pagination**: default page size 25; options: 25 / 50 / 100.
- **Row selection**: single-select. Selecting a row loads the user into the form in **Edit** mode.
- **Keyboard**: Up/Down arrow moves selection; Enter opens in **Edit** (focuses first field).
- **Empty state**: If no rows after filter → show “No users match the current filters.” with a **Clear Filters** link.
- **Loading state**: skeleton rows; no layout shift.
- **Error state**: Show inline error with **Retry** button.

### 5.3 User Form (Right Pane)
**Title**: *New User* (create) or *Edit User* (edit)

**Fields & Widgets:**
- **Username** — text input; placeholder `e.g., jdoe`.
- **Display Name** — text input; placeholder `e.g., Jane Doe`.
- **Phone** — text input; placeholder `+905555555555` (E.164).
- **Email** — text input; placeholder `user@example.com`.
- **User Roles** — multi-select dropdown; options: `Guest`, `Admin`, `SuperAdmin`.
- **Enabled** — checkbox (default checked for new users).

**UX details:**
- Tabbing order matches the visual order.
- Field-level validation runs on blur and on submit.
- Invalid fields show helper text and `aria-invalid="true"`.
- Dirty tracking: leaving the page with unsaved changes prompts a confirmation dialog.
- When switching selected row in the grid while form is dirty, prompt:  
  “Discard unsaved changes?” → **Discard / Stay**.
- For **Edit** mode, read-only fields: none (unless locked by the server via role policies); still, `id`, `createdAt`, `updatedAt` are never editable (not shown in the form).

### 5.4 Enabled/Disabled Behavior
- A disabled user cannot sign in. The UI never hides disabled users **unless** the “Hide Disabled Users” checkbox is on.
- Toggling `Enabled` on the form marks it dirty.

---

## 6) Initial State (First Render)

- Fetch first page: `GET /api/users?enabled=true&page=1&pageSize=25&sort=id,asc` if **Hide Disabled Users** is checked (default).  
- Grid shows data; the first row is **not** auto-selected. The detail pane shows **New User** (blank form).  
- **Save User** is disabled.

---

## 7) API Endpoints (Minimal Contract)

> Actual paths can be prefixed (e.g., `/v1`). These are examples the UI expects; backend can adapt with a mapping layer.

- **List users**  
  `GET /api/users?enabled=<bool?>&q=<text?>&page=<n>&pageSize=<n>&sort=<field,dir>`  
  **200** → `{ data: User[], page: number, pageSize: number, total: number }`

- **Get one**  
  `GET /api/users/{id}` → **200** `{ ...User }`

- **Create**  
  `POST /api/users` body `User` _without_ `id/createdAt/updatedAt` → **201** `{ ...User }`  
  Errors: **409** (duplicate email/username), **422** (validation).

- **Update**  
  `PUT /api/users/{id}` body `Partial<User>` (or full) → **200** `{ ...User }`  
  Errors: **409**, **422**, **404**.

- **Server-side filtering** (optional): `q` matches username/email (contains, case-insensitive).

**Error response shape** (recommended):
```json
{ "error": "VALIDATION_ERROR", "details": [{ "field": "email", "message": "Already in use" }] }
```

---

## 8) Validation & Formatting (UI)

- On blur, fields validate; on submit, all fields validate.
- Email input transforms to lowercase.
- Phone input: allow leading `+`, digits only; strip spaces/dashes on submit.
- Username input: force lowercase; disallow spaces.

---

## 9) Accessibility (A11y)

- All inputs have associated `<label>` elements; use `aria-describedby` for helper/error text.
- Focus outlines must be visible; maintain logical tab order.
- Role dropdown: use an accessible multi-select (keyboard operable; screen-reader friendly).
- Table headers use `<th scope="col">`; row selection uses `aria-selected` semantics.
- Respect 4.5:1 contrast ratio (WCAG AA).
- Announce toast messages via `aria-live="polite"` region.

---

## 10) Internationalization (i18n)

- All labels and messages pulled from i18n dictionaries. Default locale: `en`.  
- Date/time fields (`createdAt`, `updatedAt`) not rendered for now; if added, format per locale.

---

## 11) Performance

- Debounce list filter inputs (300ms).  
- Paginate (do not fetch all users).  
- Use **virtualized rows** if `total > 1,000`.  
- Cache last list query in memory for quick back/forward navigation.

---

## 12) Security

- All requests require auth (JWT or session).  
- CSRF protection for state-changing requests.  
- Prevent role escalation on the client; **server validates** that the current user can assign given roles.  
- Audit backend logs on create/update (actor, timestamp, diff).

---

## 13) Analytics (optional)

- Log: screen view, create success, update success, validation errors.  
- Anonymize PII in client-side analytics.

---

## 14) QA Acceptance Criteria (happy paths & edge cases)

1. **Create user** with valid inputs → grid refreshes, toast appears, new row visible.  
2. **Duplicate email/username** on save → inline field error(s), no data loss.  
3. **Toggle Hide Disabled Users** hides rows where `enabled = false`.  
4. **Switch row with dirty form** prompts confirmation; selecting **Discard** drops local changes.  
5. Keyboard navigation in grid works; Enter opens focus on **Username** in edit form.  
6. **Roles** multi-select allows selecting multiple roles; cannot save with zero roles.  
7. Leaving route with dirty form prompts a confirm dialog.  
8. All form errors are announced to screen readers.

---

## 15) Visual/Styling Notes

- Buttons: Primary = brand color; secondary = neutral; disabled has 50% opacity and forbidden cursor.  
- Table header shows sort indicator ▲/▼.  
- Row selection uses a highlighted background (not only color; also add left border for color-blind users).
- Form width: ~420–520px; consistent spacing (8px base grid).

---

## 16) Open Questions / Future Enhancements

- Bulk role assignment?  
- Server-side column filters vs client-side only?  
- Invite flow & password lifecycle?

---

## 17) Example Payloads

**Create (POST /api/users):**
```json
{
  "username": "adminuser",
  "displayName": "Admin User",
  "phone": "+905551112233",
  "email": "admin@piworks.net",
  "roles": ["Admin"],
  "enabled": true
}
```

**Update (PUT /api/users/12):**
```json
{
  "displayName": "Test User Updated",
  "roles": ["Guest", "Admin"],
  "enabled": false
}
```

---

## 18) Implementation Notes (Frontend)

- Suggested stack: React + TypeScript + React Query (data) + a11y-first component library.  
- Keep form state isolated from the grid state.  
- Use optimistic UI only if server conflict handling is robust; otherwise prefer “save then refresh”.

---

## 19) Change Log

- **v1.0** — Initial spec.
