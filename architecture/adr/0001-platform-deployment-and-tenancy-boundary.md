# ADR 0001: Tablio platform deployment and tenancy boundary

## Status

Accepted (2026-08-15) — amended 2026-08-15 (stage tunnel contract, AAAA, token split)

## Date

2026-08-15

## Context

Tablio is a multi-tenant hospitality SaaS. The first code will run in two environments (WSL stage and HEL1 production) and will serve many restaurants from a shared API host. If tenancy, host surfaces, and release privileges are left implicit, a later retrofit will leak data or couple stage to production.

This ADR locks those boundaries **before** the platform skeleton. Implementation details (scripts, CI steps, Redis indexes, container IDs) belong in the API README and implementation plan, not here.

The governing rule:

> **The host selects the surface. Authentication selects the tenant.**

## Decision

### 1. Backend stack

The product API is **Django + Django REST Framework + Celery**.

FastAPI is not the product API. It may appear later as a sidecar (for example realtime or heavy export), not as a replacement for the SaaS core.

### 2. Two environments

The same application image runs in two isolated runtimes.

| Branch | Environment | Hosts | Ingress |
|--------|-------------|-------|---------|
| `develop` | WSL stage | `admin-stage.tablio.hr`, `api-stage.tablio.hr` | Cloudflare Tunnel → local Traefik |
| `main` | HEL1 production | `admin.tablio.hr`, `api.tablio.hr` | Cloudflare DNS (proxied A, and AAAA only if HEL1 IPv6 is routed to Traefik) → public HEL1 IP → Traefik |

On HEL1, Traefik does **not** create DNS records. It routes and terminates TLS. DNS create/update is a Cloudflare API concern of the production release path. A-only production records are valid until IPv6 is actually routed to Traefik.

Stage ingress is a **remotely-managed** Cloudflare Tunnel. The standing contract:

- Public hostnames `admin-stage.tablio.hr` and `api-stage.tablio.hr` have tunnel ingress to `http://traefik:80` **without rewriting `Host`**.
- Each hostname also has a proxied CNAME to `<tunnel-id>.cfargotunnel.com`.
- CNAME alone is not sufficient. Ingress alone is not sufficient. Both are required.
- The cloudflared connector runs on WSL only, on the Docker network that Traefik uses. It does not run on HEL1.
- Local Traefik must accept the tunnel hop on the HTTP entrypoint. That hop is not a public HTTP surface.

`develop` never deploys to HEL1. `main` never changes the stage tunnel or `*-stage` DNS. Using the Cloudflare API from HEL1 to bootstrap stage records is a one-off, not the standing owner of stage.

### 3. Host isolation — surface only

The `Host` header selects **admin vs API**, not a tenant.

- `/admin/` is served only on admin hosts.
- `/api/v1/` is served only on API hosts.
- Cross-surface requests return **404**.
- An unknown `Host` returns **400**.
- A stage instance does not accept production hosts, and a production instance does not accept stage hosts.

Shared hosts `api.tablio.hr` and `api-stage.tablio.hr` do **not** identify a tenant. A tenant does not need its own public domain.

### 4. Tenant resolution — authentication selects the tenant

Trust order:

1. Host determines surface only (admin or API).
2. The API key determines the tenant.
3. `request.tenant` is set **only** from a valid API key.
4. The client must not choose a tenant via header, query parameter, or body.
5. A tenant identifier in the URL must **never** override the tenant of the API key.

There is no default tenant. Fail closed.

### 5. Data isolation model

- One database and one schema per environment.
- Every tenant-owned model has a required `tenant_id`.
- Isolation is **application-level**. The first version is not database-per-tenant and does not use PostgreSQL RLS.
- Unique constraints that belong to a business namespace must include the tenant.
- Unprotected `.objects.all()`, `.first()`, and lookup by public ID alone are forbidden for **tenant-owned data on request and Celery paths**.
- Migrations, platform admin, and explicit operator tools may use an unscoped manager, but only on a **clearly marked path outside the tenant API**.

### 6. Django admin is the operator surface

`/admin/` in the first cut is the **platform / operator** admin.

- It is not a tenant portal.
- Tenant users do not sign in there.
- A superuser may cross tenant boundaries only as an explicit operator privilege.

A future Tablio administrative frontend is a separate surface. Django admin does not become that product.

### 7. API keys and HTTP authentication contract

API keys are stored as **hash + prefix only**. The raw key is shown once at creation. Key headers are not logged.

| Condition | Response |
|-----------|----------|
| Unknown, expired, or deactivated key | **401** |
| Valid key bound to a suspended or deactivated tenant | controlled **404** |
| Insufficient scope | **403** |

### 8. Celery tenant boundary

The tenant boundary holds outside the HTTP request.

- Every tenant task receives an explicit `tenant_id`.
- The task re-checks that the tenant exists and is active.
- A worker process has no global or “current” tenant.
- A task must not process another tenant’s data by object ID alone.

### 9. Separated databases, secrets, and deploy privileges

- Each environment has its own Postgres instance and its own `SECRET_KEY`.
- Stage deploy must not touch HEL1 or production DNS.
- Production deploy must not touch the stage tunnel or `*-stage` DNS.
- Stage and production Cloudflare upsert secrets live in **separate GitHub Environments**. A production environment must not hold the stage tunnel id. A stage environment must not hold production deploy host or production-only secret names.
- The Cloudflare DNS upsert credential is **not** the Traefik ACME/TLS credential. Same account, two credentials, two jobs. Sharing one all-zones token for both jobs is a temporary operational shortcut, not this decision.
- No Cloudflare token lives in the API repository.

### 10. PostGIS only via the Docker network

Application containers reach Postgres by the service DNS name on the external Docker network `postgis`.

They must not use a container ID. They must not use host `127.0.0.1` from inside application containers. Host localhost remains for operators on the WSL or HEL1 machine only.

Readiness must succeed against the Tablio database, not merely against a listening PostGIS container.

### 11. Promote PR is the only production path

The only `develop` → `main` path is a **Promote to production** pull request.

- `develop` never deploys to HEL1.
- `main` never changes stage.
- Production deploy runs only from `main` after that promote.

## Rejected alternatives

- **Database-per-tenant** — rejected for the first version because of operational complexity.
- **Tenant from request header or body** — rejected because the client can forge it.
- **Default tenant** — rejected because of data-leak risk.
- **Shared admin/API host** — rejected because it weakens the security boundary.
- **Direct deploy `develop` → production** — rejected.
- **One Cloudflare token for DNS upsert and Traefik ACME** — rejected because rotation and blast radius are different jobs.
- **CNAME-only stage without tunnel ingress** — rejected because the hostname never reaches WSL Traefik.
- **Stage tunnel connector on HEL1** — rejected because production must not own the stage hop.
- **HEL1 as the standing owner of stage DNS or the stage tunnel** — rejected. Bootstrap via the Cloudflare API is not the ongoing release path.

## Consequences

### Positive

- Host and tenant are separate concerns, so a shared API host cannot accidentally become “the” tenant.
- Release is repeatable: stage from `develop`, production only after Promote.
- Blast radius is smaller: stage credentials and deploy rights cannot change production. Separate GitHub Environments keep the stage tunnel id out of production and production deploy host out of stage.
- Scoped querysets and explicit Celery `tenant_id` reduce accidental cross-tenant reads.

### Negative

- Two environments mean more infrastructure and two configuration sets.
- Stricter queryset and Celery patterns slow the first code and require discipline in review.

### Neutral

- The same image runs in both environments; runtime configuration stays separate.
- A tenant does not need a custom domain to operate on the shared API host.
- Stage Traefik listening on HTTP for the tunnel hop is an implementation of this ingress contract, not a decision to expose stage on the public internet without TLS. Edge TLS remains at Cloudflare.

## See also

- API platform skeleton: [tablio-hr/api#1](https://github.com/tablio-hr/api/pull/1) (merge `cfb6a380dd365b84aa681b97ca318aa1fd6690f9`)

## Out of scope

This ADR does not specify container IDs, script names, Redis database indexes, workflow filenames, or CI step lists. Those belong in the API README and the implementation plan.

It does not decide venue, floor-plan, reservation, inventory, or staff domain models. Those belong in a later product-domain ADR. It does not decide a tenant portal or reception login.

## Amendment — 2026-08-15: POS DeviceCredential may select the tenant

The original Decision 4 trust order and “the client must not choose a tenant” remain in the original text.

ADR 0019 owns POS device registration and `DeviceCredential`. Authentication may select the tenant via an API key **or** a valid POS `DeviceCredential`. The client still must not send `tenant_id`. Machine API keys remain for non-POS integrations.

This amendment does not change host isolation, application-level tenancy, Django admin as the platform surface, or the Promote-to-production path.
