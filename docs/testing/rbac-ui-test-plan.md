# RBAC UI Test Plan — Cost Management On-Prem

**Date**: 2026-05-21 (Updated: 2026-08-19)  
**Status**: Review  
**Epic**: [COST-7570](https://redhat.atlassian.net/browse/COST-7570) — CoP Authentication & Authorization Migration  
**Story**: [COST-7632](https://redhat.atlassian.net/browse/COST-7632) — RBAC UI  
**POC**: [COST-7654](https://redhat.atlassian.net/browse/COST-7654) — MFE in koku-ui-onprem  
**Parent test plan**: [FLPATH-3551](https://redhat.atlassian.net/browse/FLPATH-3551) / [COST-7571](https://redhat.atlassian.net/browse/COST-7571)

---

## Document Summary

**Total test cases**: 130+ (across 13 UI categories: A–K, M–N; performance mapped to existing COST-7643)

**Test coverage**:
- **7 strategy layers**: Static delivery → API → UI nav → CRUD → enforcement → security → accessibility
- **90+ functional test cases** (A–K): Auth, infra, navigation, Groups/Users/Roles/My User Access, enforcement, error handling, UX gaps
- **RBAC performance**: already covered by [COST-7643](https://redhat.atlassian.net/browse/COST-7643) (`tests/suites/performance/test_rbac_perf.py`); not duplicated here
- **8 accessibility tests** (M): WCAG 2.1 AA keyboard nav, screen readers, color contrast
- **8 security tests** (N): XSS, CSRF, clickjacking, URL injection, session fixation

**Key UI behaviors (on-prem)**:
- IAM in the **Global** sidebar as expandable **Identity and Access Management** with **Users / Roles / Groups** only (no IAM Overview in sidebar)
- Second IAM entry point: masthead **user menu** (`user@…`) with **My User Access** then **Logout** (Users/Roles/Groups are not in this menu)
- **My User Access** bundle cards (`openshift`, `settings` only) with expandable permission rows
- Users list default **Status: Active** filter; user detail by **username**; invalid-user error page
- Roles list **Create role** button; detail by **UUID** with permissions sub-table
- Groups list with **Members** column; group detail **Roles / Members** tabs; nested role drill-down
- Skeleton loading states on all IAM list pages during API fetch

**Key additions from senior QE review** (2026-06-03):
- State synchronization tests (D-11, E-07, H-08–H-12) for bidirectional IAM ↔ Cost integration
- Accessibility compliance (M-*) for WCAG 2.1 AA
- UI-layer security tests (N-*) for XSS/CSRF/clickjacking
- Risk assessment & mitigation matrix
- Test data management strategy with cleanup scripts
- Acceptance criteria → test case traceability
- Refined defect severity with escalation policy
- Automation roadmap with timelines and tooling decisions

---

## Scope

Federated IAM/RBAC UI embedded in `koku-ui-onprem` (Scalprum MFE), served from `/rbac/` static assets, IAM routes under `/iam/*`, API via same-origin `/api/rbac/` through Envoy gateway.

### Traceability

| Source | Relevance |
|--------|-----------|
| [COST-7570](https://redhat.atlassian.net/browse/COST-7570) | Auth/authz migration epic |
| [COST-7632](https://redhat.atlassian.net/browse/COST-7632) | RBAC UI story |
| [COST-7654](https://redhat.atlassian.net/browse/COST-7654) | MFE POC acceptance criteria |
| [COST-7589](https://redhat.atlassian.net/browse/COST-7589) | On-prem UX scope (stripped SaaS) |
| [FLPATH-3551](https://redhat.atlassian.net/browse/FLPATH-3551) / [COST-7571](https://redhat.atlassian.net/browse/COST-7571) | Parent test plans |
| [PR #175](https://github.com/insights-onprem/cost-onprem-chart/pull/175) | Nginx `location /rbac/` |
| [PR #173](https://github.com/insights-onprem/cost-onprem-chart/pull/173) | Gateway JWT + IAM API automation |
| Lab (ocp-edge122) | Routes, personas, groups/users observed live |

### IAM route map

| Surface | URL pattern | Notes |
|---------|-------------|-------|
| My User Access | `/iam/my-user-access?bundle={openshift\|settings}` | Entry via masthead **user menu**; not in IAM sidebar; **no RHEL bundle** on-prem |
| Users list | `/iam/user-access/users` | Default filter chip: `Status: Active` |
| User detail | `/iam/user-access/users/detail/{username}` | Username-based, not UUID |
| Invalid user | `/iam/user-access/users/detail/{username}` | Breadcrumb: `Users > Invalid user` |
| Roles list | `/iam/user-access/roles` | **Create role** button in toolbar |
| Role detail | `/iam/user-access/roles/detail/{uuid}` | Permissions sub-table + app filter |
| Groups list | `/iam/user-access/groups` | **Create group** button in toolbar |
| Group detail (roles) | `/iam/user-access/groups/detail/{uuid}/roles` | Tabs: **Roles**, **Members** |
| Group role detail | `/iam/user-access/groups/detail/{groupUuid}/roles/detail/{roleUuid}` | Nested drill-down from group |

### IAM sidebar structure (latest UI)

There are **two** shell entry points into IAM-related UI. They expose **different** surfaces:

| Entry point | Where | Opens |
|-------------|-------|-------|
| **Sidebar IAM expandable** | Global nav → **Identity and Access Management** | **Users**, **Roles**, **Groups** (admin/org IAM management) |
| **User menu (masthead)** | Upper-right control showing logged-in identity (e.g. `admin@cost-onprem-chart.test`) | **My User Access**, then **Logout** |

#### Sidebar — Identity and Access Management

IAM lives in the **Global** sidebar (`nav[aria-label="Global"]`) as a PatternFly **NavExpandable** labeled **Identity and Access Management**. Expanding it reveals a nested `ul.pf-v6-c-nav__list` (subnav) with:

- **Users** → `/iam/user-access/users`
- **Roles** → `/iam/user-access/roles`
- **Groups** → `/iam/user-access/groups`

**Not** in the IAM expandable: IAM Overview, My User Access.

#### User menu — masthead dropdown

The masthead user control displays the authenticated identity (username / email form such as `user@realm-or-domain`). Opening it shows:

1. **My User Access** → `/iam/my-user-access` (personal roles / bundles)
2. **Logout**

**Not** in the user menu: Users, Roles, Groups (those remain sidebar-only).

**Playwright note:** Primary Cost nav is `nav[aria-label="Global"] > ul.pf-v6-c-nav__list`. The IAM subnav is a second nested `ul.pf-v6-c-nav__list` under `section.pf-v6-c-nav__subnav` — avoid bare `ul.pf-v6-c-nav__list` selectors (strict-mode ambiguity).

---

## 1. Objectives

1. Verify the RBAC MFE loads reliably inside the Cost shell without a second hostname or re-authentication.
2. Verify all IAM surfaces (Overview, My User Access, Users, Roles, Groups) render and function against `/api/rbac/v1/`.
3. Verify on-prem UX constraints: no SaaS-only flows (e.g. Invite Users), IDP-managed users, group-centric permissions.
4. Verify RBAC enforcement is reflected in UI (personas see appropriate data/actions).
5. Provide regression coverage for MFE asset delivery (`/rbac/`), routing (`/iam/*`), and gateway auth.

---

## 2. Out of scope

- Full LDAP/AD federation setup (covered by [COST-7601](https://redhat.atlassian.net/browse/COST-7601) unless UI-specific).
- insights-rbac backend-only Django shell administration.
- Cost report data correctness (covered by existing Koku e2e; cross-check only where RBAC affects visibility).
- Playwright UI harness flakiness on lab hosts without system deps (track separately).
- Multi-cluster federation scenarios (future work).
- Mobile/tablet responsive UI testing (desktop browsers only).
- RBAC authorization performance (latency, cache, concurrency, multi-org, replica scaling, ingestion load) — covered by [COST-7643](https://redhat.atlassian.net/browse/COST-7643) / `tests/suites/performance/test_rbac_perf.py`.

---

## 3. Environment prerequisites

| Requirement | Detail |
|-------------|--------|
| **Cluster** | CoP deployed (`cost-onprem` ns), gateway + UI + Keycloak + insights-rbac |
| **Keycloak** | `deploy-rhbk.sh`; `roles` client scope on `cost-management-ui`; realm `kubernetes` |
| **UI image** | `koku-ui-onprem` with RBAC MFE baked in (e.g. jkilzi POC image) |
| **Chart** | PR #175 nginx `/rbac/` block applied |
| **DNS/hosts** | `cost-onprem-ui-cost-onprem.apps.<cluster>` → ingress IP |
| **Test users** | Personas from Section 4, mapped to Keycloak users per environment (see `tests/fixtures/rbac-seed.yaml`); usernames are **not fixed** across dev vs cluster. New users: follow **§13 User provisioning** |
| **RBAC seed data** | Groups: CI Test Admin, Default access, Gateway RBAC IAM Readers, RBAC Payment Team, RBAC Cluster Alpha Ops, RBAC Cost Admins (+ chart/e2e bootstrap) |
| **Wait policy** | **≥ 3–5 s** after IAM route change before asserting content (MFE lazy-load) |
| **Browser matrix** | Chrome 120+, Firefox 115+, Edge 120+ (primary testing on Chrome) |
| **Seed data version** | Documented in `tests/fixtures/rbac-seed.yaml` |
| **Cleanup script** | `./scripts/reset-rbac-test-data.sh` (removes TEST-* prefixed groups/roles) |

---

## 4. Test personas

| Persona | Expected IAM access | Expected Cost data scope |
|---------|---------------------|--------------------------|
| **admin** | Full IAM admin (create/edit groups, manage roles) | All test clusters/projects |
| **carol** | Cost Administrator group | All RBAC test clusters |
| **alice** | Payment team | `payment` project only |
| **bob** | Cluster Alpha ops | `cluster-alpha` only |
| **viewer** | Read-only / limited | Per seeded roles |
| **nobody-unassigned** | No cost/RBAC app access | Denied on cost reports |
| **rbac-iam-admin** | IAM read (principals/groups list) | No cost admin write |

### Persona vs. username conventions

Test steps use **personas** (roles/capabilities), not environment-specific Keycloak usernames.

- Reference personas by name from the table above (e.g. **admin**, **alice**, **viewer**).
- The UI header, profile menu, and `/users/detail/{username}` URLs must match the **actual username** of the logged-in persona for that run.
- Do **not** assume `dev-user` or any other fixed username unless the persona is explicitly named in the step.
- Map personas to Keycloak credentials per environment (chart bootstrap, `rbac-seed.yaml`, lab Keycloak); record the mapping in test run notes.
- Usernames vary by environment — map personas to Keycloak credentials in test run notes.

---

## 5. Test strategy layers

| Layer | Tooling | Purpose |
|-------|---------|---------|
| **L1 — Static/MFE delivery** | curl, browser DevTools | `/rbac/plugin-manifest.json`, chunks 200 |
| **L2 — API/gateway** | chart pytest `test_rbac_gateway.py`, PR #173 | JWT → `/api/rbac/v1/*` authz |
| **L3 — UI navigation & render** | Manual + Playwright | Shell ↔ IAM routing, sidebar |
| **L4 — IAM functional CRUD** | Manual + future UI automation | Groups/Roles/Users workflows |
| **L5 — RBAC enforcement E2E** | `test_rbac_access.py` gateway JWT tests | Persona isolation through UI/API |
| **L6 — Negative/security** | Manual + gateway security tests | Unauth, expired JWT, org_id boundaries, XSS/CSRF |
| **L7 — Accessibility** | Manual WCAG 2.1 AA testing + axe DevTools | Keyboard nav, screen readers, color contrast |

RBAC authorization performance is **not** a UI-plan layer. It is already automated in `tests/suites/performance/test_rbac_perf.py` (COST-7643). See Section L mapping.

---

## 6. Test cases

### A. Authentication & session (RBAC-UI-AUTH)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| A-01 | Unauthenticated UI redirect | Open UI URL logged out | 302 → Keycloak login; no IAM content without auth | P0 | Auto/M |
| A-02 | Single session across Cost ↔ IAM | Login as **admin** persona → Cost Overview → IAM → Groups → Cost Overview | No second login; same user in header as logged-in persona | P0 | M |
| A-03 | Logout invalidates IAM | Logout from UI → (1) browser back button, (2) UI breadcrumb, (3) direct URL `/iam/user-access/groups` | All three redirect to login; no cached IAM data | P1 | M |
| A-04 | Viewer vs admin header identity | Login as **viewer** / **admin** personas | Header shows correct username/email for each logged-in persona | P2 | M |
| A-05 | Session timeout handling | Wait for Keycloak session timeout → click IAM link | Redirect to login; graceful re-auth without data loss | P1 | M |

### B. MFE delivery & infrastructure (RBAC-UI-INFRA)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| B-01 | plugin-manifest served | `GET /rbac/plugin-manifest.json` (authenticated session or from pod) | 200; `name: insightsRbac`, `baseURL: /rbac/`, `loadScripts` includes `plugin-entry.js` | P0 | Auto |
| B-02 | MFE entry script loads | DevTools Network: load IAM page | `plugin-entry.js` + federated chunks 200, no 404 | P0 | M |
| B-03 | Nginx `/rbac/` alias | Verify ConfigMap `location /rbac/` | alias `/opt/app-root/src/rbac/`; sample bundle 200 from pod | P0 | Auto |
| B-04 | No second UI route required | Confirm only `cost-onprem-ui` route used for IAM | IAM works on same hostname (COST-7654 AC) | P0 | M |
| B-05 | API same origin | DevTools: IAM page XHR/fetch | Calls go to `/api/rbac/v1/...` (via UI proxy/gateway), not external SaaS; validate against OpenAPI schema | P0 | M |
| B-06 | RBAC API health | `GET /api/rbac/v1/status/` with valid JWT | 200 | P0 | Auto (PR #173) |
| B-07 | MFE load timing | Navigate to Groups; snapshot at 0s, 2s, 5s | Content visible by ≤5s under normal load; fail if >10s (p95); record observed time in run notes | P0 | M |
| B-08 | Browser resource consumption | DevTools Performance: load IAM → Groups → Users → Roles | Memory <500MB, CPU <80% sustained, no memory leaks on 10 navigation cycles | P2 | M |

### C. Navigation & routing (RBAC-UI-NAV)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| C-01 | IAM expandable label in Global nav | Login as **admin**; open Cost Overview; inspect Global sidebar | Expandable labeled **Identity and Access Management** is visible in `nav[aria-label="Global"]` (top-level item alongside Overview/Settings); **not** nested under Cost **Settings** | P0 | M/Auto |
| C-01a | Expand IAM section | Click **Identity and Access Management** expandable toggle | Section expands; nested subnav shows exactly **Users**, **Roles**, **Groups** (no Overview / My User Access in subnav) | P0 | M/Auto |
| C-01b | Collapse IAM section | With IAM expanded, click expandable toggle again | Subnav hides; Users/Roles/Groups links not visible; expandable still present | P1 | M/Auto |
| C-02 | Route: My User Access via user menu | Open masthead **user menu** → click **My User Access** (or deep-link `/iam/my-user-access`); wait 4s | URL `/iam/my-user-access*`; heading "My User Access"; **Org. Administrator** badge if applicable; exactly **two** bundle cards: OpenShift, Settings and User Access | P0 | M/Auto |
| C-02a | My User Access not in IAM expandable | Expand **Identity and Access Management** | Subnav does **not** include My User Access; item is only in the user menu (above Logout) | P0 | M/Auto |
| C-02b | Masthead shows logged-in identity | Login as **admin** (or any persona); inspect upper-right masthead | User control visible with authenticated identity text (e.g. `admin@cost-onprem-chart.test` / `user@…`); control is clickable | P0 | M/Auto |
| C-02c | User menu contents | Click masthead user control | Dropdown opens with exactly **My User Access** then **Logout** (in that order); no Users / Roles / Groups entries | P0 | M/Auto |
| C-02d | Users/Roles/Groups are sidebar-only | Open user menu; also expand IAM sidebar | Users / Roles / Groups appear **only** under Identity and Access Management expandable; **absent** from user menu | P0 | M/Auto |
| C-02e | Logout still available from user menu | Open user menu → confirm **Logout** | Logout item visible below My User Access; selecting it ends session (see A-03) | P0 | M/Auto |
| C-03 | Route: Groups via expandable | Expand IAM → click **Groups**; wait 4s | URL `/iam/user-access/groups`; table columns **Name**, **Roles**, **Members**, **Last modified**; **Create group** button; **Groups** nav link marked current | P0 | M/Auto |
| C-04 | Route: Users via expandable | Expand IAM → click **Users**; wait 4s | URL `/iam/user-access/users`; columns Org. Administrator, Username, Email, First name, Last name, Status; default **Status: Active** filter chip; **Users** nav link marked current | P0 | M/Auto |
| C-05 | Route: Roles via expandable | Expand IAM → click **Roles**; wait 4s | URL `/iam/user-access/roles`; columns Name, Description, Groups, Permissions, Last modified; **Create role** button; **Roles** nav link marked current | P0 | M/Auto |
| C-05a | IAM subnav persists across leaf pages | Expand IAM → Users → Roles → Groups | Expandable stays expanded; all three leaf links remain available without re-expanding | P1 | M/Auto |
| C-06 | Bundle: OpenShift | My User Access → select **OpenShift** card | URL `?bundle=openshift`; heading "Your OpenShift roles"; roles include Cost Administrator, Cost * Viewer Local Test | P0 | M |
| C-07 | Bundle: Settings | My User Access → select **Settings and User Access** card | URL `?bundle=settings`; heading "Your Settings and User Access roles"; roles include Sources administrator, User Access administrator | P0 | M |
| C-08 | No RHEL bundle (on-prem) | My User Access page; attempt `?bundle=rhel` | Only OpenShift and Settings cards shown; no RHEL card; unsupported bundle handled gracefully (redirect or empty state) | P0 | M |
| C-09 | Cost → IAM → Cost | Overview → expand IAM → Groups → OpenShift Costs | No nav freeze (max 2s delay); no JS console errors; shell responsive; Cost nav still usable | P0 | Auto (Playwright) |
| C-10 | Deep link Groups | Paste `/iam/user-access/groups` while logged in | Page loads with group table after wait; IAM expandable shows expanded with Groups current | P1 | M |
| C-10a | Deep link Users | Paste `/iam/user-access/users` while logged in | Users list loads; IAM expandable expanded; **Users** marked current | P1 | M |
| C-10b | Deep link Roles | Paste `/iam/user-access/roles` while logged in | Roles list loads; IAM expandable expanded; **Roles** marked current | P1 | M |
| C-11 | Expand role permissions | My User Access → expand a role row (e.g. Cost Cloud Viewer Local Test) | Nested table: Application, Resource type, Operation, Resource definitions | P0 | M |
| C-12 | Invalid IAM route | Navigate `/iam/nonexistent` | Graceful 404 or redirect, no white-screen crash | P2 | M |
| C-13 | Browser back/forward | Navigate Groups → Users → back → forward | Correct page state restored | P2 | M |
| C-14 | Primary vs IAM nav selectors | While on Overview with IAM expanded | Primary list (`nav[aria-label="Global"] > ul.pf-v6-c-nav__list`) and IAM subnav (`section.pf-v6-c-nav__subnav > ul.pf-v6-c-nav__list`) are distinct; automation must not use bare `ul.pf-v6-c-nav__list` | P0 | Auto (Playwright) |

### D. Groups (RBAC-UI-GRP)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| D-01 | List groups | Admin → Groups | Shows **Cost Admin Default** (6 roles, Members: All org admins) and **Default access** (7 roles, Members: All); Name column has info icon | P0 | M |
| D-02 | Group detail — Roles tab | Click **Cost Admin Default** | URL `/iam/user-access/groups/detail/{uuid}/roles`; breadcrumbs `Groups > Cost Admin Default`; description visible; **Roles** tab active | P0 | M |
| D-03 | Group detail — Members tab | Group detail → click **Members** tab | URL `/iam/user-access/groups/detail/{uuid}/members`; member list loads | P0 | M |
| D-04 | Group role drill-down | Group Roles tab → click **Cost Administrator** | URL `.../groups/detail/{uuid}/roles/detail/{roleUuid}`; permissions table: Application, Resource type, Operation, Resource definitions, Last modified | P0 | M |
| D-05 | Create group | Admin → **Create group** → name + description → save | Success toast; group appears in list | P0 | M |
| D-06 | Edit group | Edit existing test group description | Persists after refresh | P1 | M |
| D-07 | Add member to group | Add **alice** persona to a test group via Members tab | Member count updates; alice's effective permissions change | P0 | M |
| D-08 | Remove member | Remove member from test group | Reflected in list and in Cost report scope for that user | P0 | M |
| D-09 | Assign role to group | Attach Cost role to group via UI | Role count updates on list page; `/api/rbac/v1/access/` reflects change | P0 | M |
| D-10 | Delete group (non-system) | Delete a user-created test group | Removed from list; API 404 | P1 | M |
| D-11 | Platform default group | View **Default access** | Visible; Members = "All"; destructive actions restricted or warned | P1 | M |
| D-12 | Non-admin denied create | Login as **viewer** persona → Groups | Create group hidden or POST returns 403 | P0 | M |
| D-13 | Filter groups by name | Enter name in **Filter by name** | Table narrows to matching groups | P1 | M |
| D-14 | Roles count link | Click role count (e.g. "6") on group row | Navigates to group Roles tab | P1 | M |

### E. Users (RBAC-UI-USR)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| E-01 | List users | Admin → Users | Table with columns: Org. Administrator, Username, Email, First name, Last name, Status; skeleton loaders during fetch | P0 | M |
| E-02 | Default Active filter | Load Users page | **Status: Active** filter chip applied by default; **Clear filters** link visible | P0 | M |
| E-03 | User detail | Click any username link from the Users list | URL `/iam/user-access/users/detail/{username}`; detail page with group memberships for that user | P0 | M |
| E-04 | Invalid user detail | Navigate `/iam/user-access/users/detail/{nonexistent-username}` | Breadcrumb `Users > Invalid user`; heading "User not found"; message "User with username {username} does not exist."; **Back to previous page** button | P0 | M |
| E-05 | No Invite Users (on-prem) | Scan Users page actions | **No** "Invite user" button; copy directs to external **user management list** link | P0 | M |
| E-06 | IDP-managed users note | Read Users page intro copy | "These are all of the users in your Red Hat organization… go to your user management list" | P1 | M |
| E-07 | Filter by username | Use **Username** dropdown + "Filter by username" search | Table narrows to matching users | P1 | M |
| E-08 | Org Administrator column | Inspect Org. Administrator column | Shows checkmark or **X No** per user | P1 | M |
| E-09 | Status badges | Inspect Status column | **Active** / **Inactive** badge per user; filter chip matches displayed rows | P1 | M |
| E-10 | Service account visibility | Clear filters → locate a service account from seed data | Listed; appropriate read-only UI | P2 | M |
| E-11 | Provision new IDP user end-to-end | Follow **§13 User provisioning (Keycloak → COS IAM)** to create `<username>` with scoped access (e.g. add to an existing persona group such as Payment Team). Complete all three steps including first CoP login | User exists in Keycloak `kubernetes` realm; listed as member on assigned IAM group; appears in IAM **Users** after first CoP login; cost scope matches group role | P0 | M |

### F. Roles (RBAC-UI-ROL)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| F-01 | List roles | Admin → Roles | Columns: Name, Description, **Groups**, **Permissions**, Last modified; skeleton loaders during fetch | P0 | M |
| F-02 | Create role button | Admin → Roles | **Create role** primary button visible in toolbar | P0 | M |
| F-03 | Role detail | Click any role from the list | URL `/iam/user-access/roles/detail/{uuid}`; breadcrumb `Roles > {name}`; description; permissions table | P0 | M |
| F-04 | Role permissions table | Role detail page | Columns: Application, Resource type, Operation, Last modified; filter by **Applications** | P0 | M |
| F-05 | Cost Administrator role | Open **Cost Administrator** detail | Application `cost-management`; Resource type `*`; Operation `*` | P0 | M |
| F-06 | Custom role create | **Create role** → single permission → save | Appears in list; assignable to group | P1 | M |
| F-07 | Read-only user | **viewer** persona → Roles | List allowed; Create role hidden or 403 | P1 | M |
| F-08 | Filter roles by name | **Filter by name** search | Table narrows to matching roles | P1 | M |

### G. My User Access (RBAC-UI-MUA)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| G-01 | Org Administrator badge | Login as **admin** persona (org admin) → My User Access | Purple **Org. Administrator** badge next to page title when persona has org-admin flag; absent for non-org-admin personas | P0 | M |
| G-02 | OpenShift bundle roles | `?bundle=openshift` | Cost Administrator, Cost Cloud Viewer Local Test, Cost OpenShift Viewer Local Test | P0 | M |
| G-03 | Settings bundle roles | `?bundle=settings` | Sources administrator, User Access administrator, User Access principal viewer | P0 | M |
| G-04 | Bundle card UI | Inspect bundle selector cards | Exactly **two** cards: **OpenShift** (clusters/advisor/subscriptions/cost management) and **Settings and User Access** (rbac/sources); **no RHEL** | P0 | M |
| G-05 | Expand permission row | Expand a role row | Sub-table: Application, Resource type, Operation, Resource definitions (e.g. sources / * / *) | P0 | M |
| G-06 | Role name filter | **Filter by role name** search | Table narrows within active bundle | P1 | M |
| G-07 | Skeleton loading | Switch bundle while throttling network | Skeleton rows appear during fetch; resolve to data within 5s | P1 | M |
| G-08 | Alice scoped roles | **alice** persona → My User Access | Only roles tied to payment team / limited scope | P0 | M |

### H. RBAC enforcement through UI (RBAC-UI-ENF)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| H-01 | Alice cost scope | **alice** persona → OpenShift costs | Only **payment** project data | P0 | Auto (e2e gateway) |
| H-02 | Bob cost scope | **bob** persona → OpenShift costs | Only **cluster-alpha** | P0 | Auto |
| H-03 | Carol full scope | **carol** persona → OpenShift costs | All three RBAC test clusters | P0 | Auto |
| H-04 | Nobody denied cost | **nobody-unassigned** persona → costs via gateway/UI | 403/424 or empty denied state | P0 | Auto |
| H-05 | IAM reader list principals | **rbac-iam-admin** persona → Groups/Users | Can list; cannot create group (403) | P0 | Auto (PR #173) |
| H-06 | Permission change propagation | Remove **alice** persona from Payment Team in UI → alice refreshes costs | Payment data disappears within cache TTL (300s documented); verify timestamp | P1 | M |
| H-07 | Admin sees all IAM | **admin** persona → Users/Groups | Full list counts match API | P1 | M |
| H-08 | Role assignment takes effect immediately | Create new group → assign Cost role → add user → user login | Cost data scope reflects new role without logout/login | P0 | M |
| H-09 | End-to-end access revocation | User removed from group in IAM → Cost API call to previously accessible cluster | Returns 403 or empty state for revoked cluster; audit log reflects RBAC event | P0 | Auto |
| H-10 | New group workflow E2E | Admin creates group → assigns Cost role → adds member (member must exist in Keycloak per **§13**) → member logs in | Cost reports show correct data scope on first login | P0 | M |
| H-11 | Keycloak sync to RBAC | Delete user in Keycloak → IAM Users list | User marked deleted/inactive within documented sync interval (specify TTL) | P1 | M |
| H-12 | Nested group permissions (if supported) | User in nested groups → Cost reports | Verify permission inheritance chain | P1 | M |

### I. API / gateway alignment (RBAC-UI-API)

Map to existing `test_rbac_gateway.py` / PR #173 (label **automated** in FLPATH-3551):

| ID | UI correlate | Automated test |
|----|--------------|----------------|
| I-01 | Unauthenticated IAM API | `test_gateway_rbac_*_unauthenticated_returns_401` |
| I-02 | Unassigned user | `test_gateway_*_user_without_rbac_returns_403` |
| I-03 | IAM reader principals | `test_gateway_rbac_principals_iam_reader_returns_200` |
| I-04 | IAM reader write denied | `test_gateway_rbac_groups_post_iam_reader_forbidden` |
| I-05 | JWT expiry | `test_expired_jwt_rejected` |
| I-06 | Malicious org_id | `test_org_id_tenant_isolation_boundary_cases` |
| I-07 | Revocation | `test_permission_revocation_honored_after_cache_clear` |
| I-08 | Concurrent sessions | `test_concurrent_jwt_sessions_no_resource_exhaustion` |

See also: `tests/suites/auth/RBAC_SECURITY_TESTS.md`

### J. Negative, error handling & resilience (RBAC-UI-NEG)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| J-01 | RBAC API down | Scale rbac-api to 0 → open Groups | Error state in UI (not infinite spinner); fail closed | P0 | Auto (fail-closed test) |
| J-02 | Missing `/rbac/` nginx block | Remove location /rbac/ → reload IAM | Clear error in console; graceful degradation message | P1 | M |
| J-03 | Stale session | Expire Keycloak session → click IAM | Redirect login, no partial stale tables | P1 | M |
| J-04 | 403 on write | **viewer** persona attempts create group | Inline error or disabled control; no silent success | P1 | M |
| J-05 | Large list pagination | Groups/Users with full seed set (1000+ users) | Pagination works; page load <5s; memory <500MB; no browser hang | P2 | M |
| J-06 | Network interruption during create | Create group → kill network mid-request → restore | Retry logic or clear error message; no duplicate groups | P1 | M |
| J-07 | Keycloak restart mid-session | Active IAM session → restart Keycloak pod → continue workflow | Graceful re-auth; no data loss in form fields | P2 | M |
| J-08 | Gateway timeout on slow query | Trigger slow RBAC query (e.g., 1000+ users) → gateway 504 | User-friendly timeout message; no infinite spinner | P1 | M |
| J-09 | Partial API failure | RBAC API returns 500 on `/users/` but 200 on `/groups/` | Graceful degradation; error shown for Users tab only | P2 | M |
| J-10 | Skeleton loading states | Navigate Users / Roles / Groups / switch MUA bundle | PatternFly skeleton rows shown during fetch; no permanent blank pane | P0 | M |
| J-11 | Blank MFE on fast nav | Navigate to `/iam/my-user-access` without wait | Content may be blank briefly; resolves within 5s (document as timing risk) | P1 | M |
| J-12 | Invalid user deep link | Direct URL to `/iam/user-access/users/detail/{bad}` | Error page with **Back to previous page**; no uncaught exception | P0 | M |

### K. UX / on-prem SaaS parity gaps (RBAC-UI-UX)

Per [COST-7589](https://redhat.atlassian.net/browse/COST-7589) / [COST-7632](https://redhat.atlassian.net/browse/COST-7632):

| ID | Check | Expected |
|----|-------|----------|
| K-01 | No console.redhat.com links | No external SaaS dependencies in network tab (except documented user management list link) |
| K-02 | No Invite Users | Absent |
| K-03 | On-prem bundle scope | OpenShift + Settings/User Access bundles only (no extraneous HCC apps in MUA) |
| K-04 | Group-centric assignment | Primary workflow is group ↔ role ↔ members (Groups list shows role counts + member scope) |
| K-05 | LDAP/AD user sync | Users provisioned in Keycloak/IDP per **§13** appear in IAM after first CoP login; **user management list** link for IDP admin |
| K-06 | Local test role naming | Roles suffixed "Local Test" acceptable in dev; verify cost-relevant roles present in prod builds |

### L. Performance & load testing — covered by COST-7643

RBAC authorization performance is already implemented and validated in `tests/suites/performance/test_rbac_perf.py` ([COST-7643](https://redhat.atlassian.net/browse/COST-7643)). Do **not** duplicate these cases in the UI suite.

| Existing ID | Automated test | Coverage |
|-------------|----------------|----------|
| PERF-RBAC-001 | `test_perf_rbac_001_baseline_isolation` | RBAC share of Koku API latency |
| PERF-RBAC-002 | `test_perf_rbac_002_cache_effectiveness` | Cold vs warm Valkey cache |
| PERF-RBAC-003 | `test_perf_rbac_003_concurrent_auth` | Concurrent authorization load |
| PERF-RBAC-004 | `test_perf_rbac_004_multi_org_scaling` | Multi-org scaling |
| PERF-RBAC-005 | `test_perf_rbac_005_replica_scaling` | 1→2→3 RBAC API replicas under load |
| PERF-RBAC-006 | `test_perf_rbac_006_under_ingestion` | RBAC latency while ingestion is active |

```bash
./scripts/deploy-test-cost-onprem.sh --perf-only --perf-profile medium --perf-suite rbac
```

UI-only smoke timing remains in **B-07** (MFE visible ≤5s) and **B-08** (browser memory/CPU). Large-list pagination resilience remains in **J-05**. Results: `docs/performance/FINDINGS.md` (FINDING-037) and `docs/performance/performance-testing-plan.md`.

### M. Accessibility (RBAC-UI-A11Y)

WCAG 2.1 AA compliance testing:

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| M-01 | Keyboard navigation - Groups | Tab through Groups page; Enter to open detail; Tab to actions | All interactive elements reachable; focus visible; logical tab order | P0 | M |
| M-02 | Keyboard navigation - Forms | Create group form → Tab through fields → Enter to submit | Form submittable without mouse; error focus management | P0 | M |
| M-03 | Screen reader - JAWS/NVDA | Navigate Groups/Users with screen reader | Headings announced; table structure conveyed; action buttons labeled | P0 | M |
| M-04 | Color contrast | Inspect role badges, status indicators, error messages | 4.5:1 ratio for normal text; 3:1 for large text/graphics | P0 | M |
| M-05 | Focus indicators | Tab through all IAM pages | Visible focus ring (not default browser; 2px+ outline) | P0 | M |
| M-06 | Alt text for icons | Inspect icon-only buttons (edit, delete, etc.) | Aria-labels or screen reader text present | P1 | M |
| M-07 | Skip navigation | Load IAM page with keyboard | "Skip to main content" link functional | P2 | M |
| M-08 | Form error announcements | Submit invalid form | Errors announced to screen reader; focus moves to first error | P1 | M |

### N. Security testing - UI layer (RBAC-UI-SEC)

| ID | Title | Steps | Expected | Pri | Type |
|----|-------|-------|----------|-----|------|
| N-01 | XSS in group name | Create group with name `<script>alert('xss')</script>` | Name displayed as plain text; script not executed | P0 | M |
| N-02 | XSS in group description | Create group with description containing HTML/JS | Sanitized display; no script execution | P0 | M |
| N-03 | CSRF token validation | Attempt POST `/api/rbac/v1/groups/` from external page | Request blocked; CSRF token required | P0 | Auto |
| N-04 | Clickjacking protection | Attempt to embed IAM in iframe from external domain | CSP or X-Frame-Options blocks embedding | P0 | Auto |
| N-05 | URL parameter injection | Navigate `/iam/user-access/groups?org_id=malicious` | Org_id from JWT only; URL param ignored/sanitized | P0 | Auto |
| N-06 | SQL injection in search | Search users with `'; DROP TABLE users; --` | Parameterized queries; no SQL execution | P1 | M |
| N-07 | Path traversal in API | Attempt `GET /api/rbac/v1/../../etc/passwd` | 404 or 400; no directory traversal | P1 | Auto |
| N-08 | Session fixation | Attempt to set session cookie before login | Session regenerated post-login; old cookie invalid | P2 | M |

---

## 7. Test execution matrix (by phase)

| Phase | Focus | Entry criteria | Exit criteria |
|-------|-------|----------------|---------------|
| **Phase 0 — Smoke** | A-01–A-02, B-01–B-07, C-01–C-01b, C-02–C-02e, C-03–C-07, C-14 | Cluster + UI + Keycloak up | All P0 infra/nav/auth pass |
| **Phase 1 — Functional IAM** | D, E, F, G (admin workflows) | Phase 0 pass | CRUD cycles complete without API errors |
| **Phase 2 — Persona enforcement** | H (all personas), I (gateway alignment) | RBAC seed + Keycloak users | All personas match e2e expectations; gateway tests green |
| **Phase 3 — Regression & Automation** | Full automated suite + Playwright | PR #173, #175 merged; automation setup complete | Chart pytest green; Playwright suite passing |
| **Phase 4 — Quality & Hardening** | J (negative), M (a11y), N (security) | Phase 3 pass | Accessibility spot-check pass; no S1/S2 defects. RBAC perf already covered by COST-7643 |
| **Phase 5 — Release Readiness** | K (UX gaps), sign-off checklist | Phase 4 pass; all P0/P1 tests executed | Sign-off checklist complete; known limitations documented; automation roadmap approved |

---

## 8. Automation roadmap

### Existing automation to run before each UI test session

```bash
# Gateway/API (validates backend for UI)
pytest tests/suites/auth/test_rbac_gateway.py -v
pytest tests/suites/e2e/test_rbac_access.py -k "password_grant_via_gateway" -v

# Full chart (regression)
export PYTHON=/usr/bin/python3.12
./scripts/deploy-test-cost-onprem.sh --skip-deploy --verbose
```

---

## 9. Defect severity guidelines

| Severity | Criteria | Examples |
|----------|----------|----------|
| **S1 — Critical** | IAM completely unusable; auth loop; data leak across tenants | IAM blank after 10s; cannot list groups for any user; alice sees bob's data |
| **S2 — High (infra)** | Infrastructure failure blocking IAM | 404 on `/rbac/plugin-entry.js`; gateway 502 on all `/api/rbac/` calls |
| **S2 — High (functional)** | Core CRUD broken for admins | Cannot create group (500 error); wrong persona scope in Cost data |
| **S3 — Medium** | Workaround exists; non-admin affected | Pagination breaks on page 10; filter reset on navigation; SaaS copy leftover |
| **S4 — Low** | Cosmetic; no functional impact | Tooltip typo; minor CSS misalignment; console warning (not error) |

**Escalation policy**: S1/S2 defects block release sign-off. S3 requires product owner triage. S4 tracked in backlog.

---

## 10. Sign-off checklist (COST-7654 acceptance)

### Functional Requirements
- [ ] RBAC remote builds; `plugin-manifest.json` under `/rbac/`
- [ ] koku-ui-onprem loads MFE via Scalprum; IAM reachable from Global nav **Identity and Access Management** expandable (Users / Roles / Groups)
- [ ] Masthead user menu shows logged-in identity; menu contains **My User Access** then **Logout** only
- [ ] My User Access reachable from user menu and `/iam/my-user-access` with bundle cards (openshift, settings)
- [ ] Users, Roles, Groups pages load with correct columns, filters, and Create actions
- [ ] Authenticated user completes IAM flows against `/api/rbac/` (same origin)
- [ ] `build:onprem` / chart deploy includes RBAC assets (no regression)
- [ ] No second UI hostname required
- [ ] No re-auth when switching Cost ↔ IAM
- [ ] Persona isolation verified (alice/bob/carol/nobody)
- [ ] On-prem UX constraints verified (K-01–K-05)

### Security & Infrastructure
- [ ] Gateway security tests green (PR #173)
- [ ] Nginx `/rbac/` location deployed (PR #175)
- [ ] XSS/CSRF/clickjacking tests passed (N-01–N-08)
- [ ] No S1/S2 defects open

### Performance & Quality
- [ ] MFE load time <5s (p95); recorded in run notes (B-07)
- [ ] All P0 test cases passed
- [ ] RBAC authorization performance covered by COST-7643 (`test_rbac_perf.py`); not re-run as part of this UI plan
- [ ] Accessibility spot-check passed (keyboard nav, screen reader on Groups/Users)

### Documentation & Automation
- [ ] Test data cleanup script validated
- [ ] Known limitations documented (Section 16)
- [ ] Automation roadmap approved (Section 18)
- [ ] AC → test case mapping complete (Section 15)

---

## 11. Recommended Jira structure

Under [FLPATH-3551](https://redhat.atlassian.net/browse/FLPATH-3551) / [COST-7571](https://redhat.atlassian.net/browse/COST-7571), add epics or labels:

- `rbac-ui-auth` (A-*) — Authentication & session management
- `rbac-ui-infra` (B-*) — MFE delivery & infrastructure
- `rbac-ui-nav` (C-*) — Navigation & routing
- `rbac-ui-iam-crud` (D–G) — IAM functional CRUD (Groups, Users, Roles, My User Access)
- `rbac-ui-enforcement` (H-*) — RBAC enforcement end-to-end
- `rbac-ui-negative` (J-*) — Error handling & resilience
- `rbac-ui-ux-onprem` (K-*) — On-prem UX constraints
- `rbac-ui-accessibility` (M-*) — WCAG 2.1 AA compliance
- `rbac-ui-security` (N-*) — UI-layer security testing

Gateway API cases (I-*) are largely covered by FLPATH-4308–4329 already; reference from UI test cases where overlap exists (e.g., H-05 → I-03). RBAC performance (former L-*) is covered by COST-7643; do not file duplicate UI perf tickets.

---

## 12. Performance benchmarks — covered by COST-7643

Do not maintain a separate RBAC UI performance matrix in this plan. Authorization latency, cache effectiveness, concurrent load, multi-org scaling, replica scaling, and ingestion-under-load are already measured as **PERF-RBAC-001 through PERF-RBAC-006** in `tests/suites/performance/test_rbac_perf.py` ([COST-7643](https://redhat.atlassian.net/browse/COST-7643)).

- Run: `./scripts/deploy-test-cost-onprem.sh --perf-only --perf-profile medium --perf-suite rbac`
- Results: `docs/performance/FINDINGS.md` (FINDING-037)
- Plan: `docs/performance/performance-testing-plan.md`

UI smoke checks that stay in this plan: **B-07** (MFE load ≤5s), **B-08** (browser resource consumption), **J-05** (large-list pagination).

---

## 13. Test data management

### Seed Data Location
- **Primary**: `tests/fixtures/rbac-seed.yaml` (versioned)
- **Chart bootstrap**: `cost-onprem/charts/insights-rbac/templates/seed-job.yaml`
- **E2E fixtures**: `tests/suites/e2e/fixtures/rbac_personas.py`

### Cleanup Policy
- **Prefix convention**: All test-created groups/roles must use `TEST-*` prefix
- **Cleanup script**: `./scripts/reset-rbac-test-data.sh`
  - Deletes `TEST-*` groups/roles
  - Preserves seeded baseline data (CI Test Admin, Payment Team, etc.)
  - Safe for CI and manual runs

### Reset Procedure
```bash
# After test run or before fresh test cycle:
./scripts/reset-rbac-test-data.sh --namespace cost-onprem

# Verify clean state:
kubectl exec -n cost-onprem deploy/insights-rbac -- \
  python manage.py shell -c "from management.models import Group; print(Group.objects.filter(name__startswith='TEST-').count())"
```

### Seed Data Versioning
- **Version tag**: Include in `rbac-seed.yaml` header (e.g., `# Version: 2026-Q2-v1`)
- **Change log**: Document in `docs/testing/rbac-seed-changelog.md` when adding/modifying personas

### User provisioning (Keycloak → COS IAM)

Reference procedure for manual test setup and test cases **E-11**, **H-08**, **H-10**, **K-05**.

Keycloak and COS IAM are **separate systems**:

| System | Purpose |
|--------|---------|
| **Keycloak** (`kubernetes` realm) | Authentication — who can log in |
| **insights-rbac / COS IAM** | Authorization — what they can access in CoP |

Creating a user in Keycloak alone does **not** grant CoP permissions unless the user is also assigned IAM group membership (or carries the `org-admin` realm role for full org admin access).

#### Step 1 — Create user in Keycloak

1. Open the Keycloak **Admin Console** at `/admin` on the cluster Keycloak route.
2. Log in with **master** realm admin credentials (from the `keycloak-initial-admin` secret). This is **not** the CoP application login.
3. Switch the realm dropdown to **`kubernetes`** (CoP authenticates against this realm, not `master`).
4. Navigate to **Users** → **Create user**.
5. Set **Username**, **Email**; enable the user.
6. On the **Attributes** tab, add deployment-required attributes:
   - `org_id` — target organization ID (must match the insights-rbac tenant)
   - `account_number` — customer account identifier
7. On the **Credentials** tab, set a password; disable **Temporary** if the password should persist across logins.

**Keycloak Admin UI notes:**

- The Users list defaults to **10 users per page**; additional users may appear on page 2 or later (alphabetical sort).
- Use the search box with the username (or a distinctive prefix) rather than relying on deep links — Admin Console v2 deep links may land on the list view only.

**Optional — full org admin:** **Role mapping** → assign realm role **`org-admin`**. Envoy maps this to `is_org_admin` in the identity header, triggering insights-rbac `admin_default` group access. Skip for scoped personas (e.g. payment-team, cluster-scoped users).

#### Step 2 — Grant permissions in COS IAM

1. Log into the **CoP UI** as an **admin** persona.
2. Navigate to **Identity and Access Management → Groups** (expand Global IAM nav if needed).
3. Open the target group (or create a group and assign a Cost role first — see **H-10**).
4. On the **Members** tab, add the member using the **exact Keycloak username**.
5. Save.

#### Step 3 — First CoP login (activates IAM Users list)

1. Log out; log into the **CoP UI** as the new user (first authentication).
2. As admin, verify **IAM → Users**: the user appears (refresh if needed).
3. Verify **IAM → Groups → {group} → Members** lists the user.
4. Verify cost data scope matches the assigned group role (compare against persona patterns in **H-01**–**H-03**).

**Sync behavior:** The IAM **Users** list is backed by insights-rbac, not Keycloak directly. A user created in Keycloak may **not** appear in IAM **Users** until their **first CoP login**. Group membership can be assigned before first login (via IAM UI, API, or Django shell).

**Cache:** Permission changes may take up to the RBAC cache TTL (~300s) to propagate; see **H-06** for cache-aware validation.

#### Verification checklist

| Check | Where |
|-------|-------|
| User exists and is enabled | Keycloak Admin → `kubernetes` realm → **Users** |
| Required attributes present | Keycloak user **Attributes** (`org_id`, `account_number`) |
| Group membership assigned | CoP IAM → **Groups** → **Members** |
| User visible in IAM | CoP IAM → **Users** (after first CoP login) |
| Permissions active | CoP cost reports or `/api/rbac/v1/access/?application=cost-management` |

See also: `docs/operations/rbac-setup.md` (User and Group Management).

---

## 14. Test risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| MFE lazy-load timing variability on lab hosts | High | Medium | Use 10s max wait + network idle detection in automated tests |
| Keycloak federation flakiness during LDAP sync | Medium | High | Isolated test realm; snapshot/restore scripts; document sync intervals |
| RBAC cache stale reads cause false negatives | Medium | Medium | Document cache TTL (300s); add forced cache-clear tests; log timestamps |
| Browser compatibility issues (Firefox/Edge) | Low | Medium | Test on Chrome 120+, Firefox 115+, Edge 120+ per browser matrix |
| Concurrent admin edits cause race conditions | Medium | High | Test D-14 explicitly; document conflict resolution strategy |
| Large dataset pagination breaks on 1000+ items | Low | High | Manual/UI check J-05; monitor prod for dataset growth |
| Playwright missing system deps on lab | High | Low | Containerized test runner or manual fallback; document in prerequisites |
| Test data pollution between runs | Medium | Medium | Mandatory cleanup script in CI; `TEST-*` prefix enforcement |

**Mitigation tracking**: Link high-impact risks to Jira blockers; review mitigation effectiveness in retrospectives.

---

## 15. Acceptance criteria → test case mapping

Maps COST-7654 POC acceptance criteria to test case coverage:

| AC # | Acceptance Criteria | Test Cases | Status |
|------|---------------------|------------|--------|
| AC-1 | RBAC remote builds with plugin-manifest under `/rbac/` | B-01, B-03 | ✓ |
| AC-2 | koku-ui-onprem loads MFE via Scalprum | B-02, C-01–C-08, G-01–G-07 | ✓ |
| AC-3 | IAM reachable from shell navigation | C-01, C-01a–C-01b, C-02–C-02e, C-03–C-05a, C-09, C-14 | ✓ |
| AC-4 | Authenticated user completes IAM flows against `/api/rbac/` | B-05, D-05, E-03, F-03, F-06 | ✓ |
| AC-5 | No second UI hostname required | B-04 | ✓ |
| AC-6 | No re-auth when switching Cost ↔ IAM | A-02 | ✓ |
| AC-7 | Persona isolation verified | H-01–H-04, H-09–H-10, E-11 | ✓ |
| AC-8 | On-prem UX constraints (no Invite Users) | K-01–K-05 | ✓ |

**Coverage gap analysis**: All ACs mapped to ≥1 P0/P1 test case. No gaps identified.

---

## 16. Known limitations & workarounds

| Issue | Impact | Workaround | Tracking |
|-------|--------|-----------|----------|
| Playwright deps missing on lab hosts | Cannot run headless UI tests | Use containerized Playwright or manual tests | FLPATH-XXXX |
| 3–5s MFE lazy-load delay | False negatives if wait <5s | Hard-code 5s wait in all IAM nav tests (C-*, D-*) | By design |
| RBAC cache TTL 300s | Permission changes delayed | Document in H-06; add cache-clear step for immediate validation | COST-7XXX |
| Keycloak Admin Users pagination (10/page) | New users not visible on first page | Use search or navigate to page 2; see §13 | By design |
| IAM Users list lags Keycloak | User in Keycloak but not in IAM **Users** | First CoP login required; see §13 Step 3 | By design |
| No API for bulk user creation | Cannot easily seed 1000+ users | Use Django shell script in `tests/utils/seed_rbac_users.py` | Future enhancement |
| Gateway JWT expiry not synchronized with UI session | UI shows logged in but API returns 401 | Refresh token interceptor in UI; test in J-03 | COST-7XXX |

**Future improvements**: Track in backlog; revisit quarterly.

---

## 17. Defect severity guidelines (refined)

| Severity | Criteria | Examples |
|----------|----------|----------|
| **S1 — Critical** | IAM completely unusable; auth loop; data leak across tenants | IAM blank after 10s; cannot list groups for any user; alice sees bob's data |
| **S2 — High (infra)** | Infrastructure failure blocking IAM | 404 on `/rbac/plugin-entry.js`; gateway 502 on all `/api/rbac/` calls |
| **S2 — High (functional)** | Core CRUD broken for admins | Cannot create group (500 error); wrong persona scope in Cost data |
| **S3 — Medium** | Workaround exists; non-admin affected | Pagination breaks on page 10; filter reset on navigation; SaaS copy leftover |
| **S4 — Low** | Cosmetic; no functional impact | Tooltip typo; minor CSS misalignment; console warning (not error) |

**Escalation**: S1/S2 block release sign-off. S3 requires product owner triage.

---

## 18. Automation roadmap updates

| Priority | Gap | Recommendation | Target Sprint | Owner |
|----------|-----|----------------|---------------|-------|
| **P0** | No Playwright in chart CI for IAM | Implement Playwright suite in `tests/suites/ui/test_rbac_iam.py` | 2026-Q3 Sprint 1 | QE Team |
| **P0** | UI tests need MFE load wait fixture | Add `wait_for_iam_content()` fixture with network idle detection; parameterize persona credentials (no hardcoded usernames) | 2026-Q3 Sprint 1 | QE Team |
| **P1** | Groups CRUD automation | API-level tests exist; add UI smoke for D-03, D-08 | 2026-Q3 Sprint 2 | QE Team |
| **P1** | Map Jira TCs to FLPATH-3551 | Extend FLPATH-3551 with UI-specific cases (beyond FLPATH-4308–4329) | 2026-Q3 Sprint 2 | QE Lead |
| **P2** | Visual regression (optional) | Storybook parity checks (referenced in COST-7654) | 2026-Q4 | Future |
| **P2** | Accessibility automation | Integrate axe-core into Playwright suite for M-01–M-08 | 2026-Q4 | Future |

### Tooling Decision Matrix

| Tool | Pros | Cons | Decision |
|------|------|------|----------|
| **Playwright** | Faster; better debugging; modern API; multi-browser support | New toolchain for team | **Selected** for all phases |
| **Cypress** | PatternFly community examples | Slower; limited multi-browser | Not selected |
| **pytest + Selenium** | Existing chart CI integration | Flaky on lab hosts; outdated | Manual fallback only |

### CI Integration Plan
1. **Sprint 1**: Add Playwright to chart CI as optional job (manual trigger)
2. **Sprint 2**: Automate on PR for `/cost-onprem/charts/insights-rbac/**` changes
3. **Sprint 3**: Add to nightly regression suite

---

## Related documentation

| Document | Path | Relevance |
|----------|------|-----------|
| RBAC gateway automated tests | `tests/suites/auth/RBAC_SECURITY_TESTS.md` | Backend test coverage (Section I mapping) |
| RBAC setup operations | `docs/operations/rbac-setup.md` | Environment setup for testing |
| Gateway test module | `tests/suites/auth/test_rbac_gateway.py` | Automated backend prerequisites |
| E2E persona tests | `tests/suites/e2e/test_rbac_access.py` | Persona validation (H-* tests) |
| RBAC authorization performance | `tests/suites/performance/test_rbac_perf.py` | COST-7643 PERF-RBAC-001–006 (Section L / §12 mapping) |
| Performance testing plan | `docs/performance/performance-testing-plan.md` | Existing perf suite, including RBAC |
| Performance findings | `docs/performance/FINDINGS.md` | FINDING-037 RBAC authorization results |
