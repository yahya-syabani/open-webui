# Implementation Plan: Security & Compliance
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Data isolation, encryption, authentication, compliance (SOC 2, GDPR)

---

## Overview

A SaaS system handling user data and payment information must implement comprehensive security controls. This plan covers data isolation (preventing one customer's data leaking to another), encryption at rest and in transit, strong authentication, API security, and compliance with SOC 2 and GDPR frameworks.

---

## Current State (Open WebUI Baseline)

- No multi-tenancy → no tenant isolation logic
- SQLite unencrypted at rest
- No secrets management
- Basic JWT auth, no MFA
- No audit logging
- No data deletion/GDPR compliance flow

---

## Target Security Architecture

```
External Attacker
        │
        ▼
   WAF (AWS Shield, Cloudflare)
        │
   ▼ (DDoS mitigated)
   
   TLS 1.3 (in-transit encryption)
        │
        ▼
   API Gateway (request validation)
        │
        ▼
   Authentication (JWT + MFA)
        │
        ▼
   Authorization (RBAC + org_id checks)
        │
        ▼
   Query Layer (tenant_id always filtered)
        │
        ▼
   Data at Rest (AES-256 encrypted)
        │
        ▼ (Encrypted DB, encrypted S3, encrypted Redis)
   
   Audit Logs (immutable, centralized)
```

---

## Phase 1: Data Isolation (Tenant Segmentation)

### 1.1 Query Filter Middleware

Every database query must filter by `organization_id`. Create a base repository class that enforces this:

```python
# backend/open_webui/repositories/base.py

from abc import ABC, abstractmethod
from uuid import UUID
from sqlalchemy import and_

class TenantAwareRepository(ABC):
    """
    Base repository enforcing tenant isolation.
    Every query is scoped to the current organization.
    """

    def __init__(self, db_session):
        self.db = db_session
        self._tenant_id = None

    def set_tenant(self, org_id: UUID):
        """Set the current tenant context."""
        self._tenant_id = org_id

    async def get_by_id(self, model_class, id: UUID) -> object:
        """Get resource, checking it belongs to current tenant."""
        result = await self.db.execute(
            select(model_class)
            .where(
                and_(
                    model_class.id == id,
                    model_class.organization_id == self._tenant_id
                )
            )
        )
        return result.scalar_one_or_none()

    async def list_for_tenant(self, model_class, filters=None):
        """List all resources for current tenant."""
        query = select(model_class).where(
            model_class.organization_id == self._tenant_id
        )
        if filters:
            for col, val in filters.items():
                query = query.where(getattr(model_class, col) == val)
        result = await self.db.execute(query)
        return result.scalars().all()

    async def delete(self, model_class, id: UUID):
        """Delete resource (checks tenant before delete)."""
        obj = await self.get_by_id(model_class, id)
        if not obj:
            raise HTTPException(404, "Resource not found")
        await self.db.delete(obj)
        await self.db.commit()
```

### 1.2 Dependency Injection of Tenant Context

In request handlers, always inject the current organization and use it:

```python
async def get_current_user_with_org(
    token: str = Depends(oauth2_scheme),
    org_header: Optional[str] = Header(None, alias="X-Organization-ID")
) -> UserOrgContext:
    """Parse JWT and validate org membership."""
    user = jwt.decode(token)
    org_id = org_header or user.get("current_org_id")
    
    # Verify user is member of this org
    membership = await db.query(OrganizationMember).filter(
        OrganizationMember.user_id == user.id,
        OrganizationMember.organization_id == org_id
    ).first()
    
    if not membership:
        raise HTTPException(403, "Not a member of this organization")
    
    return UserOrgContext(
        user_id=user.id,
        org_id=org_id,
        role=membership.role,
        tenant_id=org_id  # pass tenant context
    )

@router.get("/api/chats")
async def list_chats(
    ctx: UserOrgContext = Depends(get_current_user_with_org),
    db: AsyncSession = Depends(get_db)
):
    """List all chats for current org."""
    repo = ChatRepository(db)
    repo.set_tenant(ctx.tenant_id)  # Set tenant context
    chats = await repo.list_for_tenant(Chat)
    return chats
```

### 1.3 Row-Level Security (SQL-Level)

For maximum safety, implement PostgreSQL Row Level Security (RLS) policies:

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE chats ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE usage_logs ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see chats in their org
CREATE POLICY chats_org_isolation ON chats
  USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE POLICY chats_org_isolation_insert ON chats
  WITH CHECK (organization_id = current_setting('app.current_org_id')::uuid);

-- Before each query, set the app.current_org_id session variable
-- SQL: SET app.current_org_id TO '123e4567-e89b-12d3-a456-426614174000';
```

In Python:
```python
async def set_tenant_context(db_session, org_id: UUID):
    await db_session.execute(
        text(f"SET app.current_org_id TO '{org_id}'")
    )
```

---

## Phase 2: Encryption

### 2.1 Encryption at Rest

**Database:**
- AWS RDS: Enable `StorageEncrypted` with KMS key
```hcl
resource "aws_db_instance" "postgres" {
  ...
  storage_encrypted = true
  kms_key_id = aws_kms_key.rds.arn
  ...
}
```

**Redis:**
- ElastiCache: Enable `at_rest_encryption_enabled` and `transit_encryption_enabled`
```hcl
resource "aws_elasticache_replication_group" "redis" {
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token = random_password.redis_auth.result
}
```

**S3 (for backups, logos, files):**
```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
  }
}
```

### 2.2 Encryption in Transit

- **TLS 1.3** enforced for all external connections
- **Nginx/ALB** terminates SSL (get cert from AWS Certificate Manager or Let's Encrypt)
- **Internal service-to-service**: mTLS (mutual TLS) for additional security

Enable mTLS in Kubernetes:
```bash
# Using Linkerd service mesh (simplest mTLS setup)
kubectl annotate namespace production linkerd.io/inject=enabled
```

### 2.3 Field-Level Encryption (Sensitive Data)

For SSO tokens, OAuth secrets, payment data — encrypt at application level before storing:

```python
# services/encryption.py
import cryptography.fernet

class FieldEncryption:
    def __init__(self, key: bytes):
        self.cipher = cryptography.fernet.Fernet(key)

    def encrypt(self, plaintext: str) -> str:
        return self.cipher.encrypt(plaintext.encode()).decode()

    def decrypt(self, ciphertext: str) -> str:
        return self.cipher.decrypt(ciphertext.encode()).decode()

# Initialize with key from AWS Secrets Manager
FIELD_ENCRYPTION = FieldEncryption(
    cryptography.fernet.Fernet.generate_key()  # or load from env
)

# Usage in ORM model:
class User(Base):
    id        = Column(UUID, primary_key=True)
    sso_token = Column(String)  # encrypted JSON

    @property
    def sso_token_decrypted(self) -> dict:
        if not self.sso_token:
            return None
        return json.loads(FIELD_ENCRYPTION.decrypt(self.sso_token))

    @sso_token_decrypted.setter
    def sso_token_decrypted(self, value: dict):
        self.sso_token = FIELD_ENCRYPTION.encrypt(json.dumps(value))
```

---

## Phase 3: Authentication & Authorization

### 3.1 Strengthened JWT Implementation

```python
# services/auth.py
import jwt
from datetime import datetime, timedelta

def create_jwt_token(user_id: str, org_id: str, role: str) -> str:
    payload = {
        "sub":              user_id,           # subject (user ID)
        "org":              org_id,            # organization ID
        "role":             role,              # org role
        "iat":              datetime.utcnow(), # issued at
        "exp":              datetime.utcnow() + timedelta(hours=1),  # expires in 1 hour
        "jti":              secrets.token_hex(16),  # JWT ID (for revocation)
    }
    return jwt.encode(
        payload,
        os.environ["JWT_SECRET_KEY"],
        algorithm="HS256",
        headers={"kid": "signing-key-v1"}  # key ID for rotation
    )

async def verify_jwt_token(token: str) -> dict:
    """Verify JWT and check for revocation."""
    try:
        payload = jwt.decode(
            token,
            os.environ["JWT_SECRET_KEY"],
            algorithms=["HS256"]
        )
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")
    
    # Check if token is revoked (e.g., user logged out)
    is_revoked = await redis_client.get(f"revoked_token:{payload['jti']}")
    if is_revoked:
        raise HTTPException(401, "Token has been revoked")
    
    return payload

async def logout(token: str):
    """Revoke JWT by storing jti in Redis."""
    payload = jwt.decode(token, os.environ["JWT_SECRET_KEY"])
    ttl = int(payload["exp"]) - int(time.time())
    await redis_client.setex(f"revoked_token:{payload['jti']}", ttl, "1")
```

### 3.2 Multi-Factor Authentication (MFA)

Require MFA for admin users and as an option for all users:

```python
# Install: pyotp, qrcode

@router.post("/api/auth/mfa/setup")
async def setup_mfa(ctx = Depends(get_current_user)):
    """Generate MFA secret and QR code."""
    secret = pyotp.random_base32()
    qr = qrcode.QRCode()
    qr.add_data(pyotp.totp.TOTP(secret).provisioning_uri(
        name=ctx.user.email,
        issuer_name="Antigravity IDE"
    ))
    qr.make()
    img = qr.make_image()
    
    # Store secret temporarily (not yet confirmed)
    await redis_client.setex(f"mfa_pending:{ctx.user.id}", 600, secret)
    
    return {
        "secret": secret,
        "qr_data_url": qr_to_data_url(img),
        "backup_codes": [generate_backup_code() for _ in range(10)],
    }

@router.post("/api/auth/mfa/confirm")
async def confirm_mfa(code: str, ctx = Depends(get_current_user)):
    """Verify TOTP code and enable MFA."""
    secret = await redis_client.get(f"mfa_pending:{ctx.user.id}")
    if not secret:
        raise HTTPException(400, "No pending MFA setup")
    
    if not pyotp.TOTP(secret).verify(code):
        raise HTTPException(400, "Invalid code")
    
    # Save to DB
    await db.execute(
        "UPDATE users SET mfa_enabled=true, mfa_secret=$1 WHERE id=$2",
        FIELD_ENCRYPTION.encrypt(secret), ctx.user.id
    )
    await redis_client.delete(f"mfa_pending:{ctx.user.id}")
    return {"status": "enabled"}

@router.post("/api/auth/login")
async def login(email: str, password: str, mfa_code: Optional[str] = None):
    user = await find_user_by_email(email)
    if not user or not verify_password(password, user.password_hash):
        raise HTTPException(401, "Invalid credentials")
    
    if user.mfa_enabled:
        if not mfa_code:
            return {"requires_mfa": True, "mfa_token": create_temporary_mfa_token(user.id)}
        
        secret = FIELD_ENCRYPTION.decrypt(user.mfa_secret)
        if not pyotp.TOTP(secret).verify(mfa_code):
            raise HTTPException(401, "Invalid MFA code")
    
    token = create_jwt_token(user.id, user.current_org_id, user.role)
    return {"access_token": token}
```

---

## Phase 4: API Security

### 4.1 API Key Management

For service-to-service auth (e.g., WebHook verification, internal tools):

```python
# models/api_keys.py
class APIKey(Base):
    __tablename__ = "api_keys"
    
    id              = Column(UUID, primary_key=True, default=uuid4)
    organization_id = Column(UUID, ForeignKey("organizations.id"))
    name            = Column(String(128))
    key_hash        = Column(String(256))  # Never store plaintext
    prefix          = Column(String(8))    # sk_live_abc123...
    last_used_at    = Column(DateTime)
    expires_at      = Column(DateTime)
    permissions     = Column(JSONB, default={})
    created_at      = Column(DateTime, default=datetime.utcnow)

def create_api_key(org_id: str, name: str) -> str:
    prefix = "sk_live_" if ENV == "production" else "sk_test_"
    random_suffix = secrets.token_urlsafe(24)
    full_key = f"{prefix}{random_suffix}"
    
    key_hash = hashlib.sha256(full_key.encode()).hexdigest()
    
    db.execute("""
        INSERT INTO api_keys (organization_id, name, key_hash, prefix)
        VALUES ($1, $2, $3, $4)
    """, org_id, name, key_hash, full_key[:16])
    
    return full_key

async def validate_api_key(key: str) -> Optional[APIKey]:
    key_hash = hashlib.sha256(key.encode()).hexdigest()
    result = await db.execute(
        "SELECT * FROM api_keys WHERE key_hash=$1 AND expires_at > NOW()",
        key_hash
    )
    api_key = result.scalar_one_or_none()
    if api_key:
        await db.execute(
            "UPDATE api_keys SET last_used_at=NOW() WHERE id=$1",
            api_key.id
        )
    return api_key
```

### 4.2 Request Validation & Rate Limiting

See rate limiting and abuse detection in the Rate Limiting plan.

Additional: Input validation with Pydantic schemas:
```python
class ChatCompletionRequest(BaseModel):
    messages: List[MessageSchema] = Field(..., max_items=100)
    model: str = Field(..., max_length=128)
    temperature: float = Field(default=0.7, ge=0, le=2)
    max_tokens: int = Field(default=2048, ge=1, le=128_000)
    
    @validator("messages")
    def validate_messages(cls, v):
        total_length = sum(len(m.content) for m in v)
        if total_length > 1_000_000:
            raise ValueError("Total message content exceeds max length")
        return v
```

---

## Phase 5: Audit Logging

### 5.1 Comprehensive Audit Log

Track all sensitive actions:

```sql
CREATE TABLE audit_logs (
  id              UUID PRIMARY KEY,
  organization_id UUID,
  user_id         UUID,
  action          VARCHAR(128),   -- user.login, user.mfa_enabled, payment.charged, data.exported
  resource_type   VARCHAR(64),
  resource_id     VARCHAR(128),
  status          VARCHAR(32),    -- success | failure
  ip_address      INET,
  user_agent      TEXT,
  metadata        JSONB,          -- additional context
  created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_org_date ON audit_logs(organization_id, created_at DESC);
```

```python
async def audit_log(
    org_id: str,
    user_id: str,
    action: str,
    resource_type: str,
    resource_id: str,
    status: str = "success",
    metadata: dict = None,
    request: Request = None,
):
    ip_addr = request.client.host if request else None
    user_agent = request.headers.get("user-agent") if request else None
    
    await db.execute("""
        INSERT INTO audit_logs
        (organization_id, user_id, action, resource_type, resource_id,
         status, ip_address, user_agent, metadata)
        VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9)
    """, org_id, user_id, action, resource_type, resource_id,
        status, ip_addr, user_agent, json.dumps(metadata or {}))
```

Usage:
```python
@router.post("/api/auth/login")
async def login(credentials: LoginRequest, request: Request):
    try:
        user = authenticate(credentials)
        await audit_log(
            org_id=user.org_id,
            user_id=user.id,
            action="user.login",
            resource_type="user",
            resource_id=user.id,
            status="success",
            request=request
        )
        return {"token": create_jwt_token(user.id, user.org_id)}
    except AuthError:
        await audit_log(
            org_id=None,
            user_id=None,
            action="user.login",
            resource_type="user",
            resource_id=credentials.email,
            status="failure",
            metadata={"reason": "invalid_credentials"},
            request=request
        )
        raise HTTPException(401, "Invalid credentials")
```

---

## Phase 6: GDPR Compliance

### 6.1 Data Access & Export

Users can request all their data:

```python
@router.post("/api/user/export-data")
async def request_data_export(ctx = Depends(get_current_user)):
    """Initiate a GDPR data export for current user."""
    job_id = secrets.token_hex(16)
    
    # Queue async job
    await redis.rpush("data_export_jobs", json.dumps({
        "job_id": job_id,
        "user_id": ctx.user.id,
        "org_id": ctx.org.id,
        "created_at": datetime.utcnow().isoformat(),
    }))
    
    await audit_log(
        org_id=ctx.org.id,
        user_id=ctx.user.id,
        action="user.data_export_requested",
        resource_type="user",
        resource_id=ctx.user.id
    )
    
    return {
        "job_id": job_id,
        "status": "pending",
        "message": "Your data export is being prepared. You'll receive a link via email shortly."
    }

# Worker task
async def process_data_export(job_id: str, user_id: str, org_id: str):
    """Collect all user data and create downloadable zip."""
    data = {
        "user": await get_user_json(user_id),
        "chats": await get_user_chats_json(user_id, org_id),
        "usage": await get_user_usage_json(user_id, org_id),
        "invoices": await get_user_invoices_json(user_id, org_id),
        "export_date": datetime.utcnow().isoformat(),
    }
    
    # Create zip file
    with zipfile.ZipFile(f"/tmp/export-{job_id}.zip", "w") as z:
        z.writestr("data.json", json.dumps(data, indent=2))
    
    # Upload to S3
    download_url = await upload_to_s3(f"exports/{job_id}.zip", ...)
    
    # Send email with download link (expires in 7 days)
    await send_email(
        to=user.email,
        subject="Your Antigravity IDE Data Export",
        body=f"Download your data: {download_url}"
    )
```

### 6.2 Right to Be Forgotten (Deletion)

User can request account deletion with all associated data:

```python
@router.post("/api/user/delete")
async def request_account_deletion(password: str, ctx = Depends(get_current_user)):
    """Initiate account deletion (30-day grace period)."""
    if not verify_password(password, ctx.user.password_hash):
        raise HTTPException(401, "Invalid password")
    
    # Set deletion flag
    await db.execute(
        "UPDATE users SET deletion_requested_at=NOW(), deletion_at=NOW()+INTERVAL'30 days' WHERE id=$1",
        ctx.user.id
    )
    
    # Send confirmation email with cancellation link
    await send_email(
        to=ctx.user.email,
        subject="Account Deletion Scheduled",
        body=f"Your account will be permanently deleted in 30 days. Cancel: {cancellation_link}"
    )
    
    await audit_log(
        org_id=ctx.org.id,
        user_id=ctx.user.id,
        action="user.deletion_requested",
        resource_type="user",
        resource_id=ctx.user.id
    )
    
    # Log them out immediately
    await revoke_all_tokens(ctx.user.id)
    
    return {"status": "deletion_scheduled", "final_deletion_date": "..."}

# Scheduled job (runs daily)
async def process_account_deletions():
    """Hard-delete users whose grace period has expired."""
    users_to_delete = await db.execute("""
        SELECT id, organization_id FROM users
        WHERE deletion_at <= NOW()
    """)
    
    for user in users_to_delete:
        # Delete all user data in cascade
        await db.execute("DELETE FROM users WHERE id=$1", user.id)
        
        await audit_log(
            org_id=user.organization_id,
            user_id=user.id,
            action="user.deleted",
            resource_type="user",
            resource_id=user.id
        )
```

---

## Phase 7: Compliance Frameworks

### 7.1 SOC 2 Type II

Implement controls across:
- **CC (Common Criteria)**: Configuration & change management, access control, encryption
- **PI (Principles)**: Availability, processing integrity, confidentiality, privacy

Key evidence:
- Change log (Git commits, feature flags)
- Access control matrix (RBAC roles)
- Encryption audit report (keys, algorithms)
- Incident response plan
- Annual SOC 2 audit

### 7.2 GDPR Article 32 (Security)

Required measures:
- ✅ Encryption in transit (TLS 1.3)
- ✅ Encryption at rest (AES-256, KMS)
- ✅ Access control (org_id isolation)
- ✅ Pseudonymization (no PII in logs)
- ✅ Confidentiality & integrity (audit logs)
- ✅ Availability (backups, RTO/RPO)
- ✅ Regular testing (security audit annually)

### 7.3 Privacy Policy & DPA

- Privacy Policy: publicly available, updated annually
- Data Processing Agreement: signed by customer before signup
- Sub-processor list: AWS, Stripe, SendGrid, etc.

---

## Implementation Checklist

- [ ] Tenant isolation enforced at DB and ORM layer
- [ ] Row-level security (RLS) policies deployed
- [ ] All traffic uses TLS 1.3 (nginx → origin)
- [ ] Database encrypted at rest (AWS KMS)
- [ ] Redis encrypted at rest and in transit
- [ ] S3 bucket encrypted with customer-managed keys
- [ ] JWT tokens include `jti` for revocation
- [ ] MFA available for all users, required for admins
- [ ] API keys hashed and stored securely
- [ ] All DB queries validated with Pydantic schemas
- [ ] Audit log captures: login, MFA, role changes, data export, deletion
- [ ] GDPR data export endpoint working
- [ ] GDPR deletion with 30-day grace period
- [ ] Secrets Manager (AWS) storing all API keys
- [ ] Audit log table encrypted
- [ ] Backup encryption enabled
- [ ] Privacy Policy + DPA template prepared
- [ ] SOC 2 readiness checklist completed
- [ ] Annual security audit scheduled
