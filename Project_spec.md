# Super Shop Management — Project Spec

> Stack: **Next.js + NestJS + Prisma + PostgreSQL + Redis/BullMQ + Socket.IO**  
> Large multi-branch supermarket: inventory (batch/expiry FIFO/FEFO), POS (offline-capable sync), purchase orders, suppliers, promotions, loyalty, and double-entry accounting ledger.

---

## 1. Overview & Goals

### Product
Enterprise-style supermarket operations system for multiple branches: centralized catalog, per-branch stock with batch/lot tracking, cashier POS with barcode scanning, promotions engine, loyalty points, supplier POs, and accounting ledger.

### Engineering goals
- **FEFO/FIFO batch allocation** with transactional stock movements.
- **Offline-first POS** sync protocol (idempotent sale upload, conflict rules).
- Promotions engine: compose stackable/non-stackable rules with deterministic pricing.
- Double-entry **ledger** for sales, COGS, payments, supplier bills.
- Multi-branch RBAC; real-time register status.

### Non-goals
- Full ERP manufacturing MRP.
- Hardware driver binaries (abstract barcode/printer/cashdrawer interfaces).

---

## 2. Roles & Permissions

| Role | Scope | Permissions |
|------|-------|-------------|
| `PLATFORM_ADMIN` | All | Branches, global catalog, tax, accounting periods |
| `BRANCH_MANAGER` | Branch | Stock adjustments, staff, local pricing overrides (if allowed), reports |
| `PURCHASING` | Org | Suppliers, POs, GRN |
| `CASHIER` | Branch register | Sell, refund (policy), hold carts |
| `INVENTORY_CLERK` | Branch | Receive stock, cycle count |
| `ACCOUNTANT` | Org | Ledger, close periods, exports |
| `CUSTOMER` (loyalty app) | Self | Points, coupons |

Permissions stored as `Permission.code` strings; `RolePermission` join; branch membership `UserBranchRole`.

---

## 3. Feature Modules

1. **Org & Branches** — company, branches, registers, tax rates, currencies (BDT primary).
2. **Catalog** — products, variants, barcode (EAN/UPC), categories, UOM, tax class, images.
3. **Inventory** — batches (`lotNumber`, `expiryDate`, `receivedAt`, `costMinor`), stock per branch+batch, movements (`PURCHASE`, `SALE`, `ADJUST`, `TRANSFER`, `RETURN`, `WASTE`).
4. **Purchasing** — suppliers, PO workflow, goods receipt notes (GRN), invoice matching.
5. **POS** — cart, barcode add, promotions apply, split tender (cash/card/MFS), receipts, parks/holds, refunds/exchanges.
6. **Offline sync** — device queue, vector clock / monotonic `clientSaleId`, server apply.
7. **Promotions** — rules engine (see §4).
8. **Loyalty** — tiers, earn/burn, coupons.
9. **Transfers** — inter-branch stock transfer orders.
10. **Accounting** — chart of accounts, journal entries, period lock.
11. **Reports** — sales flash, shrinkage, ageing expiry, dead stock, GP%.
12. **Audit** — immutable event log for price overrides and voided lines.

---

## 4. Advanced Business Logic

### 4.1 FEFO allocation on sale
```
need = qty
batches = stockBatches(branch, product)
  .filter(availableQty > 0)
  .sort(by expiryDate ASC NULLS LAST, then receivedAt ASC)  // FEFO then FIFO
for b in batches:
  take = min(need, b.availableQty)
  create StockMovement SALE linked to SaleLineBatch
  decrement b.availableQty
  need -= take
  if need == 0: break
if need > 0: fail INSUFFICIENT_STOCK (unless allow negative stock flag for managers)
```
Run inside serializable or row-locked transaction on batch rows `FOR UPDATE`.

### 4.2 Promotions engine
Rule types: `PERCENT_OFF_ITEM`, `AMOUNT_OFF_ITEM`, `BXGY`, `BUNDLE`, `CART_PERCENT`, `CART_AMOUNT`, `CATEGORY_PERCENT`, time window, min qty, customer segment, payment method.
Evaluation order:
1. Load active campaigns for branch/now.
2. Expand cart lines.
3. Apply non-stackable best-of per line (max saving) OR stackable sequence by `priority`.
4. Cap discounts; never below cost if `preventBelowCost`.
5. Output priced cart + applied promotion ids (audit).

Determinism: same cart + same campaigns → same result (sort rules by priority, id).

### 4.3 Offline POS sync
- Each sale has `clientSaleId` (UUID) unique per register; server upserts by idempotency.
- Device stores ops in IndexedDB/SQLite; sync when online.
- Conflicts: if batch insufficient at sync time → `Sale.status=PENDING_REVIEW`, do not silently drop; notify manager.
- Clock skew: server `receivedAt` authoritative; client timestamps stored for audit only.
- Sequence: register sends `lastAckSeq`; server returns new events (price book updates, void commands).

### 4.4 COGS & ledger on sale
```
Dr Cash/Card clearing   gross
Dr Discount expense     discount   (or contra revenue)
Cr Sales revenue        net sales
Dr COGS                 sum(batch.cost * qty)
Cr Inventory asset      sum(batch.cost * qty)
```
Period-locked journals cannot be posted.

### 4.5 Edge cases
- Partial refund: restock to original batches if known else new RETURN batch.
- Expiry waste job nightly: propose write-off for expired batches.
- Price override requires manager PIN + reason code.
- Barcode collision across products → reject at catalog create.
- Weighted items (kg): qty decimal(12,3); barcode may embed price (document parse rules).

---

## 5. State Machines

**PurchaseOrder**: `DRAFT → SUBMITTED → APPROVED → PARTIAL_RECEIVED → RECEIVED → CLOSED | CANCELLED`  
**TransferOrder**: `DRAFT → IN_TRANSIT → RECEIVED | CANCELLED`  
**Sale**: `OPEN → PARKED → COMPLETED | VOID` ; offline may `COMPLETED_LOCAL → SYNCED | PENDING_REVIEW`  
**PromotionCampaign**: `DRAFT → SCHEDULED → ACTIVE → ENDED | CANCELLED`  
**AccountingPeriod**: `OPEN → CLOSED → LOCKED`

---

## 6. Prisma Schema Draft (core)

```prisma
model Organization { id String @id @default(cuid()) name String }
model Branch {
  id String @id @default(cuid())
  orgId String
  code String
  name String
  timezone String @default("Asia/Dhaka")
  @@unique([orgId, code])
}
model Register {
  id String @id @default(cuid())
  branchId String
  code String
  devicePublicKey String?
  @@unique([branchId, code])
}
model Product {
  id String @id @default(cuid())
  orgId String
  sku String
  name String
  barcode String?
  uom String @default("EA")
  taxClassId String?
  trackBatch Boolean @default(true)
  isWeighted Boolean @default(false)
  @@unique([orgId, sku])
  @@unique([orgId, barcode])
}
model ProductPrice {
  id String @id @default(cuid())
  productId String
  branchId String?
  priceMinor Int
  currency String @default("BDT")
  validFrom DateTime
  validTo DateTime?
}
model StockBatch {
  id String @id @default(cuid())
  branchId String
  productId String
  lotNumber String?
  expiryDate DateTime? @db.Date
  receivedAt DateTime
  costMinor Int
  qtyOnHand Decimal @db.Decimal(14,3)
  qtyReserved Decimal @db.Decimal(14,3) @default(0)
  @@index([branchId, productId, expiryDate])
}
model StockMovement {
  id String @id @default(cuid())
  batchId String?
  branchId String
  productId String
  type String
  qty Decimal @db.Decimal(14,3)
  refType String?
  refId String?
  createdAt DateTime @default(now())
  @@index([branchId, productId, createdAt])
}
model Supplier { id String @id @default(cuid()) orgId String name String }
model PurchaseOrder {
  id String @id @default(cuid())
  orgId String
  branchId String
  supplierId String
  status String
  lines Json
}
model GoodsReceipt {
  id String @id @default(cuid())
  poId String
  receivedAt DateTime @default(now())
}
model Sale {
  id String @id @default(cuid())
  branchId String
  registerId String
  cashierId String
  clientSaleId String
  status String
  totalMinor Int
  discountMinor Int
  taxMinor Int
  currency String @default("BDT")
  completedAt DateTime?
  syncState String @default("SYNCED")
  @@unique([registerId, clientSaleId])
}
model SaleLine {
  id String @id @default(cuid())
  saleId String
  productId String
  qty Decimal @db.Decimal(14,3)
  unitPriceMinor Int
  lineDiscountMinor Int
}
model SaleLineBatch {
  id String @id @default(cuid())
  saleLineId String
  batchId String
  qty Decimal @db.Decimal(14,3)
  costMinor Int
}
model PromotionCampaign {
  id String @id @default(cuid())
  orgId String
  name String
  status String
  stackable Boolean @default(false)
  priority Int @default(100)
  startsAt DateTime
  endsAt DateTime
  rules Json
}
model LoyaltyAccount {
  id String @id @default(cuid())
  customerId String @unique
  points Int @default(0)
  tier String @default("STANDARD")
}
model LedgerAccount {
  id String @id @default(cuid())
  orgId String
  code String
  name String
  type String // ASSET LIABILITY EQUITY REVENUE EXPENSE
  @@unique([orgId, code])
}
model JournalEntry {
  id String @id @default(cuid())
  orgId String
  periodId String
  memo String?
  postedAt DateTime @default(now())
  sourceType String?
  sourceId String?
}
model JournalLine {
  id String @id @default(cuid())
  entryId String
  accountId String
  debitMinor Int @default(0)
  creditMinor Int @default(0)
}
model AccountingPeriod {
  id String @id @default(cuid())
  orgId String
  name String
  startsOn DateTime @db.Date
  endsOn DateTime @db.Date
  status String
}
```

Add partial checks: journal lines balanced (`SUM(debit)=SUM(credit)`) enforced in service before post.

---

## 7. REST API (high level)

Modules: Auth, Org, Catalog, Inventory, Purchasing, Pos, Sync, Promotions, Loyalty, Accounting, Reports.

Key endpoints:
- `POST /pos/sales` (idempotent `Idempotency-Key` or `clientSaleId`)
- `POST /pos/sales/:id/refund`
- `POST /sync/push` / `GET /sync/pull?sinceSeq=`
- `POST /inventory/adjustments`
- `POST /inventory/transfers`
- `POST /purchasing/pos` … `/receive`
- `POST /promotions/preview` (dry-run price cart)
- `GET /reports/expiry-ageing`
- `POST /accounting/journals`
- `POST /accounting/periods/:id/close`

---

## 8. Real-time

- `register:{id}`: remote void, price book bump, manager approve overlay.
- `branch:{id}`: low-stock alerts, PO received.

---

## 9. Jobs

- Expiry scan / waste suggestions
- Loyalty point expiry
- Nightly sales aggregation
- Sync retry for PENDING_REVIEW escalation
- Period close reminders

---

## 10. Security

- Register device pairing with rotating device tokens.
- Manager override PIN hashed; step-up for voids > threshold.
- Branch data isolation in queries.
- PCI: never store full PAN; tokenized card via provider.

---

## 11. Caching

- Redis price book + active promotions per branch (invalidate on publish).
- Product barcode lookup cache.

---

## 12. Testing

- FEFO allocation unit tests with mixed expiry.
- Promo engine property tests (non-negative, deterministic).
- Parallel POS sales depleting same batch → no negative without flag.
- Offline sync idempotency (duplicate clientSaleId).
- Journal balance invariant.

---

## 13. Folder Structure

```
super-shop-management/
  Project_spec.md
  frontend/   # admin + POS PWA (Next.js)
  backend/
    prisma/
    src/modules/{auth,catalog,inventory,pos,sync,promotions,purchasing,loyalty,accounting,reports}
```

POS frontend route group `(pos)` should work offline-capable (service worker).

---

## 14. Env vars (names only)

`DATABASE_URL`, `REDIS_URL`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `APP_URL`, `CORS_ORIGINS`, `S3_*`, `SMTP_URL`, `SMS_*`, `POS_OFFLINE_MAX_HOURS`, `LOYALTY_POINT_VALUE_MINOR`, `DEFAULT_TAX_RATE_BPS`

---

## 15. Milestones

1. Catalog + branches + auth  
2. Batches + movements + FEFO  
3. Online POS + receipts  
4. Promotions engine  
5. Purchasing + GRN  
6. Offline sync protocol  
7. Loyalty + ledger + reports  

---

## 16. Acceptance Criteria

- [ ] Selling 5 units allocates from earliest expiry batches first.
- [ ] Duplicate `clientSaleId` does not double-decrement stock.
- [ ] BXGY and cart percent compose per stackable rules deterministically.
- [ ] Closed accounting period rejects new journals.
- [ ] Inter-branch transfer reduces A and increases B only after receive.
- [ ] PWA POS can complete a sale offline and sync later.
- [ ] Low stock and expiry reports return correct ageing buckets.
- [ ] Cashier cannot post inventory adjustments.

---

## 17. Agent Notes

- Implement `InventoryService.allocateFeFo` as single chokepoint for outbound stock.
- `PricingService.preview(cart, context)` pure function used by POS and promotions admin simulator.
- Use Decimal.js / Prisma Decimal; never JS number for money/qty.
- Seed: 2 branches, 200 products, batches with staggered expiry, 3 cashiers, sample promos.
