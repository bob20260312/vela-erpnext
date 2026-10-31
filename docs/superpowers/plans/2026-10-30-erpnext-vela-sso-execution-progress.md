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

The target Vela checkout is `/media/ubuntu/win_work/vela-main-ws/vela` on `feat/onboarding-ceo-workbench`, with extensive pre-existing uncommitted changes. The implementation remains uncommitted and unstaged; preserve all changes in that checkout.

The existing ERPNext site was rebuilt and migrated twice with `scripts/pm-pilot/erpnext-up.sh`. Both runs exited 0; the first synced SSO DocTypes and fixtures, and the second completed as an upgrade/idempotence check. Runtime reports ERPNext `16.52.1`, Frappe `16.53.0`, and contract `1.0.0`; capabilities report PKCE/JIT/authorization sync/session revocation. SQL inspection confirms `Vela ERPNext` module registration, the `(issuer, subject)` unique index, and managed-grant Custom Fields. The HTTPS contract probe passed with ready health and no secret-shaped response fields.

The local Vela core is only partially running. `make dev up` failed during mail-connector image build because Docker could not create a bridge veth (`Device does not exist`). Agent Manager and Vite web were restarted through their local service paths; the OIDC simulator's HTTPS discovery is ready, but its `local` provider is not published by Vela. Bootstrap status reports that first-admin setup is closed; public provider discovery is empty and the admin provider endpoint returns 401 without a bearer token. Therefore no live configure call, browser login, or real revocation test was run. The compose release lock specifies ERPNext `v16.52.1` at source commit `ade850bbd6a7412e9994a9637d2c5c713771632a`; runtime reports Frappe `16.53.0`.

Joint acceptance remains **pending**. When the stack is restored, pin ERPNext/Frappe and contract versions, install/migrate the app, provision tenant credentials, run the acceptance script, then execute one real browser flow from Vela Apps through JIT and permission checks; revoke membership and verify existing ERPNext sessions are denied within the agreed SLA. Record versions, timestamps, and evidence in `docs/acceptance/erpnext-sso/README.md` before declaring the full goal complete.
