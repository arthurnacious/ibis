# API contract (v1)

Base path: `/api/v1`. JSON only. Field names are **camelCase**. Money is an integer
number of **cents** (`*Cents` fields) in the company currency. IDs are UUID strings.
Timestamps are RFC 3339 UTC strings; dates are `YYYY-MM-DD`.

The source of truth is the backend code (`backend/src/**/dto.rs`). The frontend
mirrors these shapes in each feature's `*.models.ts`; when an endpoint changes,
update this file and the owning feature's model file in the same change.

## Envelopes

Success (NestForge `ResponseEnvelope`):

```json
{ "success": true, "data": { }, "meta": { "page": 1, "per_page": 25, "total": 120 } }
```

`meta` is present only on paginated lists. `DELETE` endpoints return `{ "success": true, "data": null }`.

Error (NestForge `HttpException` body):

```json
{
  "statusCode": 403,
  "error": "Forbidden",
  "code": "lock_mode_read_only",
  "message": "Lock Mode is read-only. Sign in to make changes.",
  "details": { "errors": [{ "field": "email", "message": "email must be a valid email" }] },
  "requestId": "req-1728480000000-42"
}
```

| status | `code` | meaning |
|---|---|---|
| 400 | `bad_request` / `validation_failed` | malformed input; `details.errors[]` lists field issues |
| 401 | `unauthorized` | missing/expired/invalid access token or session |
| 403 | `forbidden` | authenticated but missing the permission |
| 403 | `lock_mode_read_only` | mutating request made with a Lock Mode (demo) session |
| 403 | `no_company_context` | the session has no current company selected |
| 404 | `not_found` | resource does not exist **in the current company** (cross-tenant ids are 404, never 403) |
| 409 | `conflict` | unique constraint (e.g. duplicate customer code) or state conflict |
| 500 | `internal_server_error` | generic message only; details are logged server-side with `requestId` |

## Authentication model

* **Access token**: short-lived (15 min) HS256 JWT returned in JSON bodies, held **in memory only** by the
  SPA and sent as `Authorization: Bearer <token>`. Claims: `sub` (user id), `sid` (session id),
  `cid` (company id or null), `mode` (`"live"` | `"demo"`), `iat`, `exp`.
* **Refresh token**: opaque random 256-bit value in cookie `erp_rt`
  (`HttpOnly; SameSite=Strict; Path=/api/v1/auth; Secure` outside local dev). Stored server-side only as a
  SHA-256 hash. Rotated on every refresh; presenting an already-rotated token revokes the whole session
  (reuse detection).
* **CSRF**: cookie-authenticated endpoints (`/auth/refresh`, `/auth/logout`) require the header
  `X-Requested-With: erp-web` (forces a CORS preflight cross-site) **and** an `Origin`, if present, that is in
  `CORS_ALLOWED_ORIGINS`. Bearer-authenticated endpoints are not CSRF-exposed.
* Every request with a bearer token is resolved server-side: JWT verified → session not revoked/expired →
  (if `cid`) active membership for `(sub, cid)` → role permissions loaded. Tenant id is **never** taken from
  the request body, path or a header.

### `SessionView` (returned by login/refresh/me/switch)

```json
{
  "accessToken": "eyJ...",           // omitted from GET /auth/me
  "expiresIn": 900,                  // seconds; omitted from GET /auth/me
  "mode": "live",                    // "live" (Normal Mode) | "demo" (Lock Mode)
  "user": { "id": "uuid", "email": "a@b.co", "displayName": "Ada Lovelace", "avatarUrl": null, "isPlatformAdmin": false },
  "company": { "id": "uuid", "name": "Harbour Supplies", "currency": "ZAR", "isDemo": false } ,  // null if none
  "role": { "id": "uuid", "name": "Company administrator" },   // null if no company
  "permissions": ["customers:read", "customers:create"],
  "companies": [{ "id": "uuid", "name": "Harbour Supplies", "isDemo": false, "roleName": "Company administrator" }]
}
```

## Endpoints

Permission in brackets is enforced server-side. `auth` = any valid session. `live` = Normal Mode session only.

### Platform
| method | path | perm | notes |
|---|---|---|---|
| GET | `/health` | – | `{ status: "ok", database: "ok" }` |
| GET | `/auth/config` | – | `{ googleClientId: string \| null, demoEnabled: boolean }` |

### Auth
| method | path | perm | body → data |
|---|---|---|---|
| POST | `/auth/login` | – | `{ email, password }` → `SessionView` + sets cookie. 401 `invalid_credentials` on any failure (same message for unknown email / wrong password). |
| POST | `/auth/google` | – | `{ credential }` (Google ID token from Google Identity Services) → `SessionView` + cookie. Token verified server-side (signature via Google JWKS, `aud`, `iss`, `exp`, `email_verified`). Creates the user if new; **never** creates memberships. |
| POST | `/auth/demo` | – | `{}` → `SessionView` (`mode: "demo"`, demo company, read-only permissions) + cookie. 404 if demo disabled. |
| POST | `/auth/refresh` | cookie + CSRF header | `{}` → `SessionView` + rotated cookie. 401 clears cookie. |
| POST | `/auth/logout` | cookie + CSRF header | `{}` → `null`; revokes session, clears cookie. Always 200. |
| GET | `/auth/me` | auth | `SessionView` without token fields |
| POST | `/auth/switch-company` | auth, live | `{ companyId }` → `SessionView` with new access token. 404 if no active membership. |

### Companies & invitations
| method | path | perm | notes |
|---|---|---|---|
| POST | `/companies` | auth, live | `{ name, legalName?, currency (ISO 4217, 3 letters), country? }` → `Company`; caller becomes Company administrator. Does **not** switch context. |
| GET | `/company` | `company:read` | current company `Company` |
| PUT | `/company` | `company:manage` | `{ name, legalName?, currency, country?, taxNumber?, email?, phone?, address? }` → `Company` |
| GET | `/me/invitations` | auth, live | pending invitations addressed to the caller's verified email: `[{ id, companyId, companyName, roleName, invitedBy, createdAt, expiresAt }]` |
| POST | `/me/invitations/{id}/accept` | auth, live | creates the membership → `{ companyId }` |

`Company`: `{ id, name, legalName, currency, country, taxNumber, email, phone, address, isDemo, createdAt }`

### Members, roles, permissions
| method | path | perm | notes |
|---|---|---|---|
| GET | `/members` | `members:read` | `[{ id, userId, email, displayName, roleId, roleName, status: "active"\|"suspended", joinedAt, lastActiveAt }]` |
| PUT | `/members/{id}` | `members:manage` | `{ roleId, status }`. 409 `last_admin` if it would leave no active member holding `members:manage` + `roles:manage`. |
| DELETE | `/members/{id}` | `members:manage` | same last-admin protection |
| GET | `/invitations` | `members:read` | company's pending invitations `[{ id, email, roleId, roleName, invitedBy, createdAt, expiresAt }]` |
| POST | `/invitations` | `members:manage` | `{ email, roleId }` → invitation (7-day expiry). 409 if already member/pending. |
| DELETE | `/invitations/{id}` | `members:manage` | revoke |
| GET | `/permissions` | `roles:read` | catalog `[{ key, module, label, description }]` |
| GET | `/roles` | `roles:read` | `[{ id, name, description, isSystem, templateKey, permissions: string[], memberCount }]` |
| POST | `/roles` | `roles:manage` | `{ name, description?, permissions: string[] }` (unknown keys → 400) |
| PUT | `/roles/{id}` | `roles:manage` | same body. System roles: name/permissions of `company_admin` template are immutable (409). |
| DELETE | `/roles/{id}` | `roles:manage` | 409 if system role or members still assigned |
| GET | `/audit-log?page&pageSize` | `audit:read` | paginated `[{ id, action, entityType, entityId, actorEmail, summary, createdAt }]` |

### Business data
| method | path | perm | notes |
|---|---|---|---|
| GET | `/dashboard/summary` | `dashboard:read` | see below |
| GET | `/customers?search&status&page&pageSize&sort` | `customers:read` | paginated `CustomerListItem[]`. `status` ∈ `active,inactive,on_hold`. `sort` ∈ `name,-name,balance,-balance,createdAt,-createdAt` (default `name`). `pageSize` ≤ 100 (default 25). `search` matches name, code, email (case-insensitive). |
| GET | `/customers/{id}` | `customers:read` | `Customer` |
| POST | `/customers` | `customers:create` | `CustomerInput` → `Customer` (201-equivalent 200) |
| PUT | `/customers/{id}` | `customers:update` | `CustomerInput` → `Customer` |
| DELETE | `/customers/{id}` | `customers:delete` | 409 `has_invoices` if invoices reference it |
| GET | `/invoices?customerId&status&page&pageSize` | `invoices:read` | paginated `[{ id, number, customerId, customerName, issueDate, dueDate, status, totalCents, amountDueCents }]`; `status` ∈ `draft,awaiting_approval,approved,paid,overdue,void` |

`CustomerInput`: `{ code (1–20, unique per company), name (2–120), email?, phone?, taxNumber?, status, creditLimitCents (≥ 0), paymentTermsDays (0–365), addressLine1?, city?, country?, notes? (≤ 2000) }`

`Customer`: `CustomerInput` + `{ id, balanceCents, overdueCents, createdAt, updatedAt }`.
`CustomerListItem`: `{ id, code, name, email, phone, city, status, balanceCents, overdueCents, updatedAt }`.

`DashboardSummary`:
```json
{
  "currency": "ZAR",
  "metrics": { "receivablesCents": 0, "overdueCents": 0, "revenueMtdCents": 0, "revenuePrevMonthCents": 0,
               "activeCustomers": 0, "openInvoices": 0, "employees": 0, "lowStockItems": 0 },
  "revenueByMonth": [{ "month": "2026-05", "revenueCents": 0 }],      // last 6 months, oldest first
  "agedReceivables": [{ "bucket": "current", "amountCents": 0 }],     // current, 1-30, 31-60, 61-90, 90+
  "recentInvoices": [ /* invoice list item, max 6 */ ],
  "recentActivity": [{ "id": "uuid", "summary": "...", "actorName": "...", "createdAt": "..." }]  // max 8
}
```

## Permission catalog

| key | module |
|---|---|
| `dashboard:read` | Workspace |
| `customers:read`, `customers:create`, `customers:update`, `customers:delete` | Sales & CRM |
| `invoices:read`, `invoices:create`, `invoices:approve` | Finance |
| `employees:read` | People & HRM |
| `inventory:read` | Inventory |
| `reports:read` | Reports |
| `members:read`, `members:manage` | Administration |
| `roles:read`, `roles:manage` | Administration |
| `company:read`, `company:manage` | Administration |
| `audit:read` | Administration |

Role templates created for every new company: **Company administrator** (`company_admin`, all keys),
**Manager** (`manager`, all `:read` + `customers:create|update`, `invoices:create|approve`, no admin
manage keys), **Employee** (`employee`, `dashboard:read`, `customers:read`, `invoices:read`,
`inventory:read`). The demo company's **Demo viewer** role holds only `:read` keys.

Platform administrator is a user-level flag (`users.is_platform_admin`), separate from company roles; it
grants **no** company data access by itself.
