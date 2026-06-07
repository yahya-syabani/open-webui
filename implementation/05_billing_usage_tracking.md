# Implementation Plan: Billing & Usage Tracking
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Stripe integration, subscription management, usage-based billing, invoicing

---

## Overview

Open WebUI has zero billing infrastructure. For SaaS, you need a complete billing system: subscription plans, payment collection, usage metering, overage charges, invoicing, and a self-serve upgrade/downgrade flow. This plan integrates Stripe as the payment processor and billing engine, with usage data flowing from LiteLLM and Open WebUI's own usage logs into Stripe Meters for consumption-based pricing.

---

## Current State (Open WebUI Baseline)

- No payment integration
- No subscription concept
- No usage metering
- No invoice generation
- No upgrade/downgrade flow

---

## Target Architecture

```
User Action (chat, upgrade click)
          │
          ▼
  Open WebUI Backend
          │
    ┌─────┴──────────────────────┐
    │                            │
    ▼                            ▼
Stripe API                 Usage Logs DB
(subscriptions,            (PostgreSQL)
 payments, invoices)
    │
    ▼
Stripe Webhooks
    │
    ▼
Open WebUI Webhook Handler
(provision plan, update user, suspend account)
```

---

## Phase 1: Stripe Account & Product Setup

### 1.1 Create Stripe Products & Prices

In your Stripe dashboard (or via API during bootstrap):

```python
# scripts/stripe_setup.py — run once during SaaS setup

import stripe
stripe.api_key = os.environ["STRIPE_SECRET_KEY"]

# --- Products ---
free_product = stripe.Product.create(
    name="Antigravity Free",
    metadata={"plan": "free"}
)

pro_product = stripe.Product.create(
    name="Antigravity Pro",
    metadata={"plan": "pro"}
)

enterprise_product = stripe.Product.create(
    name="Antigravity Enterprise",
    metadata={"plan": "enterprise"}
)

# --- Prices (flat subscription) ---
pro_monthly = stripe.Price.create(
    product=pro_product.id,
    unit_amount=1900,        # $19.00 USD
    currency="usd",
    recurring={"interval": "month"},
    metadata={"plan": "pro", "interval": "monthly"}
)

pro_yearly = stripe.Price.create(
    product=pro_product.id,
    unit_amount=19000,       # $190.00/year (2 months free)
    currency="usd",
    recurring={"interval": "year"},
    metadata={"plan": "pro", "interval": "yearly"}
)

# --- Usage-based metered price (for overages) ---
pro_token_overage = stripe.Price.create(
    product=pro_product.id,
    unit_amount=1,           # $0.00001 per token ($1 per 100k tokens)
    currency="usd",
    recurring={"interval": "month", "usage_type": "metered"},
    billing_scheme="per_unit",
    metadata={"type": "token_overage", "plan": "pro"}
)

print(f"Pro monthly price ID: {pro_monthly.id}")
print(f"Pro overage price ID: {pro_token_overage.id}")
```

### 1.2 Stripe Meter for Token Usage

```python
# Create a Stripe Meter for token tracking
meter = stripe.billing.Meter.create(
    display_name="AI Token Usage",
    event_name="ai_tokens_used",
    default_aggregation={"formula": "sum"},
    value_settings={"event_payload_key": "tokens"},
    customer_mapping={
        "type": "by_id",
        "event_payload_key": "stripe_customer_id"
    }
)
STRIPE_METER_ID = meter.id
```

---

## Phase 2: Database Schema for Billing

```sql
CREATE TABLE subscriptions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id       UUID REFERENCES organizations(id) ON DELETE CASCADE,
  stripe_customer_id    VARCHAR(128) UNIQUE,
  stripe_subscription_id VARCHAR(128) UNIQUE,
  stripe_price_id       VARCHAR(128),
  plan                  VARCHAR(32) DEFAULT 'free',   -- free | pro | enterprise
  status                VARCHAR(32) DEFAULT 'active', -- active | past_due | canceled | trialing
  current_period_start  TIMESTAMP,
  current_period_end    TIMESTAMP,
  trial_end             TIMESTAMP,
  cancel_at_period_end  BOOLEAN DEFAULT FALSE,
  created_at            TIMESTAMP DEFAULT NOW(),
  updated_at            TIMESTAMP DEFAULT NOW()
);

CREATE TABLE invoices (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id       UUID REFERENCES organizations(id),
  stripe_invoice_id     VARCHAR(128) UNIQUE,
  amount_due            INTEGER,     -- in cents
  amount_paid           INTEGER,
  currency              VARCHAR(8),
  status                VARCHAR(32), -- draft | open | paid | void | uncollectible
  invoice_pdf_url       TEXT,
  period_start          TIMESTAMP,
  period_end            TIMESTAMP,
  created_at            TIMESTAMP DEFAULT NOW()
);

CREATE TABLE payment_methods (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id          UUID REFERENCES organizations(id),
  stripe_payment_method_id VARCHAR(128) UNIQUE,
  type                     VARCHAR(32),  -- card | bank_account
  last4                    VARCHAR(4),
  brand                    VARCHAR(32),
  exp_month                INTEGER,
  exp_year                 INTEGER,
  is_default               BOOLEAN DEFAULT FALSE,
  created_at               TIMESTAMP DEFAULT NOW()
);
```

---

## Phase 3: Stripe Integration Service

### 3.1 Stripe Service Layer

Create `services/billing.py`:

```python
import stripe
from open_webui.models.subscriptions import Subscription
from open_webui.config.plans import get_plan_limits

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
STRIPE_WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]
STRIPE_PRO_MONTHLY_PRICE_ID = os.environ["STRIPE_PRO_MONTHLY_PRICE_ID"]
STRIPE_PRO_YEARLY_PRICE_ID  = os.environ["STRIPE_PRO_YEARLY_PRICE_ID"]

async def create_stripe_customer(org_id: str, email: str, name: str) -> str:
    """Create Stripe customer for a new organization. Called during org signup."""
    customer = stripe.Customer.create(
        email=email,
        name=name,
        metadata={"organization_id": org_id}
    )
    await db.execute(
        "INSERT INTO subscriptions (organization_id, stripe_customer_id, plan) VALUES ($1,$2,'free')",
        org_id, customer.id
    )
    return customer.id

async def create_checkout_session(
    org_id:    str,
    price_id:  str,
    success_url: str,
    cancel_url:  str,
) -> str:
    """Create Stripe Checkout session — returns URL to redirect user to."""
    sub = await get_subscription(org_id)
    session = stripe.checkout.Session.create(
        customer=sub.stripe_customer_id,
        mode="subscription",
        line_items=[{"price": price_id, "quantity": 1}],
        success_url=success_url + "?session_id={CHECKOUT_SESSION_ID}",
        cancel_url=cancel_url,
        subscription_data={
            "trial_period_days": 14,         # 14-day free trial
            "metadata": {"organization_id": org_id},
        },
        allow_promotion_codes=True,
        metadata={"organization_id": org_id},
    )
    return session.url

async def create_billing_portal_session(org_id: str, return_url: str) -> str:
    """Stripe Customer Portal — self-serve plan changes, cancel, update card."""
    sub = await get_subscription(org_id)
    session = stripe.billing_portal.Session.create(
        customer=sub.stripe_customer_id,
        return_url=return_url,
    )
    return session.url

async def cancel_subscription(org_id: str, at_period_end: bool = True):
    """Cancel subscription. Defaults to cancel at end of billing period."""
    sub = await get_subscription(org_id)
    stripe.Subscription.modify(
        sub.stripe_subscription_id,
        cancel_at_period_end=at_period_end
    )
    await db.execute(
        "UPDATE subscriptions SET cancel_at_period_end=$1 WHERE organization_id=$2",
        at_period_end, org_id
    )
```

---

## Phase 4: Webhook Handler

Webhooks are how Stripe tells your backend about payment events asynchronously.

### 4.1 Webhook Router

`routers/billing_webhooks.py`:

```python
@router.post("/webhooks/stripe", include_in_schema=False)
async def stripe_webhook(request: Request):
    payload   = await request.body()
    sig_header = request.headers.get("stripe-signature")

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, STRIPE_WEBHOOK_SECRET
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        raise HTTPException(400, "Invalid webhook signature")

    # Idempotency: skip already-processed events
    if await is_event_processed(event.id):
        return {"status": "already_processed"}

    await process_stripe_event(event)
    await mark_event_processed(event.id)
    return {"status": "ok"}
```

### 4.2 Event Handlers

```python
async def process_stripe_event(event: stripe.Event):
    handlers = {
        "checkout.session.completed":         handle_checkout_completed,
        "customer.subscription.created":      handle_subscription_created,
        "customer.subscription.updated":      handle_subscription_updated,
        "customer.subscription.deleted":      handle_subscription_deleted,
        "invoice.payment_succeeded":          handle_payment_succeeded,
        "invoice.payment_failed":             handle_payment_failed,
        "customer.subscription.trial_will_end": handle_trial_ending,
    }
    handler = handlers.get(event.type)
    if handler:
        await handler(event.data.object)
    else:
        logger.info(f"Unhandled Stripe event: {event.type}")

async def handle_subscription_updated(subscription):
    org_id = subscription.metadata.get("organization_id")
    new_plan = get_plan_from_price(subscription.items.data[0].price.id)

    await db.execute("""
        UPDATE subscriptions
        SET plan=$1, status=$2, current_period_start=$3,
            current_period_end=$4, updated_at=NOW()
        WHERE stripe_subscription_id=$5
    """, new_plan, subscription.status,
        datetime.fromtimestamp(subscription.current_period_start),
        datetime.fromtimestamp(subscription.current_period_end),
        subscription.id)

    # Update org plan immediately
    await db.execute("UPDATE organizations SET plan=$1 WHERE id=$2", new_plan, org_id)

    # Invalidate cached org config
    await cache_del(f"org:config:{org_id}")

    # Update LiteLLM virtual key limits
    await update_litellm_key_for_org(org_id, new_plan)

    logger.info(f"Org {org_id} plan updated to {new_plan}")

async def handle_payment_failed(invoice):
    org_id = await get_org_id_by_customer(invoice.customer)
    # Grace period: don't immediately suspend, send warning first
    await send_payment_failed_email(org_id, invoice.amount_due)
    await schedule_suspension(org_id, grace_period_days=3)

async def handle_subscription_deleted(subscription):
    org_id = subscription.metadata.get("organization_id")
    # Downgrade to free, don't delete data
    await db.execute(
        "UPDATE organizations SET plan='free' WHERE id=$1", org_id
    )
    await db.execute(
        "UPDATE subscriptions SET plan='free', status='canceled' WHERE organization_id=$1",
        org_id
    )
    await send_cancellation_email(org_id)
```

---

## Phase 5: Usage-Based Billing (Metered Tokens)

### 5.1 Report Token Usage to Stripe

After every LLM response, report consumed tokens to Stripe Meters:

```python
# services/usage_tracker.py — extend existing log_usage()

async def log_usage(user_id, org_id, model, prompt_tokens, completion_tokens, ...):
    total = prompt_tokens + completion_tokens

    # 1. Write to local DB (for quota checking — see rate limiting plan)
    await write_usage_to_db(...)

    # 2. Report to Stripe Meter (for billing)
    sub = await get_subscription(org_id)
    if sub.plan != "free" and sub.stripe_customer_id:
        await report_tokens_to_stripe(sub.stripe_customer_id, total)

async def report_tokens_to_stripe(stripe_customer_id: str, tokens: int):
    """
    Report token usage to Stripe Meter.
    Stripe aggregates these into the next invoice automatically.
    """
    stripe.billing.MeterEvent.create(
        event_name="ai_tokens_used",
        payload={
            "stripe_customer_id": stripe_customer_id,
            "tokens": str(tokens),
        },
        timestamp=int(time.time()),
    )
```

### 5.2 Overage Billing Logic

Plans include a base token allotment. Usage above that triggers per-token charges automatically via the metered Stripe price attached to the subscription as a second line item.

When creating subscription, attach both flat price + metered price:
```python
stripe.Subscription.create(
    customer=stripe_customer_id,
    items=[
        {"price": PRO_MONTHLY_PRICE_ID},           # $19/mo flat
        {"price": PRO_TOKEN_OVERAGE_PRICE_ID},      # $0.00001/token over limit
    ],
)
```

---

## Phase 6: Billing API Endpoints

### 6.1 Backend Routes

`routers/billing.py`:

```python
@router.get("/api/billing/plans")
async def list_plans():
    """Public endpoint — no auth required."""
    return {
        "plans": [
            {"id": "free",       "name": "Free",       "price_monthly": 0,   "price_yearly": 0},
            {"id": "pro",        "name": "Pro",        "price_monthly": 19,  "price_yearly": 190},
            {"id": "enterprise", "name": "Enterprise", "price_monthly": None, "price_yearly": None},
        ]
    }

@router.get("/api/billing/subscription")
async def get_subscription(ctx = Depends(require_permission("view_billing"))):
    sub = await get_subscription(ctx.org.id)
    return sub

@router.post("/api/billing/checkout")
async def create_checkout(
    plan_id: str,
    interval: str = "monthly",
    ctx = Depends(require_permission("manage_billing"))
):
    price_id = get_price_id(plan_id, interval)
    url = await create_checkout_session(
        org_id=ctx.org.id,
        price_id=price_id,
        success_url=f"{FRONTEND_URL}/billing/success",
        cancel_url=f"{FRONTEND_URL}/billing",
    )
    return {"checkout_url": url}

@router.post("/api/billing/portal")
async def billing_portal(ctx = Depends(require_permission("manage_billing"))):
    url = await create_billing_portal_session(
        org_id=ctx.org.id,
        return_url=f"{FRONTEND_URL}/billing",
    )
    return {"portal_url": url}

@router.get("/api/billing/invoices")
async def list_invoices(ctx = Depends(require_permission("view_billing"))):
    sub = await get_subscription(ctx.org.id)
    invoices = stripe.Invoice.list(customer=sub.stripe_customer_id, limit=24)
    return {"invoices": [format_invoice(inv) for inv in invoices.data]}

@router.get("/api/billing/usage")
async def get_usage_summary(ctx = Depends(require_permission("view_billing"))):
    return await get_org_usage_summary(ctx.org.id)
```

---

## Phase 7: Frontend Billing Pages

### 7.1 Pages to Build

- `/billing` — current plan, usage bar, upgrade CTA
- `/billing/plans` — plan comparison table with pricing
- `/billing/success` — post-checkout success confirmation
- `/billing/invoices` — invoice history with PDF download links

### 7.2 Upgrade Flow (Svelte)

```svelte
<!-- src/routes/billing/+page.svelte -->
<script>
  import { subscription, usageSummary } from '$lib/stores/billing';

  async function handleUpgrade(planId) {
    const res = await fetch('/api/billing/checkout', {
      method: 'POST',
      body: JSON.stringify({ plan_id: planId, interval: 'monthly' })
    });
    const { checkout_url } = await res.json();
    window.location.href = checkout_url;  // redirect to Stripe Checkout
  }

  async function handleManageBilling() {
    const res = await fetch('/api/billing/portal', { method: 'POST' });
    const { portal_url } = await res.json();
    window.location.href = portal_url;  // redirect to Stripe Portal
  }
</script>
```

---

## Phase 8: Trial Management

### 8.1 14-Day Free Trial

All new Pro signups get a 14-day trial (no credit card required for trial start):

```python
# During checkout session creation
subscription_data={
    "trial_period_days": 14,
    "trial_settings": {
        "end_behavior": {"missing_payment_method": "cancel"}
    }
}
```

### 8.2 Trial Expiry Notifications

Stripe sends `customer.subscription.trial_will_end` 3 days before trial ends:
```python
async def handle_trial_ending(subscription):
    org_id = subscription.metadata.get("organization_id")
    days_remaining = 3
    await send_trial_ending_email(org_id, days_remaining)
    # Show in-app banner
    await set_org_flag(org_id, "trial_ending_banner", True)
```

---

## Environment Variables Reference

```bash
# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
STRIPE_PRO_MONTHLY_PRICE_ID=price_...
STRIPE_PRO_YEARLY_PRICE_ID=price_...
STRIPE_PRO_OVERAGE_PRICE_ID=price_...
STRIPE_METER_ID=mtr_...

# App URLs (for Stripe redirects)
FRONTEND_URL=https://yoursaas.com
```

---

## Implementation Checklist

- [ ] Stripe products and prices created (free, pro monthly, pro yearly, overage)
- [ ] Stripe Meter configured for `ai_tokens_used` event
- [ ] `subscriptions`, `invoices`, `payment_methods` tables created
- [ ] `create_stripe_customer` called on org signup
- [ ] Checkout session flow working end-to-end (test with Stripe test cards)
- [ ] Stripe webhook endpoint deployed and signature verification passing
- [ ] `subscription.updated` webhook updates org plan and LiteLLM key
- [ ] `payment_failed` webhook sends email + schedules suspension
- [ ] Token usage reporting to Stripe Meter after every LLM call
- [ ] `/api/billing/subscription` returns correct plan data
- [ ] `/api/billing/portal` redirects to Stripe self-serve portal
- [ ] Invoice list renders with PDF download links
- [ ] 14-day trial working (Stripe test clock used for testing)
- [ ] Trial ending email sent 3 days before expiry
- [ ] Downgrade to free (not deletion) on subscription canceled
