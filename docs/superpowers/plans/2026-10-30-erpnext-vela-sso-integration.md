# ERPNext–Vela SSO Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans or superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the existing Vela ERPNext companion app so an organization can configure Vela SSO once, provision authorized ERPNext users on first login, and keep Vela-managed access synchronized without losing local ERPNext grants.

**Architecture:** Implement the ERPNext side in the existing `integrations/erpnext/app/vela_erpnext` Frappe app. Keep the existing OIDC Authorization Code + PKCE and live entitlement lookup, add site-level secret-backed settings and capability/configure/health APIs, map identities by `(issuer, sub)`, and reconcile only grants marked as Vela-owned. The Vela web adapter readiness and its connection UI remain in the companion Vela plan; its adapter must consume the exact ERPNext API contract defined here.

**Tech Stack:** ERPNext `v16.52.1` companion image; Frappe dependency `>=16.50.0,<17.0.0` (record the exact runtime version from the target Bench); Python version from the pinned image (the current ERPNext source requires Python `>=3.14`); Frappe DocTypes and hooks, PyJWT, `requests`, Python `unittest`, existing Vela identity-federation Go service and PostgreSQL migrations.

**Spec:** `docs/superpowers/specs/2026-10-30-erpnext-vela-sso-adapter-design.md`; source requirements: `/media/ubuntu/win_work/vela-main-ws/vela/docs/superpowers/specs/2026-10-30-erpnext-vela-sso-integration-requirements.md`; companion Vela adapter plan: `/media/ubuntu/win_work/vela-main-ws/vela/docs/superpowers/plans/2026-10-30-erpnext-vela-sso-adapter.md`.

## Global Constraints

- Identity key is `(issuer, sub)`; email and display name are profile fields and must never silently link accounts.
- OIDC uses Authorization Code + PKCE S256, one-time browser-bound state, nonce, issuer/audience/azp/signature/time checks, HTTPS and exact callback validation.
- OIDC RP secret and Vela entitlement client secret are separate Frappe Password fields; neither secret nor configure request body may appear in logs, ordinary config exports or API readbacks.
- Authorized Vela entitlement is required at login and each protected request. Entitlement failures deny access; no stale authorization cache is allowed in v1.
- Only `custom_vela_managed=1` role and User Permission rows may be changed or removed by sync. Local role grants, local User Permissions and DocShare remain intact.
- Vela SSO v1 maps only roles in `DEFAULT_ROLE_MAP` and `ROLE_REQUIRED_SCOPES`; unmapped/empty business roles deny login. Administrator, System Manager and other forbidden roles are never granted by SSO.
- Vela-deactivated accounts are disabled and their sessions cleared. Re-enable requires a newer active `lifecycle_version`; a locally disabled User remains disabled.
- ERPNext capability mismatch or unhealthy configuration must not be reported as a ready Vela connection. Component tests do not count as cross-system acceptance.

## Execution Status (2026-10-30)

- Tasks 1–4 implementation and current component-suite verification are complete; see `docs/superpowers/plans/2026-10-30-erpnext-vela-sso-execution-progress.md` for commands and evidence limits.
- Task 5 now has a public contract probe, a Bench-side version-bound active/revoked acceptance runner, and a joint-evidence template. The pinned local Bench install/migrate ran twice successfully, and the HTTPS public contract probe reports ready. Restricted Vela configurator bootstrap and real browser acceptance remain pending: the local provider is unpublished, first-admin bootstrap is closed, and the provider administration API requires an authenticated Vela administrator.
- Historical red-phase output for the TDD checklist is not retained as current evidence. Commit steps remain unchecked to preserve the user's existing uncommitted repository changes; no work was staged or committed.

## Files and Ownership

All implementation paths below are relative to `/media/ubuntu/win_work/vela-main-ws/vela`, the Vela repository:

- `integrations/erpnext/app/vela_erpnext/vela_erpnext/sso.py`: OIDC callback plus `capabilities`, `configure`, and `health` APIs.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/permission_sync.py`: entitlement client, managed grant projection, per-request fail-closed enforcement.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/identity.py`: composite identity lookup, JIT provisioning and legacy binding migration helpers.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_settings/`: singleton site configuration and encrypted secrets.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_identity/`: issuer/subject mapping and authorization sync versions.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/fixtures/custom_field.json`: Vela-owned markers for User Role and User Permission rows; legacy User fields retained read-only through rollback window.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/patches.txt` and `patches/v0_2/`: idempotent migration of existing subject bindings and unique constraints.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/`: unit and Bench acceptance coverage.
- `integrations/erpnext/app/vela_erpnext/vela_erpnext/hooks.py`: retain `before_request` enforcement and register migration/setup hooks only where needed.
- `integrations/erpnext/scripts/ensure-site-apps.sh`, `integrations/erpnext/README.md`: app install/upgrade and secure one-time configure/bootstrap instructions.
- Existing Vela entitlement contract consumed without a new per-user binding API: `POST /api/v1/internal/erpnext/entitlements`; response includes `user_id`, `organization_id`, `active`, `lifecycle_version`, `scope_revision`, `roles`, and `scopes`. The separate Vela adapter plan owns capability projection and readiness UI; this plan defines the ERPNext endpoint contract it must call.

## Review Focus

- Same `sub` under different issuers and migration of old single-field unique bindings: test that identity lookup and the DB constraint distinguish `(issuer, sub)` and that conflicting legacy rows stop migration without data loss (Task 2).
- Local roles, User Permissions, and DocShare during role/scope removal: test that only Vela-marked rows are removed and local rows survive (Task 3).
- Same email with a different subject, missing email, and repeat login: test conflict denial, deterministic `username`, and stable identity reuse without email matching (Task 2).
- Vela inactive state, stale `lifecycle_version`/`scope_revision`, local disable, and Vela outage: test disable/revoke/re-enable rules and fail-closed behavior before request handling (Task 4).
- Repeated configure, rotated secrets, wrong tenant/configurator role, malformed callback, and secret-bearing exceptions: test idempotence, least privilege, redaction, safe redirect and secret rotation (Task 1).

---

### Task 1: Add machine-managed SSO settings and capability APIs

**Files:**
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_settings/vela_sso_settings.json`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_settings/vela_sso_settings.py`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_sso_settings.py`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/patches/v0_2/migrate_sso_settings.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/sso.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/permission_sync.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/role_policy.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/hooks.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/fixtures/custom_field.json`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/patches.txt`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_permission_sync.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_sso.py`
- Modify: `integrations/erpnext/README.md`

**Interfaces:**
- Produces: `capabilities() -> dict`, `configure(**payload) -> dict`, `health() -> dict`, and `get_sso_settings(frappe, include_secrets: bool = False) -> dict` in `sso.py`. Default settings reads are secret-free; OIDC/token-exchange code opts into Password retrieval internally.
- Produces: `EntitlementSnapshot` and `get_entitlement_snapshot(frappe, provider: str, subject: str) -> EntitlementSnapshot` in `permission_sync.py`, adapting the existing no-store API response including `active`, `user_id`, `organization_id`, `lifecycle_version`, `scope_revision`, `roles`, and `scopes`.
- Configure accepts `contract_version`, `issuer`, `oidc_client_id`, `oidc_client_secret`, `redirect_uri`, `vela_api_base`, `provider`, `organization_id`, `entitlement_client_id`, `entitlement_client_secret`, `role_map_version`, `callback_allowlist`, and `secret_overlap_seconds` in `[0, 86400]`. On rotation, the server moves the old current secrets into previous Password fields; callers never submit previous secrets. It returns only `config_revision`, `issuer`, `client_id_digest`, and status.
- Expose Frappe methods at `GET /api/method/vela_erpnext.sso.capabilities`, `POST /api/method/vela_erpnext.sso.configure`, and `GET /api/method/vela_erpnext.sso.health`; the Vela adapter must use these exact paths and response fields.
- `configure` is POST-only and checks the dedicated `Vela SSO Configurator` Frappe role plus `custom_vela_config_organization_id == payload.organization_id`. The deployment bootstrap provisions that service user once and stores its API key/secret only in Vela secret management. The `health` response never includes any credential.
- `capabilities()` returns `vela_sso_contract_version="1.0.0"`, ERPNext/Frappe build strings, supported capability booleans and readiness state. `health()` returns config revision, issuer/client ID digest and entitlement dependency health only.
- Frappe Password fields store OIDC and entitlement credentials separately. Reconfiguration with the same payload is a no-op; rotation accepts current and previous credentials plus an overlap expiry. `role_policy.py`/`after_migrate` create the dedicated configure role; the one-time authorized installer provisions its API user/key and writes that credential directly to Vela secret management.

- [x] **Step 1: Add settings/API tests**

Add tests named `test_configure_requires_configurator_role`, `test_configure_is_idempotent`, `test_configure_rotates_secrets_with_overlap`, `test_configure_rejects_wrong_site_organization`, `test_capabilities_reports_contract_and_build`, `test_health_never_returns_secrets`, `test_site_config_secrets_migrate_into_password_store_and_are_removed`, `test_entitlement_snapshot_maps_versions_and_roles`, `test_entitlement_snapshot_rejects_tenant_mismatch`, `test_oidc_token_exchange_uses_previous_secret_during_overlap`, `test_entitlement_client_uses_previous_secret_during_overlap`, and `test_previous_secret_is_not_used_after_expiry`. Assert that a secret in a rejected payload does not appear in captured logs or error response.

- [ ] **Step 2: Run the focused tests and confirm they fail**

Run from Vela repository root: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_sso_settings vela_erpnext.tests.test_permission_sync vela_erpnext.tests.test_sso -v`

Expected: FAIL because settings/API functions and fields are not implemented.

- [x] **Step 3: Implement the settings DocType and APIs**

Declare singleton fields for contract/config revisions, OIDC issuer/client/callback, Vela API/provider/organization, entitlement client ID, role-map version, callback allowlist, enabled state, and current/previous secret expiry. Use separate current/previous Password fields for OIDC and entitlement secrets. Rotation moves the previous current value server-side and caps overlap at 86,400 seconds; try current first and previous only after authentication failure before expiry. Add a dedicated service role with no general ERPNext DocType permissions and a hidden read-only `User.custom_vela_config_organization_id`; enforce role and organization match inside `configure`. Set contract version to `1.0.0`. Implement the three APIs with safe error messages and structured logs that omit request bodies.

- [x] **Step 4: Move existing config readers to the settings interface**

Update `_settings()` in `sso.py` and `_settings()` in `permission_sync.py` to read only the singleton. The idempotent `migrate_sso_settings` patch copies existing `site_config.vela_oidc` and `site_config.vela_permission_sync` values into Password fields, verifies the stored values, and then removes both legacy keys from `site_config`; if verification fails, abort without deleting the source keys. Do not retain a runtime fallback to plaintext site config.

- [x] **Step 5: Run focused tests and verify passing behavior**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_sso_settings vela_erpnext.tests.test_permission_sync vela_erpnext.tests.test_sso -v`

Expected: PASS; health/capabilities contain no secret, and reapplying the same configuration keeps the same revision.

- [ ] **Step 6: Commit the settings/API slice**

Commit settings DocType, APIs, tests, and README updates together.

### Task 2: Create composite identity mapping and JIT ERPNext users

**Files:**
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_identity/vela_sso_identity.json`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/vela_erpnext/doctype/vela_sso_identity/vela_sso_identity.py`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/identity.py`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_identity.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/sso.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/fixtures/custom_field.json`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/patches/v0_2/migrate_sso_identity.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/patches.txt`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_sso.py`

**Interfaces:**
- Consumes: `get_sso_settings(frappe, include_secrets=True)` and `EntitlementSnapshot` / `get_entitlement_snapshot(frappe, provider, subject)` from Task 1.
- Produces: `identity_key(issuer: str, subject: str) -> str`, `find_identity(issuer: str, subject: str) -> dict | None`, `provision_user(claims: dict, snapshot: EntitlementSnapshot) -> str`, and `upsert_identity(issuer: str, subject: str, user: str, snapshot: EntitlementSnapshot) -> None`.
- `Vela SSO Identity` fields: `issuer`, `subject`, `provider`, `vela_user_id`, `organization_id`, `user`, `active`, `lifecycle_version`, `scope_revision`, `managed_by_vela`, and `last_synced_at`; enforce database unique `(issuer, subject)`.
- Deterministic username is `vela_` plus lowercase Base32 of SHA-256 over `issuer + NUL + subject`, truncated to 32 characters. Email is profile data only. If a different user owns the supplied email, reject with a generic callback error and an internal audit event. If no email exists and target Frappe requires one, use the deterministic username at the configured `.invalid` domain.

- [x] **Step 1: Add mapping/provisioning tests**

Add tests named `test_identity_key_includes_issuer`, `test_same_sub_under_two_issuers_creates_distinct_mappings`, `test_legacy_mapping_is_reused`, `test_jit_username_is_deterministic`, `test_existing_email_for_different_subject_is_rejected`, `test_missing_email_uses_nonroutable_identity`, `test_jit_rejects_privileged_or_unmapped_role`, `test_jit_does_not_send_welcome_mail_or_set_password`, and `test_jit_audit_contains_redacted_identity_and_grants_without_tokens`.

- [ ] **Step 2: Run the focused tests and confirm they fail**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_identity -v`

Expected: FAIL because the identity DocType and JIT helpers do not exist.

- [x] **Step 3: Add identity DocType and composite unique constraint**

Create the DocType with restricted read/write permissions. Add an idempotent migration that creates a database unique constraint on `(issuer, subject)`; do not rely on a fixture's field-level unique flag for this composite key.

- [x] **Step 4: Migrate current User subject fields without deleting them**

Backfill one identity row for each User with both legacy issuer and subject. If any mapping points to a missing/duplicate ERPNext user, fail migration with the affected User names and remediation instructions; do not silently discard or merge rows. Remove the single-column unique constraint from `custom_vela_oidc_subject` only after the backfill succeeds, retain the hidden fields read-only for one release, and make identical legacy `bind_subject` calls idempotently update the mapping.

- [x] **Step 5: Implement JIT provisioning and update the callback**

After token verification, query entitlement by configured `provider` + `sub`; require matching `organization_id`, `active=true`, a stable `user_id`, at least one mapped business role, and required scopes before creating a User. Create System Users without a welcome email or local password, store the identity mapping, sync profile/language, and only then call `login_as`. Reuse mapped Users on repeat logins; never search by email to select an account.

- [x] **Step 6: Run identity and callback tests**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_identity vela_erpnext.tests.test_sso -v`

Expected: PASS for legacy mapping, first-login JIT, repeat login, email conflict denial, and provider failure without session creation.

- [ ] **Step 7: Commit the identity/JIT slice**

Commit identity DocType, migration, callback changes, tests, and patch registration together.

### Task 3: Preserve local grants while reconciling Vela permissions

**Files:**
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/fixtures/custom_field.json`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/permission_sync.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_permission_sync.py`

**Interfaces:**
- Consumes: `EntitlementSnapshot` from Task 1 and `Vela SSO Identity` from Task 2.
- Produces: `_reconcile(frappe, user_name: str, erp_roles: list[str], permissions: list[tuple[str, str]]) -> None`; only rows with `custom_vela_managed=1` are changed or deleted.

- [x] **Step 1: Add ownership tests**

Add/update tests named `test_reconcile_preserves_local_role_profile_and_roles`, `test_reconcile_preserves_local_user_permissions_and_doc_shares`, `test_reconcile_removes_only_revoked_vela_role_rows`, `test_reconcile_marks_new_user_permissions_as_vela_managed`, and `test_local_grant_overlapping_vela_role_survives_vela_revoke`.

- [ ] **Step 2: Run focused permission tests and confirm failure**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_permission_sync -v`

Expected: FAIL because current `_reconcile` replaces all roles/User Permissions and deletes direct DocShare.

- [x] **Step 3: Add managed-source markers**

Add hidden `custom_vela_managed` Check fields to `Has Role` and `User Permission`; update fixtures and migration so existing rows default to local (`0`). Do not mark historical grants as Vela-owned without explicit evidence.

- [x] **Step 4: Reconcile only Vela-owned rows**

Diff desired Vela roles against `Has Role` rows marked managed; preserve `role_profile_name` and every unmarked row. Diff desired scopes against Vela-marked User Permissions; preserve unmarked permissions and all DocShare rows. If the same role/scope already exists locally, leave it local so a later Vela revoke cannot remove it. Keep strict user permission enforcement and the existing reviewed `ROLE_POLICY` checks.

- [x] **Step 5: Run permission tests and verify passing behavior**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_permission_sync -v`

Expected: PASS; Vela role/scope removals clear only managed grants, while local roles, profile, permissions and shares remain unchanged.

- [ ] **Step 6: Commit the permission-ownership slice**

Commit fixtures, reconciliation, migration and tests together.

### Task 4: Enforce current entitlement and revocation on every managed request

**Files:**
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/permission_sync.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/identity.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/hooks.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_permission_sync.py`
- Modify: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/test_sso.py`

**Interfaces:**
- Produces: `authorize_and_sync(frappe, user_name: str, issuer: str, subject: str) -> None`; identity-scoped lock covers snapshot retrieval and reconciliation. Use lock key `vela_sso_sync:` + SHA-256 of `issuer + NUL + subject`, wait at most 8 seconds, and release in `finally`. Lock timeout, API error, tenant mismatch, older lifecycle/scope revision or missing required scope raises `OIDCError` and denies the request.

- [x] **Step 1: Add lifecycle/revision tests**

Add tests named `test_entitlement_uses_no_store_live_snapshot`, `test_stale_lifecycle_version_cannot_reenable_user`, `test_newer_active_lifecycle_version_reenables_vela_disabled_user`, `test_local_disable_is_never_overridden`, `test_snapshot_reconciliation_is_serialized_per_identity`, `test_inactive_snapshot_disables_user_and_clears_all_sessions`, and `test_entitlement_outage_denies_managed_request`.

- [ ] **Step 2: Run focused lifecycle tests and confirm failure**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_permission_sync -v`

Expected: FAIL for stale-version handling and serialization; existing fail-closed cases remain passing.

- [x] **Step 3: Enforce monotonic lifecycle/scope versions and identity lock**

Persist `lifecycle_version` and `scope_revision` on `Vela SSO Identity`; reject any snapshot older than the stored values. Acquire a per-identity Frappe cache lock before fetching and reconciling a snapshot, and release it in `finally`. If the lock cannot be acquired within the bounded timeout, roll back and deny access.

- [x] **Step 4: Route managed requests through identity-backed authorization**

Keep `before_request = ["vela_erpnext.permission_sync.enforce_managed_session"]`. Resolve managed users from the identity DocType rather than relying only on User custom fields. Fetch the current entitlement before protected handling; inactive users are disabled and `frappe.sessions.clear_sessions(user=..., force=True)` is called. A locally disabled user stays disabled. Only a newer active lifecycle version can re-enable a Vela-lifecycle-disabled user.

- [x] **Step 5: Run focused tests and confirm behavior**

Run: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest vela_erpnext.tests.test_permission_sync vela_erpnext.tests.test_sso -v`

Expected: PASS for live authorization, version ordering, inactive-session clearing, and fail-closed outages.

- [ ] **Step 6: Commit the lifecycle slice**

Commit identity status, locking, request enforcement and tests together.

### Task 5: Package the contract and run version-bound acceptance

**Files:**
- Modify: `integrations/erpnext/README.md`
- Modify: `integrations/erpnext/scripts/ensure-site-apps.sh`
- Create: `integrations/erpnext/app/vela_erpnext/vela_erpnext/tests/live_sso_acceptance.py`
- Create: `docs/acceptance/erpnext-sso/<acceptance-date>.md` after a real joint run
- Coordinate with the Vela plan's capability/configuration/readiness tasks; do not duplicate its UI adapter.

**Interfaces:**
- Consumes: `capabilities`, `configure`, `health`, identity/JIT, permission sync and request enforcement from Tasks 1–4.
- The Bench acceptance runner records exact ERPNext/Frappe/contract versions, config revision, a redacted subject fingerprint, ERPNext username, live mapped roles/scopes, and revoked-session results. Browser redirect/locale and the Vela build are recorded by the browser acceptance evidence.

- [x] **Step 1: Add a version-bound Bench acceptance runner**

Add `run()` as a Bench-side helper that asserts expected ERPNext/Frappe build and contract version, validates current identity/roles/scopes for the dedicated acceptance subject, and in revoked mode invokes the managed request hook and verifies session-row deletion. First/repeat login, wrong issuer/audience/nonce, same-email conflict, safe redirect/locale, role/scope update, and the user-visible Vela-to-ERPNext flow still require real browser acceptance.

- [x] **Step 2: Run component coverage for the complete companion app**

Run from Vela root: `PYTHONPATH=integrations/erpnext/app/vela_erpnext python3 -m unittest discover -s integrations/erpnext/app/vela_erpnext/vela_erpnext/tests -p 'test_*.py'`

Expected: PASS for app-level suites. This result does not establish cross-system acceptance.

- [x] **Step 3a: Verify install and upgrade idempotence on the pinned image**

On the preserved local site, `scripts/pm-pilot/erpnext-up.sh` ran twice successfully with ERPNext `v16.52.1` and Frappe `16.53.0`. The first run registered both SSO DocTypes and their schema, ran the settings patch and fixtures/after-migrate hooks, and reported the site ready. The second run completed successfully. SQL inspection confirmed `Vela ERPNext` module ownership, the composite `(issuer, subject)` unique index, and four required authorization Custom Fields. This proves local migration/upgrade idempotence; it does not test rollback under a failing migration.

- [ ] **Step 3b: Bootstrap the restricted Vela configuration user**

Use the authorized site-administration credential to create the restricted Frappe API user bound to the Vela organization, transfer its API key/secret directly into Vela secret management, then discard the bootstrap credential. The Vela adapter must store only a secret reference and use the API key/secret for `configure`.

- [ ] **Step 4: Run real browser joint acceptance only when Vela capability is ready**

Run the app acceptance runner against the improved ERPNext image, real Vela session, real Vela identity-federation endpoint, and the Vela adapter contract. Verify that Vela marks the connection ready only after capability and health checks pass. Do not substitute simulator, mocks, historical logs, or a separate component run.

- [ ] **Step 5: Record the real joint run evidence**

Create `docs/acceptance/erpnext-sso/<acceptance-date>.md` with both builds, contract versions, redacted identity, role/scope results, redirect, deactivation visibility time, and session-denial evidence. If a real browser, Vela session, or target deployment is unavailable, leave the result as “component verified; joint acceptance pending” and list the missing evidence.

- [ ] **Step 6: Commit packaging and acceptance artifacts**

Commit deployment/documentation changes with the acceptance record only after the stated run actually occurred.
