---
title: "The x-tenant-id Pattern: Multi-Tenant API Without Multi-Tenant Complexity"
date: 2026-05-23 12:00:00 +0530
categories: [Engineering, Backend]
tags: [multi-tenancy, api-design, node, express, jwt, architecture]
description: A single request header, combined with JWT auth, can scope your entire API to a tenant context without separate databases or subdomains. Here's the pattern and its tradeoffs.
---

When you're building a multi-tenant SaaS, the first architectural question is usually: how do you keep tenant data isolated? The options range from separate databases per tenant (maximum isolation, maximum cost) to a shared database with row-level filtering (minimum cost, more careful coding required).

But there's an equally important question that gets less attention: **how does your API know which tenant context a request belongs to?**

This post covers a pattern we've used in production: a custom request header for tenant scoping, combined with JWT authentication. Simple to implement, easy to audit, and flexible enough to support multi-tenant access from a single user account.

## The Three Common Approaches

### 1. Subdomain-based (`tenant.yourdomain.com`)

The tenant is encoded in the hostname. Each subdomain routes to the same backend, which extracts the tenant from the `Host` header.

**Good:** Intuitive, visible in the URL.
**Bad:** Requires wildcard TLS certs, more complex DNS setup, awkward in development, doesn't work for mobile API clients the same way.

### 2. URL path-based (`/api/tenants/{tenantId}/...`)

The tenant identifier is part of every route path.

**Good:** RESTful, self-documenting.
**Bad:** Bloats all route definitions, requires every endpoint to include the tenant segment, makes API versioning messier.

### 3. Header-based (`x-tenant-id: <id>`)

A custom header carries the tenant context. Routes stay clean. The tenant scope is resolved in middleware before the handler runs.

**Good:** Routes stay simple, middleware handles scoping uniformly, works well with JWT auth, easy to test.
**Bad:** Less visible (the tenant isn't in the URL), requires clients to always include the header.

We use the header approach.

## The Implementation

The API accepts two forms of auth:

1. A JWT token in the `Authorization` header — identifies *who* is making the request
2. A tenant ID in the `x-tenant-id` header — identifies *on behalf of which tenant*

```http
POST /api/v1/members
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
x-tenant-id: tenant_01GZ8K3X7Y
Content-Type: application/json
```

### Middleware

Auth middleware runs first and validates the JWT. Tenant middleware runs second and validates the `x-tenant-id` against the authenticated user's permitted tenants:

```javascript
// middleware/requireAuth.js
export async function requireAuth(req, res, next) {
  const token = extractBearerToken(req.headers.authorization);
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  try {
    const payload = verifyJwt(token);
    req.user = payload;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

```javascript
// middleware/requireTenantContext.js
export async function requireTenantContext(req, res, next) {
  const tenantId = req.headers['x-tenant-id'];
  if (!tenantId) return res.status(400).json({ error: 'x-tenant-id header required' });

  // Verify the authenticated user has access to this tenant
  const membership = await db.membership.findFirst({
    where: {
      userId: req.user.id,
      tenantId,
      status: 'active',
    },
  });

  if (!membership) return res.status(403).json({ error: 'Access denied' });

  req.tenantId = tenantId;
  req.role = membership.role;
  next();
}
```

Route handlers then have `req.tenantId` and `req.role` available. All database queries in those handlers include `where: { tenantId: req.tenantId }`.

### Route Registration

Tenant-scoped routes apply both middlewares. Public routes (auth endpoints, health checks) apply neither:

```javascript
// Public
router.post('/auth/login', loginHandler);
router.get('/health', healthHandler);

// Tenant-scoped
router.use('/members', requireAuth, requireTenantContext, membersRouter);
router.use('/invoices', requireAuth, requireTenantContext, invoicesRouter);
router.use('/settings', requireAuth, requireTenantContext, settingsRouter);
```

Middleware is applied at the router level, not per-handler. New routes under a tenant-scoped prefix automatically inherit the context without any extra work.

## Multi-Tenant Access From One Account

The header pattern makes something else easy: a single user account accessing multiple tenants.

A super-admin or management tool needs to query across tenants or switch context without re-authenticating. With the header pattern this is trivial — issue one JWT for the user, then pass different `x-tenant-id` values per request:

```javascript
// Management dashboard switching tenant context
async function fetchMembersForTenant(tenantId) {
  return api.get('/members', {
    headers: {
      'Authorization': `Bearer ${userToken}`,
      'x-tenant-id': tenantId,
    }
  });
}
```

With subdomain or path-based approaches, the same scenario requires different base URLs or duplicated route structures.

## Adding Defense-in-Depth: Row-Level Security

The header + middleware pattern handles application-layer tenant isolation. For an additional layer at the database level, PostgreSQL's Row-Level Security can enforce isolation even if a bug in application code omits a `tenantId` filter:

```sql
-- Policy: users can only see rows belonging to their current tenant
ALTER TABLE members ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON members
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

At the start of each request, set the current tenant on the DB connection:

```javascript
await db.$executeRaw`SELECT set_config('app.current_tenant_id', ${req.tenantId}, true)`;
```

Now even a query that forgets `WHERE tenant_id = ?` returns empty results instead of leaking data. The middleware is the first line of defense; RLS is the backstop.

## Tradeoffs to Know Before Adopting

**Clients must always send the header.** This is a discipline requirement. Forgetting the header returns a 400, which is fast to debug, but it's a paper cut during initial integration. Document it clearly and consider a helpful error message: `"x-tenant-id header is required for this endpoint. See docs for details."`

**The header is visible in logs.** Tenant IDs aren't secrets — they're identifiers — but make sure your log sanitization rules treat them consistently with other metadata.

**Switching tenants mid-session is application-layer logic.** The API doesn't know or care about "the current tenant" in session state. The client always tells the API which tenant context to use. This is explicit, which is good, but requires the client to manage that state.

## Summary

The `x-tenant-id` header pattern is not glamorous, but it's effective. Routes stay clean. Middleware handles scoping uniformly. A single JWT works across multiple tenant contexts. And the pattern composes naturally with database-level isolation when you need it.

For most multi-tenant APIs where tenants are organizations (not individual users), this pattern is worth considering before reaching for the subdomain or path-based alternatives.
