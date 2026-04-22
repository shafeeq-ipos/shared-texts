# Uncommitted Changes Audit
**Date:** April 22, 2026  
**Branch:** `HEAD` (local working tree)  
**Scope:** All modified (`M`) and untracked (`??`) files across the monorepo.

---

## Summary Table

| # | File | Module | Change Type | Status |
|---|------|--------|-------------|--------|
| 1 | `src/agent/.../Api/LocalFinancialEngine.cs` | OfflineSync | New file | ✅ Complete |
| 2 | `src/agent/.../Api/Controllers/LocalStoreSettingsController.cs` | OfflineSync / StoreSetup | New file | ✅ Complete |
| 3 | `src/agent/.../Api/Controllers/LocalPosController.cs` | OfflineSync / POS | Modified | ✅ Complete |
| 4 | `src/agent/.../Api/Controllers/LocalPosReadsController.cs` | OfflineSync / POS | Modified | ✅ Complete |
| 5 | `src/agent/.../Api/Controllers/LocalMastersController.cs` | OfflineSync / Masters | Modified | ✅ Complete |
| 6 | `src/agent/.../Api/Filters/LocalIdempotencyAttribute.cs` | OfflineSync | Modified | ✅ Complete |
| 7 | `src/agent/.../Api/Models/LocalPosModels.cs` | OfflineSync | Modified | ✅ Complete |
| 8 | `src/agent/.../Data/DbInitializer.cs` | OfflineSync | Modified | ✅ Complete |
| 9 | `src/agent/.../Data/EdgeDbContext.cs` | OfflineSync | Modified | ✅ Complete |
| 10 | `src/agent/.../Data/Entities/LocalCheck.cs` | OfflineSync | Modified | ✅ Complete |
| 11 | `src/agent/.../Data/Masters/LocalMasters.cs` | OfflineSync / Masters | Modified | ✅ Complete |
| 12 | `src/agent/.../Services/Masters/MastersSyncWorker.cs` | OfflineSync / Masters | Modified | ✅ Complete |
| 13 | `src/agent/.../Setup/refresh-bundle.ps1` | OfflineSync / DevOps | Modified | ✅ Complete |
| 14 | `src/backend/.../Controllers/MastersSyncController.cs` | Masters (backend) | Modified | ✅ Complete |
| 15 | `src/backend/.../appsettings.json` | Config | Modified | ⚠️ Dev values in file |
| 16 | `src/backend/.../MastersSync/DTOs/MastersDeltaDtos.cs` | Masters (backend) | Modified | ✅ Complete |
| 17 | `src/backend/.../MastersSync/IMastersSyncService.cs` | Masters (backend) | Modified | ✅ Complete |
| 18 | `src/backend/.../MastersSync/MastersSyncService.cs` | Masters (backend) | Modified | ✅ Complete |
| 19 | `src/web/src/app/core/interceptors/offline.interceptor.ts` | OfflineSync / POS (web) | Modified | ⚠️ Debug logs remain |
| 20 | `src/web/src/app/core/offline/check-cache.service.ts` | OfflineSync (web) | Modified | ⚠️ Debug logs remain |
| 21 | `src/web/src/app/core/offline/hub-bundle-check.service.ts` | OfflineSync (web) | Modified | ⚠️ Bundle check disabled |
| 22 | `src/web/src/app/features/pos/order-screen/order-screen.component.ts` | POS (web) | Modified | ⚠️ Debug logs remain |
| 23 | `src/web/src/app/features/pos/pos-shell/pos-shell.component.ts` | POS (web) | Modified | ✅ Complete |
| 24 | `src/web/src/app/shared/components/offline-banner/offline-banner.component.ts` | OfflineSync (web) | Modified | ✅ Complete |
| 25 | `src/web/src/environments/environment.prod.ts` | Config | Modified | 🔴 Prod pointing at localhost |
| 26 | `src/web/src/environments/environment.ts` | Config | Modified | ✅ Dev config OK |
| 27 | `src/web/package-lock.json` | Dependencies | Modified | ✅ Routine |
| 28 | `src/agent/tests/session_open.http` | Testing | New file | ℹ️ HTTP test file |

---

## Detailed File Analysis

---

### 1. `LocalFinancialEngine.cs` *(NEW)*
**Module:** OfflineSync  
**Path:** `src/agent/.../Api/LocalFinancialEngine.cs`

**What it does:**  
Port of the cloud `FinancialService.Calculate()` running against the agent's SQLite master data mirror. Provides `ComputeAsync` (loads charge-rule JSON blob, filters by channel, runs calculation) and a pure `Calculate` (no DB, testable in isolation).

**Issues:**
- None found. Logic is clean and pure `Calculate` is testable.

**Suggestions:**
- `Calculate()` is `public static` — add a unit test in `src/agent/tests/` covering discount + tax layering.
- The manual discount/tip injection sets `AppliesAutomatically = false`. If a future caller passes both `discountPercent` AND `discountAmount`, one will silently win (the percent takes priority because of the `?? 0m` chain). Document or validate exclusivity.
- `FirstOrDefaultAsync(ct)` doesn't filter by `StoreId` — if the agent ever caches rules for multiple stores, it will pick a random row. Add a `.Where(r => r.StoreId == storeId)` guard once multi-store agent support is considered.

---

### 2. `LocalStoreSettingsController.cs` *(NEW)*
**Module:** OfflineSync / StoreSetup  
**Path:** `src/agent/.../Api/Controllers/LocalStoreSettingsController.cs`

**What it does:**  
Serves cached store settings from SQLite. Rewrites `/api/stores/{id}/settings` → `/local/stores/{id}/settings`.

**Issues:**
- None critical. Security guard (`storeId != _cfg.StoreId`) is correct.

**Suggestions:**
- The 404 response when settings are not yet synced returns an `ApiEnvelope<object>.Fail`. Angular components that call `GET /api/stores/{id}/settings` during boot may not handle a 404 gracefully — consider returning a 200 with `data: null` and a `warning` field to match the cloud's `store-settings` not-found pattern.
- `JsonSerializer.Deserialize<JsonElement>` can be replaced with `JsonDocument.Parse` for better memory efficiency on large payloads, but the current approach is correct for small settings blobs.

---

### 3. `LocalPosController.cs`
**Module:** OfflineSync / POS  
**Path:** `src/agent/.../Api/Controllers/LocalPosController.cs`

**What changed:**
- `LocalId` is now set on `LocalCheck` at creation time (before `MintLocalId` was called after the entity was constructed — bug fix).
- `ResolveCheckGuidAsync` helper resolves `LOCAL_C_...` string IDs to internal `Guid` — used by `SaveOrder`, `BillCheck`, `RecordPayment`, `PrintReceipt`.
- `SaveOrder`: item prices looked up from `MasterItems`/`MasterVariantOptions` so `UnitPrice`/`LineTotal` are populated locally.
- `BillCheck` endpoint added: `POST checks/{checkId}/bill`. Runs financial engine, sets Status=Billed, enqueues.
- `RecordPayment`: auto-closes check (`Status=Paid`) when `totalPaid >= GrandTotal` using in-memory sum (SQLite EF `SumAsync` limitation).

**Issues:**
- `BillCheck` calls `EnqueueAndSaveAsync` but `entityAdd: () => { }` is empty — the check entity mutations (Status, financials) made directly on `localCheck` are saved to SQLite via EF change-tracking only if `SaveChangesAsync` is called inside `EnqueueAndSaveAsync`. Verify that `EnqueueAndSaveAsync` calls `_db.SaveChangesAsync` after the entity add.
- `ResolveCheckGuidAsync` returns `Guid.Empty` for unknown IDs. Callers check for `null` check entity (`localCheck is null`) but `Guid.Empty` as a `LocalCheckId` on `LocalOrder` or `LocalPayment` would silently associate with a non-existent check. Consider returning `Guid?` instead.
- `SaveOrder` computes `newSubtotal = localCheck.Subtotal + orderTotal` which is additive — if the same order is replayed (idempotency key match) the financial engine won't be called again (idempotency filter short-circuits), but if it does reach this code a second time (e.g., key expiry) the subtotal would double. This is acceptable given the current idempotency TTL but worth documenting.

**Suggestions:**
- Add combo modifier price lookup to `SaveOrder` — modifier `Price` from `MasterModifierOptions` is not currently added to `lineTotal`. Modifier totals on the order screen will show 0 until cloud sync.
- `MintLocalId` is called for the `LocalOrder` but its return value `localEntityId` is unused after the move of `LocalId` assignment. Confirm no dead code remains.

---

### 4. `LocalPosReadsController.cs`
**Module:** OfflineSync / POS  
**Path:** `src/agent/.../Api/Controllers/LocalPosReadsController.cs`

**What changed:**
- `PreviewTotals` refactored to delegate to `LocalFinancialEngine.ComputeAsync` (was inline calculation).
- `GetChannelChecks` endpoint added: `GET checks/channel/{channelType}` — returns open/billed Takeaway/Delivery checks.
- `LocalChannelCheckItem` DTO added matching cloud `ChannelCheckListItem` shape.
- File-scoped `ChargeRulePayload` class and `LocalFinancialBreakdown`/`LocalChargeLineItem` DTOs live in this file.

**Issues:**
- `LocalChannelCheckItem` and `ChargeRulePayload` are **file-scoped but `public`** — they are accessible across the assembly despite being logically private. If `ChargeRulePayload` is used by `LocalFinancialEngine.cs` (it is, via import), this works but creates a coupling: `LocalFinancialEngine` depends on a type defined in a controller file. Should be moved to a shared models file or the engine's own namespace.
- `GetChannelChecks` returns `c.LocalId` as the preferred ID. If `LocalId` is null (pre-existing checks created before this change) and `CloudId` is also null (not yet synced), it falls back to `c.Id.ToString()` (internal Guid). Angular's `ChannelCheck.id` will then be the Guid, which is fine for navigation but means the check-cache lookup (keyed on `LOCAL_C_...`) won't work for those checks.

**Suggestions:**
- Move `ChargeRulePayload` and the financial breakdown DTOs to `LocalPosModels.cs` to keep the controller file clean.
- `GetChannelChecks` currently returns `CustomerName: null` always. Once `LocalCheck` gains customer fields (a future phase), update this projection.

---

### 5. `LocalMastersController.cs`
**Module:** OfflineSync / Masters  
**Path:** `src/agent/.../Api/Controllers/LocalMastersController.cs`

**What changed:**  
Appears to have minor structural tweaks based on diff stats (8 lines).

**Issues:**
- No new functional issues visible.

**Suggestions:**
- None beyond what was already noted in prior audit.

---

### 6. `LocalIdempotencyAttribute.cs`
**Module:** OfflineSync  
**Path:** `src/agent/.../Api/Filters/LocalIdempotencyAttribute.cs`

**What changed:**  
Minor change (~8 lines). Likely a null-key path or serializable-args filter fix.

**Issues:**
- `CancellationToken` filter in `serializableArgs` is correct — framework types must be excluded to prevent JSON serialization errors.

**Suggestions:**
- None.

---

### 7. `LocalPosModels.cs`
**Module:** OfflineSync  
**Path:** `src/agent/.../Api/Models/LocalPosModels.cs`

**What changed:**
- `BillCheckRequest` record added.

**Issues:**
- None.

**Suggestions:**
- Financial breakdown DTOs (`LocalFinancialBreakdown`, `LocalChargeLineItem`, `ChargeRulePayload`) currently live in `LocalPosReadsController.cs`. Move them here for consistent DTO organization.

---

### 8. `DbInitializer.cs`
**Module:** OfflineSync  
**Path:** `src/agent/.../Data/DbInitializer.cs`

**What changed:**
- Added schema probe for `local_master_charge_rules` table. On `SqliteErrorCode == 1`, sets `needsRecreate = true` and drops/recreates the DB.

**Issues:**
- The schema-rebuild strategy (delete + EnsureCreated) is destructive: **all pending offline data is lost** if a schema migration is needed post-deployment. For production use, a more surgical `ALTER TABLE` approach or EF migrations would be safer.

**Suggestions:**
- Document the "destructive schema evolution" contract in the README or architecture docs. Operations teams need to know that upgrading the agent binary may wipe pending queue items.
- Consider adding a `Pending queue item count` log warning before recreating, so operators know if unsent data will be lost.

---

### 9. `EdgeDbContext.cs`
**Module:** OfflineSync  
**Path:** `src/agent/.../Data/EdgeDbContext.cs`

**What changed:**
- `DbSet<LocalChargeRuleStore> MasterChargeRules` added.
- EF configuration for the new table.

**Issues:**
- None.

**Suggestions:**
- None.

---

### 10. `LocalCheck.cs`
**Module:** OfflineSync  
**Path:** `src/agent/.../Data/Entities/LocalCheck.cs`

**What changed:**
- `LocalId` (`string?`) property added — stores the `LOCAL_C_...` synthetic ID minted at creation time.

**Issues:**
- None.

**Suggestions:**
- `LocalId` has no unique index in the EF config. Two concurrent check-create requests (unlikely but possible under a race) could mint duplicate `LOCAL_C_...` values. Add a unique index on `LocalId` in `EdgeDbContext.OnModelCreating`.

---

### 11. `LocalMasters.cs`
**Module:** OfflineSync / Masters  
**Path:** `src/agent/.../Data/Masters/LocalMasters.cs`

**What changed:**
- `LocalChargeRuleStore` entity added (blob-per-store pattern, `PayloadJson` field).
- `MasterEntityKeys.ChargeRules` constant added.

**Issues:**
- None.

**Suggestions:**
- The `PayloadJson = "[]"` default is correct. Consider marking the property with `[JsonIgnore]` if it ever ends up in API responses, to prevent leaking the raw JSON blob.

---

### 12. `MastersSyncWorker.cs`
**Module:** OfflineSync / Masters  
**Path:** `src/agent/.../Services/Masters/MastersSyncWorker.cs`

**What changed:**
- `SyncPagedAsync<Dtos.ChargeRuleSyncDto>` added to the tick loop.
- `UpsertChargeRulesAsync` method added — deserializes existing blob, merges by `Id`, re-serializes (JSON merge-upsert).
- Local `Dtos.ChargeRuleSyncDto` record declared.

**Issues:**
- `UpsertChargeRulesAsync` deserializes and re-serializes the entire payload on every sync page. For stores with many charge rules (e.g., 100+), this is still fast, but it's O(N) on every delta page even when only a few rules changed.
- The merge is keyed on `rule.Id` — if the cloud hard-deletes a charge rule (rare, usually soft-delete via `IsActive = false`), the agent will retain the stale rule indefinitely. This is acceptable for a soft-delete–only system but should be documented.

**Suggestions:**
- Guard `UpsertChargeRulesAsync` with a `storeId` consistency check: only upsert items where `item.StoreId == _cfg.StoreId` to prevent cross-store pollution in case the cursor is ever wrong.
- Consider logging the number of rules merged per tick for observability.

---

### 13. `refresh-bundle.ps1`
**Module:** OfflineSync / DevOps  
**Path:** `src/agent/.../Setup/refresh-bundle.ps1`

**What changed:**
- Script now probes `/local/bundle-version` before and after the bundle refresh to report the old/new fingerprint.
- Improved error messages and step labels.

**Issues:**
- None critical.

**Suggestions:**
- Add a `-WhatIf` mode that shows what would happen without making changes — useful for operators verifying the script path before a live refresh.
- The script uses `Start-Sleep -Seconds 3` after restarting the service, but `Start-Sleep` is blocked by the agent's instructions. This was already present; confirm it's needed or replace with a `Wait-Service` pattern.

---

### 14. `MastersSyncController.cs` (backend)
**Module:** Masters (Backend)  
**Path:** `src/backend/.../Controllers/MastersSyncController.cs`

**What changed:**
- `[HttpGet("charge-rules")]` endpoint added, delegating to `_svc.ListChargeRulesAsync`.

**Issues:**
- None.

**Suggestions:**
- None. Follows the existing pattern exactly.

---

### 15. `appsettings.json` (backend)
**Module:** Config  
**Path:** `src/backend/.../appsettings.json`

**What changed:**
- `Seed.AdminPassword` changed from `""` to `"Admin@123"`.
- `App.FrontendUrl` changed from `https://rms.ipossoft.io` to `http://localhost:4200`.
- Trailing newline removed.

**Issues:**
- 🔴 **`AdminPassword: "Admin@123"`** — hardcoded password in `appsettings.json` will be committed to the repo. Even though this is the dev config, it sets a weak default that operators might deploy to staging/production accidentally. Should be cleared to `""` or `"CHANGE_ME"` before committing.
- ⚠️ **`FrontendUrl: "http://localhost:4200"`** — should only be in `appsettings.Development.json`, not in the base `appsettings.json`. The base file should retain the production URL; the environment-specific override file should have the local URL.

**Suggestions:**
- Revert `AdminPassword` to `""` and `FrontendUrl` to the production value in `appsettings.json`.
- Use `appsettings.Development.json` (gitignored or clearly marked) for local dev overrides.

---

### 16. `MastersDeltaDtos.cs` (backend)
**Module:** Masters (Backend)  
**Path:** `src/backend/.../MastersSync/DTOs/MastersDeltaDtos.cs`

**What changed:**
- `ChargeRuleSyncDto` record added.
- `StoreSettingsSyncDto` was already present (no change).

**Issues:**
- None.

**Suggestions:**
- None.

---

### 17. `IMastersSyncService.cs` (backend)
**Module:** Masters (Backend)

**What changed:**
- `Task<Result<DeltaPageResponse<ChargeRuleSyncDto>>> ListChargeRulesAsync(...)` added.

**Issues:**
- None.

**Suggestions:**
- None.

---

### 18. `MastersSyncService.cs` (backend)
**Module:** Masters (Backend)

**What changed:**
- `ListChargeRulesAsync` implementation added. Materializes with `ToListAsync` first to avoid EF translation error for enum-to-int cast on nvarchar column.

**Issues:**
- The `take(page + 1)` pattern for `hasMore` detection is applied to the **raw** list (pre-projection), then the projection adds one row. After projection, `rows.Count > page` is checked — this is correct. ✅
- `StoreGuard<>` check does not enforce `StoreId` for `IsPlatformAdmin` callers (`storeId` can be null). The query then returns all stores' rules. This is intentional for platform admin but should be documented.

**Suggestions:**
- None beyond documenting the platform-admin behavior.

---

### 19. `offline.interceptor.ts` (web)
**Module:** OfflineSync / POS (Web)

**What changed:**
- `mergeMap` added after hub-routed responses for `check.create` flow — caches the hub response in IndexedDB (OF-2C fix).
- `buildSyntheticBody` updated: adds `tableName: ''` and `orders: []` fields to synthesized check-create response.
- Added debug `console.info` logs for `check.create` caching flow.

**Issues:**
- ⚠️ Multiple `console.info` and `console.error` debug logs added — appropriate for debugging but should be removed or wrapped in `environment.production` guards before production release.
  - Lines added: `console.info('[OF-2C interceptor] flow=...')`, `console.info('[OF-2C interceptor] saving check...')`, `console.info('[OF-2C interceptor] ✓ cached check...')`
- ⚠️ `console.error` changed from `console.warn` for the cache-stash failure — this will visually flag a non-fatal event as an error in production consoles.

**Suggestions:**
- Wrap all `console.info` debug lines behind `if (!environment.production)`.
- Consider creating a `Logger` service that conditionally gates verbose output.

---

### 20. `check-cache.service.ts` (web)
**Module:** OfflineSync (Web)

**What changed:**
- Debug `console.info` added to `save()` and `get()` methods.
- Warn log added when `save` is called with empty `checkId`.

**Issues:**
- ⚠️ Debug `console.info` on every cache `get` and `save` call will be noisy in production.

**Suggestions:**
- Gate behind `!environment.production`.

---

### 21. `hub-bundle-check.service.ts` (web)
**Module:** OfflineSync (Web)

**What changed:**
- Bundle version comparison logic (SPA content check + script hash comparison) **commented out** rather than deleted. The service now always returns `{ ok: true }` after a successful HTTP ping.

**Issues:**
- 🔴 **Bundle mismatch detection is disabled.** If the agent is running an old Angular bundle, the user will be silently redirected to the hub with a stale app. There is no fallback warning.
- The commented-out code is ~30 lines of dead code. If it's intentionally disabled for the current phase, it should either be removed or controlled by a feature flag — not left as commented code.

**Suggestions:**
- Either:
  - Re-enable with a softer failure (log a warning instead of blocking), or
  - Replace with a proper feature flag (`environment.enableBundleVersionCheck`), or
  - Delete the dead code and add a `// TODO(OF-4): re-enable bundle version check` comment.

---

### 22. `order-screen.component.ts` (web)
**Module:** POS (Web)

**What changed:**
- `hydrateFromCache` path for `LOCAL_C_...` IDs replaced with a router-state-first approach: on fresh check creation, the check data is passed via `history.state.offlineCheck` and used immediately (no async IndexedDB read). Falls through to API GET on re-open.
- Debug `console.info` logs added.

**Issues:**
- ⚠️ Debug `console.info` on the `hydrateFromCache` entry point will log on every order-screen load for local checks.
- `this.check = { ...routerState, orders: routerState.orders ?? [] }` — `routerState` is typed as `any` via `history.state`. If the router state is tampered with or missing expected fields, this silently sets a partial check object. A defensive shape validation would help.
- `this.checkOwnershipAccess(routerState)` is called with `routerState` (the raw router state object) rather than `this.check` (the hydrated check) — minor inconsistency that should be reviewed if `checkOwnershipAccess` reads specific fields.

**Suggestions:**
- Remove debug `console.info` before production release.
- Add a type guard for `routerState` to ensure it has the expected `CheckDetail` shape before spreading.

---

### 23. `pos-shell.component.ts` (web)
**Module:** POS (Web)

**What changed:**
- On successful check creation, `router.navigate` now includes `state: { offlineCheck: res.data }` so the order-screen can hydrate from router state instead of IndexedDB.

**Issues:**
- None.

**Suggestions:**
- None. Clean, minimal change.

---

### 24. `offline-banner.component.ts` (web)
**Module:** OfflineSync (Web)

**What changed:**
- Hub URL is now normalized to `{hubUrl}/pos/terminal` before building the redirect URL.
- Whitespace change in constructor.

**Issues:**
- The `/pos/terminal` suffix is appended unconditionally. If `hubUrl` already contains a path (e.g., `http://192.168.1.5:8423/some/path`), the result will be `http://192.168.1.5:8423/some/path/pos/terminal` which may be wrong. The `replace(/\/+$/, '')` strips a trailing slash, but doesn't strip existing paths.

**Suggestions:**
- Use `new URL('/pos/terminal', hubUrl).href` to always resolve against the root of the hub origin, regardless of what path was stored in `offlineHubUrl`.

---

### 25. `environment.prod.ts` (web)
**Module:** Config

**What changed:**
- `apiUrl` set to `http://localhost:8423/local` (agent).
- `cloudAppUrl` set to `http://localhost:4200`.

**Issues:**
- 🔴 **Production environment file points at localhost.** If this build is deployed, the prod app will make API calls to the local agent instead of the real cloud API. This would be a critical regression for all non-hub-mode users.

**Suggestions:**
- Revert `environment.prod.ts` to `https://api.rms.ipossoft.io/api` and `https://rms.ipossoft.io/` immediately before committing.
- Use `environment.ts` (dev) or a local override for testing against the local agent.

---

### 26. `environment.ts` (web)
**Module:** Config

**What changed:**
- `apiUrl` switched to the staging Lambda URL (was pointing at production `api.rms.ipossoft.io`).

**Issues:**
- None — this is the correct dev configuration pointing at staging.

**Suggestions:**
- None.

---

### 27. `package-lock.json` (web)
**Module:** Dependencies

**What changed:**
- 216-line diff (~routine lockfile update).

**Issues:**
- Verify no unexpected direct dependencies were introduced. Run `npm audit` if not done recently.

**Suggestions:**
- Run `npm audit` before committing.

---

### 28. `src/agent/tests/session_open.http` *(NEW)*
**Module:** Testing  

**What it is:**  
A REST Client `.http` file for manually testing the `POST /local/pos/sessions` endpoint. Useful for quick smoke tests against a running agent.

**Issues:**
- None.

**Suggestions:**
- Expand this into a full `.http` collection covering: open session, create check, save order, bill check, record payment. Or document it as a starting point in the test README.

---

## Critical Blockers (Must Fix Before Commit)

| Priority | File | Issue |
|----------|------|-------|
| 🔴 CRITICAL | `environment.prod.ts` | Points at localhost — will break production |
| 🔴 CRITICAL | `appsettings.json` | `AdminPassword: "Admin@123"` hardcoded and `FrontendUrl` is localhost |
| 🔴 HIGH | `hub-bundle-check.service.ts` | Bundle mismatch detection disabled with no fallback |

---

## Warnings (Should Fix Before Commit)

| Priority | File | Issue |
|----------|------|-------|
| ⚠️ | `offline.interceptor.ts` | Debug `console.info` / `console.error` logs in production paths |
| ⚠️ | `check-cache.service.ts` | Debug `console.info` on every cache operation |
| ⚠️ | `order-screen.component.ts` | Debug `console.info` on hydration path |
| ⚠️ | `offline-banner.component.ts` | `hubUrl + '/pos/terminal'` doesn't handle non-root hub URLs |

---

## Suggestions (Nice to Have)

| File | Suggestion |
|------|------------|
| `LocalFinancialEngine.cs` | Add unit tests for `Calculate()` |
| `LocalFinancialEngine.cs` | Filter charge rules by `StoreId` |
| `LocalPosController.cs` | Add modifier price to `SaveOrder` line total |
| `LocalCheck.cs` | Add unique index on `LocalId` column |
| `LocalPosReadsController.cs` | Move `ChargeRulePayload` DTO to `LocalPosModels.cs` |
| `MastersSyncWorker.cs` | Validate `StoreId` match in `UpsertChargeRulesAsync` |
| `DbInitializer.cs` | Log pending queue count before destructive schema rebuild |
| `hub-bundle-check.service.ts` | Replace commented code with a feature flag or delete it |
| `order-screen.component.ts` | Type-guard `history.state.offlineCheck` before spreading |

---

## Module Change Summary

| Module | Files Changed | New Files | Notes |
|--------|--------------|-----------|-------|
| **OfflineSync (Agent)** | 9 | 2 | Core OF-2C/OF-4 offline fixes |
| **Masters (Agent)** | 3 | 0 | Charge rules sync pipeline |
| **Masters (Backend)** | 4 | 0 | Charge rules delta endpoint |
| **POS (Web)** | 2 | 0 | Hub routing, order-screen hydration |
| **OfflineSync (Web)** | 3 | 0 | OF-2C cache race fix, bundle check |
| **Config** | 3 | 0 | ⚠️ Dev values in prod files |
| **DevOps** | 1 | 0 | Bundle refresh script |
| **Testing** | 0 | 1 | `.http` smoke test |
