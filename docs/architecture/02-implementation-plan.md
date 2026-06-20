# Workspace Architecture — Implementation Plan

> **For: Antigravity IDE / AI Coding Agents**
> **Reference: `01-architecture-decisions.md` — read before implementing**
> **Prerequisites: `00-comprehensive-documentation.md` for codebase understanding**

---

## How to Use This Plan

1. Read the full plan once before starting
2. Execute phases in order — each phase depends on the previous
3. After each phase: verify with the listed check command
4. If a verification fails: fix before proceeding
5. Build frontend after every file change: `NODE_OPTIONS=--max-old-space-size=8192 npx vite build --logLevel error`
6. Never batch changes — one file at a time, verify, then next

---

## Phase 1: Database Migrations

### 1.1: Create workspace tables migration

**File:** `backend/open_webui/migrations/versions/xxxx_add_workspace_tables.py`

Create a new Alembic migration that adds three tables:

```python
# Table: workspace
#   id (Text, PK, UUID4)
#   organization_id (Text, NOT NULL)
#   name (Text, NOT NULL)
#   description (Text, nullable)
#   credit_quota (BigInteger, nullable)
#   credit_consumed (BigInteger, default=0)
#   quota_period_start (DateTime, NOT NULL, default=utcnow)
#   status (Text, default='active')  -- active/archived/deleted
#   deleted_at (DateTime, nullable)
#   created_at (DateTime, NOT NULL, default=utcnow)
#   updated_at (DateTime, NOT NULL, default=utcnow)

# Table: workspace_member
#   id (Text, PK, UUID4)
#   workspace_id (Text, FK → workspace.id)
#   user_id (Text, NOT NULL)
#   role (Text, NOT NULL)  -- owner/manager/member
#   created_at (DateTime, NOT NULL, default=utcnow)

# Table: workspace_invitation
#   id (Text, PK, UUID4)
#   workspace_id (Text, FK → workspace.id)
#   inviter_id (Text, NOT NULL)
#   invitee_id (Text, NOT NULL)
#   role (Text, NOT NULL)  -- member/manager
#   status (Text, default='pending')  -- pending/accepted/declined/expired
#   expires_at (DateTime, NOT NULL, default=utcnow+7days)
#   accepted_at (DateTime, nullable)
#   created_at (DateTime, NOT NULL, default=utcnow)

# Indexes:
#   idx_workspace_org on workspace(organization_id)
#   idx_workspace_member_ws on workspace_member(workspace_id)
#   idx_workspace_member_user on workspace_member(user_id)
#   idx_workspace_invitation_ws on workspace_invitation(workspace_id, status)
#   idx_workspace_invitation_user on workspace_invitation(invitee_id, status)
```

**Verification:**
```bash
cd backend && ../venv/bin/alembic upgrade head && ../venv/bin/alembic current
# Should show the new revision as head
```

### 1.2: Add workspace_id to existing tables

**File:** `backend/open_webui/migrations/versions/xxxx_add_workspace_id_columns.py`

Add `workspace_id` column to these tables (all NOT NULL):

```python
# Tables to ALTER:
#   chat, chat_message, model, knowledge, knowledge_file, knowledge_directory,
#   prompt, memory, file, folder

# For each table:
#   op.add_column(table, sa.Column('workspace_id', sa.Text(), nullable=False))
#   op.create_index(f'idx_{table}_workspace', table, ['organization_id', 'workspace_id'])

# Note: Since this is a fresh/vanilla clone with no data, NOT NULL is safe.
# If any rows exist, they must be backfilled first.
```

**Verification:**
```bash
cd backend && ../venv/bin/alembic upgrade head
cd backend && ../venv/bin/python -c "
from open_webui.internal.db import engine
from sqlalchemy import inspect
inspector = inspect(engine.sync_engine)
for table in ['chat', 'model', 'knowledge', 'prompt', 'memory', 'file', 'folder']:
    cols = [c['name'] for c in inspector.get_columns(table)]
    assert 'workspace_id' in cols, f'MISSING workspace_id in {table}'
print('All workspace_id columns present')
"
```

---

## Phase 2: Backend Models

### 2.1: Create workspace model

**File:** `backend/open_webui/models/workspaces.py` (NEW)

Methods needed:
- `insert_new_workspace(org_id, name, created_by)` → auto-creates default workspace on org creation
- `get_workspace_by_id(ws_id)`
- `get_workspaces_by_org_id(org_id)` → list all workspaces in org
- `get_user_workspaces(user_id, org_id)` → workspaces user is member of
- `update_workspace_by_id(ws_id, data)`
- `archive_workspace_by_id(ws_id)`
- `delete_workspace_by_id(ws_id)` → soft-delete (status='deleted', deleted_at=now)

**Verification:** Python import succeeds + unit test

### 2.2: Create workspace_member model

**File:** `backend/open_webui/models/workspace_members.py` (NEW)

Methods needed:
- `add_member(ws_id, user_id, role='member')`
- `remove_member(ws_id, user_id)`
- `get_member(ws_id, user_id)` → returns membership + role
- `get_members_by_workspace(ws_id)` → list all members
- `update_role(ws_id, user_id, new_role)`

**Verification:** Python import succeeds

### 2.3: Create workspace_invitation model

**File:** `backend/open_webui/models/workspace_invitations.py` (NEW)

Methods needed:
- `create_invitation(ws_id, inviter_id, invitee_id, role)`
- `get_invitation_by_id(inv_id)`
- `get_pending_invitations_for_user(user_id)`
- `get_invitations_by_workspace(ws_id, status_filter=None)`
- `accept_invitation(inv_id, user_id)` → atomic: status='accepted' + create member
- `decline_invitation(inv_id, user_id)`
- `cancel_invitation(inv_id)`
- `expire_stale_invitations()` → batch: pending → expired if expires_at < now

**Verification:** Python import succeeds

### 2.4: Add workspace_id to existing model methods

For each of these model files, add `workspace_id: str | None = None` parameter to ALL read methods (get_*, list_*, search_*) and delete methods:

- `models/chats.py` — `get_chat_list_by_user_id()`, `get_chat_by_id()`, etc.
- `models/chat_messages.py` — all aggregation/query methods
- `models/models.py` — `get_models()`, `get_model_by_id()`
- `models/knowledge.py` — `get_knowledge_bases()`, query methods
- `models/prompts.py` — `get_prompts()`, search methods
- `models/memories.py` — `get_memories()`, search methods
- `models/files.py` — `get_files()`, `get_file_by_id()`
- `models/folders.py` — `get_folders()`, tree methods

Pattern:
```python
async def get_xxx_by_id(self, id, organization_id=None, workspace_id=None, db=None):
    async with get_async_db_context(db) as session:
        stmt = select(Xxx).filter_by(id=id)
        stmt = apply_tenant_filter(stmt, Xxx, organization_id)
        stmt = apply_workspace_filter(stmt, Xxx, workspace_id)  # NEW
        result = await session.execute(stmt)
        return result.scalars().first()
```

**Verification:**
```bash
grep -rn "apply_workspace_filter" backend/open_webui/models/ --include="*.py" -c
# Expect: 20+ calls across 8 files
```

---

## Phase 3: Backend Middleware & Utilities

### 3.1: Add apply_workspace_filter utility

**File:** `backend/open_webui/utils/tenant_db.py`

```python
def apply_workspace_filter(stmt, model, workspace_id):
    """Filter query by workspace_id. No-op when workspace_id is None."""
    if workspace_id is not None and hasattr(model, 'workspace_id'):
        return stmt.filter(model.workspace_id == workspace_id)
    return stmt
```

**Verification:**
```bash
python -c "from open_webui.utils.tenant_db import apply_workspace_filter; print('OK')"
```

### 3.2: Add workspace header middleware

**File:** `backend/open_webui/utils/asgi_middleware.py`

Add `WorkspaceHeaderMiddleware` class (pure ASGI) that extracts `X-Workspace-ID` header:
```python
class WorkspaceHeaderMiddleware:
    async def __call__(self, scope, receive, send):
        if scope['type'] == 'http':
            headers = dict(scope.get('headers', []))
            ws_id = headers.get(b'x-workspace-id', b'').decode()
            # Store raw value — validation happens in dependency
            scope['state']['_raw_workspace_id'] = ws_id if ws_id else None
        await self.app(scope, receive, send)
```

**Register in main.py** — add to middleware chain (after AuthTokenMiddleware, before routers).

**Verification:** Server starts without error. Check request.state has workspace_id for test requests.

### 3.3: Add workspace validation dependency

**File:** `backend/open_webui/utils/auth.py` (or new `utils/workspace.py`)

```python
async def get_workspace_context(request: Request, user = Depends(get_verified_user)):
    ws_id = getattr(request.state, '_raw_workspace_id', None)
    if not ws_id:
        return None
    
    from open_webui.models.workspace_members import WorkspaceMembers
    membership = await WorkspaceMembers.get_member(ws_id, user.id)
    if not membership:
        raise HTTPException(403, "Not a member of this workspace")
    
    request.state.workspace_id = ws_id
    request.state.workspace_role = membership.role
    return ws_id

async def get_optional_workspace_context(request: Request):
    """Does NOT require membership — used for org-scoped endpoints."""
    return getattr(request.state, '_raw_workspace_id', None)
```

**Verification:** Dependency resolves in router tests.

### 3.4: Add context isolation safety net

**File:** `backend/open_webui/utils/middleware.py`

Add `verify_context_isolation(messages, current_workspace_id)` function:
```python
async def verify_context_isolation(messages, current_workspace_id):
    """Scan assembled context for cross-workspace source leakage."""
    for msg in messages:
        for source in msg.get('sources', []):
            source_ws = source.get('workspace_id')
            if source_ws and source_ws != current_workspace_id:
                log.warning(f'CROSS_WORKSPACE_LEAK: source {source["id"]} '
                           f'from ws {source_ws} leaked into ws {current_workspace_id}')
                # In production: strip the source
                # In dev: raise error
                if ENV == 'dev':
                    raise ContextIsolationError(...)
```

Call this in `process_chat_payload()` after context assembly, before model dispatch.

**Verification:** Test with deliberately misconfigured source → expect error in dev.

---

## Phase 4: Backend API Endpoints

### 4.1: Workspace invitation endpoints

**File:** `backend/open_webui/routers/workspaces.py` (NEW)

7 endpoints:
```
POST   /api/v1/workspaces/{ws_id}/invitations
GET    /api/v1/workspaces/{ws_id}/invitations
DELETE /api/v1/workspaces/{ws_id}/invitations/{inv_id}
GET    /api/v1/invitations/me
POST   /api/v1/invitations/{inv_id}/accept
POST   /api/v1/invitations/{inv_id}/decline
```

Auth: `get_verified_user`. Permission checks for workspace management endpoints.

**Register in main.py** with prefix `/api/v1`.

**Verification:** curl each endpoint with valid/invalid auth + memberships.

### 4.2: Workspace CRUD endpoints

Add to workspaces.py:
```
GET    /api/v1/workspaces          ← list user's workspaces in current org
POST   /api/v1/workspaces          ← create workspace (org admin or permission)
GET    /api/v1/workspaces/{ws_id}  ← get workspace details
PUT    /api/v1/workspaces/{ws_id}  ← update (name, description, quota)
DELETE /api/v1/workspaces/{ws_id}  ← archive or soft-delete

GET    /api/v1/workspaces/{ws_id}/members
POST   /api/v1/workspaces/{ws_id}/members  ← add member directly (bypass invitation)
DELETE /api/v1/workspaces/{ws_id}/members/{user_id}
PUT    /api/v1/workspaces/{ws_id}/members/{user_id}/role
```

**Verification:** Full CRUD cycle via curl.

### 4.3: Bootstrap default workspace

In `routers/orgs.py` or wherever org creation happens:
```python
async def create_organization(name, owner_user_id):
    org = await Organizations.insert(name=name)
    # Auto-create default workspace
    default_ws = await Workspaces.insert_new_workspace(
        org_id=org.id, name="Default", created_by=owner_user_id
    )
    # Owner becomes workspace owner
    await WorkspaceMembers.add_member(
        ws_id=default_ws.id, user_id=owner_user_id, role='owner'
    )
    return org, default_ws
```

**Verification:** Create org → verify "Default" workspace exists → verify owner is workspace owner.

### 4.4: Update existing routers for workspace scoping

For each workspace-scoped router, add workspace context dependency and pass `workspace_id` to model methods:

- `routers/chats.py` — `Depends(get_workspace_context)`
- `routers/models.py` — `Depends(get_workspace_context)`
- `routers/knowledge.py` — `Depends(get_workspace_context)`
- `routers/prompts.py` — `Depends(get_workspace_context)`
- `routers/memories.py` — `Depends(get_workspace_context)`
- `routers/files.py` — `Depends(get_workspace_context)`
- `routers/folders.py` — `Depends(get_workspace_context)`

**Verification:** Each router endpoint tested with/without X-Workspace-ID header.

### 4.5: Credit deduction integration

Add to chat completion pipeline (`main.py` or `utils/middleware.py`):

```python
# Pre-call (in process_chat_payload):
estimated_cost = estimate_chat_cost(form_data, model)
workspace = await Workspaces.get_workspace_by_id(ws_id)
if workspace.credit_quota:
    await ensure_quota_period(workspace)
    if workspace.credit_consumed + estimated_cost > workspace.credit_quota:
        raise HTTPException(402, "Workspace quota exceeded")

org = await Organizations.get_organization(org_id)
if org.credit_balance < estimated_cost:
    raise HTTPException(402, "Insufficient org credits")

# Post-call (in process_chat_response):
actual_cost = calculate_actual_cost(response.usage, model)
await deduct_credits(org_id, ws_id, user_id, actual_cost, model_id, tokens_in, tokens_out)
```

**Verification:** Test with workspace at quota limit → expect 402. Test with sufficient quota → expect deduction.

---

## Phase 5: Frontend Stores

### 5.1: Extend tenant.ts with workspace stores

**File:** `src/lib/stores/tenant.ts`

Add:
```typescript
// Workspace stores
export const currentWorkspace = writable<Workspace | null>(null);
export const userWorkspaces = writable<Workspace[]>([]);
export const workspaceRole = writable<'owner' | 'manager' | 'member' | null>(null);
export const workspaceInvitations = writable<Invitation[]>([]);
export const workspaceQuota = writable<{quota: number, consumed: number} | null>(null);

// Workspace switch event
export const workspaceSwitchEvent = writable<number>(0);

// Derived
export const currentWorkspaceId = derived(currentWorkspace, ($ws) => $ws?.id ?? null);

// Actions
export async function switchWorkspace(ws: Workspace | null) {
    const oldWsId = get(currentWorkspace)?.id;
    
    // Leave old Socket.IO room
    if (oldWsId && socket) {
        socket.emit('leave-workspace', { workspace_id: oldWsId });
    }
    
    currentWorkspace.set(ws);
    workspaceRole.set(null);
    
    if (ws) {
        localStorage.setItem('current_ws_id', ws.id);
        workspaceSwitchEvent.update(n => n + 1);
        
        // Join new Socket.IO room
        if (socket) {
            socket.emit('join-workspace', { workspace_id: ws.id });
        }
    } else {
        localStorage.removeItem('current_ws_id');
    }
}

export function workspaceHeaders(): Record<string, string> {
    const ws = get(currentWorkspace);
    return ws?.id ? { 'X-Workspace-ID': ws.id } : {};
}

export function getActiveWorkspaceId(): string | null {
    return get(currentWorkspace)?.id ?? null;
}
```

**Update exports in `stores/index.ts`:**
```typescript
export {
    currentWorkspace, userWorkspaces, workspaceRole, workspaceInvitations,
    workspaceQuota, workspaceSwitchEvent, currentWorkspaceId,
    switchWorkspace, workspaceHeaders, getActiveWorkspaceId
} from './tenant';
```

**Verification:**
```bash
npx tsc --noEmit
# Should compile without errors
```

### 5.2: Add workspace switch subscriber

**File:** `src/lib/stores/index.ts`

Add subscriber (near the existing `orgSwitchEvent` subscriber):
```typescript
workspaceSwitchEvent.subscribe(() => {
    // Reset workspace-scoped stores
    chats.set(null);
    pinnedChats.set([]);
    chatId.set('');
    models.set([]);
    knowledge.set(null);
    prompts.set(null);
    memories.set(null);
    files.set(null);
    folders.set(null);
});
```

**Verification:**
```bash
grep -n "workspaceSwitchEvent" src/lib/stores/index.ts
# Should show subscriber
```

### 5.3: Update initTenant for workspace initialization

**File:** `src/lib/stores/tenant.ts`

Extend `initTenant()`:
```typescript
export async function initTenant(token: string, userId?: string) {
    // ... existing org initialization ...
    
    // Load workspaces for current org
    const org = get(currentOrg);
    if (org) {
        const wss = await listWorkspaces(token, org.id);
        userWorkspaces.set(wss);
        
        // Restore last active workspace
        const savedWsId = localStorage.getItem('current_ws_id');
        const ws = wss.find(w => w.id === savedWsId) || wss[0] || null;
        currentWorkspace.set(ws);  // Direct set — no switchEvent
        
        if (ws && userId) {
            // Resolve workspace role
            const members = await listWorkspaceMembers(token, ws.id);
            const membership = members.find(m => m.user_id === userId);
            workspaceRole.set(membership?.role ?? 'member');
        }
    }
    
    // Load pending invitations
    const invites = await getMyInvitations(token);
    workspaceInvitations.set(invites.filter(i => i.status === 'pending'));
}
```

**IMPORTANT**: `initTenant()` sets `currentWorkspace` DIRECTLY — does NOT fire `workspaceSwitchEvent`. The event fires only on explicit user-triggered workspace switches.

**Verification:** App loads with correct workspace. No chat list wipe on initial load.

---

## Phase 6: Frontend API Client

### 6.1: Add workspaceHeaders to all API modules

For every API module in `src/lib/apis/*/index.ts`:

1. Add import:
```typescript
import { workspaceHeaders, getActiveWorkspaceId } from '$lib/stores/tenant';
```

2. Add to fetch headers:
```typescript
headers: {
    ...orgHeaders(),
    ...workspaceHeaders(),  // NEW
    'Content-Type': 'application/json',
    ...(token && { authorization: `Bearer ${token}` })
}
```

**Modules to update:** ALL 24+ API modules (simpler to add everywhere than remember which need it).

**Verification:**
```bash
for f in src/lib/apis/*/index.ts; do
  if ! grep -q 'workspaceHeaders' "$f"; then
    echo "MISSING: $f"
  fi
done
# Should have zero output
```

### 6.2: Create workspace API module

**File:** `src/lib/apis/workspaces/index.ts` (NEW)

```typescript
export async function listWorkspaces(token: string, orgId: string): Promise<Workspace[]> { ... }
export async function createWorkspace(token: string, orgId: string, data): Promise<Workspace> { ... }
export async function getWorkspace(token: string, wsId: string): Promise<Workspace> { ... }
export async function updateWorkspace(token: string, wsId: string, data): Promise<Workspace> { ... }
export async function deleteWorkspace(token: string, wsId: string): Promise<void> { ... }

export async function listWorkspaceMembers(token: string, wsId: string): Promise<Member[]> { ... }
export async function addWorkspaceMember(token: string, wsId: string, userId: string, role: string): Promise<void> { ... }
export async function removeWorkspaceMember(token: string, wsId: string, userId: string): Promise<void> { ... }
export async function updateMemberRole(token: string, wsId: string, userId: string, role: string): Promise<void> { ... }

export async function createInvitation(token: string, wsId: string, inviteeId: string, role: string): Promise<Invitation> { ... }
export async function getMyInvitations(token: string): Promise<Invitation[]> { ... }
export async function acceptInvitation(token: string, invId: string): Promise<void> { ... }
export async function declineInvitation(token: string, invId: string): Promise<void> { ... }
```

**Verification:** TypeScript compilation passes.

---

## Phase 7: Frontend Components

### 7.1: WorkspaceSwitcher component

**File:** `src/lib/components/workspace/WorkspaceSwitcher.svelte` (NEW)

Placement: Top of Sidebar, below OrgSwitcher (conditional on multiple orgs).

Features:
- Shows current workspace name + role badge (owner/manager/member)
- Dropdown with all user's workspaces in current org
- Active workspace highlighted
- Pending invitation count badge
- "Create Workspace" button (gated: org admin or permission)
- "Manage Workspace" link → admin panel (gated: owner/manager)
- "Show archived" toggle

**Import into Sidebar.svelte** (above chat list section).

**Verification:** Manual browser test — switch workspaces, verify UI updates.

### 7.2: InvitationsPanel component

**File:** `src/lib/components/workspace/InvitationsPanel.svelte` (NEW)

Shown when badge is clicked or when user has no active workspace with pending invitations.

Features:
- Lists pending invitations with: workspace name, role, inviter name, time remaining
- Accept button → calls API, switches to workspace on accept
- Decline button → calls API, removes from list
- Expired invitations shown as disabled

**Verification:** Manual test — invite user, verify panel shows, accept/decline works.

### 7.3: CreateWorkspaceModal component

**File:** `src/lib/components/workspace/CreateWorkspaceModal.svelte` (NEW)

Modal triggered from WorkspaceSwitcher dropdown.

Fields:
- Name (required)
- Description (optional)
- Credit quota (optional, tokens/month)

On create:
1. API call to create workspace
2. User auto-added as owner
3. `switchWorkspace(newWs)` called
4. Modal closes, user sees new workspace

**Verification:** Create workspace → verify it appears in switcher + user is owner.

### 7.4: No-Workspace state in (app) layout

**File:** `src/routes/(app)/+layout.svelte`

Add conditional rendering:
```svelte
{#if !$currentWorkspace}
    <InvitationsPanel />
{:else}
    <!-- Existing chat UI -->
{/if}
```

**Verification:** New user with no workspace → see InvitationsPanel, no chat UI.

### 7.5: Workspace management page (admin)

**File:** `src/routes/(app)/admin/workspaces/[id]/+page.svelte` (NEW)

4 tabs:
1. **General** — name, description, archive button
2. **Members** — list, invite, role change, remove
3. **Credits** — quota setting, consumed/limit bar, period info
4. **Danger Zone** — archive workspace, delete workspace (both with confirmation modals)

Permission gating:
- Members tab: owner/manager can manage, members can view
- Credits tab: owner + org admin
- Danger Zone: owner + org admin

**Add link from WorkspaceSwitcher:** "Manage Workspace" → `/admin/workspaces/{wsId}`

**Verification:** Navigate to admin workspace page → verify tabs render with correct permissions.

---

## Phase 8: Socket.IO Integration

### 8.1: Backend Socket.IO handlers

**File:** `backend/open_webui/socket/main.py`

Add handlers:
```python
@sio.on('leave-workspace')
async def leave_workspace(sid, data):
    ws_id = data.get('workspace_id')
    if ws_id:
        await sio.leave_room(sid, f'ws:{ws_id}')

@sio.on('join-workspace')
async def join_workspace(sid, data):
    ws_id = data.get('workspace_id')
    user = get_user_from_session(sid)
    if not user:
        raise ConnectionRefusedError('Not authenticated')
    
    # Verify membership
    membership = await WorkspaceMembers.get_member(ws_id, user.id)
    if not membership:
        raise ConnectionRefusedError('Not a workspace member')
    
    await sio.enter_room(sid, f'ws:{ws_id}')
    # Emit current credit state
    workspace = await Workspaces.get_workspace_by_id(ws_id)
    await sio.emit('credit_updated', {
        'consumed': workspace.credit_consumed,
        'quota': workspace.credit_quota
    }, to=sid)
```

**Verification:** Connect via WebSocket, join workspace, leave workspace, verify room membership.

### 8.2: Scope chat events to workspace rooms

In `process_chat_response()`:
```python
# Instead of emitting to user:{user_id} only:
await sio.emit('chat:completion', event_data, to=f'ws:{workspace_id}')
await sio.emit('chat:completion', event_data, to=f'user:{user_id}')  # Keep user room for personal events
```

**Verification:** Two users in different workspaces — verify chat events don't leak.

---

## Phase 9: Testing

### 9.1: Create workspace isolation test suite

**Directory:** `backend/tests/workspace_isolation/`

Test files:
- `test_db_queries.py` — 8 tests (one per workspace-scoped model)
- `test_api_endpoints.py` — API-level isolation tests
- `test_shared_resources.py` — Cross-workspace sharing via access grants
- `test_credit_deduction.py` — Quota enforcement and deduction accuracy
- `test_invitation_flow.py` — Full invitation lifecycle
- `test_socket_isolation.py` — Socket.IO room isolation

**Pattern per test:**
```python
async def test_chat_isolation():
    org = await create_test_org()
    ws_a = await create_test_workspace(org, "A")
    ws_b = await create_test_workspace(org, "B")
    
    # Write in workspace A
    chat = await Chats.insert_new_chat(..., workspace_id=ws_a.id)
    
    # Read from workspace A → should see chat
    chats_a = await Chats.get_chats(workspace_id=ws_a.id)
    assert chat.id in [c.id for c in chats_a]
    
    # Read from workspace B → should NOT see chat
    chats_b = await Chats.get_chats(workspace_id=ws_b.id)
    assert chat.id not in [c.id for c in chats_b]
```

**Run:** 
```bash
cd backend && ../venv/bin/python -m pytest tests/workspace_isolation/ -v
```

---

## Phase 10: Verification & Cleanup

### 10.1: Full isolation regression

```bash
cd backend
../venv/bin/python -m pytest tests/workspace_isolation/ -v
# Expect: ALL PASS (0 failures)
```

### 10.2: Frontend build

```bash
cd /path/to/open-webui
NODE_OPTIONS=--max-old-space-size=8192 npx vite build --logLevel error
# Expect: Build successful
```

### 10.3: Server smoke test

```bash
# Start server
cd backend && ../venv/bin/uvicorn open_webui.main:app --port 8080

# Health check
curl http://localhost:8080/health
# Expect: {"status": true}
```

### 10.4: Workspace E2E flow

Manual or scripted:
1. Create org → verify default workspace created
2. Create second workspace → verify appears in switcher
3. Invite user to workspace → verify notification
4. Accept invitation → verify membership
5. Switch workspaces → verify data isolation
6. Set quota → verify enforcement
7. Archive workspace → verify read-only
8. Delete workspace → verify graveyard

---

## File Change Summary

| Phase | Files Created | Files Modified |
|-------|--------------|---------------|
| 1 (Migrations) | 2 | 0 |
| 2 (Models) | 3 | 8 |
| 3 (Middleware) | 0 | 3 |
| 4 (API) | 1 | 8 |
| 5 (Stores) | 0 | 2 |
| 6 (API Client) | 1 | 24+ |
| 7 (Components) | 4 | 2 |
| 8 (Socket.IO) | 0 | 1 |
| 9 (Tests) | 6 | 0 |
| **Total** | **17 new** | **48+ modified** |

---

## Risk Register

| Risk | Mitigation |
|------|-----------|
| Workspace switch event fires on init → wipes chat list | `initTenant()` sets `currentWorkspace` directly, no event |
| orgHeaders() spread conflict with workspaceHeaders() | workspaceHeaders() always FIRST in spread |
| Cross-workspace data leak in RAG | Vector DB collection namespacing + middleware safety net |
| Credit deduction race condition | Optimistic locking (CAS on version columns) |
| Socket.IO room leak on rapid workspace switch | Leave old room BEFORE joining new room |
| Broken Sidebar after component changes | Sidebar is 1659 lines — use single-line patch(), never multi-line replace |
