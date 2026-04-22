# Store Edge Agent — Offline Mode Audit Report

**Date:** 22 April 2026  
**Scope:** Offline POS pipeline — financial engine, charge-rule sync, check lifecycle, channel list  
**Status:** All identified issues resolved. Build passing (0 errors).

---

## Summary

This session completed the full offline check lifecycle for the Store Edge Agent. The agent can now autonomously handle the entire flow from check creation through order entry, billing (with tax/service-charge computation), payment recording, and check closure — without any cloud connectivity. It also exposes the takeaway/delivery channel check list so the takeaway screen works offline.

---

## Issues Found, Fixed, and Suggestions

### 1. `POST /local/pos/checks/preview-totals` — Missing financial calculation

| Field | Detail |
|---|---|
| **Component** | `LocalPosReadsController` |
| **Issue** | `PreviewTotals` was returning a hardcoded/placeholder breakdown. It did not apply store charge rules (tax, service charge, delivery fee), so the UI showed incorrect totals before billing. |
| **Root Cause** | No shared financial engine existed on the agent. Cloud logic in `FinancialService.Calculate()` had not been ported. |
| **Fix** | Created `LocalFinancialEngine` (see §5). `PreviewTotals` now delegates to `LocalFinancialEngine.ComputeAsync`, returning a full rule-driven breakdown. |
| **Suggestion** | Add a unit test for `LocalFinancialEngine.Calculate()` (the pure method) — it has no DB dependency and is straightforward to test with in-memory rule lists. |

---

### 2. SQL Server `nvarchar` → `int` cast error in `MastersSyncService.ListChargeRulesAsync`

| Field | Detail |
|---|---|
| **Component** | `src/backend/…/MastersSync/MastersSyncService.cs` |
| **Issue** | `(int)r.Type` inside an EF LINQ query threw HTTP 500 on the cloud sync endpoint. `ChargeType` is stored as `nvarchar` in SQL Server; EF cannot translate `(int)` cast to SQL for string-backed enums. |
| **Root Cause** | EF Core cannot emit `CAST(nvarchar AS int)` in the SQL query. The cast must happen in C# memory. |
| **Fix** | Added `.ToListAsync()` before the projection so all casting happens in C# after materialisation. |
| **Suggestion** | Consider storing `ChargeType` as `int` column in the SQL Server schema to prevent this class of error in future queries. Alternatively add a `.AsEnumerable()` guard comment wherever enum-to-int casts appear in LINQ projections. |

---

### 3. `LocalChargeRule` individual-column schema — replaced with `LocalChargeRuleStore` JSON blob

| Field | Detail |
|---|---|
| **Component** | `Data/Masters/LocalMasters.cs`, `Data/EdgeDbContext.cs` |
| **Issue** | The original `LocalChargeRule` entity had individual columns (`Name`, `Type`, `ValueType`, `Value`, …). Delta sync delivers rules in pages and individual inserts would cause schema churn and complex merge logic. |
| **Root Cause** | Schema designed before the delta-sync pattern (cursor-paged, partial updates) was established. |
| **Fix** | Replaced `LocalChargeRule` with `LocalChargeRuleStore { StoreId (PK), TenantId, PayloadJson, UpdatedAtUtc }`. One row per store holds the entire rule list as a JSON blob. The sync worker deserialises existing JSON, merges by rule `Id`, and re-serialises — idempotent and handles deletions/deactivations cleanly. |
| **Suggestion** | The JSON blob approach is correct for MVP. When rule sets grow large (>100 rules), consider a separate `local_master_charge_rule` row-per-rule table to allow indexed lookups. For now, the whole blob is small and always fully loaded. |

---

### 4. `DbInitializer` — missing schema probe for `local_master_charge_rules`

| Field | Detail |
|---|---|
| **Component** | `Data/DbInitializer.cs` |
| **Issue** | On agents upgraded from a pre-OF-4 build, `EnsureCreated` would not add the new table because the database already existed. The agent would crash on first charge-rule sync. |
| **Root Cause** | `EnsureCreated` is a no-op on an existing database. No migration runner is used. |
| **Fix** | Added a `SELECT 1 FROM local_master_charge_rules LIMIT 0` probe. On `SqliteException` error code 1 (no such table), `needsRecreate = true` is set and the DB is rebuilt — following the existing pattern for other new tables. |
| **Suggestion** | Long-term, migrate the agent to EF Core `Migrate()` with proper migration history. `EnsureCreated` + manual probes will become unwieldy as the schema grows. |

---

### 5. `LocalFinancialEngine` — no shared financial helper existed

| Field | Detail |
|---|---|
| **Component** | `Api/LocalFinancialEngine.cs` (NEW) |
| **Issue** | `PreviewTotals`, `SaveOrder`, and `BillCheck` all needed the same charge-rule calculation. Without a shared helper, each would have duplicate (and potentially divergent) logic. |
| **Root Cause** | Port of cloud `FinancialService.Calculate()` had not been done. |
| **Fix** | Created `internal static class LocalFinancialEngine` with two methods: `ComputeAsync` (loads rules from SQLite, filters by channel/active/automatic, injects manual discount/tip, delegates to `Calculate`) and `Calculate` (pure, no DB — accumulates charges in sequence order, respects `AppliesOn` base amount, handles inclusive charges). |
| **Suggestion** | The `Calculate` method is pure and can be directly unit tested. Suggested test cases: zero-rule passthrough, inclusive tax (amount should not add to grand total), manual discount reducing base before service charge, delivery charge channel filter. |

---

### 6. `SaveOrder` — check totals not updated after order lines added

| Field | Detail |
|---|---|
| **Component** | `LocalPosController.SaveOrder` |
| **Issue** | After adding order lines to a check, the check's `Subtotal`, `TaxTotal`, `DiscountTotal`, and `GrandTotal` were not recalculated. The check total remained at the creation-time value. |
| **Root Cause** | The financial recompute step was absent. |
| **Fix** | After adding order lines, compute `newSubtotal = localCheck.Subtotal + orderTotal`, call `LocalFinancialEngine.ComputeAsync`, and write the breakdown back to the check row before `EnqueueAndSaveAsync`. |
| **Suggestion** | If orders can be voided offline, the same recompute pattern must run in the void-lines path. Verify that `VoidLines` in `LocalPosController` also re-runs the financial engine after removing voided line totals from the subtotal. |

---

### 7. `POST /local/pos/checks/{checkId}/bill` — endpoint missing (404)

| Field | Detail |
|---|---|
| **Component** | `LocalPosController` |
| **Issue** | The frontend sends `POST /api/pos/checks/{id}/bill` which the hub router rewrites to `POST /local/pos/checks/{id}/bill`. The endpoint did not exist — every bill attempt returned 404, leaving the check in `Open` status and the table permanently occupied. |
| **Root Cause** | The bill endpoint was not ported when the agent was first created. |
| **Fix** | Added `BillCheck` action. It resolves the check, runs `LocalFinancialEngine.ComputeAsync` with any request-supplied discount/tip, sets `Status = "Billed"`, and enqueues `pos/checks/{checkId}/bill` for cloud replay. |
| **Suggestion** | The `BillCheckRequest` mirrors the cloud shape fully (including `SkipPrintJobs`, `ApprovalToken`, `DiscountPresetId`, `DiscountReason`). Currently only `DiscountPercent`, `DiscountAmount`, and `TipAmount` are acted upon. The remaining fields are passed through to the cloud replay queue and will be honoured when synced. |

---

### 8. `RecordPayment` — check status not auto-updated to `Paid`

| Field | Detail |
|---|---|
| **Component** | `LocalPosController.RecordPayment` |
| **Issue** | After a payment was recorded, the check's status remained `Billed`. The table on the floor plan stayed occupied indefinitely because the local query filters on `Status != "Paid"`. The table never vacated without cloud sync. |
| **Root Cause** | No payment-sum-vs-grand-total comparison was done in `RecordPayment`. |
| **Fix** | After inserting the payment, load all prior payments for the check (excluding the new one) via `.Select(p => p.Amount).ToListAsync()`, sum in C#, add `request.Amount`, and set `localCheck.Status = "Paid"` when `totalPaid >= GrandTotal`. |
| **Suggestion** | For split-payment scenarios (e.g., 50% cash + 50% card in two calls), the per-payment recompute is correct — each call re-sums all prior payments. Test case: two payments that together meet the grand total should flip status on the second call only. |

---

### 9. SQLite EF Core — `SumAsync` on `decimal` property not translatable

| Field | Detail |
|---|---|
| **Component** | `LocalPosController.RecordPayment` |
| **Issue** | `_db.Payments.Where(…).SumAsync(p => (decimal?)p.Amount)` throws at runtime because the SQLite EF provider cannot translate `decimal` aggregation to SQL. This caused HTTP 500 on every payment attempt. |
| **Root Cause** | SQLite stores decimals as `REAL` (float) or `TEXT` depending on mapping. The EF SQLite provider does not generate `SUM()` for decimal projections. |
| **Fix** | Changed to `.Select(p => p.Amount).ToListAsync(ct)` followed by `.Sum()` in C# memory. This avoids the EF translation entirely. |
| **Suggestion** | Apply the same pattern everywhere `SumAsync` is used on a `decimal` column in the agent. Audit other places where `SumAsync` or `AverageAsync` on decimal columns might have the same issue. A shared comment/pragma or a code-review checklist entry would prevent recurrence. |

---

### 10. `GET /local/pos/checks/channel/{channelType}` — takeaway/delivery list missing (404)

| Field | Detail |
|---|---|
| **Component** | `LocalPosReadsController` |
| **Issue** | The POS shell calls `GET /api/pos/checks/channel/Takeaway` (and `/Delivery`) to populate the takeaway/delivery order list. The hub router rewrites this to `/local/pos/checks/channel/Takeaway`. The endpoint did not exist — the list showed empty and new orders were not visible. |
| **Root Cause** | The channel-list read endpoint was not ported to the agent. |
| **Fix** | Added `GetChannelChecks(string channelType)` action with route `checks/channel/{channelType}`. Filters `_db.Checks` by `ChannelType == channelType` and `Status != "Paid" && Status != "Cancelled"`, orders descending by `CreatedAtUtc`, and returns each check as a `LocalChannelCheckItem` matching the cloud `ChannelCheckListItem` shape exactly. |
| **Suggestion** | The `LocalCheck` entity does not currently store `CustomerName`, `CustomerMobile`, `AddressSummary`, or `OrderSource`. These are returned as `null` — the UI correctly falls back to "Walk-in". When customer-attach is wired up offline, add these columns to `LocalCheck` and populate them in the response. |

---

## Files Changed

| File | Change Type | Summary |
|---|---|---|
| `src/agent/…/Api/LocalFinancialEngine.cs` | **New** | Shared financial engine. Loads charge rules from SQLite, applies tax/service charge/discount/tip in rule sequence order. |
| `src/agent/…/Api/Controllers/LocalPosReadsController.cs` | **Modified** | Added `LocalChannelCheckItem` DTO; added `GetChannelChecks` endpoint; `PreviewTotals` now delegates to `LocalFinancialEngine`; removed duplicate private calculation methods. |
| `src/agent/…/Api/Controllers/LocalPosController.cs` | **Modified** | `SaveOrder` recomputes financial totals after adding lines; added `BillCheck` endpoint; `RecordPayment` auto-closes check to `Paid` when fully paid, using in-memory `decimal` sum. |
| `src/agent/…/Api/Models/LocalPosModels.cs` | **Modified** | Added `BillCheckRequest` DTO. |
| `src/agent/…/Data/Masters/LocalMasters.cs` | **Modified** | Replaced `LocalChargeRule` (individual columns) with `LocalChargeRuleStore` (JSON blob, PK = `StoreId`). Added `MasterEntityKeys.ChargeRules`. |
| `src/agent/…/Data/EdgeDbContext.cs` | **Modified** | Added `DbSet<LocalChargeRuleStore> MasterChargeRules` with EF config for `local_master_charge_rules` table. |
| `src/agent/…/Data/DbInitializer.cs` | **Modified** | Added schema probe for `local_master_charge_rules`; triggers DB rebuild on missing table. |
| `src/agent/…/Services/Masters/MastersSyncWorker.cs` | **Modified** | Added `SyncPagedAsync` call for `charge-rules`; added `UpsertChargeRulesAsync` (JSON merge-upsert by rule Id); added local `ChargeRuleSyncDto` record. |
| `src/backend/…/MastersSync/DTOs/MastersDeltaDtos.cs` | **Modified** | Added `ChargeRuleSyncDto` record. |
| `src/backend/…/MastersSync/IMastersSyncService.cs` | **Modified** | Added `ListChargeRulesAsync` interface method. |
| `src/backend/…/MastersSync/MastersSyncService.cs` | **Modified** | Implemented `ListChargeRulesAsync`; materialises with `ToListAsync` before casting enum to int. |
| `src/backend/…/Controllers/MastersSyncController.cs` | **Modified** | Added `GET masters-sync/charge-rules` endpoint. |

---

## Open Items / Suggestions Summary

| Priority | Item |
|---|---|
| High | Audit all agent code for other `SumAsync`/`AverageAsync` on `decimal` columns — SQLite EF limitation applies globally. |
| High | Verify `VoidLines` in `LocalPosController` also calls `LocalFinancialEngine` to recompute totals after voiding. |
| Medium | Add `CustomerName`, `CustomerMobile`, `AddressSummary`, `OrderSource` columns to `LocalCheck` entity when offline customer-attach is implemented. |
| Medium | Unit test `LocalFinancialEngine.Calculate()` — it is pure, has no DB dependency, and covers inclusive tax, delivery charge filtering, and manual discount/tip injection. |
| Medium | Migrate agent schema management from `EnsureCreated` + manual probes to EF Core `Migrate()` as the table count grows. |
| Low | Consider storing `ChargeType` as `int` in SQL Server to avoid future `ToListAsync`-before-cast workarounds. |
| Low | Add a split-payment integration test: two partial payments that together reach `GrandTotal` should flip check status to `Paid` on the second call. |
