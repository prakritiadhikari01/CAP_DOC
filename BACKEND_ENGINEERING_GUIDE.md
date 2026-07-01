# NIC College Ambassador Program — Backend Engineering Guide
**The single reference to check before writing any code.**

Version 1.0 · Backend: Django + DRF · DB: PostgreSQL · Architecture: Modular Monolith
Traces to: SDS v0.1 (1 Jul 2026), SRS v0.1 (25 Jun 2026)

---

## 1. Project Vision — Why This Exists

The platform centralizes NIC's College Ambassador Program:
- Recruit ambassadors (public application → admin decision)
- Give ambassadors a self-service dashboard (profile, reports, events)
- Give NIC staff an admin console (review, approve, publish, manage)
- Publish verified public content (directory, events, innovation stories)

The backend must provide: secure auth, role-based access, a verified-content workflow, and a scalable versioned API.

---

## 2. The One Rule That Shapes Everything: Verified Content Model

> **Nothing an ambassador submits is ever written directly to a public or admin-trusted record.**

Every profile edit or activity report is stored as a **pending change request**, isolated from the published record. An admin must explicitly approve it before it becomes visible anywhere (public site, dashboard summary, analytics).

```
Ambassador → Submit Change Request → Verification Queue (pending)
                                            ↓
                                    Admin Review
                                    ↙          ↘
                              Approved        Rejected (+ reason)
                                    ↓                ↓
                        Merged into Published    Ambassador can
                        Profile (atomic w/       revise & resubmit
                        AUDIT_LOG write)         (new row, history kept)
```

**Why it's built this way (not simple CRUD + status flag):** a moderation-queue pattern with two separate tables means unverified data can *never* leak into a public view through a missed filter — there's structurally no join path from pending data to public/dashboard endpoints.

**Non-negotiable implementation rules:**
1. `AMBASSADOR_PROFILE` (published) is written **only** inside the `approve` action, in a DB transaction together with the `AUDIT_LOG` insert.
2. Public/dashboard-summary endpoints query `AMBASSADOR_PROFILE` (or approved `ACTIVITY_REPORT` rows) directly — never through `PROFILE_CHANGE_REQUEST`.
3. Rejection **requires** non-empty `review_notes`.
4. Resubmission after rejection = **new row**, never mutate the rejected one (preserves history).
5. Ambassadors see only their own change requests; admins see all — enforced by DRF permission classes, not frontend routing.
6. Concurrent edit to the same section while one is already pending → flagged for admin, never silently overwritten.

---

## 3. System Actors & Use Cases

| Actor | Description |
|---|---|
| **Visitor** | Unauthenticated public user |
| **Ambassador** | Approved rep, logs in with org email + password |
| **Administrator** | NIC staff, review & management permissions |

### Visitor
| Use Case | API |
|---|---|
| Submit application | `POST /api/v1/applications` |
| Browse ambassador directory | `GET /api/v1/ambassadors` |
| View ambassador detail | `GET /api/v1/ambassadors/{id}` |
| View events | `GET /api/v1/events` |
| View innovation stories | `GET /api/v1/stories` |

### Ambassador (authenticated)
| Use Case | API |
|---|---|
| Login | `POST /api/v1/auth/login` |
| View own dashboard summary | reads published data only |
| Submit profile change | `POST /api/v1/me/profile/change-requests` |
| List own change requests | `GET /api/v1/me/profile/change-requests` |
| Submit activity report | `POST /api/v1/me/reports` |
| List own reports | `GET /api/v1/me/reports` |
| Register for event | `POST /api/v1/events/{id}/register` |

**Detailed flow — Submit Profile Change:**
Precondition: authenticated, active AMBASSADOR record.
1. Ambassador edits a field → 2. Frontend submits `PROFILE_CHANGE_REQUEST` → 3. Stored as `pending`, receipt confirmed → 4. Published profile unchanged → 5. Admin notified.
Postcondition: pending request exists; public/dashboard view unaffected until reviewed.
Alt flow: existing pending request for the same section may be superseded rather than duplicated.

### Administrator
| Use Case | API |
|---|---|
| Review applications | `GET /api/v1/admin/applications` |
| Approve/reject application | `PATCH /api/v1/admin/applications/{id}/decision` |
| Review pending changes | `GET /api/v1/admin/change-requests` |
| Approve change | `PATCH /api/v1/admin/change-requests/{id}/approve` |
| Reject change | `PATCH /api/v1/admin/change-requests/{id}/reject` |
| Review/approve/reject reports | `GET/PATCH /api/v1/admin/reports...` |
| Manage events | `POST/PUT/DELETE /api/v1/admin/events` |
| Author/publish stories | `POST/PUT/DELETE /api/v1/admin/stories` |
| View/edit/deactivate ambassadors | `GET/PATCH /api/v1/admin/ambassadors` |
| View analytics | `GET /api/v1/admin/analytics` |

**Detailed flow — Review & Approve/Reject:**
Precondition: authenticated admin; ≥1 pending item exists.
1. Admin views pending item + proposed data → 2. Approves (merge into published + audit log) or rejects with notes → 3. Ambassador notified.
Exception: concurrent edit → flagged, not silently overwritten.

---

## 4. Database Design

### Core entities

| Entity | Purpose | Key fields |
|---|---|---|
| **USER** | Central identity/auth record. Created only on application approval (ambassador) or admin provisioning. | id, organization_email, password_hash, role, must_change_password, is_active, created_at |
| **APPLICATION** | One row per recruitment submission | id, name, email, college, phone, skills, motivation, status (submitted→under_review→approved/rejected) |
| **AMBASSADOR** | Membership/account record | id, user_id, college, joined_date, status, level |
| **AMBASSADOR_PROFILE** | The **only published**, publicly-readable content record. 1:1 child of AMBASSADOR. Mutated only by the approve action. | id, ambassador_id (unique), bio, skills (JSON), projects, achievements, linkedin, photo |
| **PROFILE_CHANGE_REQUEST** | One row per proposed edit, scoped to ONE section | id, ambassador_id, section, proposed_data (JSON), status (pending/approved/rejected), review_notes, created_at |
| **ACTIVITY_REPORT** | Ambassador-submitted event report, own lifecycle (different review criteria than profile) | id, ambassador_id, title, description, date, evidence, status |
| **EVENT** | Event catalog | id, title, description, date, location, status |
| **EVENT_REGISTRATION** | M:M join, AMBASSADOR ↔ EVENT | ambassador_id, event_id |
| **INNOVATION_STORY** | Admin-authored content | id, title, content, author, status (draft/published) |
| **ATTACHMENT** | Polymorphic file ref (profile photos, report photos, recommendation letters, event photos). Only URL persisted, file lives in object storage. | owner_type, owner_id, url |
| **AUDIT_LOG** | Immutable, append-only, every state-changing admin action. Never purged. | actor, action, entity, entity_id, timestamp, changes |

### Rules
- **3NF**, normalized. JSON columns (`skills`, `proposed_data`, `social_links`) only for genuinely variable/semi-structured data — not a shortcut around relational modeling.
- FKs enforce integrity; `PROFILE_CHANGE_REQUEST.ambassador_id` / `ACTIVITY_REPORT.ambassador_id` are cascade-**restrict** on delete (preserve audit history).
- Status fields use DB-level CHECK constraints (or Django choices + migration constraints) — invalid states can't be written even by a raw script.

### Indexes (add these from day one, not as an afterthought)
| Table | Index | Reason |
|---|---|---|
| PROFILE_CHANGE_REQUEST | (status, submitted_at) | fast pending-queue listing, oldest-first |
| ACTIVITY_REPORT | (status, submitted_at) | fast pending-queue listing |
| AMBASSADOR_PROFILE | (ambassador_id) unique | enforce 1:1, fast lookup |
| EVENT | (event_date, status) | upcoming/past filtering |
| APPLICATION | (status, submitted_at) | admin review queue |
| AUDIT_LOG | (entity_type, entity_id) | fast "history for this record" |

---

## 5. Backend Modules (Django apps)

Each app under `apps/` gets its own `models.py`, `serializers.py`, `views.py`, `permissions.py`, `services.py`, `tests/`. All endpoints prefixed `/api/v1/`.

| App | Responsibility |
|---|---|
| `accounts` | Org-credential issuance, login/logout, password lifecycle, token issuance, role resolution |
| `applications` | Application intake, validation, admin decision, triggers account provisioning |
| `verification` | **Core module.** Moderation queue for profile changes + activity reports. Sole write-path to `AMBASSADOR_PROFILE` |
| `events` | Event CRUD, registrations, upcoming/past archive |
| `network` | Public ambassador directory — reads **only** published data |
| `stories` | Admin-authored innovation stories pipeline |
| `admin_dashboard` | Aggregates records, queues, analytics for admin console |
| `notifications` | Celery-based async emails (submission received, approved, rejected, etc.) |
| `audit` | Append-only audit log writes/reads |

**Layering rule — never put business logic in views:**
```
View → Serializer → Service Layer → Model
```

---

## 6. Auth Module — Detail (this is the trickiest part, read carefully)

**No third-party identity provider.** The system is the sole source of truth for credentials.

**Account provisioning — the ONLY path that creates login credentials:**
1. Anyone applies via public site — no login needed to apply.
2. Admin approves application → Auth module auto-generates `firstname.lastname@<org-domain>` + one-time temp password.
3. Temp password is **never** shown in admin UI or logged in plaintext — only in the outbound email and as a hash in DB.
4. `must_change_password = true` on the new USER → forces password change before dashboard access.
5. Passwords hashed with **Argon2**.

**Password policy:**
- Min 10 characters, checked against common-password blocklist.
- Temp passwords randomly generated (never derived from name/email), expire after ~7 days unused.
- Rate limiting on `/auth/login` and `/auth/password/forgot`.
- Failed login attempts tracked per account, temporary lockout after threshold, logged to `AUDIT_LOG`.

**Endpoints:**
| Endpoint | Method | Access | Purpose |
|---|---|---|---|
| `/auth/login` | POST | Public | Authenticate, return access/refresh token |
| `/auth/token/refresh` | POST | Authenticated | Refresh access token |
| `/auth/logout` | POST | Authenticated | Invalidate refresh token/session |
| `/auth/password/change` | POST | Authenticated | Required if `must_change_password` is true |
| `/auth/password/forgot` | POST | Public | Request reset link |
| `/auth/password/reset` | POST | Public + token | Complete reset |
| `/auth/me` | GET | Authenticated | Current user, role, linked profile id |

---

## 7. Full API Reference

Base path: `/api/v1/`. JSON everywhere; file uploads via `multipart/form-data` → return an `ATTACHMENT` reference. Pagination: limit/offset or cursor, default page size 20. Filtering via query params, e.g. `?status=pending&ambassador=<id>`.

**Recruitment (`applications`)**
- `POST /applications` — public, submit application
- `GET /applications/{id}` — public (owner by email token) / admin — check status
- `GET /admin/applications` — admin, list/filter by status
- `PATCH /admin/applications/{id}/decision` — admin, approve/reject

**Verification (`verification`)**
- `POST /me/profile/change-requests` — ambassador, submit change
- `GET /me/profile/change-requests` — ambassador, list own
- `POST /me/reports` — ambassador, submit activity report
- `GET /me/reports` — ambassador, list own
- `GET /admin/change-requests` — admin, list (default pending)
- `PATCH /admin/change-requests/{id}/approve` — admin
- `PATCH /admin/change-requests/{id}/reject` — admin (requires `review_notes`)
- `GET /admin/reports` — admin
- `PATCH /admin/reports/{id}/approve` — admin
- `PATCH /admin/reports/{id}/reject` — admin

**Events (`events`)**
- `GET /events` — public, upcoming/past
- `GET /events/{id}` — public, detail
- `POST /events/{id}/register` — ambassador
- `POST/PUT/DELETE /admin/events` — admin

**Network (`network`)**
- `GET /ambassadors` — public, directory (published only)
- `GET /ambassadors/{id}` — public, detail

**Stories (`stories`)**
- `GET /stories` — public, published only
- `POST/PUT/DELETE /admin/stories` — admin

**Admin (`admin_dashboard`)**
- `GET/PATCH /admin/ambassadors` — admin
- `GET /admin/analytics` — admin

### Example contract — Submit Profile Change
```
POST /api/v1/me/profile/change-requests
{
  "section": "skills",
  "proposed_data": { "skills": ["UI/UX", "Public Speaking", "Python"] }
}

→ 201 Created
{
  "id": "6f2c...",
  "ambassador_id": "a1b2...",
  "section": "skills",
  "status": "pending",
  "submitted_at": "2026-07-01T09:15:00Z"
}
```

### Error envelope
```json
{ "error": { "code": "...", "message": "...", "fields": {...} } }
```
`400` validation · `401` unauthenticated · `403` wrong role/ownership · `404` not found · `409` conflicting concurrent pending edits · `422` business-rule violation (e.g. reject without notes)

### RBAC matrix
| Resource | Visitor | Ambassador | Admin |
|---|---|---|---|
| Public content (events, stories, directory) | Read | Read | Read |
| Applications | Create own | — | Read/Decide all |
| Own profile change requests | — | Create/Read own | Read all, Approve/Reject |
| Own activity reports | — | Create/Read own | Read all, Approve/Reject |
| Events management | — | Register only | Full CRUD |
| Innovation stories | — | — | Full CRUD |
| Ambassador records | — | Read/Edit own settings | Read/Edit/Deactivate all |
| Analytics | — | — | Read |

---

## 8. Coding Standards

**Naming:** Models → `PascalCase` (e.g. `AmbassadorProfile`). Functions/variables → `snake_case` (e.g. `approve_change_request()`).

**Django rule:** business logic never lives in views. `View → Serializer → Service Layer → Model`.

**Every feature, before it's mergeable, needs:**
- [ ] Model
- [ ] Migration
- [ ] Serializer
- [ ] API endpoint
- [ ] Permission class enforced at view level (not just frontend)
- [ ] Tests
- [ ] Swagger/OpenAPI updated
- [ ] README updated if setup changed
- [ ] Reviewed by the other developer

**Commit style:**
```
feat(auth): add JWT login
fix(profile): validate change request
docs(api): update endpoints
```

---

## 9. Git Workflow

You already have the repo + cookiecutter template, so this is just the ongoing discipline:

- `main` — production
- `develop` — integration branch
- `feature/<module-name>` — one branch per feature, branched from `develop`
- PR into `develop`, requires the other dev's review before merge
- `main` only receives merges from `develop` at release points

---

## 10. Sprint Plan — Equal 50/50 Ownership

No dev "owns" a module. Every feature gets a **primary implementer** and a **secondary reviewer/tester**, then they swap on the next feature. Both write code, both review, both test, both document.

| Sprint | Goal | Dev A leads | Dev B leads |
|---|---|---|---|
| 0 (Day 1-3) | Foundation — no business logic | Repo structure, Django init, settings (base/dev/prod) | DB design doc, API doc skeleton, env setup doc |
| 1 | Auth + Recruitment | User model, JWT | Permissions, auth tests |
| 2 | Recruitment complete | Application APIs | Admin review/decision APIs |
| 3 | Ambassador Profile + Verification | Profile models | Verification workflow (change requests) |
| 4 | Events + Stories | Events module | Stories module |
| 5 | Production readiness | Testing, security | Deployment, performance |

Roles swap each sprint — whoever reviewed last sprint implements this sprint's equivalent piece, and vice versa.

**Daily rhythm (6 hr/day):**
1. 15-min standup — yesterday / today / blockers
2. ~5 hrs coding
3. End of day: log to `docs/DAILY_LOGS/day-x.md` (completed, problems, tomorrow)

---

## 11. Testing Strategy

- Unit tests per app (`apps/<name>/tests/`), run in CI on every PR.
- Migration checks + lint run in CI on every PR.
- Full test suite + migration dry-run against disposable DB on merge to `main`.
- Priority coverage: verification workflow (approve/reject transactions), auth (password policy, lockouts), RBAC permission classes.

---

## 12. Deployment Checklist

- **Environments:** Development (Docker Compose: Django, Postgres, Redis, Mailhog) → Staging (mirrors prod, seeded test data) → Production (auto-scaled, managed Postgres w/ backups, object storage, monitoring)
- **CI/CD:** PR → lint + unit tests + migration check → merge to main → build image, full suite, migration dry-run → auto-deploy staging → **manual approval gate** for production
- **Backups:** daily automated Postgres backups, point-in-time recovery ≥30 days retention; object storage versioned separately; `AUDIT_LOG` never purged
- **Security:** HTTPS everywhere; secrets in a secrets manager (never in source control); all admin/ambassador endpoints require auth + DRF role-based permission classes at view level; file uploads validated for type/size and scanned

---

## 13. Definition of Done (checklist to paste into every PR)

```
☐ Code written
☐ Database migration created
☐ API endpoint created
☐ Swagger/OpenAPI updated
☐ Unit tests written and passing
☐ Permissions checked (role-based, at view level)
☐ README/docs updated
☐ Reviewed by the other developer
☐ No lint errors
```

---

## 14. Golden Rule to Remember

Before writing any code for a feature, walk this order:
**Understand requirement → check database impact → design API → implement → test → document → review → merge.**

The verification/moderation pattern (Section 2) is the one architectural decision the whole system hinges on — when in doubt about where to write data, the answer is: *never directly to the published table, always through a pending row + an explicit approve transaction.*
