# ERPNext–Vela SSO Adapter execution progress

Plan: `docs/superpowers/plans/2026-10-30-erpnext-vela-sso-integration.md`
Design: `docs/superpowers/specs/2026-10-30-erpnext-vela-sso-adapter-design.md`
Vela-side plan: `/media/ubuntu/win_work/vela-main-ws/vela/docs/superpowers/plans/2026-10-30-erpnext-vela-sso-adapter.md`

## Original outcome and acceptance boundary

Enable a Vela employee to enter ERPNext from the Vela application list using organization-level SSO, create or reuse the ERPNext identity by `(issuer, sub)`, receive only approved Vela-managed ERPNext roles and business scopes, and lose access when Vela authorization or membership is revoked. The target is a real cross-system user flow, not a successful isolated adapter test.

Component tests do not constitute joint acceptance. Full acceptance requires a pinned ERPNext/Frappe build installed and migrated, configured Vela tenant and credential vault, a real Vela browser session, successful ERPNext JIT/login and authorization checks, and observed session denial/revocation.

## Implemented

- ERPNext companion contract: versioned capabilities/health/configure endpoints, secret-safe singleton settings and rotation, legacy settings migration, dedicated configurator restrictions, independent `(issuer, sub)` identity records and migration, deterministic JIT identity, exact email collision rejection, and no local password/welcome-email provisioning.
- Live authorization: ERPNext pulls a no-store Vela entitlement snapshot, validates tenant/identity/active state, roles/scopes and monotonic revisions before protected requests; identity-scoped lock serializes updates; only Vela-managed grants are reconciled and local grants are preserved; missing authorization fails closed and inactive users have sessions cleared.
- Vela configuration: Agent Manager validates tenant-owned credential references, resolves encrypted values in memory, provisions via bounded HTTPS HTTP adapter, checks contract compatibility and stores safe config revision/reference data.
- Employee entry: organization app readiness is based on a live ERPNext capability/health probe; the app offers an authenticated Vela redirect to ERPNext without requiring a per-employee ERPNext connection or exposing upstream URL in app JSON.
- Acceptance harness and evidence template: `scripts/acceptance/erpnext-sso/check-contract.sh` and `docs/acceptance/erpnext-sso/README.md`.

## Verification evidence

Executed on 2026-10-30:

- ERPNext companion Python suite: `PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 pytest -q integrations/erpnext/app/vela_erpnext/vela_erpnext/tests` — **95 passed, 3 subtests passed**. The repository's `/usr/bin/python3` uses PyJWT 2.3; the JWK code was corrected to avoid a newer-only `PyJWK.algorithm_name` property, then the full suite passed.
- Vela Go packages: `go test ./services/agent-manager/internal/erpnextsso ./services/agent-manager/internal/handlers ./services/agent-manager/cmd/agent-manager ./services/agent-manager/internal/server` — passed.
- Vela identity federation packages: `go test ./services/identity-federation/internal/handlers ./services/identity-federation/internal/repo` — passed.
- Web application focused tests: `pnpm exec vitest run src/views/apps/ApplicationsPage.test.tsx src/views/apps/VelaErpPage.test.tsx --reporter=verbose` — **24 passed**. jsdom emits its known unimplemented cross-document navigation warning for the external-login link; the tests pass.
- Web application production build: `pnpm build` — TypeScript and Vite production bundle succeeded. Vite reported large bundle chunk advisories (largest generated JS chunk about 7.9 MB); this does not block the SSO build but remains a frontend performance follow-up.
- Acceptance script syntax: `bash -n scripts/acceptance/erpnext-sso/check-contract.sh` — passed earlier; contract calls were not run against a live endpoint.

These tests cover component behavior and the negotiated API contract with test doubles. A local Bench install/upgrade was later verified as recorded below; tests do not verify real Vela OIDC, cross-system cookies/session behavior, or production revocation latency.

## Environment and remaining acceptance

The target Vela checkout is `/media/ubuntu/win_work/vela-main-ws/vela` on `feat/onboarding-ceo-workbench`. The SSO implementation and associated integration/startup work were subsequently committed and pushed through `0d0cde53`; the Vela working tree is clean apart from generated Python cache directories.

The existing ERPNext site was rebuilt and migrated twice with `scripts/pm-pilot/erpnext-up.sh`. Both runs exited 0; the first synced SSO DocTypes and fixtures, and the second completed as an upgrade/idempotence check. Runtime reports ERPNext `16.52.1`, Frappe `16.53.0`, and contract `1.0.0`; capabilities report PKCE/JIT/authorization sync/session revocation. SQL inspection confirms `Vela ERPNext` module registration, the `(issuer, subject)` unique index, and managed-grant Custom Fields. The HTTPS contract probe passed with ready health and no secret-shaped response fields.

The local Vela core is only partially running. `make dev up` failed during mail-connector image build because Docker could not create a bridge veth (`Device does not exist`). Agent Manager and Vite web were restarted through their local service paths; the OIDC simulator's HTTPS discovery is ready, but its `local` provider is not published by Vela. Bootstrap status reports that first-admin setup is closed; public provider discovery is empty and the admin provider endpoint returns 401 without a bearer token. Therefore no live configure call, browser login, or real revocation test was run. The compose release lock specifies ERPNext `v16.52.1` at source commit `ade850bbd6a7412e9994a9637d2c5c713771632a`; runtime reports Frappe `16.53.0`.

Joint acceptance remains **pending**. When the stack is restored, pin ERPNext/Frappe and contract versions, install/migrate the app, provision tenant credentials, run the acceptance script, then execute one real browser flow from Vela Apps through JIT and permission checks; revoke membership and verify existing ERPNext sessions are denied within the agreed SLA. Record versions, timestamps, and evidence in `docs/acceptance/erpnext-sso/README.md` before declaring the full goal complete.

## Pushed-commit revalidation (2026-11-01)

Vela source was initially revalidated at `0d0cde5351d534a6ae07de791b7be51de9f6917c`; focused authorization follow-up is now pushed at `b07adf25`. The companion docs branch is `feat/erpnext-vela-sso-docs` at `5de241d192` plus this follow-up. ERPNext remains `16.52.1`, Frappe `16.53.0`, and `vela_erpnext` `0.1.0` on the existing `frontend` site.

- ERPNext companion suite: **95 passed, 3 subtests passed**.
- Agent Manager SSO/handler/proxy/plugin-registry/command/server Go packages passed; identity-federation handler/repo and authorization-policy repo packages passed. The first acceptance run found a stale authorization handler mock; focused revalidation and the discovered role-set tenant-isolation gap are recorded below.
- Authorization-policy delegated-assignment PostgreSQL integration and Ontology registry PostgreSQL integration tests both failed before test execution when Testcontainers could not start its reaper container: Docker cannot attach a veth to `docker0` (`Device does not exist`). No app database reset was done.
- Ontology normal cmd/reindex package tests passed; storage package has no test files. ERPNext site bootstrap and dev-startup shell contract tests passed. The local release-lock consistency check passed, but the upstream/mirror tag probe hit a GnuTLS/RPC interruption.
- Web SSO page tests: **24 passed**; production build passed with the existing large-chunk advisory. The live ERPNext contract probe passed with contract `1.0.0` and ready health. ERPNext's login endpoint returned a PKCE authorization redirect to the local Keycloak simulator; this did not complete a callback or establish a Vela session.
- Read-only runtime evidence: the ERPNext Compose/site are UP with the pinned image and app versions; the Desk `vela-all-apps` Page exists; login serves the OIDC asset; `vela-erpnext` and `vela-zentao` registry rows are `builtin|enabled|ready`.
- Vela backend remains partial; first-admin bootstrap is closed, the local provider list is empty, and provider administration returns 401 without an authenticated admin. No Vela configure call, browser JIT, mapped-role readback, or session revocation was performed. The VVT CLI is unavailable, so this is a developer acceptance record, not a VVT campaign sign-off.

Overall acceptance remains **not passed** until the same authorized test identity completes the Vela Apps → ERPNext login/JIT/authorization/revocation flow. Full details are recorded in the Vela repository at `docs/acceptance/2026-11-01-submitted-feature-acceptance.md`.

### Authorization role-source and cross-tenant Role Set follow-up

- Updated the handler mock to expect the accepted tenant-or-global role lookup; `go test ./services/authorization-policy/internal/handlers ./services/authorization-policy/internal/repo -count=1` passes.
- Added regression coverage for adding a global role to a tenant Role Set and rejecting another tenant's role. The new cross-tenant assertion failed before the fix because the repository write path accepted the foreign role.
- `AddRoleToRoleSetWithActorSnapshot` now locks and validates the referenced role: tenant Role Sets accept same-tenant/global roles, while global Role Sets accept only global roles. The delegated-assignment implementation retains the explicit Role Set membership boundary.
- `TestDelegatedAssignmentChecksAndWritesAtomically` and `TestDelegationRequiresTenantRoleSetAndSufficientRank` both pass with `-tags=integration` against a disposable isolated PostgreSQL database via `TEST_DATABASE_URL`. The first covers role-source read/revoke isolation and cross-tenant target denial in addition to Role Set membership. The second confirms another tenant cannot use the tenant Role Set.
- Docker's bridge veth failure still prevents Testcontainers startup, so these focused PostgreSQL cases used an isolated database on the existing local Postgres service. No application database was modified. This acceptance resolves the authorization-policy scope; it does not complete the separate Vela browser/JIT/revocation acceptance.
