# ADR 0019: POS Devices, Registration and Configuration

## Status

Proposed

This ADR may be documented and merged while `Proposed`.

This ADR does not authorize a device-management product, MDM vendor, PKI vendor, or POS application code.

## Date

2026-08-15

## Context

ADR 0001: the host selects the surface; authentication selects the tenant. The client must not send `tenant_id`. Machine API keys stay ADR 0001.

ADR 0011 already forbids a client-sent `device_id` and requires the authenticated device on a Viva result. ADR 0017 owns `OperatorSession` and human permissions. ADR 0018 owns cash operation mode and whether a location allows several devices on one drawer. ADR 0014 owns KDS instruction delivery.

Without this ADR, every till would share one API key, a waiter would pick `tenant_id` in the body, a cloned tablet would impersonate the register, a device override would grant itself `payment_capture`, and a location move would leave cash posting to the old drawer.

Database names, API names, and source code use English. The user interface may localize them to Croatian and other languages.

This ADR locks device trust, assignment, and configuration **before** POS implementation. Physical schema details belong in a later implementation. The semantics below must not change once accepted.

The governing rule:

```text
Device                 = physical or logical POS install
DeviceRegistration     = device ↔ tenant
DeviceAssignment       = current location and function
DeviceCredential       = proof of device identity
DeviceConfigVersion    = published source layer
EffectiveDeviceConfig  = compiled immutable result
OperatorSession        = employee using it now (0017)
```

```text
One ACTIVE POS device belongs to exactly one tenant
and exactly one location.
```

A device is not an employee. Registration does not grant `discount.apply`, `payment.accept`, or reverse. Human business actions still need an operator and ADR 0017 permissions.

Tenant and location come from the authenticated device context. The client must not send `tenant_id` or `location_id`. Moving location is an audited backend action, not undo.

## Decision

### 1. POSDevice

One app install gets a durable `device_id`. `RETIRED` is never reused. Reinstall or replacement gets a new identity.

```text
POSDevice
---------
tenant_id
device_id
name
device_type
platform
app_instance_id
status
registered_at
last_seen_at
```

Types describe purpose: `REGISTER`, `HANDHELD`, `KDS`, `SELF_SERVICE`, `BACK_OFFICE`, `PRINTER_GATEWAY`, `CUSTOMER_DISPLAY`. The backend authorizes by **capabilities**, not `if type == REGISTER`.

```text
PENDING | ACTIVE | REASSIGNING | SUSPENDED | COMPROMISED | RETIRED
```

- `PENDING` — enrollment started, no operational trust
- `ACTIVE` — may open a device session and run allowed commands
- `REASSIGNING` — location move in progress; no new business commands
- `SUSPENDED` — temporary block; may return to `ACTIVE`
- `COMPROMISED` — credentials revoked immediately
- `RETIRED` — terminal; do not reactivate

### 2. One tenant, one location

An active POS install belongs to **one tenant**. Changing tenant: revoke registration, wipe local tenant data and credentials, new registration, new credential, audit.

An active device has **one current** `DeviceAssignment`: `location_id`, validity, `assigned_by`, `assignment_generation`.

A valid `DeviceCredential` sets `request.tenant` the same way an ADR 0001 API key does. The client still cannot choose tenant. Machine API keys remain for non-POS integrations.

The device must not post business actions in two locations at once. Return to a previous location is a **new** assignment, not undo.

### 3. REASSIGNING

```text
ACTIVE
→ REASSIGNING
→ new-location EffectiveDeviceConfig APPLIED
→ ACTIVE
```

Entering `REASSIGNING` increments `assignment_generation`, revokes `OperatorSession`s, and stops **new** business commands.

Before leaving `REASSIGNING` the backend checks at least: no device-bound Payment `UNKNOWN`; no unfinished terminal operation; no active closing claim that depends on the device; operator session revoked; critical peripherals unbound; no open offline command queue if one exists (ADR 0020).

If the new config fails to apply, the device **stays `REASSIGNING`**. It must not silently return to the old location and must not run with mixed context.

### 4. Enrollment and proof-of-possession

```text
EnrollmentGrant
---------------
tenant_id
location_id
allowed_device_type
expires_at
max_uses = 1
created_by
status
```

The grant code is stored **hashed**, rate-limited, never logged, and bound to the expected tenant, location, and device type. It is **not** a standing API key.

```text
grant → public key → server challenge
→ device signature → credential issuance
```

Sending a public key is not enough. The device must sign a server challenge to prove possession of the matching private key. The grant is consumed **atomically only after** successful proof-of-possession.

The key pair is born on the device. The backend does **server-side credential issuance**, not server-side key generation.

If the backend consumes the grant and creates the device, then the response is lost:

- Retry with the **same grant and same public key**, plus a valid proof-of-possession signature, returns the **same** device / issuance result. It must not create a second device and must not leave the device permanently blocked.
- The same grant with a **different** public key is rejected and audited as a suspicious attempt.

### 5. DeviceCredential and rotation

Asymmetric key or certificate. The private key is born and stays on the device. The backend stores the public key. Requests bind to device and tenant. Revoke applies immediately to new requests.

Forbidden: one API key for all POS devices; copying a credential to a replacement; storing the private key in a cloneable backup; a client-chosen `device_id`.

```text
ISSUED | ACTIVE | ROTATING | REVOKED | EXPIRED
```

Rotation requires proof of the current private key **or** specially authorized recovery. The new key gets a **higher generation**. Old and new may overlap only for a short, controlled window. After the new key is confirmed, the old is revoked. Rollback to an older generation is forbidden. Compromise revokes **all** active generations.

Clone protection: unique local key pair; server-side **issuance**; detect illogical parallel sessions; mark `COMPROMISED` and revoke all credentials; new identity after reinstall if the old key cannot be proven. Hardware attestation is an optional extra signal, not required on every platform.

### 6. Online anti-replay

A device credential proves who sent the request. A signed online request must cover at least:

```text
HTTP method
path
body hash
device_id
credential generation
timestamp
nonce or security idempotency key
```

The backend rejects replay outside the allowed window. A **business** idempotency key and a **security** anti-replay nonce are not the same thing. The exact offline replay protocol stays ADR 0020.

### 7. Capabilities versus operator

v1 capability catalog. New codes require a docs change.

```text
ticket_entry
payment_capture
cash_drawer_control
receipt_print
kds_display
customer_display
barcode_scan
table_service
```

```text
capability ceiling     = what tenant and device type allow
effective capability   = narrower set for this location and device
UI / config preference = ordinary settings
```

“Most specific wins” applies to preferences. A device override **must not** raise the ceiling. A lower layer may only **narrow** capabilities. Expanding a protected capability (`payment_capture`, `cash_drawer_control`, and later protected codes) requires `device.security_manage` and audit.

The backend computes capability. Device acknowledgement is not a security authority. The device only confirms which `EffectiveDeviceConfig` it applied.

Human business action:

```text
effective device capability
AND operator permission (0017)
```

Machine actions (KDS ack, printer delivery ack, customer-display heartbeat, config ack, health) use device identity and a closed machine capability. They must not accept an arbitrary `staff_id`. If the device acts for a person, the operator must be known.

Examples: KDS may read production instructions, not take cash. A customer display may show a limited Ticket, not mutate it. A handheld may send orders without `cash_drawer_control`. A register may capture payment and open a mapped drawer.

### 8. OperatorSession

ADR 0017 owns `OperatorSession`. This ADR adds:

- device `ACTIVE` (not `REASSIGNING`)
- credential `ACTIVE`
- assignment covers the location
- session binds `device_id` **and** `assignment_generation`
- suspend, compromise, retire, or `REASSIGNING` revokes the session
- the session does not move to a replacement device

A registered device with no operator may run only allowed machine functions. PIN on an unknown device stays forbidden (ADR 0017).

### 9. EffectiveDeviceConfig

Source layers. Most specific wins for **preferences** only:

```text
tenant defaults
→ location configuration
→ device-type profile
→ explicit device override
```

Equal-level conflict is rejected at **publish**.

`DeviceConfigVersion` is immutable when `PUBLISHED`. An edit creates a new version (`DRAFT` | `PUBLISHED` | `RETIRED`).

Config **rollback** does not reactivate an old published version. Rollback is published as a **new** version with a **new generation** and the previous content.

The compiled result is its own immutable fact:

```text
EffectiveDeviceConfig
---------------------
effective_config_id
schema_version
source_version_ids
canonical_payload_hash
config_generation
```

The device reports `applied_effective_config_id` and the hash (`PENDING` | `APPLIED` | `FAILED` | `UNSUPPORTED`). The backend must not treat desired sources as applied. A business event snapshots `effective_config_id` and hash. If the compiled hash does not match the payload, `APPLIED` is rejected.

Atomic apply: download the full compiled payload → validate `schema_version` and hash → check required capabilities and peripherals → store locally → activate atomically → acknowledge. On failure: keep the last good config, report `FAILED` / `UNSUPPORTED`, do not claim the new id is active, show drift.

Non-critical: name, theme, button layout, display timeout. Critical: tenant/location, payment capabilities, cash-drawer mapping, fiscal printer, KDS station, credential policy, min app version. A critical change needs a new `config_generation`, operator re-auth, a successful apply ack, and may block the related business function until applied.

Device↔drawer mapping lives in this config. ADR 0018 still owns cash operation mode and whether the location allows a shared drawer.

### 10. App version, heartbeat, and time

App policy: `SUPPORTED` | `UPDATE_REQUIRED` | `BLOCKED`. `UPDATE_REQUIRED` may finish safe in-flight flows but should upgrade. `BLOCKED` must not start new business commands. Blocking must not abandon an unfinished payment without a controlled recovery path. Offline rules stay ADR 0020.

Heartbeat: status, app/OS version, applied `effective_config_id`, credential expiry, last successful sync, peripheral health, free space, device time and skew. Heartbeat is **not** proof that business messages are synced. Missing heartbeat may mark the device offline. It must **not** auto-`RETIRED`.

Server time is the authority for durable business timestamps. Store server received time, device reported time, and skew. Large skew may block a security-sensitive action or warn.

### 11. Peripherals

Config may bind printer, cash drawer, scanner, customer display, payment terminal, KDS screen. Each peripheral has a stable id and health. A printer or drawer change is audited. Printer failure **after** a business commit retries **delivery of the same document**. It must not repeat the sale or payment.

### 12. Suspend, compromise, retire, and in-flight work

Distinguish: a **new** business command; an **idempotent retry** of an already committed command; **recovery status** of an unknown operation.

After `SUSPENDED`, `COMPROMISED`, or `RETIRED`:

- a new business command is rejected
- an idempotent retry of an already committed command may return the existing result with **no new write**
- a closed recovery channel may check the status of a previously started payment
- recovery must not start a new sale, payment, or refund

- `SUSPENDED` — stop new business commands; may return to `ACTIVE` after review.
- `COMPROMISED` — revoke all active credential generations, end operator sessions, block all but the closed recovery channel, require a new credential or new device identity, raise a security event. Uses `device.security_manage`.
- `RETIRED` — terminal; revoke; end sessions; keep historical references; do not delete business records; replacement gets a new `device_id`.

### 13. Permissions

ADR 0017 owns the catalog. This ADR adds:

```text
device.enroll
device.assign
device.configure
device.suspend
device.retire
device.credential_rotate
device.security_manage
```

Expanding a protected capability ceiling requires `device.security_manage`. The backend checks the permission **and** the business rule.

### 14. Audit

Keep at least: who created the enrollment grant; who registered or approved the device; public credential fingerprint; tenant and location; assignment history; `effective_config_id` and acknowledgement; capability changes; suspend / compromise / retire; rotation and revoke; operator sessions; app version and config snapshot on a business action.

A later config, role, or assignment change does not rewrite that snapshot.

### 15. Mandatory acceptance tests

An implementation of this ADR must cover at least:

- Client-sent `tenant_id` or `location_id` ignored; context comes from the device credential and current assignment.
- Second active tenant on the same install is rejected; tenant change requires revoke, wipe, and new identity.
- Device cannot post in location B while assigned to A.
- Public key without a valid challenge signature cannot finish enrollment.
- Lost enrollment response + same grant + same key + valid proof-of-possession returns the same device; a different public key is rejected and audited.
- Grant is hashed, not logged; consume happens only after successful proof-of-possession.
- Repeated signed online request with the same nonce is rejected.
- Location move enters `REASSIGNING`; new business commands are rejected; `assignment_generation` increments; operator sessions revoked.
- Location move with a device-bound `UNKNOWN` payment is rejected.
- Failed new-location apply leaves the device `REASSIGNING` (no silent return, no mixed context).
- Return to the previous location is a new assignment, not undo.
- Shared / copied device credential is rejected; a clone parallel session can mark `COMPROMISED`.
- `RETIRED` identity is not reactivated; replacement has a new `device_id`.
- KDS / customer-display machine ack without operator is allowed; cash capture without operator is rejected.
- Handheld without `cash_drawer_control` cannot open a drawer even if the operator has `drawer.open`.
- Operator with `payment.accept` on a device without effective `payment_capture` is rejected.
- Device override cannot raise the capability ceiling.
- Equal-level config conflict is rejected at publish.
- Compiled config hash mismatch → `APPLIED` rejected; last good config remains.
- Desired sources ≠ applied `effective_config_id`; drift is visible.
- Cash payment after a critical drawer-mapping change and before that `effective_config_id` is `APPLIED` is rejected.
- Historical payment snapshot keeps the old `effective_config_id` after a later publish.
- Config rollback is a new version and new `config_generation`, not reactivation of the old published row.
- Old credential after completed rotation is rejected; generation rollback is rejected.
- Missed heartbeat marks offline, not `RETIRED`.
- `BLOCKED` app version cannot start a new business command; in-flight payment has a recovery path (detail ADR 0020).
- Printer retry after commit delivers the same document and does not create a second Payment.
- `SUSPENDED` then `ACTIVE` is allowed; `COMPROMISED` requires `device.security_manage` and a new credential or identity; `RETIRED` is terminal.
- After suspend / compromise / retire: new write rejected; idempotent retry of a committed command returns the same result; closed recovery may read a prior payment status; recovery cannot start a new sale, payment, or refund.
- Operator session does not transfer to a replacement device.

## Rejected alternatives

- One standing API key for all devices.
- Client `tenant_id` or `location_id` as context.
- A device active in two tenants or two locations.
- A registered device as a substitute for operator permission.
- A shared device credential.
- Reuse of a retired identity.
- Enrollment without proof-of-possession.
- Server-generated private keys.
- Consuming a grant before proof-of-possession.
- A second device from a lost-response retry.
- The same grant with a different public key.
- Unsigned or replayable online device requests.
- Treating a business idempotency key as anti-replay.
- Treating desired source versions as applied.
- No compiled `EffectiveDeviceConfig` id/hash.
- A device override that raises the capability ceiling.
- Device acknowledgement as capability authority.
- Silent return to the old location on failed reassignment.
- Mixed-context work while `REASSIGNING`.
- Reactivating an old config version or credential generation.
- Auto-retire on a missed heartbeat.
- A printer retry that repeats the business transaction.
- Moving an active operator session to a new location or replacement device.
- Blocking recovery of an already-started `UNKNOWN` payment.
- Deleting an old device and losing audit.
- Amending ADR 0002–0010 or 0012–0016 in this change.

## Consequences

### Positive

- A cloned tablet cannot silently become the register.
- Tenant and location cannot be forged in the request body.
- A location move cannot post cash to the old drawer under mixed config.
- A security block can still resolve an already-started `UNKNOWN` payment without a new charge.

### Negative

- Enrollment and every online device request need proof-of-possession and anti-replay.
- A failed location move leaves the device in `REASSIGNING` until an operator finishes it.
- Capability ceilings make device-level “just enable payments” fail closed.

### Neutral

- Documentation can merge without MDM, a PKI vendor, or POS application code.
- Offline queue and replay stay ADR 0020.
- ADR 0017 still owns who the operator is. ADR 0018 still owns cash mode.

## Invariants

1. Device ≠ employee ≠ OperatorSession ≠ Payment. Registration is not a business permission.
2. One `ACTIVE` device belongs to exactly one tenant and one current location. The client does not send `tenant_id` or `location_id`.
3. Enrollment consumes a hashed one-time grant only after a valid challenge signature. Same grant + same key is idempotent. A different key is rejected.
4. The backend issues credentials; it does not generate the device private key. Rotation raises generation; rollback to an older generation is forbidden. Compromise revokes all active generations.
5. A signed online request includes method, path, body hash, device, generation, timestamp, and nonce. Business idempotency ≠ anti-replay. Offline replay is ADR 0020.
6. Effective capability cannot exceed the ceiling. The backend computes capability. `EffectiveDeviceConfig` is the compiled immutable result; desired sources are not applied.
7. Location move uses `REASSIGNING`. Failure stays there. No silent return and no mixed context.
8. After suspend / compromise / retire, new writes are rejected. Idempotent retry of a committed command and closed recovery of a prior payment status are allowed. Recovery cannot start a new sale, payment, or refund.
9. Printer retry after commit delivers the same document. Missed heartbeat is not `RETIRED`.
10. Tenant isolation. Ids alone do not authorize. Historical snapshots keep `effective_config_id` and `assignment_generation`.

## Follow-up ADRs

```text
Offline POS Operation and Synchronization
```

Do not implement an offline queue, MDM, or a PKI vendor from this ADR.

## See also

- [ADR 0001: Platform deployment and tenancy boundary](0001-platform-deployment-and-tenancy-boundary.md)
- [ADR 0011: Payments and Settlement](0011-payments-and-settlement.md)
- [ADR 0017: Staff Identity, Roles and Operator Authorization](0017-staff-identity-roles-and-operator-authorization.md)
- [ADR 0018: Shifts, Cash Drawers and Daily Closing](0018-shifts-cash-drawers-and-daily-closing.md)

## Out of scope

This ADR does not define:

- offline command queue, conflict merge, local counters, offline PIN cache, sync cursor, or offline replay (ADR 0020)
- MDM vendor or OS provisioning
- app-store distribution
- concrete PKI vendor
- printer or payment-terminal wire protocols
- KDS screen UX
- exact grant TTL, replay window, rotation overlap, or clock-skew threshold
- POS screen layout
