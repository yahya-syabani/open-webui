# Implementation Plan: Multi-User & Account Management
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Multi-tenancy, user isolation, organization accounts, RBAC

---

## Overview

Open WebUI ships with a basic user/admin model backed by SQLite. For SaaS, this must be extended into a full multi-tenant system supporting organizations, workspace isolation, granular roles, SSO, and per-tenant configuration. This plan covers all structural changes to the Open WebUI codebase required to achieve this.

---

## Current State (Open WebUI Baseline)

- Single flat user table with `role` field: `admin | user | pending`
- No organization or workspace concept
- JWT-based session auth
- No SSO out of the box (OIDC available via env config but basic)
- All users share a single instance namespace

---

## Target Architecture

```
Tenant (Organization)
  └── Workspaces (optional sub-groups)
        └── Users
              └── Roles (admin, member, viewer, billing_admin)
                    └── Chats / Models / Files (all scoped to tenant_id)
```

---

## Phase 1: Database Schema Extensions

### 1.1 New Tables

**`organizations`**
```sql
CREATE TABLE organizations (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug         VARCHAR(64) UNIQUE NOT NULL,       -- used in subdomain/URL
  name         VARCHAR(255) NOT NULL,
  plan         VARCHAR(32) DEFAULT 'free',         -- free | pro | enterprise
  created_at   TIMESTAMP DEFAULT NOW(),
  updated_at   TIMESTAMP DEFAULT NOW(),
  is_active    BOOLEAN DEFAULT TRUE,
  metadata     JSONB DEFAULT '{}'                  -- custom config per tenant
);
```

**`organization_members`**
```sql
CREATE TABLE organization_members (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
  role            VARCHAR(32) DEFAULT 'member',    -- owner | admin | member | viewer | billing_admin
  invited_by      UUID REFERENCES users(id),
  joined_at       TIMESTAMP DEFAULT NOW(),
  is_active       BOOLEAN DEFAULT TRUE,
  UNIQUE(organization_id, user_id)
);
```

**`workspaces`** *(optional for enterprise tier)*
```sql
CREATE TABLE workspaces (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  name            VARCHAR(255) NOT NULL,
  description     TEXT,
  created_by      UUID REFERENCES users(id),
  created_at      TIMESTAMP DEFAULT NOW(),
  is_active       BOOLEAN DEFAULT TRUE
);
```

**`invitations`**
```sql
CREATE TABLE invitations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  email           VARCHAR(255) NOT NULL,
  role            VARCHAR(32) DEFAULT 'member',
  token           VARCHAR(128) UNIQUE NOT NULL,
  invited_by      UUID REFERENCES users(id),
  expires_at      TIMESTAMP NOT NULL,
  accepted_at     TIMESTAMP,
  created_at      TIMESTAMP DEFAULT NOW()
);
```

### 1.2 Modify Existing `users` Table

Add columns:
```sql
ALTER TABLE users ADD COLUMN current_organization_id UUID REFERENCES organizations(id);
ALTER TABLE users ADD COLUMN sso_provider VARCHAR(64);      -- google | github | azure | saml
ALTER TABLE users ADD COLUMN sso_subject   VARCHAR(255);    -- external identity subject
ALTER TABLE users ADD COLUMN is_verified   BOOLEAN DEFAULT FALSE;
ALTER TABLE users ADD COLUMN last_login_at TIMESTAMP;
ALTER TABLE users ADD COLUMN profile       JSONB DEFAULT '{}';
```

### 1.3 Tenant-Scope Existing Tables

Add `organization_id` to resource tables:
```sql
ALTER TABLE chats    ADD COLUMN organization_id UUID REFERENCES organizations(id);
ALTER TABLE documents ADD COLUMN organization_id UUID REFERENCES organizations(id);
ALTER TABLE models   ADD COLUMN organization_id UUID REFERENCES organizations(id);
ALTER TABLE prompts  ADD COLUMN organization_id UUID REFERENCES organizations(id);
ALTER TABLE tools    ADD COLUMN organization_id UUID REFERENCES organizations(id);
```

---

## Phase 2: Backend Changes (`backend/open_webui/`)

### 2.1 New Routers

**`routers/organizations.py`**
- `POST /api/organizations` — create org (triggers onboarding flow)
- `GET /api/organizations/{org_id}` — get org details
- `PUT /api/organizations/{org_id}` — update name, metadata
- `DELETE /api/organizations/{org_id}` — soft-delete
- `GET /api/organizations/{org_id}/members` — list members
- `POST /api/organizations/{org_id}/members/invite` — send invitation email
- `DELETE /api/organizations/{org_id}/members/{user_id}` — remove member
- `PUT /api/organizations/{org_id}/members/{user_id}/role` — change role

**`routers/invitations.py`**
- `GET /api/invitations/{token}` — validate token, return org + role info
- `POST /api/invitations/{token}/accept` — complete signup/join

### 2.2 Middleware: Tenant Context Injection

Create `middleware/tenant.py`:
```python
class TenantMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Resolve tenant from:
        # 1. Subdomain: acme.yoursaas.com → org slug "acme"
        # 2. Header: X-Organization-ID
        # 3. JWT claim: organization_id
        org = await resolve_tenant(request)
        request.state.organization_id = org.id if org else None
        request.state.organization = org
        return await call_next(request)
```

Register in `main.py`:
```python
app.add_middleware(TenantMiddleware)
```

### 2.3 Auth Dependency Updates

Update `utils/auth.py` — extend `get_current_user` to also resolve org membership:
```python
async def get_current_user_with_org(
    token: str = Depends(oauth2_scheme),
    org_id: Optional[str] = Header(None, alias="X-Organization-ID")
) -> UserOrgContext:
    user = decode_jwt(token)
    membership = await get_membership(user.id, org_id)
    if not membership:
        raise HTTPException(403, "Not a member of this organization")
    return UserOrgContext(user=user, org=membership.organization, role=membership.role)
```

### 2.4 RBAC Permission System

Create `utils/rbac.py`:
```python
PERMISSIONS = {
    "owner":         {"*"},
    "admin":         {"manage_members", "manage_models", "manage_settings", "use_chat", "view_billing"},
    "member":        {"use_chat", "manage_own_files", "create_prompts"},
    "viewer":        {"use_chat"},
    "billing_admin": {"view_billing", "manage_billing"},
}

def require_permission(permission: str):
    def dependency(ctx: UserOrgContext = Depends(get_current_user_with_org)):
        if permission not in PERMISSIONS.get(ctx.role, set()) and "*" not in PERMISSIONS.get(ctx.role, set()):
            raise HTTPException(403, f"Insufficient permission: {permission}")
        return ctx
    return dependency
```

Usage in routers:
```python
@router.post("/models")
async def create_model(ctx = Depends(require_permission("manage_models"))):
    ...
```

### 2.5 SSO Integration

Extend `routers/auth.py` to support:
- **Google OAuth2** — already partially available
- **GitHub OAuth2**
- **Microsoft Azure AD (OIDC)**
- **Generic SAML 2.0** — for enterprise customers

Install: `python-saml`, `authlib`

Add env config:
```
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
AZURE_TENANT_ID=
AZURE_CLIENT_ID=
AZURE_CLIENT_SECRET=
SAML_IDP_METADATA_URL=
```

---

## Phase 3: Frontend Changes (`src/`)

### 3.1 Organization Onboarding Flow

New pages:
- `/onboarding` — create org, set name, invite teammates
- `/join/{token}` — accept invitation, complete profile
- `/org/settings` — manage org name, members, SSO config
- `/org/members` — member list with role management

### 3.2 Org Switcher Component

Add to topbar/sidebar — allows users who are members of multiple orgs to switch context:
```svelte
<!-- src/lib/components/OrgSwitcher.svelte -->
<OrgSwitcher
  currentOrg={$currentOrganization}
  orgs={$userOrganizations}
  on:switch={handleOrgSwitch}
/>
```

On switch: update `X-Organization-ID` header in all subsequent API calls, refresh chat list.

### 3.3 Store Updates

Extend `src/lib/stores/` with:
- `organization.ts` — current org, orgs list
- `membership.ts` — current user's role and permissions
- `rbac.ts` — client-side permission helpers

---

## Phase 4: Email & Notification System

### 4.1 Transactional Email

Install: `fastapi-mail` or integrate with SendGrid / Resend / Postmark.

Required email templates:
- **Invitation email** — join org link, expires in 7 days
- **Welcome email** — after first login
- **Role changed** — notifies user their org role changed
- **Security alert** — new login from unrecognized device

### 4.2 Email Config
```
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
EMAIL_FROM=noreply@yoursaas.com
```

---

## Phase 5: Admin Super-Panel

A separate admin area (distinct from org-level admin) for the SaaS operator:

- List all organizations, users, plans
- Impersonate user (for support)
- Suspend/reactivate organization
- Force-reset MFA, password
- View audit log across all tenants

Protected by a separate `superadmin` role stored in a config/env, not in the DB.

---

## Phase 6: Audit Logging

Create `models/audit_log.py` and `services/audit.py`:
```sql
CREATE TABLE audit_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID,
  user_id         UUID,
  action          VARCHAR(128) NOT NULL,   -- user.invited, chat.deleted, model.created
  resource_type   VARCHAR(64),
  resource_id     VARCHAR(128),
  metadata        JSONB DEFAULT '{}',
  ip_address      INET,
  created_at      TIMESTAMP DEFAULT NOW()
);
```

Log key events: member invite/join/remove, role changes, model additions, data export, login/logout.

---

## Migration Strategy

### Step-by-Step Rollout

1. **Week 1–2:** Add `organizations` and `organization_members` tables; backfill existing users into a default org
2. **Week 3:** Deploy tenant middleware (non-breaking — falls back to default org if no header)
3. **Week 4:** Release RBAC system, update all existing endpoints to check permissions
4. **Week 5:** Frontend org switcher + settings pages
5. **Week 6:** SSO integrations (Google first, then GitHub, then Azure)
6. **Week 7:** Invitation flow end-to-end
7. **Week 8:** Audit logging + superadmin panel

### Zero-Downtime Migration

- All schema changes additive (new columns nullable, new tables)
- Feature-flag new org features behind `ENABLE_MULTI_ORG=true` env var
- Existing single-org setups continue working without change

---

## Testing Checklist

- [ ] User can register and create an organization
- [ ] Owner can invite member by email; invitation token expires correctly
- [ ] Member cannot access another org's chats
- [ ] Role changes take effect immediately (no cached stale permissions)
- [ ] SSO login creates user if not exists, links if email matches
- [ ] Suspended org blocks all logins for that org's users
- [ ] Audit log captures invite, login, role change events
- [ ] Org switcher updates all API calls correctly

---

## Dependencies

| Package | Purpose |
|---|---|
| `authlib` | OAuth2 / OIDC |
| `python-saml` | SAML 2.0 enterprise SSO |
| `fastapi-mail` | Transactional email |
| `alembic` | DB migrations |
| `pydantic-settings` | Env config management |
