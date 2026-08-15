# ADR 0017: Staff Identity, Roles and Operator Authorization

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a staff admin product, IdP vendor, shift clock, cash drawer, or POS device registration in application code.

## Date

2026-08-15

## Context

ADR 0001: the host selects the surface; authentication selects the tenant. The client must not send `tenant_id`. Django `/admin/` is the platform operator surface, not a tenant portal. Machine API keys stay ADR 0001.

ADR 0012 stores a responsible operator on the Ticket. ADR 0015 requires a separate permission for manual overbooking. ADR 0016 owns pricing thresholds and atomic approval consume, and references roles without owning the catalog.

Without this ADR, a shared shift PIN would author every ticket, a waiter in tenant A could act in tenant B, a rehired employee would revive an old POS session, a custom role could silently gain `emergency.override`, and a second membership row would let one person approve themselves.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks the staff identity and authorization domain **before** POS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
UserIdentity       = person and login
StaffMembership    = durable person↔tenant relation
MembershipEpisode  = current employment period
LocationAssignment = where they may work
Role               = named permission set
RoleVersion        = immutable permission snapshot
RoleAssignment     = which role version they hold
Permission         = which action they may execute
OperatorSession    = who is using this POS device now
Approval           = one-time grant for a sensitive action (0016)
```

A Ticket’s responsible operator is a `StaffMembership`, not a display name.

## Decision

### 1. UserIdentity

`UserIdentity` is the real person. It is not employment, a shift, or a POS role.

```text
UserIdentity
------------
stable id
name
primary login
status          # ACTIVE | LOCKED | DISABLED
MFA material
security events
```

One person may be a manager in one tenant, a waiter in another, and have no access to a third. Rights never cross tenants.

### 2. One permanent membership per tenant

Access to a tenant exists only through a `StaffMembership`.

```text
StaffMembership
---------------
tenant_id
user_identity_id
staff_number
```

```text
UNIQUE (tenant_id, user_identity_id)
```

Membership **id is permanent**. Historical tickets, approvals, and audit keep this id. Rows are never deleted.

Maker-checker compares `StaffMembership` id. A second membership for the same person in the same tenant would let them approve themselves. Rehire **always** creates a new `MembershipEpisode` on the same membership. It **never** creates a second membership.

### 3. MembershipEpisode

Rights are read only from the **current** episode.

```text
MembershipEpisode
-----------------
staff_membership_id
version
status          # INVITED | ACTIVE | SUSPENDED | ENDED
valid_from
valid_until
```

- At most one episode may be `INVITED`, `ACTIVE`, or `SUSPENDED`.
- `ENDED` is terminal **for that episode**.
- `INVITED` has no operational rights.
- `ACTIVE` may receive roles and location assignments.
- `SUSPENDED` immediately loses new operational authority.

Ending an episode increments `authorization_generation`, revokes operator sessions, expires unused approvals, and stops using that episode’s assignments. A new episode starts with **no** operational `LocationAssignment` or `RoleAssignment`. Those must be granted again. Old assignment rows stay for audit.

### 4. LocationAssignment

Active membership does **not** imply all locations.

```text
LocationAssignment
------------------
staff_membership_id
membership_episode_id
scope_type      # TENANT | LOCATION
location_id     # null when TENANT
valid_from
valid_until
status
```

One explicit `TENANT` location assignment means all **current and future** locations of that tenant.

A `TENANT` **RoleAssignment** does **not** grant POS work at a location. Location-scoped commands still need a covering `LocationAssignment`. A location manager cannot approve or act in another location.

### 5. Permission catalog

The backend authorizes by a stable permission code, never `if role == MANAGER`.

This ADR **owns** the catalog. Other ADRs name which permission their action requires. They do not invent codes locally. New codes require a docs change to this catalog.

Tenants **may** create custom roles by combining system permissions. Tenants **may not** create permission names, rename codes, or change the meaning of a system permission.

A role must **not** embed business limits (“discount up to 10 %”). `discount.apply` is this ADR; the threshold and extra approval are ADR 0016.

v1 catalog (closed starter set). `closing.perform` is reserved for ADR 0018; this ADR only names it.

```text
ticket.create
ticket.send
ticket.transfer
ticket.void
ticket.reprice
discount.apply
price.override
comp.request
comp.approve
payment.accept
payment.reverse
reservation.overbook
staff.manage
role.manage
role.assign
emergency.override
drawer.open_no_sale
closing.perform
```

### 6. Role, RoleVersion, and RoleAssignment

System roles cannot be deleted: `WAITER`, `BARTENDER`, `KITCHEN`, `SHIFT_MANAGER`, `LOCATION_MANAGER`, `TENANT_ADMIN`. Tenants may add custom roles.

```text
RoleVersion
-----------
role_id
version
permission set
```

`RoleVersion` is immutable. Editing the permission set creates a new version. Activating a new version is an audited action. An existing `RoleAssignment` does **not** move to the new version without that controlled activation.

```text
RoleAssignment
--------------
staff_membership_id
membership_episode_id
role_version_id
scope_type      # TENANT | LOCATION
scope_id
valid_from
valid_until
assigned_by
```

Two valid assignments **union** their permissions. v1 is allow-only. If explicit denies are added later, **deny wins**. Expired assignments grant nothing. A later role change does not rewrite historical audit.

### 7. Delegation boundary

```text
staff.manage   = membership / episode / location assignment
role.manage    = create or version roles
role.assign    = attach a role version to a membership episode
```

- A person must not assign themselves a stronger role without a distinct approval from another membership.
- A person must not assign or put on a role a permission they are not allowed to delegate.
- The tenant must not be left without a last active tenant administrator.
- A custom role must not bypass maker-checker. There is no self-approve flag. ADR 0016 maker-checker stays membership-id inequality.

**Protected permissions** may be granted or assigned only by an authorized tenant administrator:

```text
staff.manage
role.manage
role.assign
emergency.override
```

### 8. OperatorSession and PIN

```text
OperatorSession
---------------
device_id
staff_membership_id
membership_episode_id
location_id
authorization_generation
revoked_at
authenticated_at
expires_at
authentication_method
```

Every command requires the session’s `membership_episode_id` and `authorization_generation` to match the **current** episode and generation. A stale session is invalid even if the same membership later has a new `ACTIVE` episode.

Suspension, episode end, identity `LOCKED` / `DISABLED`, or access revoke **increments generation** and immediately invalidates old sessions (`revoked_at` set).

Device registration and trust stay ADR 0019. Moving a device to another location **ends or re-authenticates** the operator session.

Fast switch may use personal PIN, card, NFC, device biometrics bound to the person, or full login. **Every employee uses their own credential.** A shared shift PIN is rejected.

PIN is not the primary identity. It is not enough for an unknown device, user management, security settings, full tenant backend, or resetting someone else’s credentials.

PIN is hashed, rate-limited, locked after a defined number of failures, changeable without changing `UserIdentity`, and **unique within the tenant**.

### 9. Authentication selects the tenant

After identity login, the issued operational session binds **exactly one** `StaffMembership` (hence one tenant). The client must not send `tenant_id` or an arbitrary `staff_id` as the actor.

Machine API keys stay ADR 0001. Django `/admin/` stays the platform operator surface. `TENANT_ADMIN` is not a platform administrator. Platform support has **no silent** tenant business actions.

This ADR does not amend ADR 0001.

### 10. Command authorization

Every backend command checks at least:

```text
identity ACTIVE
AND current episode ACTIVE
AND LocationAssignment covers the location
AND OperatorSession valid for this device and location
AND session.membership_episode_id = current episode
AND session.authorization_generation = current generation
AND session.revoked_at is null
AND required permission present on assigned RoleVersion(s)
AND business rule satisfied
```

Frontend may hide a button. It is never the authority. Actor identity comes only from the authenticated context.

### 11. Step-up, maker-checker, and revocation

Sensitive actions (large discount, Comp, reverse, manual overbook, no-sale drawer, staff or role change) may require `step_up_required`.

Confirmation is bound to the exact action and payload, short-lived, one-time, distinct from the requester when maker-checker applies, and audited. Consume stays ADR 0016’s one transaction.

Maker-checker:

```text
requester_staff_membership_id
!= approver_staff_membership_id
```

Two PINs on the same membership do not count as two people. A second membership for the same person is impossible.

The approver is evaluated **at consume time**: active identity, current `ACTIVE` episode, covering location assignment, required approve permission, and the ADR 0016 threshold. A later role change does not void a consume that was valid then. An approver who already lost the right cannot consume.

On `SUSPENDED`, `ENDED`, lock, or security disable: increment generation, reject new commands, revoke operator sessions, block new approvals, mark unused approvals `EXPIRED`, keep historical authorship.

### 12. Authorization snapshot

Sensitive business actions persist at least: `user_identity_id`, `staff_membership_id`, `membership_episode_id`, `authorization_generation`, `operator_session_id`, tenant, location, permission used, `role_version_id`(s) that granted it, approval or emergency override, decision time, and result.

Business records reference `StaffMembership`, not a mutable display name.

### 13. Emergency override

`emergency.override` may bypass **only** an explicitly named business rule, for **one** exact action and payload.

It **never** bypasses:

- tenant isolation
- active identity and current `ACTIVE` episode
- covering `LocationAssignment`
- device trust (ADR 0019)
- acting on another tenant

One person with `emergency.override` may activate it with their **own** step-up. That activation is **not** a second-person approval. It always goes to a later review by a **different** manager (different `StaffMembership`).

If the action still requires maker-checker for law or financial control, emergency override **does not lift** that requirement.

Mandatory reason, time-boxed grant, stronger audit. Not a daily substitute for roles.

### 14. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Same person, two tenants: role in A does not authorize a command in B.
- Second `StaffMembership` for the same `(tenant_id, user_identity_id)` is rejected.
- `INVITED` / `SUSPENDED` / ended episode: command rejected; new episode has no inherited assignments.
- `TENANT` RoleAssignment without `TENANT` LocationAssignment: location POS command rejected.
- Location manager of A cannot approve in B.
- Client-sent `staff_id` ignored; actor comes from the session.
- Shared PIN rejected; tenant-duplicate PIN rejected.
- Two PINs, one membership: maker-checker rejected.
- Approver loses `comp.approve` before consume: consume rejected; unused request `EXPIRED`.
- Valid consume stays valid in audit after a later role change.
- Frontend omit-check is not sufficient; backend still enforces.
- Device location change ends or re-auths `OperatorSession`.
- After rehire, an old `OperatorSession` (previous `membership_episode_id` or old `authorization_generation`) is rejected.
- Self-assignment of a stronger role without another membership’s approval is rejected.
- Assigning or embedding a permission the actor cannot delegate is rejected.
- Removing the last active tenant administrator is rejected.
- Editing a role’s permission set creates a new `RoleVersion`; existing assignments keep the old version until audited activation.
- Emergency override without reason is rejected.
- Emergency override against another location or another tenant is rejected.
- Emergency override does not satisfy a still-required maker-checker.

## Rejected alternatives

- A shared shift account or PIN.
- Authorization by role name only.
- A tenant role that implies all locations.
- A client-supplied `staff_id` as the actor.
- Deleting former staff and losing historical authorship.
- Rewriting historical approvals after a role change.
- PIN as a universal account credential.
- Frontend-only permission checks.
- Permanent emergency override.
- Emergency that crosses tenant or location isolation.
- Emergency that replaces a legally required maker-checker.
- An approver who already lost the required right.
- Tenant-invented permission names.
- Rewriting an ended episode on rehire.
- A second `StaffMembership` for the same person in the same tenant.
- An `OperatorSession` that becomes valid again after rehire because only `staff_membership_id` is checked.
- Silently moving a `RoleAssignment` onto a new `RoleVersion`.
- Self-assignment of a stronger role.
- Assigning a permission the actor cannot delegate.
- Removing the last active tenant administrator.
- A custom role that bypasses maker-checker.
- Amending ADR 0001–0014 in this change.

## Consequences

### Positive

- One person can work for several tenants without leaking rights.
- Rehire keeps historical authorship and cannot revive an old POS session.
- Custom roles cannot invent permissions or silently escalate.
- Maker-checker cannot be gamed with a second membership or two PINs.

### Negative

- Every location-scoped command needs an explicit assignment and a live session generation check.
- Role edits require a new version and an audited activation; assignments do not follow automatically.
- Emergency override still needs a second person when maker-checker is required.

### Neutral

- Documentation can merge without a staff UI, IdP, or device registry.
- Shifts and cash drawers stay ADR 0018. Device trust stays ADR 0019.
- ADR 0016 still owns thresholds and atomic consume.

## Invariants

1. UserIdentity ≠ StaffMembership ≠ MembershipEpisode ≠ LocationAssignment ≠ Role ≠ RoleVersion ≠ Permission ≠ OperatorSession ≠ Approval.
2. `UNIQUE (tenant_id, user_identity_id)`. Rehire is a new episode on the same membership.
3. Rights come only from the current episode. `INVITED` has none. `ENDED` is terminal for that episode. Membership rows are not deleted.
4. Active membership does not imply all locations. `TENANT` RoleAssignment is not location access. A `TENANT` LocationAssignment is explicit and covers current and future locations.
5. Backend authorizes by catalog permission, not role name. Tenants compose custom roles from system permissions only. RoleVersion is immutable; assignments do not silently follow a new version.
6. Protected permissions (`staff.manage`, `role.manage`, `role.assign`, `emergency.override`) are assigned only by an authorized tenant administrator. No self-escalation, no last-admin removal, no custom-role maker-checker bypass.
7. OperatorSession is bound to episode and `authorization_generation`. Generation increment plus `revoked_at` invalidates old sessions, including after rehire.
8. Authentication selects the tenant via one bound membership. The client does not send `tenant_id` or the actor `staff_id`. Frontend is not the authority.
9. Maker-checker is membership-id inequality, evaluated at consume time. Unused approvals expire when the staff loses the right. A later role change does not rewrite a valid historical consume.
10. Emergency override bypasses only a named business rule for one payload. It never bypasses tenant, identity, episode, location, or device trust, and it does not lift a required maker-checker.
11. Sensitive actions store an authorization snapshot including `role_version_id`. Business records reference `StaffMembership`.
12. Tenant isolation. Ids alone do not authorize. `TENANT_ADMIN` is not a platform administrator.

## Follow-up ADRs

```text
Shifts, Cash Drawers and Daily Closing
POS Devices, Registration and Configuration
Audit Trail, Data Retention and Privacy
```

Do not implement shifts, device registration, or an IdP vendor from this ADR.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0012: POS Tickets, Ordering and Service Workflow](0012-pos-tickets-ordering-and-service-workflow.md)
- [ADR 0015: Reservations, Waitlist and Guest Seating](0015-reservations-waitlist-and-guest-seating.md)
- [ADR 0016: Price Lists, Discounts, Comps and Approval Rules](0016-price-lists-discounts-comps-and-approval-rules.md)

## Out of scope

This ADR does not define:

- shifts, clock-in/out, cash-drawer assignment, daily closing, or hours (ADR 0018)
- device registration, certificates, kiosk provisioning, or offline credential cache (ADR 0019)
- password-policy product UI, SSO vendor, or biometric vendor
- KDS bump, seating, or reservation CRUD permissions beyond `reservation.overbook`
- PIN length, lockout count, or MFA factors
- staff roster chrome or POS screen layout

## Amendment — 2026-08-15: Shift, drawer, wallet, and business-day permissions owned by ADR 0018

The original Decision 5 catalog ownership and “tenants cannot invent permission names” remain in the original text.

Reserved `closing.perform` is specified as `business_day.close`. `drawer.open_no_sale` stays. ADR 0018 adds:

```text
shift.clock_in
shift.clock_out
shift.correct
drawer.open
drawer.use
drawer.handover
drawer.cash_in
drawer.cash_out
drawer.safe_drop
drawer.close
drawer.variance_approve
wallet.open
wallet.close
wallet.handover
cash.transfer
safe.count
business_day.open
business_day.close
business_day.exception
```

`operator_requires_open_staff_shift` is an ADR 0018 location rule on top of this ADR’s identity, episode, location, session, and permission checks. An open shift is not a role.

This amendment does not change `UNIQUE (tenant_id, user_identity_id)`, RoleVersion, maker-checker, or emergency override.

## Amendment — 2026-08-15: Device trust and assignment owned by ADR 0019

The original Decision 8 `OperatorSession` fields and “PIN is not enough for an unknown device” remain in the original text.

ADR 0019 owns device registration, credential, and location assignment. `OperatorSession` also binds `assignment_generation`. `REASSIGNING`, suspend, compromise, or retire revokes the session. The session does not move to a replacement device.

ADR 0019 adds:

```text
device.enroll
device.assign
device.configure
device.suspend
device.retire
device.credential_rotate
device.security_manage
```

A human business action still needs effective device capability **and** an operator permission from this catalog.

This amendment does not change membership uniqueness, RoleVersion, maker-checker, or emergency override.

## Amendment — 2026-08-15: Offline authorization owned by ADR 0020

The original Decision that PIN is not enough for an unknown device, and that the backend authorizes by permission, remain in the original text.

ADR 0020 owns `OfflineAuthorization` and a short, device-bound PIN cache. The device cannot widen permissions. v1 rejects second-approver and emergency override while offline.

ADR 0020 adds:

```text
offline.lease_manage
offline.drain_exception
```

This amendment does not change membership uniqueness, RoleVersion, maker-checker, or online emergency override.
