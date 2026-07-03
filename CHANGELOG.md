# Changelog — ETechFlow Order Email Editor

All notable changes to this module. Adheres to [Semantic Versioning](https://semver.org/).

---

## [1.1.1] — 2026-07-03 — Security: portal-only licensing (removes forgeable key path)

Closes a licensing bypass. Previous versions shipped the HMAC signing secret
(`SECRET_FRAGMENTS` / `BUNDLE_SECRET_FRAGMENTS`) inside `LicenseValidator` and
validated a locally-computed key against it, so anyone with the module source
could compute a valid key for their own domain and run the module unlicensed.
A second bypass let a `production_environment = No` toggle disable licensing.

### Changed (security)

- **Validation is now portal-only.** `isValid()` honours a key only when the
  ETechFlow portal confirms it. The module ships no signing secret.
- **Removed** `computeKey()`, `computeBundleKey()`, `checkKey()`, and the
  `SECRET_FRAGMENTS` / `BUNDLE_SECRET_FRAGMENTS` constants.
- **Offline grace is now un-forgeable** — it derives solely from a cached,
  genuine portal "valid" response (48 h TTL), never from admin-settable config.
- **`isProductionEnvironment()` hardcodes `true`** so the sandbox toggle can no
  longer disable enforcement. Rewrote the unit suite with a hard forged-key test.

---

## [1.1.0] — 2026-06-04 — Portal licensing, server-IP binding & billing-period plans

Adds the eTechFlow Store Portal licensing layer to the module (the HMAC per-module
and shared bundle keys keep working for offline / suite activation).

### Added

- **Hybrid `Model/LicenseValidator.php`** — SP-XXXX subscription keys validate live
  against the eTechFlow portal with a **domain + server-IP** two-factor binding
  (strict: no offline grace, no stale issued-key fallback). HMAC per-module and
  shared bundle keys unchanged. Dev hosts + `Production Environment = No` bypass.
- **In-admin Stripe checkout + license gate** — `Controller/Adminhtml/License/`
  (`Gate`, `Checkout`, `Activated`) + dark plan-card gate page and success page.
  Billing-period plans: **Weekly $5 / Monthly $15 / Yearly $150** (one full-feature
  module, billed by period), matching the portal subscription model.
- **License + Stripe admin config** — license key, issued-key, portal URL, bundle
  key, and Stripe keys under *Stores → Config → eTechFlow → Order Email Editor*.
- **`etc/db_schema_whitelist.json`** — fixes the missing whitelist so the
  `etechflow_email_change_history` table is created on `setup:upgrade` (the 1.0.x
  package shipped without it).

### Changed

- **License-gated entry points** — the Edit History grid redirects to the gate when
  unlicensed; the Update endpoint already returns 403 via `Config::isEnabled()`.
- **Customer-account sync** is now a licensed feature (`Config::isCustomerSyncAllowed()`,
  plan flag `customer_sync`; included on all billing-period plans).

---

## [1.0.2] — 2026-05-22 — Move admin menu under eTechFlow top-level sidebar

### Changed

- **OEE admin pages relocated to a dedicated "eTechFlow" sidebar entry.** Previously the Edit History list lived under `Sales → Sales Operation`. Now it sits as an `Order Email Editor` column inside a new top-level `eTechFlow` sidebar entry (clusters with other paid-extension vendors above Magento's Stores). Matches the pattern Amasty / Magefan / MageWorx use.
- Each eTechFlow module declares the same `eTechFlow::root` + `eTechFlow::settings` + `eTechFlow::configuration` entries — Magento merges by id, so installing N modules still produces exactly one `eTechFlow` sidebar group.

### Migration

```
composer update etechflow/module-order-email-editor
bin/magento setup:upgrade
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f
bin/magento cache:flush
```

Admin URL routes unchanged (`order_email_editor/history/index` still works). No schema or behaviour changes — pure menu-layout adjustment.

---

## [1.0.0] — 2026-05-19

### Initial commercial release

Edit the customer email on a placed order. Fix typos, handle customer requests, keep an audit trail. Admin-only. Hyvä-safe by design.

#### Added

- **Edit Email button** on every order detail page (admin → Sales → Orders → pick order). Opens a modal: current email shown, new email input, optional "also update linked customer account" checkbox (hidden for guest orders).
- **Email Change History grid** at admin → Sales → Operations → Order Email Change History. Filterable Magento UI Component grid of every change made.
- **Atomic DB update** of all the places Magento stores the order email:
  - `sales_order.customer_email`
  - `sales_order_address.email` (both billing + shipping rows)
  - `sales_order_grid` + `sales_invoice_grid` + `sales_creditmemo_grid` + `sales_shipment_grid` (via Magento's core `sales_order_save_after` reindex)
  - `customer_entity.email` (if checkbox ticked + linked customer exists)
  - `quote.customer_email` + `quote_address.email` (defensive, only if quote still exists)
- **Audit log** in `etechflow_email_change_history` — admin id, admin name, old email, new email, customer-record-updated flag, IP, timestamp.
- **Per-installation HMAC license** with bundle-key support — same as every other eTechFlow module. Unlicensed installs silently hide the Edit Email button + history menu and reject the update endpoint with 403.
- **Two ACL resources**: `edit_email` (modal + update endpoint) and `view_history` (history grid). Granular per-role permissions.
- **Profiler instrumentation** — wraps the update path in an `ETechFlow_OEE_UpdateOrderEmail` Tideways span.
- **Verify CLI** — `bin/magento etechflow:oee:verify` checks DI resolution + DB table presence.
- **Hyvä-safe** — admin-only module with zero frontend assets. Hyvä themes never touch the admin.

#### Compatibility

- Magento Open Source 2.4.4 – 2.4.8
- Adobe Commerce 2.4.4 – 2.4.8
- PHP 8.1 / 8.2 / 8.3 / 8.4
- All Hyvä child themes
