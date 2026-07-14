# Team Onboarding Flow — Architecture & Implementation Plan

## Context

Digitomics InfraOps has no coherent, backend-persisted onboarding experience today. What exists is fragmented across four disconnected mechanisms: a one-time "name your team" gate, a partially-wired `workspace_setup` JSON blob with a completion bug (only one of four steps actually gates "done"), a session-only "connect your tools" nag dialog, and a fully-built but RBAC-role-based (not job-persona-based) product tour that lives entirely in `localStorage`. None of them capture a user's persona, company context, or interests, and none drive a persistent, dynamic "getting started" checklist. This plan introduces a single, backend-driven 5-stage onboarding flow triggered by team (tenant) creation, consolidating the fragmented pieces into one system, without duplicating any existing page, API, widget, or component.

Three scope decisions were confirmed with the user before finalizing this plan:
1. **Trigger scope**: onboarding fires only on self-service tenant creation (the real "Create Team" action today) — not on enterprise/platform-admin-provisioned tenants.
2. **Sequencing**: the existing team-naming gate is merged into the new flow's Stage 2 (Company Info); the existing Product Tour still runs, unchanged, immediately after the new onboarding completes.
3. **Dashboard personalization scope**: Phase 1 (this effort) delivers persona-driven Quick Actions, checklist ordering, and light nav emphasis — all backed by real, already-live data. Full widget-grid reprioritization is deferred to a documented Phase 2, because the persona-relevant dashboard widgets (`budget-tracker`, `k8s-cluster-status`, `CfoView`/`CtoView`, etc.) are currently mock-data-only and unrouted — wiring them to real data is a separate, much larger effort than onboarding itself.

---

## Part A — Current State (verified against code, not just docs)

`docs/BACKEND_ARCHITECTURE.md` and `docs/FRONTEND_ARCHITECTURE.md` are accurate for the general system but are **silent on the self-service tenant-creation path** — verified directly in code instead.

### A1. Team = Tenant (no separate model)

There is no `Team`/`Organization`/`Membership` table. "Team" is the product-facing name for `Tenant` (`backend/app/db/models.py:98-132`). `User.tenant_id` is a **required, non-nullable FK** (line 150-152) — strictly 1:1, enforced further by Postgres RLS (`_enable_tenant_rls`, `alembic/versions/0001_redesigned_schema.py:68-110`). Additional teammates join via `TenantInvitation` (lines 267-323), never via a second tenant membership.

### A2. The real "Create Team" flow

Controlled by `settings.deployment_mode` (`backend/core/config.py:184-398`):
- **Self-service mode**: `POST /api/v1/auth/register/self-service/google` and `.../self-service/password` (`backend/routes/auth_routes.py:1263,1340`) each call **`create_tenant_with_defaults()`** in `backend/services/tenant_provisioning.py:170` — creates the `Tenant` row, seeds built-in `AppRole`s, creates `TenantSettings`, makes the signer-upper `tenant_admin`. If no name given, tenant is named `"My team"` (`PROVISIONAL_TEAM_NAME`, line 76) and `TenantSettings.limits["workspace_setup"]["team_name_pending"] = true`. Frontend: `frontend/src/app/(auth)/signup/page.tsx` ("Create your free team" copy, lines 231-391), `postAuthPath()` (lines 29-31) redirects to `/dashboard/onboarding/team` when pending.
- **Enterprise mode**: users pick from existing tenants (`GET /registration-tenants`); only platform admins create tenants (`backend/routes/platform_admin_routes.py:543`, same `create_tenant_with_defaults()` call site). **Out of scope per decision above.**

`create_tenant_with_defaults()` is the single call site for both self-service and platform-admin tenant creation — the natural hook point for initializing onboarding state.

### A3. Existing onboarding-adjacent infrastructure (must not be duplicated)

| Mechanism | Location | Storage | Status |
|---|---|---|---|
| Team-naming gate | `frontend/src/app/dashboard/onboarding/team/page.tsx`, `POST /api/v1/auth/me/team` (`auth_routes.py:1674`) | `TenantSettings.limits["workspace_setup"]["team_name_pending"]` (JSON) | Working, single-step, mandatory. **Merge into new Stage 2.** |
| Workspace Setup Wizard | `GET/PUT /api/v1/settings/workspace-setup*` (`backend/routes/settings_routes.py:616-762`), `useOnboardingStatus`/`useAppSettings` (`use-app-settings.ts`) | `TenantSettings.limits["workspace_setup"]` (JSON: `steps.{profile,tools,integrations,budget}`) | Backend state exists; **no frontend UI renders it** (hook is fetched but unconsumed); completion logic bug — only `profile` step gates `completed`. **Do not extend; superseded by new tables (Part C).** Leave data in place, stop writing to it. |
| Connection Setup Prompt | `frontend/src/components/connections/connection-setup-prompt.tsx` | 3 `UserSettings` columns (migration `0004`) + sessionStorage | Working nag dialog, tenant_admin/admin only, one-time-per-session. **Retire in Phase 4** — redundant once Quick Actions (Stage 3) always offers this. |
| Product Tour | `frontend/src/components/product-tour/*`, `frontend/src/lib/product-tour/*` | **localStorage only**, 3 RBAC personas (`platform_admin`/`tenant_admin`/`standard_user`) | Fully built spotlight walkthrough. Different concept from job-function persona. **Keep as-is, just change its start-gate condition (Part E).** |
| `SetupStep` type | `frontend/src/types/integration.ts` | N/A (unused) | Dead scaffolding (`{step,title,description,action,completed}`) — zero consuming UI. **Good shape to repurpose for the new checklist item type.** |
| `CfoView`/`CtoView` | `frontend/src/components/dashboard/{cfo,cto}-view.tsx` | Mock data (`data/mock-cost-data.ts`, `mock-k8s-data.ts`) | Dead code — not imported/routed anywhere. **Not used in Phase 1** (per dashboard-scope decision). |

### A4. Dashboard today

`/dashboard/overview/page.tsx` (13 lines) renders only `<ConnectionSetupPrompt />` + `<BillingOverview />`. The 12 components under `components/dashboard/` (`metric-card`, `budget-tracker`, `cost-trend-chart`, `department-spend`, `savings-opportunities`, `spend-by-service`, `k8s-cluster-status`, `resource-utilization`, `deployment-activity`, `capacity-planner`, plus `cfo-view`/`cto-view`) are self-contained `Card`s, mostly mock-data-driven, **not composed into the live dashboard at all**. There is currently zero persona-based dashboard composition in the running app.

### A5. Integration/connection catalogs (source of truth for Quick Actions)

Two independent catalogs, both real and API-exposed — reuse both, don't re-model either:
- **Composio collaboration tools**: `backend/Tools/integrations/registry.py` → `INTEGRATIONS` dict (slack, microsoftteams, gmail, github, googledocs, googledrive, googlesheets, slackbot) → `GET /integrations/catalog` → `/dashboard/integrations`.
- **Cloud/billing connectors**: `backend/Tools/connections/catalog.py` → `CODEBASE_CONNECTION_TOOLS` (20 providers incl. AWS, GCP, **Azure** [fully implemented end-to-end], GoDaddy, Jira, Trello, Databricks, etc.) → `GET /api/v1/tool-connections/catalog` → `/dashboard/connections` (single page, `CONNECTORS` array, bespoke wizard via `connectionStep` state, reusable `connector-picker.tsx`/`connector-card.tsx`).

**Not implemented at all** (would need net-new integration work, out of scope for onboarding): Oracle Cloud, live Kubernetes (dashboard widget is mock; a real unused GKE-sync backend pipeline exists but no frontend consumes it), Docker, Prometheus, Grafana, Loki (the last four are this app's *own* observability tooling, not customer-connectable integrations). **The recommendation engine must only ever surface actions for things present in the two catalogs above** — this is what keeps the config-driven design honest rather than promising connectors that don't exist.

### A6. Reusable UI primitives (`frontend/src/components/ui/`, 28 files)

Have already: `card`, `button`, `dialog`, `select`, `input`, `label`, `badge`, `progress` (used by `budget-tracker`, `k8s-cluster-status`), `accordion`, `status-badge`, `sheet` (built on `@base-ui/react`, **currently unused by anything** — good candidate for the floating widget's expand interaction). Missing: `popover`, a shared `stepper`/wizard primitive (both `/dashboard/connections` and `custom-tool-wizard.tsx` implement ad hoc step state independently), and any checklist/todo-list component (none exists anywhere in the repo).

### A7. State management conventions to follow

Server state → React Query (`queryKeys.*`, 5 min default staleTime, tenant-scoped invalidation via `invalidateTenantScopedQueries`). Client UI state → Zustand only for things with no backend yet. `api-client.ts`'s `apiFetch` + `ApiError` pattern is universal across `services/*.ts`. This plan introduces zero new state-management patterns — it slots into the existing ones.

---

## Part B — Key Architectural Decisions

1. **Dedicated relational tables, not another JSON blob.** The existing `workspace_setup` JSON blob already produced a real bug (3 of 4 steps don't gate completion) because JSON blobs can't enforce shape or be queried/indexed. New onboarding state gets proper tables (Part C).
2. **Hybrid checklist: derive from live data wherever possible, persist only what can't be derived.** "Connected AWS" is computed by querying `cloud_connections`/`tool_connections` directly — never trusted from a client-reported flag. Only genuinely non-derivable acks (e.g., "viewed welcome guide") get a small persisted table. This is what makes progress **impossible for a client to manipulate** (a stated requirement) — most of the checklist has no writable "mark complete" endpoint at all.
3. **Hard gate vs. soft checklist.** Stages 1-2 (persona, company info/interests) are a blocking redirect gate, exactly like today's team-naming gate. Stage 3 (Quick Actions) ends the gate — clicking "Go to Dashboard" (or acting on any action) calls `/complete`. Stages 4-5 (Getting Started Hub, floating widget) are ongoing, non-blocking, and persist independently of the "onboarding completed" flag, matching the spec's "never show onboarding again" (refers to the wizard) vs. "checklist continues to track state" (ongoing).
4. **New persona concept, kept separate from `TourPersona`.** `TourPersona` (`platform_admin`/`tenant_admin`/`standard_user`) is an RBAC-role concept driving a UI walkthrough. The new `persona_key` (CEO, CTO, DevOps Engineer, FinOps Practitioner, etc.) is a job-function concept driving recommendations. They coexist and are never conflated.
5. **Config-driven, not database-driven, catalogs.** Personas, interests, and static quick actions live in a version-controlled Python config module (`backend/core/onboarding_config.py`), not admin-editable DB tables. This satisfies "adding a persona shouldn't require code changes" (it's a data-only diff) without building a full admin-CRUD UI that nobody asked for.
6. **Persona/company-info captured once, at the tenant level.** Onboarding is a team-level setup activity performed by the tenant creator (always `tenant_admin` per `tenant_provisioning.py`). It is not per-user. The schema reserves room for a future per-user dashboard-view override without any migration changes now.
7. **Safe backfill: only tenants created after this ships ever see onboarding.** Every tenant that exists at migration time is marked `status='completed'` — no retroactive onboarding is forced onto existing users.

---

## Part C — Database Design

New Alembic revision `0008_team_onboarding.py` (chains from `0007`).

**`tenant_onboarding`** (1 row per tenant; replaces `team_name_pending` and the `workspace_setup` step model going forward)
| Column | Type | Notes |
|---|---|---|
| `tenant_id` | UUID, PK, FK `tenants.id` ON DELETE CASCADE | |
| `status` | enum: `not_started`, `in_progress`, `completed` | |
| `current_stage` | int (1=persona, 2=company info, 3=quick actions) | drives resume-after-refresh |
| `persona_key` | str, nullable | validated against `onboarding_config.ONBOARDING_PERSONAS` at write time |
| `team_name`, `company_name`, `industry`, `company_size`, `monthly_cloud_spend`, `region`, `business_type`, `primary_goals` | str/text/JSON as appropriate | Stage 2 fields |
| `created_by_user_id` | FK `users.id` | who initiated |
| `started_at`, `completed_at`, `created_at`, `updated_at` | timestamptz | |

**`tenant_onboarding_interests`** — `id`, `tenant_id` (FK), `interest_key`, `created_at`; unique on `(tenant_id, interest_key)`.

**`onboarding_task_state`** — `id`, `tenant_id` (FK), `task_key`, `status` (`done`/`dismissed`), `completed_at`, `completed_by_user_id`. Only used for the small set of non-derivable checklist items.

Both new tenant-scoped tables are added to the RLS-enabled table list (same mechanism as every other tenant table, per `0001_redesigned_schema.py`'s `_enable_tenant_rls` helper) — this is what makes cross-tenant access structurally impossible, not just an app-level check.

**Backfill (in the same migration):**
- Every existing row in `tenants` → insert `tenant_onboarding` with `status='completed', completed_at=now()`.
- Any tenant currently mid-flow (`workspace_setup.team_name_pending == true`) → `status='in_progress', current_stage=1` instead, so no one gets silently force-completed while actively naming their team.
- Leave `TenantSettings.limits['workspace_setup']`/`['team_name_pending']` JSON keys in place untouched (harmless, no longer read by new code) — no destructive cleanup migration.

---

## Part D — Backend Architecture

**New files:**
- `backend/core/onboarding_config.py` — static config: `ONBOARDING_PERSONAS` (key, label, icon, description, `recommended_interests`, `dashboard_priority` [reserved for Phase 2]), `ONBOARDING_INTERESTS` (key, label, category, `maps_to_tool_codes`/`maps_to_integration_ids`, `maps_to_route`), `STATIC_QUICK_ACTIONS` (invite_team → admin users page, enable_ai_operator → chat, create_budget → budget page, configure_alerts → monitoring).
- `backend/services/onboarding/recommendation_engine.py` — pure functions: `get_recommended_interests(persona_key)`, `get_recommended_quick_actions(persona_key, interests, tenant_id, db)` (cross-references `Tools/integrations/registry.py` + `Tools/connections/catalog.py` + `STATIC_QUICK_ACTIONS`, marks each connected/not via existing `cloud_connections`/`tool_connections`/`ComposioUserConnection` queries — reuses existing repository/service functions, never re-declares catalog data), `compute_checklist(tenant_id, db)`.
- `backend/services/onboarding/onboarding_service.py` — state transitions (init, save persona, save company info + interests, complete, complete/dismiss task), wraps the two tables in Part C.
- `backend/schemas/onboarding.py` — Pydantic request/response models (proper module, unlike `settings_routes.py`'s inline-schema pattern — minor convention improvement).
- `backend/routes/onboarding_routes.py` — new router, prefix `/api/v1/onboarding`.
- `backend/tests/test_onboarding.py` — mirrors existing `test_team_name_pending.py`.

**Endpoints** (mapped to the requested Create Team → ... → Load Dashboard chain):

| Endpoint | Method | Purpose | Auth |
|---|---|---|---|
| *(none — happens inside `create_tenant_with_defaults()`)* | — | **Initialize Onboarding**: one added call creates the `tenant_onboarding` row in the same transaction as tenant creation | n/a |
| `/api/v1/onboarding/status` | GET | **Get Onboarding Status**: status, current_stage, persona, company info, interests, computed checklist, progress %, est. time remaining — the one endpoint gating wizard-vs-dashboard | any authenticated tenant user |
| `/api/v1/onboarding/persona` | PUT | **Save Persona** (Stage 1), advances `current_stage` to 2 | tenant_admin/admin |
| `/api/v1/onboarding/company-info` | PUT | **Save Step / Save Interests** (Stage 2: company fields + interests), advances to 3; reuses existing `slugify_team_name`/tenant-rename logic from `tenant_provisioning.py` | tenant_admin/admin |
| `/api/v1/onboarding/complete` | POST | **Complete Onboarding** — ends the wizard gate | tenant_admin/admin |
| `/api/v1/onboarding/tasks/{task_key}/complete` | POST | **Complete Task** — only for non-derivable checklist items | tenant_admin/admin |

No "Load Dashboard" endpoint is introduced — that's the existing, unmodified `/dashboard/overview` + billing/connections APIs; onboarding status merely gates which UI renders.

**Modified files:**
- `backend/services/tenant_provisioning.py` — add the `tenant_onboarding` row creation inside `create_tenant_with_defaults()`.
- `backend/routes/auth_routes.py` — `POST /me/team` becomes a thin, backward-compatible wrapper delegating to `onboarding_service` (kept alive, not deleted, to avoid breaking any caller not found in this investigation).
- `backend/core/container.py` — wire new service(s) (existing DI pattern).
- `backend/main.py` — register new router (one line, existing pattern).
- `backend/services/permissions_catalog.py` — add a `team.onboarding` permission key for consistency with the RBAC catalog convention.

**Security** (structural, not just app-logic):
- RLS on both new tables (Part C) — cross-tenant reads/writes are impossible at the DB layer regardless of app bugs.
- Write endpoints require `tenant_admin`/`admin` (mirrors `ConnectionSetupPrompt`'s existing `canSeeConnectionSetupPrompt` role gate) — a regular teammate cannot mark onboarding complete or alter another team's setup.
- Most checklist items have **no client-writable completion state at all** — they're computed from real connection/user/budget tables, so there's nothing to manipulate.
- Audit log events (`onboarding.persona_set`, `onboarding.company_info_set`, `onboarding.completed`) via existing `audit_log_service.py`, consistent with today's `tenant.self_service_create`/`tenant.team_name_set` events.

---

## Part E — Frontend Architecture

**New files:**
- `src/app/dashboard/onboarding/page.tsx` — replaces `team/page.tsx`; full-viewport (DashboardShell already special-cases any `/dashboard/onboarding/*` path for chrome-less rendering — reused, not rebuilt). Hosts a 3-stage wizard using local step state (consistent with the existing ad hoc pattern in `/dashboard/connections` and `custom-tool-wizard.tsx`, rather than inventing a new stepper system for a single flow).
- `src/components/onboarding/{persona-step,company-info-step,quick-actions-step,onboarding-progress-indicator,getting-started-hub,setup-progress-widget}.tsx`.
- `src/components/ui/stepper.tsx` — small, optional new primitive (three call-sites duplicating step-state logic is a minor smell worth fixing while touching this area).
- `src/services/onboarding-service.ts`, `src/hooks/use-team-onboarding.ts`, `src/types/onboarding.ts`.

**Reused, unmodified:** `ui/card`, `button`, `select`, `input`, `label`, `badge`, `progress`, `accordion`, `status-badge`, `sheet` (first real consumer, for the floating widget's expand interaction), `connections/connector-picker.tsx` + `connector-card.tsx` (Quick Actions grid), all 12 `components/dashboard/*` widgets (untouched — not used in Phase 1), `BillingOverview` (untouched), both integration/connection catalogs (read-only consumption via existing services).

**Wizard content, mapped to existing destinations (no duplicate setup pages):**
- Stage 3 "Connect AWS/GCP/Azure/Slack/…" tiles link to `ROUTES.DASHBOARD_CONNECTIONS`/`ROUTES.DASHBOARD_INTEGRATIONS` with a `?from=onboarding&connector=<id>` query param. `/dashboard/connections` already has an internal `startConnectorWizard(connector, {step})` function (confirmed in code) — reading the query param to auto-open the right connector is a small, additive change, not a rebuild. "Invite Team" links to the existing admin/users page; "Enable AI Operator" to chat; "Create Budget" to the budget page; "Configure Alerts" to monitoring.
- No hard "return to onboarding" redirect trap: the floating widget (Stage 5, global, persists across all dashboard pages until fully done) is the return mechanism — the user finishes connecting AWS on its real page and can resume via the widget whenever they choose, rather than being bounced back.

**Routing/gating changes:**
- `frontend/src/components/layout/dashboard-shell.tsx` — extend the existing `team_name_pending` redirect conditional to check the new `useTeamOnboarding()` status (`status !== 'completed'` → redirect to `/dashboard/onboarding`, unless already on that path). Exempt platform-admin tenant-workspace-view sessions (reuse existing `platformAdminSnapshot`/`isViewingTenantWorkspace` distinction) so impersonating admins are never redirected into a tenant's wizard. Exempt non-admin roles from the hard redirect entirely (only `tenant_admin`/`admin` can progress the wizard; other roles just see the dashboard with the Getting Started Hub in read-only view).
- `proxy.ts` (edge) — unchanged; onboarding status requires a DB call and gating already happens client-side today (same for `team_name_pending`), keeping edge logic fast and stateless.
- `frontend/src/app/dashboard/onboarding/team/page.tsx` — replaced; add a redirect shim from the old path to `/dashboard/onboarding` for any stale links.
- `frontend/src/components/product-tour/tour-provider.tsx` — swap its existing `team_name_pending` start-gate for `onboardingStatus.status === 'completed'`, so the tour still correctly waits for the fuller new flow.
- `frontend/src/app/(auth)/signup/page.tsx` — `postAuthPath()` now checks the new onboarding status instead of the old flag.

**State management (no duplicate state):**
- Backend/DB is sole source of truth for progress — nothing onboarding-related in `localStorage` (satisfies "do not rely on Local Storage").
- React Query: `useTeamOnboarding()` (new name, avoids collision with the existing deprecated `useOnboardingStatus` in `use-app-settings.ts`), query key `queryKeys.onboarding.status()`, short staleTime (~30s) with refetch-on-focus enabled for this query specifically, so returning from Connect-AWS promptly reflects in the checklist. Mutations (`usePersonaMutation`, `useCompanyInfoMutation`, `useCompleteOnboardingMutation`, `useCompleteTaskMutation`) invalidate that key on success — the same pattern already used by action-center mutations.
- Zustand/component state only for the ephemeral "which wizard tab is showing" — never for progress truth.

**Dashboard integration (Phase 1 scope):**
- `src/app/dashboard/overview/page.tsx` — add `<GettingStartedHub />` above `<BillingOverview />`; auto-hides once all *required* checklist items are done.
- `components/layout/sidebar.tsx` — light-touch nav-group reordering by persona (e.g., FinOps → Budget/Cost-explorer group first) — small edit to existing ordering logic, not a new component.
- Phase 2 (documented, not built now): live-wire the mock dashboard widgets and build an actual persona-weighted widget grid on Overview, using the `dashboard_priority` field already reserved in `onboarding_config.py` — zero backend schema changes needed when that phase starts.

**Retire in Phase 4:** `connection-setup-prompt.tsx` and its session-storage helper — its "connect your tools" nag becomes redundant once Quick Actions always offers this during onboarding.

---

## Part F — Recommendation Engine (config-driven chain)

```
persona_key (config)
  → recommended_interests (config) ∪ user-selected interests (tenant_onboarding_interests)
    → cross-reference live catalogs: Tools/integrations/registry.py + Tools/connections/catalog.py + STATIC_QUICK_ACTIONS (config)
      → filter to only what actually exists as a connector/action today
        → sort by persona weight
          → Stage 3 Quick Actions list — AND → Stage 4/5 checklist items
            → "done" status computed live from cloud_connections / tool_connections / ComposioUserConnection / users / budget tables
```
Adding a persona or interest = one entry in `onboarding_config.py` (data change). Adding a new real integration = one entry in the existing catalogs (already how those catalogs work) — it then automatically becomes available as a recommendable action with zero changes to `recommendation_engine.py` logic.

---

## Part G — Sequence (new tenant, happy path)

```
Signup (self-service) → create_tenant_with_defaults()
  [+ inserts tenant_onboarding row: status=in_progress, current_stage=1]
→ JWT issued → frontend: useTeamOnboarding() → status != completed
→ redirect /dashboard/onboarding
→ Stage 1: PUT /onboarding/persona → current_stage=2
→ Stage 2: PUT /onboarding/company-info (+interests) → current_stage=3
→ Stage 3: GET /onboarding/status (computed Quick Actions)
   → user clicks "Connect AWS" → /dashboard/connections?from=onboarding&connector=aws
   → completes connection (existing flow, untouched)
   → (no explicit "complete" call — status will reflect it on next fetch)
→ user clicks "Go to Dashboard" → POST /onboarding/complete
→ redirect /dashboard/overview
   → GettingStartedHub + floating widget render remaining tasks (Stage 4/5, ongoing)
→ Product Tour auto-starts (gate now keys off onboarding completion)
```
Returning user, later login: `status == completed` → straight to dashboard, no onboarding, per spec.

---

## Part H — Edge Cases

- **Abandon mid-flow**: resumes at `current_stage` on next login — server-persisted, satisfies "resume from last completed step."
- **Invited non-admin teammate joins a tenant whose onboarding is still `in_progress`**: not redirected into the wizard (write/gate endpoints are tenant_admin/admin-only); sees dashboard with the Getting Started Hub in a read-only view of tenant-wide progress.
- **Checklist item flips back** (e.g., AWS later disconnected): reflects reality since it's derived live — this only affects the ongoing Stage 4/5 checklist, never re-triggers the completed wizard gate.
- **Platform admin viewing a tenant via switch-tenant**: exempted from the tenant's onboarding gate (reuses existing `platformAdminSnapshot`/`isViewingTenantWorkspace` distinction).
- **Concurrent Stage 2 submits by two admins**: last-write-wins is acceptable for this low-stakes form data; no locking needed.

---

## Part I — Performance

- `GET /onboarding/status` computes checklist state on read — start uncached (small queries against already-indexed tenant-scoped tables); add a short-TTL Redis cache keyed by tenant_id only if profiling shows it's a hot path (mirrors the existing `action_center_cache.py`/60s user-cache pattern already in the codebase — reuse it rather than inventing a new caching approach if needed later).
- Config lookups are static in-memory data — zero runtime cost.
- New tables are tiny (≤1 row per tenant + a handful of interest/task rows) — no special indexing beyond the standard tenant_id + unique constraints already specified.

---

## Part J — Migration & Rollback Strategy

- **Migration**: one new additive Alembic revision (`0008`); no changes to existing tables' columns; safe backfill rule (Part C) ensures zero impact on existing tenants. Backend can ship and be deployed independently of the frontend (new router is purely additive).
- **Optional kill-switch**: wrap the DashboardShell redirect check in an existing `tenant_feature_flags` flag (reusing `feature_flags_service.py`, already in the codebase) for an instant rollback path without a deploy, if desired.
- **Rollback**: flip the flag (or revert the one frontend PR) to stop gating — existing dashboard behavior returns immediately. `alembic downgrade` cleanly drops the 3 new, isolated tables (nothing else has an FK into them). No destructive changes were made to `Tenant`, `User`, `TenantSettings`, or `UserSettings`.

---

## Part K — Implementation Roadmap

| Phase | Scope | Est. |
|---|---|---|
| 0 | Backend foundation: migration, models, config, recommendation engine (unit-testable in isolation), `onboarding_service`, routes, wired into `create_tenant_with_defaults()`. Fully testable via API before any frontend work. | 2-3 days |
| 1 | Frontend wizard Stages 1-2 (persona, company info), service/hook layer, DashboardShell gating swap, retire `team/page.tsx`. | 3-4 days |
| 2 | Stage 3 Quick Actions (reusing `connector-picker`/`connector-card`), query-param deep-link into `/dashboard/connections`, `/complete` wiring, Product Tour gate swap. | 2-3 days |
| 3 | Stage 4 Getting Started Hub (Overview page) + Stage 5 floating widget (global, DashboardShell-mounted), auto-hide logic, sidebar light-touch persona ordering. | 2-3 days |
| 4 | Cleanup: retire `ConnectionSetupPrompt`, repurpose/retire dead `SetupStep` type, update `docs/BACKEND_ARCHITECTURE.md`/`docs/FRONTEND_ARCHITECTURE.md`, test coverage pass, manual QA (resume-after-refresh, multi-admin, platform-admin exemption). | 1-2 days |
| 5 *(documented only, not built now)* | Live-wire mock dashboard widgets to real data; build the actual persona-weighted widget grid using the already-reserved `dashboard_priority` config field. | separate effort |

---

## Part L — Verification Plan

- **Backend**: `pytest backend/tests/test_onboarding.py` — tenant creation auto-creates the onboarding row; RLS blocks cross-tenant reads/writes; persona/company-info save + resume across requests; checklist computation reflects live connection-table changes; completion gate behavior.
- **Manual E2E** (dev servers via existing `run.sh`/`start-dev.sh`): self-service signup → confirm redirect to `/dashboard/onboarding` → walk all 3 stages → refresh mid-flow to confirm resume at the correct stage → connect a sandbox/test AWS credential → confirm the checklist item flips → confirm the floating widget appears and its auto-hide behavior once required tasks are done → log out/in again to confirm onboarding never reappears → log in as an invited non-admin teammate to confirm they're not gated into the wizard.
- **Regression**: log in as a pre-existing tenant/user created before this migration and confirm no onboarding redirect ever appears (backfill rule holds).
