# Slice 1: Tenant, location and authorization context

## Status

Locked (2026-08-16)

This plan authorizes API work. ADR 0017, 0027, and 0028 stay `Proposed`.
This slice implements a declared subset. It does not accept those ADRs.

API PR A must not start before this document is merged.

## Goal

Prove, from PostgreSQL constraints through an API command and tests, that every
later business operation knows:

```text
who acts
→ for which tenant
→ at which location
→ through which active membership episode
→ with which permission
```

## Out of scope

Product, tax, warehouse ledger, POS Ticket, fiscalization, SaaS billing, POS
`OperatorSession`, PIN, device trust, custom role editor, `emergency.override`,
maker-checker consume, hash-chain audit, privacy, OAuth, webhooks, IdP/SSO/MFA,
cookie session / CSRF.

## Existing platform

The API skeleton already implements [ADR 0001](../adr/0001-platform-deployment-and-tenancy-boundary.md):
host selects surface; authentication selects tenant; `ApiApplication` hash +
prefix; controlled 404 on a suspended tenant; `for_request_tenant()`;
`TenantTask` with an explicit `tenant_id`.

`TenantMembership` (Django `User` ↔ `Tenant`) is a stub. Remove it from the
staff path. Django `User` remains the platform operator on `/admin/` only.
Tenant people are not `AUTH_USER_MODEL`.

`BusinessLocation` is the place entity from [ADR 0003](../adr/0003-warehouse-and-inventory.md).
Do not invent a second Location type. `location_id` in ADR 0017 refers to it.

## Permission catalog change

[ADR 0017](../adr/0017-staff-identity-roles-and-operator-authorization.md) adds:

```text
location.view
location.manage
```

`staff.manage` and `role.assign` already exist. `role.manage` is not a slice 1
API. `location.view` does not grant mutate. `location.manage` does not imply
all locations.

## Authorization context

```text
identity ACTIVE
AND tenant ACTIVE
AND current episode status = ACTIVE
AND now ∈ [episode.valid_from, episode.valid_until)
AND session.episode_id = current episode
AND session.authorization_generation = membership.authorization_generation
AND session.expires_at > now
AND session.revoked_at is null
AND LocationAssignment covers the target (TENANT or that LOCATION)
AND assignment status active AND now ∈ [assignment.valid_from, assignment.valid_until)
AND required permission on the union of RoleVersions whose scope
    intersects the covering LocationAssignment
AND resource.tenant_id = session.tenant_id
AND target location is_active when the command is a new business operation
```

Status and interval are both authority. `INVITED` has no rights. Rehire is a
new episode on the same membership and inherits no assignments.

Rules:

1. The client must not send `tenant_id`. Tenant comes from the staff session or
   from an API key on the machine path.
2. Login binds exactly one `StaffMembership`. A person with several tenants
   selects `staff_membership_id`, not `tenant_id`.
3. Machine `ApiApplication` must not execute `staff.manage` or `location.manage`.
4. `StaffAccessSession` is not POS `OperatorSession`. It has no `device_id`.
   It still carries `membership_episode_id` and `authorization_generation`.
5. Body `staff_id` is ignored or rejected. The actor comes from the session.
6. A request must not carry a staff Bearer token and an API key together
   (400). Staff routes accept only a staff session. Machine routes accept only
   an API key.

### Atomic generation bump

Live generation mismatch invalidates a session. Do not `UPDATE` every
`StaffAccessSession.revoked_at` on an authority change. Set `revoked_at` only
on explicit logout (and optionally an expiry job).

Every authority change, in one transaction:

```text
SELECT StaffMembership … FOR UPDATE
→ change episode / assignment / identity state
→ increment authorization_generation
→ write AuditEvent SUCCESS
→ commit
```

Includes: activate / suspend / end episode; add or revoke location or role
assignment; lock / disable identity. Tenant suspend does not bump every
membership: `authorize()` already requires `tenant ACTIVE`.

### Login identifier

Before compare and store: trim, Unicode NFKC, email casefold. Persist only the
normalized form. PostgreSQL `UNIQUE` on `Lower(primary_login)`.

The same external 401 (same `code`, no hint) for: unknown identity; wrong
password; `LOCKED` / `DISABLED`; missing, inactive, or foreign selected
membership. Verify a dummy hash on unknown identity so timing does not
enumerate accounts.

### Session token

Bearer in `Authorization`. No cookie session in this slice.

- at least 256 bits of entropy
- raw value shown once at login
- store prefix + hash only
- absolute `expires_at` (duration in the API README)
- never in URL, logs, audit, or error payload
- logout is idempotent (already revoked or unknown prefix → 204)

## Models and PostgreSQL invariants

English names. `public_id` (UUID) is the external id. Lookup is always
tenant-bound.

Tenant-owned tables carry `tenant_id`. Cross-table tenant equality is **not**
a `CHECK` on another row. Prove it with composite foreign keys. The domain
service is an extra guard, not the only one.

```text
StaffMembership          UNIQUE (id, tenant_id)
MembershipEpisode        UNIQUE (id, tenant_id)
                         FK (staff_membership_id, tenant_id)
                           → StaffMembership(id, tenant_id)
LocationAssignment       tenant_id NOT NULL
                         FK (staff_membership_id, tenant_id)
                         FK (membership_episode_id, tenant_id)
                         FK (location_id, tenant_id) when LOCATION
RoleAssignment           tenant_id NOT NULL
                         FK (staff_membership_id, tenant_id)
                         FK (membership_episode_id, tenant_id)
                         FK (location_id, tenant_id) when LOCATION
StorageArea              FK (location_id, tenant_id)
                           → BusinessLocation(id, tenant_id)
```

**Tenant.** Add `public_id`. Status stays `active` | `suspended`.

**BusinessLocation.** `tenant_id`, `public_id`, `name`, `timezone`, `is_active`.
`UNIQUE (tenant_id, public_id)`, global `UNIQUE (public_id)`,
`UNIQUE (tenant_id, lower(name))`. Deactivated rows stay readable with
`location.view` and must not be the target of a new business operation. No
address, municipality, legal entity, or fiscal device.

**StorageArea.** Created in the same transaction as the location.
`code = MAIN` (stable internal code). `UNIQUE (location_id, code)`. At most
one default storage per location. Retry must not create a second row. No
inventory API.

**UserIdentity.** Normalized `primary_login`, Django password hasher, status
`ACTIVE | LOCKED | DISABLED`.

**StaffMembership.** `UNIQUE (tenant_id, user_identity_id)`,
`UNIQUE (tenant_id, staff_number)`, `authorization_generation >= 0`,
`ON DELETE PROTECT`. Rows are never deleted.

**MembershipEpisode.** `tenant_id`, `version > 0`, status
`INVITED | ACTIVE | SUSPENDED | ENDED`,
`valid_until IS NULL OR valid_until > valid_from`. At most one
`INVITED|ACTIVE|SUSPENDED` episode per membership. `ENDED` is terminal for
that episode.

**LocationAssignment.** Belongs to the episode (`membership_episode_id`
NOT NULL). `TENANT` ⇒ `location_id IS NULL`. `LOCATION` ⇒ `location_id`
NOT NULL. Valid only while that episode is `ACTIVE` and in interval, and
while the assignment itself is active and in interval. A `TENANT`
assignment covers current and future locations of that tenant.

**RoleAssignment.** No generic `scope_id`. Required `membership_episode_id`.
`TENANT` ⇒ `location_id IS NULL`. `LOCATION` ⇒ concrete `location_id`.
Permission on a command is the intersection of role scope and a covering
`LocationAssignment`. A location role without a location assignment is not
enough. Rehire must not see old role or location assignments.

Slice 1 seeds system `TENANT_ADMIN` with
`{staff.manage, role.assign, location.view, location.manage}`. No custom
roles and no new `RoleVersion` through the API.

**StaffAccessSession.** Bearer token bound to membership, episode, and the
generation snapshot. Every request re-checks live generation, episode,
tenant, and expiry.

## API commands

Staff Bearer on `/api/v1/`. Problem+json. Mutations require `Idempotency-Key`.

- `POST /auth/staff/login` — bind one membership; return the raw token once
- `POST /auth/staff/logout` — idempotent 204
- `GET /me/context` — identity, tenant, assignments, episode, permissions
- `GET /locations`, `GET /locations/{public_id}` → `location.view`.
  List only locations the assignment covers, except `TENANT` scope (all
  tenant locations, including deactivated)
- `POST /locations`, `PATCH`, deactivate → `location.manage`
- `POST /staff/memberships`
- `POST /staff/memberships/{id}/episodes:activate|:suspend|:end`
- `POST /staff/memberships/{id}/location-assignments`
- `POST /staff/memberships/{id}/location-assignments/{assignment_id}:revoke`
- `POST /staff/memberships/{id}/role-assignments`
- `POST /staff/memberships/{id}/role-assignments/{assignment_id}:revoke`

The client may send `location_id` as the resource target. It must not send
`tenant_id`.

### Last tenant administrator

Remove / suspend / end may proceed only if another person remains with all of:

- identity `ACTIVE`
- episode `ACTIVE` and in interval
- active, in-interval `TENANT` LocationAssignment
- active, in-interval `TENANT` `TENANT_ADMIN` RoleAssignment

Check under `SELECT … FOR UPDATE` on the affected membership rows.

### Platform bootstrap

```text
manage.py bootstrap_tenant --slug … --admin-login … --admin-password …
manage.py seed_stage_tenants
manage.py seed_stage_tenants --reset-admin-password
```

Not a tenant API. Changing the env password must not reset an existing
password without `--reset-admin-password`. Seed must not run in PR CI or on
production `main`.

## Stage seed

Sudreg identifies the company. Maps identify the venue. Registered seat ≠
venue. Tests use synthetic tenants (`demo-a`, `demo-b`).

Do not commit Sudreg PDFs, personal OIBs, or home addresses. Company OIB/MBS
are not columns on `Tenant` in this slice.

### Tenant A — `sibenik-1983`

- Tenant name: `ŠIBENIK 1983 j.d.o.o.`, timezone `Europe/Zagreb`
- Location 1: `Caffe Bar Mozart`
- Location 2: `Dvorana Baldekin` (drinks service in the hall)
- First admin: Natalija Radić, login `mozartsibenik@gmail.com`, `staff_number=1`
- `TENANT` LocationAssignment + `TENANT_ADMIN`
- Later LegalEntity notes only: MBS `110092399`, OIB `03442668895`,
  EUID `HRSR.110092399`, seat Šibenik, Stjepana Radića 42
- Venue 1: Caffe Bar Mozart, Stjepana Radića 42
- Venue 2: Sportska dvorana Baldekin, Stjepana Radića 44 — same tenant,
  second location

### Tenant B — `supina-poljica`

- Tenant name: `ŠUPINA POLJICA j.d.o.o.`, timezone `Europe/Zagreb`
- Location: `Restaurant Uzorita`
- First admin: Toni Šupe, login `tonisupe7@gmail.com`, `staff_number=1`
- `TENANT` LocationAssignment + `TENANT_ADMIN`
- Later LegalEntity notes only: MBS `110083401`, OIB `91381354893`,
  EUID `HRSR.110083401`, seat Šibenik, Šibenskih vatrogasaca 10

Seed is idempotent on `Tenant.slug` and `(tenant, location.name)`. Passwords
come from env (`SEED_ADMIN_PASSWORD_SIBENIK_1983`,
`SEED_ADMIN_PASSWORD_SUPINA_POLJICA`) or are generated and printed once.

## Audit

Two paths:

- **SUCCESS** — same transaction as the domain change. Rollback writes no
  SUCCESS. If the audit insert fails, the command rolls back.
- **DENIED** — not in the rejected command transaction. Write it in a
  separate short security transaction after 401/403 is decided. A DENIED
  write failure must not turn 403 into 500 and must not allow the command.

No hash-chain, retention, or privacy in this slice. Do not store passwords or
tokens. `SYSTEM` actors need a named service identity (bootstrap/seed).

## Idempotency

`CommandClaim` key:

```text
actor_type + actor_id + tenant_id + api_version + method + canonical_route
+ idempotency_key_hash
```

States: `IN_PROGRESS | SUCCEEDED | FAILED_FINAL`. `IN_PROGRESS` has a lease.
Auth / malformed / 429 do not consume the key. Same key + different body →
409.

Replay is not a free cache. Re-run authorization before returning a frozen
result. If re-authz fails → 401/403, not the frozen success.

## Code layout

- `apps.identity` — identity, membership, episode, assignments, role,
  session, `authorize()`
- `apps.audit` — `AuditEvent`
- `apps.api` — HTTP, Bearer vs API-key isolation, `CommandClaim`
- `apps.tenants` — `Tenant`, `BusinessLocation`, `StorageArea`

Thin views. Tenant-owned querysets on request paths go through
`for_request_tenant()` or a successor.

## Acceptance tests

- Same person, two tenants: rights in A do not authorize a command in B
- Second `StaffMembership` for the same `(tenant_id, user_identity_id)` rejected
- `INVITED` / `SUSPENDED` / ended: 403; new episode inherits neither
  location nor role assignments
- Body `tenant_id` and `staff_id` do not change the actor
- `GET /me/context` and a command see the same tenant / episode / permissions
- Location-scoped command without a covering assignment rejected;
  `TENANT` assignment covers a newly created location
- Role scope without a location assignment rejected
- Self-assignment of a stronger role rejected
- Last valid tenant admin cannot be removed
- Parallel removal of the two last admins: only one operation succeeds
- Two concurrent `episodes:activate` do not create two current episodes
- Rehire: old session (old episode or old generation) rejected
- Suspended tenant: controlled 404, no slug leak
- Login is case-insensitive and trimmed
- Same login response for unknown / wrong password / disabled / bad membership
- Expired session and expired assignment rejected
- Changing a role or location assignment immediately invalidates the old
  session generation
- Cross-tenant role or location rejected by a **database constraint**, not
  only the service
- Idempotency: retry = same result; different body = 409; parallel = one
  side-effect
- Idempotency replay after the actor is suspended → deny, not frozen success
- SUCCESS audit in the same transaction; rollback has no SUCCESS; DENIED
  write failure does not become 500 or allow the command
- Staff token fails on an API-key-only path and the reverse; both headers → 400
- Deactivated location is readable with `location.view`, not an operational target
- `StorageArea.MAIN` exists after location create; retry does not create a second
- Seed with a changed env password does not reset the stored password without
  `--reset-admin-password`
- Existing API-key contract (`/app/config`, 401/403/404) stays green

## API PR sequence

After this document merges:

1. **API PR A** — models, composite FKs, episode-bound assignments,
   `StorageArea.MAIN`, remove `TenantMembership`, `bootstrap_tenant`,
   `seed_stage_tenants`
2. **API PR B** — Bearer session, normalized login, uniform failures,
   `GET /me/context`, `authorize()`
3. **API PR C** — commands including revoke, atomic generation bump,
   SUCCESS/DENIED audit, idempotency with re-authz, acceptance tests

PR A may land alone. The slice is not done until PR C’s chain tests pass.

## Locked compromises

- Generation mismatch is the canonical revoke; `revoked_at` is logout only
- Tenant suspend is caught by `tenant ACTIVE`, not a mass generation bump
- Audit is an append-only row, not a tamper-evident chain
- Idempotency is a staff-command claim; replay always authorizes again
- `StorageArea.MAIN` is a structural seed, not the warehouse slice
- Bearer, not cookie
