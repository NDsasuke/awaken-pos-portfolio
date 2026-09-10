# Features

An honest breakdown of what exists. Anything not marked otherwise is implemented and in use.

**Legend:** ✅ Implemented · 🟡 Partially implemented · ⬜ Planned, not built

---

## Point of sale

| Feature | Status |
|---|---|
| Till screen with live product search | ✅ |
| Barcode scanning — hardware scanner, device camera, manual entry | ✅ |
| Line-level and cart-level discounts | ✅ |
| Manual price override, recorded against the operator | ✅ |
| Cash and credit payment methods | ✅ |
| Sequential per-shop, per-day receipt numbering | ✅ |
| Duplicate-submission protection | ✅ |
| Void with separate authorisation, recording both parties | ✅ |
| Quotations, convertible to sales | ✅ |
| Thermal receipt printing | ✅ |
| Full-page bill printing | ✅ |
| Silent printing without a print dialog | 🟡 Desktop only |
| Sales returns and refunds | ⬜ |

## Products and catalogue

| Feature | Status |
|---|---|
| Products, categories, sub-categories | ✅ |
| Per-shop units of measure | ✅ |
| Unit-priced goods and loose goods sold by weight or volume | ✅ |
| Category-driven custom product fields, inherited down the category tree | ✅ |
| Bulk tax reclassification | ✅ |
| Barcode and QR label sheet printing | ✅ |
| Image optimisation on upload | ✅ |
| Sinhala and transliterated search | ✅ |
| Bulk catalogue import for an established shop | ⬜ |
| Bulk product editing | ⬜ |
| Product variations | ⬜ |
| Per-unit serial / IMEI tracking | ⬜ |
| Item versus service product types | ⬜ |

## Inventory

| Feature | Status |
|---|---|
| Costed stock batches with purchase and expiry dates | ✅ |
| Oldest-batch-first consumption | ✅ |
| Stock intake by CSV file | ✅ |
| Stock intake by manual entry | ✅ |
| Manual stock decrease for damage, loss and correction | ✅ |
| Stock movement history | ✅ |
| Low-stock reporting | ✅ |
| Expiring-product reporting | ✅ |
| Automated stock consistency checking | ✅ |

## Customers and suppliers

| Feature | Status |
|---|---|
| Customer accounts | ✅ |
| Credit sales with outstanding balance tracking | ✅ |
| Payment collection with allocation across outstanding sales | ✅ |
| Supplier records and delivery history | ✅ |
| Supplier payables tracking and payment recording | ✅ |
| Instalment terms on purchases | ⬜ |

## Reporting and exports

| Report | Status |
|---|---|
| Daily sales | ✅ |
| Monthly sales | ✅ |
| Category-wise sales | ✅ |
| Sales report | ✅ |
| Cost report | ✅ |
| Profit report | ✅ |
| Stock valuation | ✅ |
| Profit-leak detector | ✅ |
| Inventory intelligence | ✅ |
| Weekly business summary | ✅ |
| Tax report by rate band | ✅ |
| CSV data exports (nine datasets) | ✅ |
| Background computation for heavy reports | ✅ |
| Purchase, expense, income, profit & loss, annual reports | ⬜ |

Planned reports appear in the product as clearly labelled placeholders rather than dead links, so
a shop is never left clicking something that silently does nothing.

## Tax

| Feature | Status |
|---|---|
| Per-product and per-category tax classification | ✅ |
| Classification inheritance from category to product | ✅ |
| Tax-inclusive pricing | ✅ |
| Tax disclosure on receipts and bills | ✅ |
| Tax figures frozen onto the sale record | ✅ |
| Tax estimates on quotations, labelled as estimates | ✅ |

## Users, roles and access

| Feature | Status |
|---|---|
| Shop-defined roles with a seniority ordering | ✅ |
| Per-feature permissions enforced in the request pipeline | ✅ |
| Permission delegation bounded by the granter's own access | ✅ |
| User deactivation, effective on the next request | ✅ |
| Retention of users with transaction history, for audit integrity | ✅ |
| Staff seat limits by subscription plan | ✅ |
| A distinct role-management permission and delegation seniority cap | ⬜ |

## Audit

| Feature | Status |
|---|---|
| Shop activity log — price overrides, stock movements, voids, access changes | ✅ |
| Separate administrative audit log for operator actions | ✅ |
| Append-only sales — no edit or delete route exists | ✅ |
| Activity log export | ✅ |
| Configurable retention windows by record type | ✅ |

## Subscriptions and platform administration

| Feature | Status |
|---|---|
| Subscription plans gating features and seats | ✅ |
| Trials, expiry and renewal reminders | ✅ |
| Manual payment recording and approval | ✅ |
| Generated PDF invoices | ✅ |
| Operator administration area | ✅ |
| Shop announcements | ✅ |
| Support ticketing | ✅ |
| Release notes | ✅ |
| Administrator-editable email templates | ⬜ |

## Security

| Feature | Status |
|---|---|
| Session authentication | ✅ |
| Google OAuth sign-in | ✅ |
| Email OTP multi-factor authentication for administrator accounts | ✅ |
| Password reset with verification | ✅ |
| Rate limiting on authentication and other sensitive endpoints | ✅ |
| Security header set including a nonce-based Content Security Policy | ✅ |
| Correct client identification behind a reverse proxy | ✅ |

## Operations

| Feature | Status |
|---|---|
| Scheduled backups | ✅ |
| Multiple backup destinations, including cloud object storage | 🟡 Primary destination in routine use |
| Backup integrity verification | ✅ |
| Documented and rehearsed restore procedure | ✅ |
| System health check suite | ✅ |
| External monitoring independent of the application host | ✅ |
| Failure alerting with recorded delivery | ✅ |
| Scheduled data retention pruning | ✅ |

## Localisation

| Feature | Status |
|---|---|
| English interface | ✅ |
| Sinhala interface | 🟡 High-traffic screens and help centre complete; a tail of lower-traffic screens remains |
| Sinhala rendering on printed receipts | ✅ |
| Separation of translated system text from untranslated shop-authored content | ✅ |
| Tamil interface | ⬜ |

## Mobile and devices

| Feature | Status |
|---|---|
| Responsive interface for tablet and phone | ✅ |
| Camera barcode scanning on mobile browsers | ✅ |
| Hardware scanner support | ✅ |
| Silent receipt printing from a mobile device | ⬜ Not achievable from a web page; a companion print bridge is designed but not built |
