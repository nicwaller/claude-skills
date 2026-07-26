---
name: pangolin-api
description: Use when the user asks about managing Pangolin (pangolin.net) resources via its REST API — creating/configuring public HTTP resources that proxy to internal services (e.g. Kubernetes ClusterIP/Service DNS names), attaching targets and health checks, or setting up shared access policies (SSO, email allowlists, IP rules). Covers the exact routes, request bodies, and a critical health-check gotcha discovered through live troubleshooting.
---

# Pangolin API

Pangolin (pangolin.net) is a reverse-proxy / access-gateway product. This skill
covers driving it directly over its REST API to create public HTTP resources
that forward to internal backends (e.g. a Kubernetes Service's internal DNS
name), attach health checks, and assign shared access policies.

## Auth & base URL

```
Authorization: Bearer <API key>
```

Base URL: `https://api.pangolin.net/v1`

The real OpenAPI spec is at `https://api.pangolin.net/v1/openapi.json` — fetch
that directly to confirm exact schemas before guessing. **Do not** use
`https://api.pangolin.net/v1/docs/openapi.json` — that path serves the Swagger
UI's HTML shell, not the spec, and is a common trap.

## Route naming quirk

Resources created through the org-level API are called **"public-resource"**
in the route path — a different family from a legacy/generic **"resource"**
path, which mostly only supports GET/read and resource-policy management.
Guessing plausible-looking routes (`POST /v1/resource`,
`/v1/org/.../site/.../resource`, etc.) reliably 404s. The routes that actually
matter:

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/org/{orgId}/resources` | List all resources in an org. Returns full objects including nested `targets[]` with `healthStatus`, `hcEnabled`, `siteId`, `siteOnline`. Best single call for an overview/triage. |
| GET | `/resource/{resourceId}` | Full detail on one resource (mode, ssl, sso, domainId, health, resourcePolicyId, etc). |
| PUT | `/org/{orgId}/public-resource` | Create a new HTTP resource. |
| PUT | `/public-resource/{resourceId}/target` | Attach a backend target to a resource. |
| GET | `/public-resource/{resourceId}/targets` | List targets on a resource, with live `hcHealth` per target. |
| POST | `/public-resource/{resourceId}` | Update a resource — this is how a shared access policy gets attached. |
| GET | `/resource-policy/{resourcePolicyId}` (also mirrored at `/public-resource-policy/{id}`) | Inspect a shared policy. |
| GET | `/org/{orgId}/resource-policies` | List all policies in an org. Needs a broader-scoped API key — see gotcha below. |
| GET | `/org/{orgId}/domains` | List domains available to attach resources under (gives `domainId`). |
| GET | `/org/{orgId}/sites` | List sites (gives `siteId` for target placement). |

## Worked example: create a resource, attach a target with health check, assign a shared policy

Placeholder IDs below (`org_123`, `domain_abc`, `site_xyz`, etc.) stand in for
real values you'd look up first via the `domains`/`sites` list endpoints.

**1. Create the public resource:**

```bash
curl -X PUT https://api.pangolin.net/v1/org/org_123/public-resource \
  -H "Authorization: Bearer $PANGOLIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-app",
    "subdomain": "my-app",
    "domainId": "domain_abc",
    "mode": "http"
  }'
```

A 409 with `"Resource with that domain already exists"` means the subdomain is
already taken under that domain — pick a different subdomain or reuse the
existing resource instead.

This returns a `resourceId` (call it `res_456` below).

**2. Attach a target — with health check fields set explicitly (see gotcha):**

```bash
curl -X PUT https://api.pangolin.net/v1/public-resource/res_456/target \
  -H "Authorization: Bearer $PANGOLIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "siteId": "site_xyz",
    "ip": "my-app.my-namespace.svc.cluster.local",
    "port": 3000,
    "method": "http",
    "mode": "http",

    "hcEnabled": true,
    "hcScheme": "http",
    "hcMode": "http",
    "hcHostname": "my-app.my-namespace.svc.cluster.local",
    "hcPort": 3000,
    "hcPath": "/",
    "hcMethod": "GET",
    "hcInterval": 5,
    "hcUnhealthyInterval": 30,
    "hcTimeout": 5,
    "hcFollowRedirects": true,
    "hcHealthyThreshold": 1,
    "hcUnhealthyThreshold": 1
  }'
```

`ip` doesn't need to be a literal IP — a hostname is fine, e.g. a Kubernetes
internal Service DNS name like `my-app.my-namespace.svc.cluster.local`.

**3. Assign a shared access policy to the resource:**

```bash
curl -X POST https://api.pangolin.net/v1/public-resource/res_456 \
  -H "Authorization: Bearer $PANGOLIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"resourcePolicyId": 42}'
```

Once a shared policy is assigned, per-resource fields like `sso` /
`emailWhitelistEnabled` / `applyRules` become *overrides on top of* the shared
policy rather than the source of truth for that resource. Inspect the policy
itself to see what it actually grants:

```bash
curl https://api.pangolin.net/v1/resource-policy/42 \
  -H "Authorization: Bearer $PANGOLIN_API_KEY"
```

The policy object contains `sso`, `applyRules`, `emailWhitelistEnabled`,
`users[]` (specific accounts granted access), `emailWhiteList[]`, and
`rules[]` (e.g. IP-based ACCEPT/DENY rules). This is how you layer IP
allowlisting on top of SSO for sensitive internal tools.

**4. Confirm health after propagation delay:**

```bash
curl https://api.pangolin.net/v1/public-resource/res_456/targets \
  -H "Authorization: Bearer $PANGOLIN_API_KEY"
```

Wait ~10-15 seconds after creating/updating a target before trusting
`hcHealth` — it can still show the old/unhealthy value until the next
health-check tick fires. An immediate re-GET right after the PUT is not final.

## Critical gotcha: silent health check failure from a null hcPort

A target has **two fully separate sets of fields**:

- Connection fields: `ip`, `port` (where traffic actually proxies to)
- Health-check fields: `hcEnabled`, `hcPath`, `hcScheme`, `hcMode`,
  `hcHostname`, `hcPort`, `hcMethod`, `hcInterval`, `hcUnhealthyInterval`,
  `hcTimeout`, `hcFollowRedirects`, `hcHealthyThreshold`,
  `hcUnhealthyThreshold`

If you create a target and leave `hcPort` null/unset — **even when the real
`port` is set correctly and traffic actually proxies through it fine** — the
health checker never comes up healthy, and the resource is stuck reporting
"unhealthy" with **no error surfaced anywhere obvious**.

There's no visible complaint in the newt agent logs either. A working target
logs proxied requests normally:

```
HTTP handler: GET / -> http://target:port
```

But a target whose health check never got configured simply **never** logs a
line like:

```
Starting health check monitoring for target N (...)
```

That absence — not an error, just a missing log line — is the tell.

**Fix: always explicitly set `hcPort` equal to the target's real `port`, and
`hcHostname` equal to the target's `ip`, when creating a target.** Never leave
these to default/null.

This was the root cause of a real incident: a working target for one app was
marked unhealthy in the dashboard purely because of a null `hcPort`, while an
equivalent target for a different app — which had explicitly set `hcPort` —
was healthy the whole time.

### Recommended health-check defaults

These mirror a known-working setup and are a reasonable starting point absent
other requirements:

```json
{
  "hcPath": "/",
  "hcScheme": "http",
  "hcMode": "http",
  "hcMethod": "GET",
  "hcInterval": 5,
  "hcUnhealthyInterval": 30,
  "hcTimeout": 5,
  "hcFollowRedirects": true,
  "hcHealthyThreshold": 1,
  "hcUnhealthyThreshold": 1
}
```

## Other gotchas

- `GET /org/{orgId}/resource-policies` (list all policies in an org) needs a
  broader-scoped API key. A resource-scoped/org key that can read individual
  policies by ID just fine may still get a 403
  `"Key does not have permission to perform this action"` on this listing
  endpoint. If that happens, fetch known policy IDs individually via
  `GET /resource-policy/{resourcePolicyId}` instead of trying to enumerate them.

## Alternative: declarative Blueprints

Pangolin also supports a declarative YAML/JSON config mechanism called
**Blueprints** — sections like `public-resources`, `private-resources`,
`public-policies`, `sites`, each entry keyed by a stable ID — applied in one
shot via:

```
PUT /org/{orgId}/blueprint
Body: {"blueprint": "<base64-encoded JSON of the YAML-equivalent config>"}
```

This can replace the whole step-by-step create/target/policy sequence above
for bulk or repeatable setups. However, the step-by-step REST calls above are
more transparent for one-off or interactive troubleshooting work, and are the
recommended default when working interactively — reach for Blueprints only
when the user explicitly wants a declarative/bulk config approach.
