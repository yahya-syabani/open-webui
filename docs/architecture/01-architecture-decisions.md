# StratLogic Open WebUI — Architecture Decisions Register

> **37 decisions from interactive architecture workshop — June 2026**
> See `00-comprehensive-documentation.md` for the full codebase analysis that informed these decisions.

---

## Decision Map

```
Business Model (1-3)
  └── Entity Hierarchy (4-6)
       ├── Workspace Lifecycle (7-10)
       ├── RBAC & Permissions (11-14)
       ├── Credit & Billing (15-18)
       └── AI Context Isolation (19-20)

Frontend Architecture (21-31)
  ├── Stores & State (21-22, 30)
  ├── Components & UX (23, 25-28, 31)
  ├── API Client (24)
  └── Socket.IO (29)

Backend Architecture (32-37)
  ├── Middleware (32)
  ├── Query Pattern (33)
  ├── Credit Pipeline (34)
  ├── Invitation API (35)
  ├── Testing Strategy (36)
  └── Migration Strategy (37)
```

---

## Section A: Business & Entity Model

### AD-1: Business Model
**Decision:** B2B SaaS — StratLogic sells to companies who provision their employees.
**Rationale:** Enterprise customers are the target market. Organizations are billing entities. Admins manage users. Individual developers are not the primary market.
**Consequences:** Org-level credit pools, admin-managed provisioning, no personal billing accounts initially.

### AD-2: Pricing Model
**Decision:** Per-usage (metered AI consumption).
**Rationale:** AI costs are usage-based. Per-seat doesn't align cost with value. The platform is an AI interface, not a general collaboration tool.
**Consequences:** Token metering required. Monthly quota system. Cost estimation pipeline. Audit trail for every deduction.

### AD-3: Credit Ownership
**Decision:** Organization-owned credit pool.
**Rationale:** B2B model means the company pays. Users don't have personal wallets. Credits stay with the org when users leave.
**Consequences:** No user-level credit balance. Deduction always checks org balance. Workspace quotas provide subdivision control.

### AD-4: Cross-Organization Sharing
**Decision:** None. Hard boundary. Option A (strict isolation).
**Rationale:** Cross-org sharing is the #1 source of complexity in multi-tenant SaaS. Not needed for v1. The data model doesn't preclude adding it later via access grants.
**Consequences:** No federated identity. No cross-org resource sharing. Simpler security model. Faster to build.

### AD-5: Workspace Purpose
**Decision:** Team/department subdivisions with strong isolation (Scenario A).
**Rationale:** Workspaces serve as collaboration containers. Engineering and Marketing need separate contexts. Models, knowledge bases, and chats should differ per team.
**Consequences:** `workspace_id` on 7 resource tables. Workspace switch cascade resets stores. Explicit membership model.

### AD-6: Workspace Membership
**Decision:** Model B — explicit membership. Users must be added to a workspace to access it.
**Rationale:** Enterprise-grade requirement. Workspaces may contain sensitive data. Opt-in access prevents accidental exposure. Matches Slack private channels.
**Consequences:** Invitation flow required. Provisioning UI. "No workspace" state for new members. Workspace role system (owner/manager/member).

### AD-7: Workspace Roles
**Decision:** Three roles: owner, manager, member. Creator is auto-owner. Org admin can override.
**Rationale:** Owner for full control. Manager for member management. Member for baseline usage. Org admin has implicit owner-level access to all workspaces.
**Consequences:** `workspace_member.role` column. Permission checks cascade: org admin → owner → manager → member.

### AD-8: Workspace Deletion
**Decision:** Soft-delete to graveyard container. Not hard-delete.
**Rationale:** Data must never be destroyed without recovery window. Audit trail preservation. GDPR compliance via later purge mechanism.
**Consequences:** `workspace.status` column (active/archived/deleted). `workspace.deleted_at` timestamp. Graveyard query filter for admin. Recovery API.

### AD-9: Workspace Archive State
**Decision:** Three-state lifecycle: active → archived → deleted (graveyard). Archive is intermediate.
**Rationale:** Archived workspaces are read-only but restorable. Completed projects shouldn't clutter the switcher. 30-day restore window before graveyard.
**Consequences:** State machine in workspace model. UI toggle "Show archived." Quota frozen on archive. No new chats in archived workspaces.

### AD-10: Default Workspace Bootstrap
**Decision:** Auto-create "Default" workspace on org creation.
**Rationale:** Users need a workspace to chat. Model B means no self-join. Chicken-and-egg solved by auto-provisioning. Admin can rename/delete later.
**Consequences:** Org creation triggers workspace creation. Owner auto-assigned. No user-facing workspace creation prompt on first login.

---

## Section B: RBAC & Permissions

### AD-11: Permission Model
**Decision:** Workspace baseline + group elevation. Allow-override (groups only add, never subtract).
**Rationale:** Workspace membership should grant basic usage (chat, upload, query). Groups grant elevated permissions (create models, manage members, share). Simpler to reason about — being in a group can only help.
**Consequences:** Two-tier permission check: baseline defaults → merge group permissions (OR logic, True wins). No subtractive permissions.

### AD-12: Group Creation
**Decision:** Admin-only. Groups are an administrative tool, not user self-organization.
**Rationale:** Groups are the sharing primitive. Admin control prevents sprawl. Simpler UX — users don't need to understand groups to use the product.
**Consequences:** Group CRUD in admin panel only. No "create group" button for regular users. Group list visible to all for sharing target selection.

### AD-13: Group Scope
**Decision:** Groups are org-scoped (not workspace-scoped).
**Rationale:** Groups are the cross-workspace sharing primitive. A group must be visible to all workspaces for sharing to work. Engineering can share with "Marketing Team" group regardless of workspace.
**Consequences:** Groups have `organization_id` but no `workspace_id`. Group members are org-level. Group permissions apply across all workspaces.

### AD-14: User Departure
**Decision:** Resources stay with the org. Ownership transfers to workspace owner or org admin. Personal data (memories) deleted.
**Rationale:** Work product belongs to the company. Audit trail preserved. Identity becomes "Former Member" for attribution.
**Consequences:** Cascade logic on org member removal. Ownership transfer for workspace-scoped resources. Memory deletion. API key revocation.

---

## Section C: Credit & Billing

### AD-15: Credit Pool Model
**Decision:** Single org-level credit pool. Workspace quotas provide subdivision control.
**Rationale:** Company buys credits once. Workspace quotas prevent one team from starving others. Simpler than per-workspace pools.
**Consequences:** `organization.credit_balance`. `workspace.credit_quota` (nullable). Deduction checks: workspace quota → org balance.

### AD-16: Quota Enforcement
**Decision:** Hard block before model call (Option A). Conservative cost estimate. Actual cost deducted post-call.
**Rationale:** Quotas are limits, not suggestions. Pre-check prevents free usage. Conservative estimate avoids mid-session blocks from estimate errors. Actual deduction is accurate.
**Consequences:** Cost estimation function. Pre-call quota check. Post-call CAS deduction. Dead quota from conservative estimates minimized by post-deduction accuracy.

### AD-17: Quota Period Reset
**Decision:** Lazy reset with atomic compare-and-swap on first access of new period.
**Rationale:** No cron dependency. No missed resets if server is down. CAS prevents race conditions. Industry standard (Stripe pattern).
**Consequences:** `workspace.quota_period_start` column. CAS update: `WHERE quota_period_start = old_period SET consumed=0, period_start=new_period`. Monthly period (configurable).

### AD-18: Credit Deduction Timing
**Decision:** Pre-check (conservative estimate) → model call → post-deduct (actual cost). Optimistic locking on org balance.
**Rationale:** Pre-check prevents overage. Post-deduct ensures accuracy. Optimistic lock prevents double-spend. Retry on CAS failure.
**Consequences:** Two-phase deduction. `organization.credit_version` for CAS. `credit_transaction` table is append-only. Socket.IO event on balance change.

---

## Section D: AI Context Isolation

### AD-19: Context Isolation Strategy
**Decision:** Defense-in-depth: DB query filters + middleware safety net.
**Rationale:** Query filters are the primary gate (every SELECT adds `WHERE workspace_id = ?`). Middleware safety net scans assembled context before model dispatch. Belt AND suspenders.
**Consequences:** `apply_workspace_filter()` on all model methods. Middleware-level `verify_context_isolation()` scanning sources for workspace_id mismatch. Prod-always mode (not dev-only).

### AD-20: Vector DB Namespacing
**Decision:** Collection names include workspace_id: `kb-{ws_id}-{kb_id}`. Memory namespace: `memory-{org_id}-{ws_id}-{user_id}`.
**Rationale:** Vector DB queries don't go through SQLAlchemy. Collection-level isolation is the only guarantee. Prevents cross-workspace RAG leakage.
**Consequences:** Collection naming convention. Migration of existing collections (if any). Retrieval pipeline must construct correct collection names.

---

## Section E: Frontend Architecture

### AD-21: Store Architecture
**Decision:** Extend `tenant.ts` — single context file for org + workspace state. Do NOT create separate workspace.ts.
**Rationale:** Single source of truth for context switching. Regression risk assessment: safe if switchWorkspace() is sole mutation path, initTenant() sets directly without firing events, all $currentWorkspace subscribers handle null.
**Consequences:** `tenant.ts` gains: `userWorkspaces`, `workspaceRole`, `workspaceInvitations`, `workspaceSwitchEvent`, `switchWorkspace()`, `workspaceHeaders()`.

### AD-22: Workspace Switch Cascade
**Decision:** Reset workspace-scoped stores on switch: chats, models, knowledge, prompts, memories, files, pinnedChats, chatId. Also folders (become workspace-scoped).
**Rationale:** Different workspace = different data. Only reset what's workspace-scoped. Org-scoped resources (tools, skills, functions, notes, tags, channels, groups) persist across switches.
**Consequences:** `workspaceSwitchEvent` subscriber in `stores/index.ts`. Separate from `orgSwitchEvent` subscriber (resets all stores).

### AD-23: Folder Scoping
**Decision:** Folders become workspace-scoped (revision from earlier org-scoped decision).
**Rationale:** Folders organize chats, which are workspace-scoped. An org-wide folder tree containing workspace-specific chats creates confusion. Clean separation per workspace.
**Consequences:** `folder.workspace_id` column needed in migration. Folders reset on workspace switch.

### AD-24: Workspace Switcher Placement
**Decision:** Option C — nested org + workspace in sidebar. Org switcher above workspace switcher (conditional on multiple orgs).
**Rationale:** Shows hierarchy explicitly. User sees both context levels. Industry standard for platforms with nested contexts (Slack, Discord).
**Consequences:** Two dropdowns in Sidebar. `OrgSwitcher` component (conditional). `WorkspaceSwitcher` component (always visible when workspace active).

### AD-25: API Header Interceptor
**Decision:** Universal `workspaceHeaders()` added to ALL API modules. No selective application.
**Rationale:** Simplest mental model. Backend ignores the header for org-scoped endpoints. No developer has to remember which modules need it.
**Consequences:** `workspaceHeaders()` utility exported from `tenant.ts`. Added to fetch headers in all 24+ API modules alongside existing `orgHeaders()`.

### AD-26: Invitation UI
**Decision:** Badge on workspace switcher (passive, persistent) + one-time toast on new invitation (active, ephemeral). Invitations panel for resolution.
**Rationale:** Badge persists until all invitations resolved. Toast alerts on new invitation. Panel shows Accept/Decline per invitation with workspace name and role.
**Consequences:** `WorkspaceSwitcher` component with badge count. `InvitationsPanel` component. Toast notification integration. REST API fallback for offline users.

### AD-27: Workspace Management
**Decision:** 4-tab workspace settings page in admin panel: General, Members, Credits, Danger Zone.
**Rationale:** Central place for workspace administration. Permission-gated tabs (Credits = owner+org admin, Danger = owner only). Matches settings page pattern already in codebase.
**Consequences:** New route: `admin/workspaces/[id]/+page.svelte`. RBACGuard gating on tabs. Member management UI (list, invite, role change, remove).

### AD-28: Permission-Gated UI
**Decision:** `RBACGuard` component wrapping UI elements. Checks: org role → workspace role → group permissions → resource grants.
**Rationale:** Consistent gating pattern. Allow-override simplifies logic. Resource-level grants for shareable resources. Matches existing RBACGuard pattern in codebase.
**Consequences:** `RBACGuard` usage across: create buttons, delete buttons, share buttons, settings links, member management UI.

### AD-29: Workspace Creation Flow
**Decision:** Switcher dropdown → "Create Workspace" button (if permitted) → modal → auto-switch to new workspace.
**Rationale:** Natural creation point. Auto-switch reduces friction. Creator becomes owner automatically.
**Consequences:** `CreateWorkspaceModal` component. Permission check: org admin OR `workspace.create` group permission. Auto-switch fires `workspaceSwitchEvent`.

### AD-30: Socket.IO Room Management
**Decision:** Dedicated `ws:{workspace_id}` rooms. Leave old room before joining new. Server-side membership validation on join.
**Rationale:** Prevents cross-workspace event leakage. Server validates membership before allowing join. Timing: leave → update store → join ensures no race condition.
**Consequences:** Socket.IO server handlers: `leave-workspace`, `join-workspace`. Frontend: `switchWorkspace()` manages room membership. Server validates `workspace_member` before allowing join.

### AD-31: No-Workspace State
**Decision:** Invitation resolution UI shown. No chat interface until workspace is active.
**Rationale:** Model B consequence. User must be in a workspace to chat. Pending invitations take priority. "Ask your admin" messaging for users with zero invitations.
**Consequences:** Conditional rendering in `(app)/+layout.svelte`. `$currentWorkspace === null` → show invitations panel or "no workspaces" message. No chat, no sidebar chat list.

### AD-32: New Store Inventory
**Decision:** 6 new stores in `tenant.ts`: `currentWorkspace`, `userWorkspaces`, `workspaceRole`, `workspaceInvitations`, `workspaceSwitchEvent`, `workspaceQuota`.
**Rationale:** Minimum necessary for workspace functionality. `workspaceSwitchEvent` fires cascade. `workspaceQuota` provides real-time credit visibility.
**Consequences:** All new stores writable. `workspaceSwitchEvent` subscriber in `stores/index.ts`. `workspaceHeaders()` derived from `currentWorkspace`.

---

## Section F: Backend Architecture

### AD-33: Workspace Middleware
**Decision:** Two-layer: ASGI extraction (WorkspaceHeaderMiddleware, before auth) + FastAPI validation dependency (after auth). Defense-in-depth: `apply_workspace_filter` also validates.
**Rationale:** Extraction is universal (header parsing). Validation is opt-in per-endpoint (auth-gated). Query utility is safety net (catches forgotten dependencies).
**Consequences:** `WorkspaceHeaderMiddleware` in ASGI chain. `get_validated_workspace_context` dependency. `apply_workspace_filter` with lazy validation.

### AD-34: Query Pattern
**Decision:** Simple filter for non-shareable resources (chats, memories, files). OR query with access grants for shareable resources (models, KBs, prompts).
**Rationale:** Chats/memories never shared across workspaces — simple filter sufficient. Models/KBs can be shared via access grants — need UNION with grant checks.
**Consequences:** Two query patterns. `apply_workspace_filter()` for simple cases. Custom OR queries for shareable resources. Frontend tags shared resources with `shared_from_workspace_id`.

### AD-35: Credit Deduction Pipeline
**Decision:** Pre-call: estimate cost → check workspace quota (if set) → check org balance. Post-call: calculate actual cost → atomic dual deduction (workspace consumed + org balance) with CAS.
**Rationale:** See AD-16 and AD-18.
**Consequences:** Cost estimation function (input tokens * price + max_output * price). CAS on `workspace.quota_period_start` and `organization.credit_version`. Retry on CAS failure. Credit transaction record.

### AD-36: Invitation API
**Decision:** 7 endpoints: create, list (workspace), cancel, list mine, accept, decline. Socket.IO notification + REST fallback for offline users.
**Rationale:** Full invitation lifecycle. Workspace-scoped list for managers. User-scoped list for invitees. REST fallback because Socket.IO events are ephemeral.
**Consequences:** `workspace_invitation` table. 7 API endpoints. Socket.IO event: `user:{id}` room. GET `/invitations/me` called on app load + socket reconnect.

### AD-37: Testing Strategy
**Decision:** 4-layer defense: DB query tests → API endpoint tests → full isolation suite → middleware safety net.
**Rationale:** Regression is the #1 risk. Every model method tested for cross-workspace leakage. CI runs full suite on every PR. Middleware safety net catches leaks in production.
**Consequences:** Test directory: `tests/workspace_isolation/`. Test pattern: write in workspace A, read from workspace B, assert not visible. Shared resource test: grant access, assert visible.

### AD-38: Migration Strategy
**Decision:** Two Alembic migrations: (1) new tables (workspace, workspace_member, workspace_invitation), (2) ALTER existing tables (add workspace_id + indexes). NOT NULL columns, composite indexes. Vanilla clone = no data backfill needed.
**Rationale:** Fresh database. Clean schema. Composite indexes on (organization_id, workspace_id) match query pattern. Two files for dependency order (parent tables first, child columns second).
**Consequences:** Migration 1: `xxxx_add_workspace_tables.py`. Migration 2: `xxxx_add_workspace_id_columns.py`. All migrations idempotent. `downgrade()` for rollback.

---

## Appendix: Decision Summary Table

| # | Topic | Decision |
|---|-------|----------|
| 1 | Business model | B2B SaaS |
| 2 | Pricing | Per-usage (metered) |
| 3 | Credit ownership | Organization pool |
| 4 | Cross-org sharing | None (hard boundary) |
| 5 | Workspace purpose | Team subdivision (Scenario A) |
| 6 | Workspace membership | Explicit (Model B) |
| 7 | Workspace roles | owner/manager/member |
| 8 | Workspace deletion | Soft-delete to graveyard |
| 9 | Workspace archive | Active → Archived → Graveyard |
| 10 | Bootstrap | Auto-create "Default" workspace |
| 11 | Permission model | Baseline + group elevation, allow-override |
| 12 | Group creation | Admin-only |
| 13 | Group scope | Org-scoped |
| 14 | User departure | Resources stay, memories deleted |
| 15 | Credit pool | Org-level with workspace quotas |
| 16 | Quota enforcement | Hard block pre-call |
| 17 | Quota reset | Lazy CAS on first access |
| 18 | Deduction timing | Pre-check + post-deduct |
| 19 | Context isolation | Defense-in-depth + middleware |
| 20 | Vector DB namespacing | workspace_id in collection names |
| 21 | Store architecture | Extend tenant.ts |
| 22 | Workspace switch cascade | Reset 8 workspace-scoped stores |
| 23 | Folder scoping | Workspace-scoped |
| 24 | Workspace switcher | Nested org + workspace (Option C) |
| 25 | API interceptor | Universal workspaceHeaders() |
| 26 | Invitation UI | Badge + toast + panel |
| 27 | Workspace management | 4-tab admin page |
| 28 | Permission gating | RBACGuard component |
| 29 | Creation flow | Switcher → modal → auto-switch |
| 30 | Socket.IO rooms | Dedicated ws:{id} rooms |
| 31 | No-workspace state | Invitation resolution UI |
| 32 | New stores | 6 stores in tenant.ts |
| 33 | Workspace middleware | Two-layer extraction + validation |
| 34 | Query pattern | Simple filter vs OR with grants |
| 35 | Credit pipeline | Pre-check + atomic post-deduction |
| 36 | Invitation API | 7 endpoints + Socket.IO + REST |
| 37 | Testing strategy | 4-layer defense |
| 38 | Migration strategy | 2 Alembic files, clean schema |
