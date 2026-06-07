# Implementation Plan: Custom Branding
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** White-labeling, per-tenant theming, custom domains, logo/color management

---

## Overview

Open WebUI ships with a fixed "Open WebUI" identity — name, logo, colors, and favicon hardcoded or lightly configurable via env vars. For SaaS, you need two levels of branding: your own SaaS identity replacing all Open WebUI references, and per-tenant custom branding allowing enterprise customers to show their own logo and colors to their users. This plan covers both layers.

---

## Current State (Open WebUI Baseline)

- App name controlled via `WEBUI_NAME` env var
- Logo replaceable via `WEBUI_LOGO` env var (single global logo)
- No per-organization branding
- No custom domain support
- No theme color customization
- No white-label API responses

---

## Target Architecture

```
Request arrives at yoursaas.com (or acme.yoursaas.com)
          │
          ▼
  Tenant resolution (subdomain or header)
          │
          ▼
  Branding config loaded (Redis cache, DB fallback)
          │
          ├── CSS variables injected (colors, fonts)
          ├── Logo URL served (S3 or CDN)
          ├── App name/favicon set per tenant
          └── Email templates branded per tenant
```

---

## Phase 1: Branding Data Model

### 1.1 Database Table

```sql
CREATE TABLE organization_branding (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id     UUID REFERENCES organizations(id) ON DELETE CASCADE UNIQUE,

  -- Identity
  app_name            VARCHAR(128),                    -- "Acme AI Assistant"
  support_email       VARCHAR(255),                    -- support@acme.com
  help_url            TEXT,                            -- link in Help menu
  privacy_url         TEXT,
  terms_url           TEXT,

  -- Visual
  logo_url            TEXT,                            -- S3 / CDN URL
  logo_dark_url       TEXT,                            -- dark mode variant
  favicon_url         TEXT,
  
  -- Colors (hex)
  primary_color       VARCHAR(7)  DEFAULT '#2563eb',   -- buttons, links
  secondary_color     VARCHAR(7)  DEFAULT '#64748b',
  accent_color        VARCHAR(7)  DEFAULT '#7c3aed',
  background_color    VARCHAR(7)  DEFAULT '#ffffff',
  sidebar_color       VARCHAR(7)  DEFAULT '#f8fafc',
  text_color          VARCHAR(7)  DEFAULT '#0f172a',

  -- Typography
  font_family         VARCHAR(128) DEFAULT 'Inter',    -- Google Fonts name
  font_url            TEXT,                            -- custom font CDN URL

  -- Custom domain
  custom_domain       VARCHAR(253),                    -- chat.acme.com
  custom_domain_verified BOOLEAN DEFAULT FALSE,
  custom_domain_dns_token VARCHAR(128),               -- CNAME verification token

  -- Feature flags (branding-related)
  hide_powered_by     BOOLEAN DEFAULT FALSE,          -- hide "Powered by Antigravity"
  hide_model_names    BOOLEAN DEFAULT FALSE,          -- show "AI Assistant" not "llama3.1"
  custom_welcome_message TEXT,
  custom_placeholder_text VARCHAR(255),

  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP DEFAULT NOW()
);
```

### 1.2 Pydantic Schema

```python
# schemas/branding.py
from pydantic import BaseModel, Field
from typing import Optional
import re

HEX_RE = re.compile(r'^#[0-9A-Fa-f]{6}$')

class BrandingUpdate(BaseModel):
    app_name:               Optional[str]   = Field(None, max_length=128)
    support_email:          Optional[str]   = None
    logo_url:               Optional[str]   = None
    logo_dark_url:          Optional[str]   = None
    favicon_url:            Optional[str]   = None
    primary_color:          Optional[str]   = None
    secondary_color:        Optional[str]   = None
    accent_color:           Optional[str]   = None
    background_color:       Optional[str]   = None
    sidebar_color:          Optional[str]   = None
    text_color:             Optional[str]   = None
    font_family:            Optional[str]   = None
    custom_domain:          Optional[str]   = None
    hide_powered_by:        Optional[bool]  = None
    hide_model_names:       Optional[bool]  = None
    custom_welcome_message: Optional[str]   = None

    @validator("primary_color", "secondary_color", "accent_color",
               "background_color", "sidebar_color", "text_color")
    def validate_hex(cls, v):
        if v and not HEX_RE.match(v):
            raise ValueError("Must be a valid hex color (e.g. #2563eb)")
        return v
```

---

## Phase 2: Branding API

### 2.1 Branding Router

`routers/branding.py`:
```python
@router.get("/api/branding")
async def get_branding(request: Request):
    """
    Public endpoint — called by frontend on every load.
    Resolves branding from subdomain or current org context.
    """
    org_id = getattr(request.state, "organization_id", None)
    if org_id:
        branding = await get_branding_cached(org_id)
    else:
        branding = get_default_branding()
    return branding

@router.put("/api/branding")
async def update_branding(
    data: BrandingUpdate,
    ctx = Depends(require_permission("manage_settings"))
):
    await upsert_branding(ctx.org.id, data)
    await cache_del(f"branding:{ctx.org.id}")
    return {"status": "updated"}

@router.post("/api/branding/logo")
async def upload_logo(
    file: UploadFile = File(...),
    variant: str = "light",             # light | dark | favicon
    ctx = Depends(require_permission("manage_settings"))
):
    """Upload logo image — stores to S3, returns CDN URL."""
    url = await upload_to_s3(file, org_id=ctx.org.id, variant=variant)
    field = {"light": "logo_url", "dark": "logo_dark_url", "favicon": "favicon_url"}[variant]
    await upsert_branding(ctx.org.id, {field: url})
    await cache_del(f"branding:{ctx.org.id}")
    return {"url": url}
```

### 2.2 Default Branding

```python
def get_default_branding() -> dict:
    """Your SaaS's own branding — shown to users not in an org context."""
    return {
        "app_name":        os.environ.get("DEFAULT_APP_NAME", "Antigravity IDE"),
        "logo_url":        os.environ.get("DEFAULT_LOGO_URL", "/static/logo.svg"),
        "primary_color":   os.environ.get("DEFAULT_PRIMARY_COLOR", "#2563eb"),
        "accent_color":    os.environ.get("DEFAULT_ACCENT_COLOR", "#7c3aed"),
        "font_family":     "Inter",
        "hide_powered_by": False,
    }
```

---

## Phase 3: Frontend Branding Injection

### 3.1 CSS Variables System

Replace all hardcoded colors in Open WebUI's CSS with CSS variables that get overridden at runtime.

In `src/app.html` or the root layout — inject branding from the API:
```html
<script>
  // Injected server-side as a <script> tag for instant rendering (no FOUC)
  window.__BRANDING__ = %sveltekit.branding%;
</script>
```

In the root `+layout.svelte`:
```svelte
<script>
  import { onMount } from 'svelte';
  import { branding } from '$lib/stores/branding';

  onMount(async () => {
    const res = await fetch('/api/branding');
    const b = await res.json();
    branding.set(b);
    applyBranding(b);
  });

  function applyBranding(b) {
    const root = document.documentElement;
    if (b.primary_color)    root.style.setProperty('--color-primary',    b.primary_color);
    if (b.secondary_color)  root.style.setProperty('--color-secondary',  b.secondary_color);
    if (b.accent_color)     root.style.setProperty('--color-accent',     b.accent_color);
    if (b.background_color) root.style.setProperty('--color-background', b.background_color);
    if (b.sidebar_color)    root.style.setProperty('--color-sidebar',    b.sidebar_color);
    if (b.text_color)       root.style.setProperty('--color-text',       b.text_color);
    if (b.font_family)      root.style.setProperty('--font-family',      `'${b.font_family}', sans-serif`);

    // Load custom font if not a system/Google font
    if (b.font_url) {
      const link = document.createElement('link');
      link.rel = 'stylesheet';
      link.href = b.font_url;
      document.head.appendChild(link);
    }

    // Update page title and favicon
    document.title = b.app_name || 'Antigravity IDE';
    if (b.favicon_url) {
      document.querySelector("link[rel='icon']")?.setAttribute('href', b.favicon_url);
    }
  }
</script>
```

### 3.2 Replace Hardcoded Colors

Update `src/app.css` — audit and replace every `#2563eb`, `blue-600`, etc.:
```css
:root {
  --color-primary:    #2563eb;
  --color-secondary:  #64748b;
  --color-accent:     #7c3aed;
  --color-background: #ffffff;
  --color-sidebar:    #f8fafc;
  --color-text:       #0f172a;
  --font-family:      'Inter', sans-serif;
}

/* Replace all instances like: */
.btn-primary { background-color: var(--color-primary); }
a           { color: var(--color-primary); }
.sidebar    { background-color: var(--color-sidebar); }
```

### 3.3 Logo Component

Create `src/lib/components/BrandLogo.svelte`:
```svelte
<script>
  import { branding } from '$lib/stores/branding';
  $: isDark = document.documentElement.classList.contains('dark');
  $: logoUrl = isDark && $branding.logo_dark_url
    ? $branding.logo_dark_url
    : $branding.logo_url ?? '/static/default-logo.svg';
</script>

<img
  src={logoUrl}
  alt={$branding.app_name ?? 'Antigravity IDE'}
  class="h-8 w-auto"
/>
```

### 3.4 "Powered By" Footer

Conditionally show/hide attribution:
```svelte
{#if !$branding.hide_powered_by}
  <footer class="text-xs text-muted">
    Powered by <a href="https://antigravity.dev">Antigravity IDE</a>
  </footer>
{/if}
```

---

## Phase 4: Custom Domains

### 4.1 Domain Verification Flow

Enterprise customers can serve the chat interface from their own domain (e.g. `chat.acme.com`).

**Step 1 — Customer submits domain:**
```python
@router.post("/api/branding/custom-domain")
async def set_custom_domain(domain: str, ctx = Depends(require_permission("manage_settings"))):
    # Validate domain format
    if not is_valid_domain(domain):
        raise HTTPException(400, "Invalid domain format")

    # Generate DNS verification token
    token = secrets.token_hex(16)
    await upsert_branding(ctx.org.id, {
        "custom_domain": domain,
        "custom_domain_dns_token": token,
        "custom_domain_verified": False,
    })
    return {
        "domain": domain,
        "verification": {
            "type":  "CNAME",
            "name":  f"_verify.{domain}",
            "value": f"{token}.verify.yoursaas.com",
            "instructions": f"Add a CNAME record pointing _verify.{domain} to {token}.verify.yoursaas.com"
        }
    }
```

**Step 2 — Verify DNS:**
```python
@router.post("/api/branding/custom-domain/verify")
async def verify_custom_domain(ctx = Depends(require_permission("manage_settings"))):
    branding = await get_branding(ctx.org.id)
    verified = await check_cname_record(
        hostname=f"_verify.{branding.custom_domain}",
        expected=f"{branding.custom_domain_dns_token}.verify.yoursaas.com"
    )
    if not verified:
        raise HTTPException(400, "DNS record not found or not propagated yet")

    await upsert_branding(ctx.org.id, {"custom_domain_verified": True})
    await provision_ssl_certificate(branding.custom_domain)   # Let's Encrypt via Certbot/Caddy
    return {"status": "verified", "domain": branding.custom_domain}
```

### 4.2 Caddy for Automatic SSL + Custom Domain Routing

Use Caddy (or Traefik) to automatically provision SSL for custom domains:

```
# Caddyfile
{
  auto_https on
}

# Wildcard subdomain for all SaaS tenants
*.yoursaas.com {
  reverse_proxy openwebui:8080 {
    header_up X-Tenant-Subdomain {labels.2}
  }
}

# Custom domain routing — loaded dynamically from DB
import /etc/caddy/custom-domains/*.conf
```

Generate per-tenant Caddy config files when a custom domain is verified:
```python
async def provision_ssl_certificate(domain: str):
    config = f"""
{domain} {{
  reverse_proxy openwebui:8080 {{
    header_up X-Custom-Domain {domain}
  }}
}}
"""
    with open(f"/etc/caddy/custom-domains/{domain}.conf", "w") as f:
        f.write(config)
    # Signal Caddy to reload
    subprocess.run(["caddy", "reload", "--config", "/etc/caddy/Caddyfile"])
```

---

## Phase 5: Branding in Emails

### 5.1 Email Template System

Create a base email template that accepts branding variables:

```html
<!-- templates/email/base.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    .btn    { background-color: {{ primary_color }}; color: white; }
    .header { background-color: {{ sidebar_color }}; }
  </style>
</head>
<body>
  <div class="header">
    <img src="{{ logo_url }}" alt="{{ app_name }}" height="40">
  </div>
  <div class="content">
    {% block content %}{% endblock %}
  </div>
  <div class="footer">
    {% if not hide_powered_by %}
    <p>Powered by <a href="https://antigravity.dev">Antigravity IDE</a></p>
    {% endif %}
    <p>© {{ app_name }} · <a href="{{ privacy_url }}">Privacy</a> · <a href="{{ terms_url }}">Terms</a></p>
  </div>
</body>
</html>
```

When sending email for a user in an org, load their org's branding and inject into the template:
```python
async def send_email_branded(org_id: str, to: str, subject: str, template: str, context: dict):
    branding = await get_branding_cached(org_id)
    ctx = {**branding, **context}
    html = render_template(f"email/{template}.html", ctx)
    await send_email(to=to, subject=subject, html=html, from_name=branding["app_name"])
```

---

## Phase 6: Branding Settings UI

### 6.1 Branding Editor Page

New page at `/org/settings/branding`:

Sections:
- **Identity** — App name, support email, help/privacy/terms URLs
- **Logo Upload** — drag-and-drop for light logo, dark logo, favicon (preview shown inline)
- **Colors** — color picker for each CSS variable with live preview
- **Typography** — font family selector (Google Fonts browse or custom URL)
- **Custom Domain** — domain input, DNS instructions, verification status badge
- **Advanced** — hide powered-by toggle, hide model names, custom welcome message

### 6.2 Live Preview Panel

The branding editor shows a side-by-side preview of the chat UI with the current settings applied — so admins see changes instantly before saving:

```svelte
<!-- src/routes/org/settings/branding/+page.svelte -->
<div class="branding-editor">
  <div class="controls">
    <!-- color pickers, logo upload, etc. -->
  </div>
  <div class="preview">
    <iframe
      src="/preview/chat"
      style="--color-primary: {draftBranding.primary_color};"
      title="Branding Preview"
    ></iframe>
  </div>
</div>
```

---

## Phase 7: S3 Asset Storage

### 7.1 Logo Upload Service

```python
# services/storage.py
import aioboto3

S3_BUCKET = os.environ["S3_BUCKET_NAME"]
CDN_URL    = os.environ["CDN_BASE_URL"]   # e.g. https://assets.yoursaas.com

async def upload_to_s3(file: UploadFile, org_id: str, variant: str) -> str:
    ext       = file.filename.rsplit(".", 1)[-1].lower()
    key       = f"branding/{org_id}/{variant}.{ext}"
    session   = aioboto3.Session()

    async with session.client("s3") as s3:
        await s3.upload_fileobj(
            file.file,
            S3_BUCKET,
            key,
            ExtraArgs={
                "ContentType": file.content_type,
                "CacheControl": "public, max-age=31536000",  # 1-year cache
                "ACL": "public-read",
            }
        )
    return f"{CDN_URL}/{key}"
```

### 7.2 Image Validation

```python
async def validate_logo(file: UploadFile):
    if file.content_type not in {"image/png", "image/svg+xml", "image/jpeg", "image/webp"}:
        raise HTTPException(400, "Logo must be PNG, SVG, JPEG, or WebP")
    content = await file.read()
    if len(content) > 2 * 1024 * 1024:   # 2MB limit
        raise HTTPException(400, "Logo must be under 2MB")
    await file.seek(0)
```

---

## Phase 8: Plan-Gating Branding Features

Not all plans get all branding features:

```python
BRANDING_FEATURES_BY_PLAN = {
    "free": {
        "custom_logo":       False,
        "custom_colors":     False,
        "hide_powered_by":   False,
        "custom_domain":     False,
        "email_branding":    False,
    },
    "pro": {
        "custom_logo":       True,
        "custom_colors":     True,
        "hide_powered_by":   False,    # Pro shows "Powered by" — only Enterprise can hide
        "custom_domain":     False,
        "email_branding":    True,
    },
    "enterprise": {
        "custom_logo":       True,
        "custom_colors":     True,
        "hide_powered_by":   True,
        "custom_domain":     True,
        "email_branding":    True,
    },
}
```

---

## Environment Variables Reference

```bash
# SaaS default branding
DEFAULT_APP_NAME=Antigravity IDE
DEFAULT_LOGO_URL=https://assets.yoursaas.com/logo.svg
DEFAULT_PRIMARY_COLOR=#2563eb
DEFAULT_ACCENT_COLOR=#7c3aed

# Asset storage
S3_BUCKET_NAME=antigravity-assets
S3_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
CDN_BASE_URL=https://assets.yoursaas.com

# Custom domain infrastructure
CADDY_CONFIG_DIR=/etc/caddy/custom-domains
DOMAIN_VERIFY_SUFFIX=verify.yoursaas.com
```

---

## Implementation Checklist

- [ ] `organization_branding` table created and migrated
- [ ] `GET /api/branding` resolves branding from org context
- [ ] CSS variables system in place — no hardcoded colors remain in components
- [ ] Logo upload working (S3 storage, CDN URL returned)
- [ ] Favicon and page title update dynamically per org
- [ ] Color picker UI in branding settings page
- [ ] Live preview iframe in branding editor
- [ ] Custom domain flow: submit → DNS instructions → CNAME verify → SSL provision
- [ ] Caddy/Traefik routes custom domains to correct app instance with org header
- [ ] Email templates use org branding variables
- [ ] "Powered by" hidden only for Enterprise plan
- [ ] `hide_model_names` flag renames models to "AI Assistant" in UI
- [ ] Branding cached in Redis with 10-minute TTL
- [ ] Branding cache invalidated on every settings update
- [ ] All branding features gate-checked against plan tier
