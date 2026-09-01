# [STORE Backend] Database Schema Updates — SAM-415

## New Schema

### Refund Table — Add Columns

- [ ] **`nta_version`**: `"v3"` for new-version transactions. Nullable (`null` = v2).
- [ ] **`customs_check_kbn`**: `1` or `9`. `1` = OK (confirmed), `9` = rejected.
- [ ] **`customs_check_date`**: `yyyymmdd`. The date `customs_check_kbn` was confirmed.
- [ ] **`customs_check_regi_date`**: `yyyymmddhhMMss`. The datetime this customs-check information was last updated.
- [ ] **`customs_checked_at`**: timestamp. The datetime *we* ingested this data. Audit/investigation use only (see below — **low priority**).
- [ ] **`sell_date`**: `yyyymmdd`. Sale/transfer date. **Required — see reasoning below.**

Going forward, `expired_at` will be calculated based on `customs_check_date`. For that reason, all customs-check-related return-value columns will be stored on the `refund` table — not on `RefundRecord`.

#### Why add `sell_date`

The sale/transfer date currently exists only inside `RefundRecord.request_body` (encrypted), so **it can't be filtered via SQL.** In v3, three separate features need to filter/calculate based on this date:

1. **Customs-confirmation expiry batch** ([SAM-450](https://linear.app/tai-matsu/issue/SAM-450/store-backendexpire-unconfirmed-v3-refunds-after-90-days-with-a-new)) — catch unconfirmed refunds where 90 days have passed since the sale date
2. **Cancel-eligibility check** ([SAM-442](https://linear.app/tai-matsu/issue/SAM-442/dashboard-frontendgate-the-cancel-button-by-nta-version-v3-allows) / [SAM-451](https://linear.app/tai-matsu/issue/SAM-451/admin-frontendgate-the-cancel-button-by-nta-version-v3-allows)) — per spec 別紙5-3 error D0022 (must be within 90 days of sale date)
3. **Settlement date filtering** ([SAM-449](https://linear.app/tai-matsu/issue/SAM-449/admin-backendsettlement-reporting-must-exclude-cancellations-and-for))

#### Why add `customs_checked_at`

**Originally noted as "required for filtering the TRJ daily batch" — that was incorrect.** The TRJ daily batch issue ([SAM-453](https://linear.app/tai-matsu/issue/SAM-453/admin-backendtrj-daily-batch-stops-working-from-2027-01-01-the-trj)) is fixed simply by hardcoding a lower-bound constant, so this column isn't needed for that.

The real reason to add it: audit/investigation purposes. "The day NTA confirmed it" (`customs_check_date`) and "the moment we found out about it" are two different things — the latter is needed when investigating ingestion delays or missed records.

**Low priority — safe to drop if you want to keep the migration smaller.**

#### Type decisions

`customs_check_kbn` only has two values today (`1`/`9`), but **recommend keeping it as `text`, not an enum** — to leave room for NTA adding values in the future.

Date fields should generally be **stored as the raw string exactly as received from NTA** (parsing failures shouldn't block ingestion). The exception is `sell_date` — the expiry batch needs to do date arithmetic on it (`sell_date + 90 days`), so a real `date` type may be easier to work with. **Decide this together with [SAM-450](https://linear.app/tai-matsu/issue/SAM-450/store-backendexpire-unconfirmed-v3-refunds-after-90-days-with-a-new)'s implementation approach.**

#### Backfill

All columns nullable. **No backfill of existing records.** v2 refunds have no concept of customs confirmation, and `sell_date` isn't used in v2 either. Only new v3 refunds will populate these going forward.

#### Model definitions live in two repos

Defined separately in the customer backend (`RefundApp/models.py`) and the admin backend (`AdminApp/models.py`). **The migration itself lives in STORE Backend, but the model definitions need to be updated in both repos.**

Also worth noting: there's already an inconsistency in `RefundRecord.send_no`'s constraint between the two repos (admin side: `unique=True`; customer side: `UNIQUE(shop_id, send_no)`). Worth considering whether to align these while we're in here.

---

### Custom Check Period Run Table (`customs_check_period_runs`) — New Table

- [ ] `id`
- [ ] `sender_id`: sender identification code (`senders.sender_id`)
- [ ] `(shop_id?)`: for SAMURAI TAX, specifying `sender_id` alone might be sufficient
- [ ] `period_from`: `yyyymmddhhMMss`. Confirmation-datetime range — from
- [ ] `period_to`: `yyyymmddhhMMss`. Confirmation-datetime range — to
- [ ] `status`: `str = Field(default="running", nullable=False, index=True)` — `running` / `succeeded` / `failed`
- [ ] `result`
- [ ] `customs_check_num`
- [ ] `error_code`
- [ ] `request_body`
- [ ] `response_body`

---

## Metadata

| Field | Value |
|---|---|
| URL | [SAM-415](https://linear.app/tai-matsu/issue/SAM-415/store-backenddatabase-schema-updates) |
| Identifier | SAM-415 |
| Status | Backlog |
| Priority | High |
| Assignee | navin.gurung@tai-matsu.jp |
| Labels | STORE Backend |
| Project | [NTA System API V3](https://linear.app/tai-matsu/project/nta-system-api-v3-ec8115730c96/overview) — NTA v3 migration. Go-live 2026-11-01. v2 runs in parallel until 2027-04-30. See project overview for sequencing, dependencies, and any un-filed gaps. |
| Project milestone | Complete Database Migration and small updates |
| Related issues | SAM-453, SAM-445, SAM-449, SAM-426 |
| Blocking | SAM-421, SAM-451, SAM-450, SAM-442, SAM-441, SAM-446 |
| Created | 2026-08-30 |
| Updated | 2026-08-31 |
