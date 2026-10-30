# Changelog

All notable changes to the `Threecommerce_OneStepCheckout` module are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.5] - 2026-10-30
### Changed
- `composer.json`: il suggerimento del tema Hyvä punta a `hyva-themes/magento2-default-theme-csp` (tema compatibile con la CSP) invece di `magento2-default-theme`. Solo metadati: nessun cambio di codice o di comportamento.

## [1.2.4] - 2026-10-19

### Fixed
- Hyvä: when the address comes back from the quote (page reload, returning to checkout), the
  address-book select now shows the matching saved address instead of "new address".

## [1.2.3] - 2026-10-19

### Fixed
- Hyvä: a logged-in customer with an empty address book now gets the address typed at checkout
  saved in the address book (as default billing/shipping), as the native checkout does for the
  first address. Before, nothing was saved and every checkout started with an empty form.

## [1.2.2] - 2026-10-19

### Fixed
- Hyvä: with a zero grand total only the "No Payment Information Required" (`free`) method is
  offered, as in the native checkout. Before, other active methods (e.g. Check / Money order) were
  listed and the first one pre-selected, and Magento rejected it when placing the order. A selected
  method that is no longer available is now reset.

## [1.2.1] - 2026-10-19

### Fixed
- Hyvä: the order summary now shows the item custom options (e.g. "Dominio: www.example.com")
  under the product name, as the Breeze summary already did (`getSummary()` exposes `options`).
- Hyvä: the payment, shipping and "billing same as shipping" radios now have a `name` per group,
  so two methods can no longer look selected at the same time (the submitted value was already
  correct).

## [1.2.0] - 2026-10-14

### Added
- Breeze: built-in base stylesheet `css/breeze-checkout.css` for the checkout and the thank-you
  page (two columns on desktop, single column with the collapsible summary below 1024px, raw `@media`
  in px — no Luma mixins). Neutral colors exposed as `--osc-*` CSS variables (zero-specificity
  `:where()` block, so a theme can override them); buttons/headings follow the admin colors
  (`--osc-btn` / `--osc-accent`, applied by `theme-vars.js`). The Breeze root now carries the
  `data-osc-accent` / `data-osc-btn` attributes for this.
- New option **Appearance → Use Built-in Styles** (`onestepcheckout/appearance/builtin_styles`,
  default Yes, Breeze only). The stylesheet is injected by a `head.additional` block with
  `ifconfig`, so with No the module CSS is not loaded at all (the theme styles the markup).
- Breeze extension points for add-ons (used by Threecommerce_StorePickup 2.x):
  - layout containers `onestepcheckout.breeze.shipping.extra` (inside the shipping-method section)
    and `onestepcheckout.breeze.recap.extra` (end of the info recap); several modules can add blocks;
  - DOM events on `#osc-breeze` (bubbling): `osc:shipping-method-change` `{code, carrierCode,
    methodCode}` and `osc:before-place-order` `{errors, tasks}` — `errors` block the order, `tasks`
    (functions returning a Promise) run after the address/shipping are saved on the cart and
    before payment.

### Changed
- Success page styles are now theme-specific: Hyvä always loads `css/checkout.css` +
  `theme-vars.js` (`hyva_onestepcheckout_index_success.xml`); Breeze loads the built-in stylesheet
  only when the option is enabled. Previously `checkout.css` was loaded on every theme.
- README rewritten for Breeze + Hyvä (requirements, configuration paths, native redirect,
  integrations with m2-italianinvoice and m2-store-pickup, styles option, extension points).
- `composer.json` description/suggestions updated for both frontends.

### Fixed (Breeze)
- The shipping methods list stayed in the loading state (`osc-breeze__methods--loading`) after the
  first estimate: `_applyShipping()` bumps the request counter used as guard, so the loading flag was
  never cleared. Invisible with the old theme-only styles, it blocked the shipping choice once the
  built-in stylesheet (`pointer-events: none`) is loaded. Now guarded by a dedicated counter.

### Fixed (Hyvä / Magewire — reviewed, not yet verified on a live Hyvä store)
- `updatedAcceptedAgreements()`, `updatedSelectedAddressId()` and `updatedBSelectedAddressId()`
  returned `void`: Magewire assigns a hook's return value back to the property, so ticking a T&C
  checkbox or choosing a saved address assigned `null` to a typed property (fatal `TypeError`).
  They now return the value.
- The "same as shipping / different billing address" radios send `"1"`/`"0"` strings into the
  `bool $billingSame` property (fatal `TypeError` under `strict_types`): new `updatingBillingSame()`
  casts to bool before assignment.
- A separate billing address was silently ignored: the place-order step read `billingSame` /
  `billing` from the Alpine `x-data` config, evaluated only once at first render (Magewire morphs do
  not refresh it). The billing address (same as shipping or separate) is now written to the quote
  server-side in `prepareOrder()`, and the GraphQL step only sets the payment method and places the
  order. This also stops `setBillingAddressOnCart` from overwriting the add-on billing fields.
- The payment code is passed in the `osc:place-order` event detail (validated server-side) instead
  of the stale config; the `change` listener keeps `window.__oscPaymentCode` on the clicked radio.
- Virtual-only carts (downloadable/virtual products) could never be ordered: no shipping method
  exists, so payment methods never loaded and `prepareOrder()` demanded a shipping method. Virtual
  carts now use the address as billing address, load payment methods directly, skip the shipping
  method, hide the billing toggle/shipping sections and read totals from the billing address.
- `$extraBilling` / `$extraAddress` (client-controlled) could set any quote-address field; keys are
  now limited to simple names outside a blocklist (`customer_id`, `quote_id`, `address_type`, …)
  with scalar values.
- `selectMethod()` accepts only codes among the collected rates; shipping prices are shown with the
  store currency format instead of `number_format()`.
- Removed `x-init="init()"` from the place-order island (Alpine 3 already calls `init()`).

## [1.1.26] - 2026-10-02

### Fixed
- Breeze: the "Place order" button stayed disabled after a validation error (e.g. Terms &
  Conditions not accepted) until the page was reloaded. Root cause: Breeze's `$.fn.find` does not
  support the jQuery `:visible` pseudo-selector, so `_scrollToError()` threw inside the place-order
  `catch` before `_placeBusy(false)` ran, and the "clear on edit" listener threw too. The error
  visibility is now a plain DOM check (`_hasError()`), the busy state is reset first on every failed
  attempt, and required agreements are validated locally before any request (the button is never
  disabled for a missing tick). Ticking the checkbox clears the error message.
- Breeze: the order summary now shows each item's custom options (e.g. *Dominio*) under the product
  name, so two lines of the same product are distinguishable (new `osc-breeze__summary-item-info` /
  `__summary-item-options` markup; styling lives in the theme).

## [1.1.25] - 2026-07-28

### Changed
- Admin config section moved under the shared **Threecommerce Plugins** tab (`threecommerce_plugins`),
  like every other Threecommerce module, instead of its own standalone tab; the section is now
  labelled **Checkout** (was "Shopify-like Checkout"). The section id stays `onestepcheckout`, so
  the stored `core_config_data` paths (`onestepcheckout/general/*`, `onestepcheckout/appearance/*`)
  and `Model/Config.php` are untouched — no config is lost.
- Dropped the remaining "Shopify" wording from the admin-visible strings (ACL resource title and the
  express-checkout / header-footer field comments) and from the `en_US` / `it_IT` translations.

### Fixed
- Place-order double-submit guard: `_placeOrder()` bails out while an order is already in flight.
- The place-order error alert now clears as soon as any field is edited (delegated `input`/`change`
  listener on the checkout root, so it also covers controls rendered later by Magewire/JS), which
  re-enables the "Place order" button instead of leaving it stuck after a failed attempt.
- Hyvä parity: the Magewire template renders the error alert with the same fixed, always-visible
  placement used by the Breeze checkout.

## [1.1.24] - 2026-07-10

### Added
- Mobile collapsible order summary (Shopify-style): on mobile the order summary moves to the top
  of the checkout as a collapsed bar showing the label + grand total; tapping it expands the full
  summary (items, coupon, totals, recap). Desktop layout is unchanged (summary stays in the sticky
  right column, always expanded).
  - New markup: `[data-osc-summary-toggle]` bar (with `[data-osc-summary-toggle-total]`) and a
    `[data-osc-summary-body]` wrapper inside the summary aside.
  - The checkout JS mirrors the `grand_total` into the toggle bar and binds the open/close toggle.
  - Styling lives in the theme (`custom/_osc-checkout.less`), per project convention.

## [1.1.23] - 2026-07-08

### Added
- Generic extension slot in the Breeze checkout so other modules can contribute extra billing
  fields without patching this module:
  - Template hook `getChildHtml('osc.billing.extra')` inside `#osc-breeze` (after the address section).
  - The checkout JS now collects any input inside `#osc-breeze` marked with `data-osc-ext="<name>"`
    into `billingAddress.extension_attributes.<name>`, and `data-osc-addr="<name>"` into the native
    billing address field `<name>` (e.g. `vat_id`), on all order flows (virtual / physical /
    separate billing). Used by `Threecommerce_ItalianInvoice` to add the invoice-request fields.
- Hyvä parity for the extension slot: `getChildHtml('billing_extra')` in the Magewire template plus
  generic `$extraBilling` / `$extraAddress` component properties (bound via `wire:model`) that
  `prepareOrder()` persists onto the quote billing address (and shipping when billing==shipping) so
  the contributing module's quote→order observer copies them. NB: Hyvä is inert on this store (active
  theme is Breeze), so this path is shipped for parity but not verified end-to-end — on Hyvä the order
  is placed via GraphQL, whose BillingAddressInput does not carry these custom fields; a separate
  billing address may need a GraphQL schema extension, to verify when Hyvä is activated.

### Changed
- The optional second street line is now rendered only when `customer/address/street_lines > 1`
  (new `ViewModel::getStreetLines()`), instead of always. Respects the store address config.

## [1.1.21] - 2026-07-07

### Fixed
- Hyvä single-page checkout hardening (follow-up to 1.1.20):
  - Contact + shipping/billing address fields now use `wire:model.lazy` (sync on blur) instead of
    `.defer`, so values entered are persisted server-side and no longer lost when a reactive
    roundtrip (address → shipping rates) morphs the DOM.
  - Selecting a shipping method now calls `selectMethod('<code>')` with the code passed explicitly,
    instead of relying on the deferred `selectedMethod` being flushed by `wire:click` — this removes
    a race where the payment methods sometimes failed to load after choosing a shipping method.
  - The hybrid place-order step reads the selected payment code from a server-rendered
    `data-osc-payment` attribute (reliably morphed by Magewire), fixing a GraphQL
    "Required parameter 'code' for 'payment_method' is missing" error when the payment method was
    auto-selected server-side in the place-order roundtrip. `window.__oscPaymentCode` stays in sync
    from the checked radio for payment add-ons (Banca Sella).
- Success (thank-you) page is styled again on the Hyvä theme. 1.1.19 moved it to Breeze-native BEM
  markup (`.osc-success__*`) styled by the Breeze theme's LESS; on Hyvä those classes had no CSS, so
  the page rendered unstyled (a full-size check SVG). The module now ships the Hyvä styling for
  `.osc-success__*` in `css/checkout.css`, and the success template carries the `data-osc-*` brand
  attributes so `theme-vars.js` applies the button color.

## [1.1.20] - 2026-07-07

### Changed
- Hyvä (Magewire) checkout is now a single-page (Shopify-like) layout, matching the Breeze
  frontend: the whole form stays visible in the left column — contact, shipping address,
  billing (when different), shipping method, payment, order note and a single "Place order"
  button — instead of the previous 4-step wizard. Shipping methods appear reactively once the
  address is complete (country + city + ZIP, on blur), and payment methods once a shipping
  method is chosen; the right column keeps the coupon + order summary.
- The single "Place order" validates the whole form server-side (Magewire), guarantees the
  address + shipping method are set on the quote, then hands off to the existing hybrid GraphQL
  place-order step via an `osc:place-order` browser event (the Alpine island now listens for it).
- The add-on extension points are preserved: StorePickup (`pickup` / `pickup_recap`, shipping
  radios still `wire:model.defer="selectedMethod"`) and Banca Sella (`payment_extra`). The
  place-order island now reads the selected payment code live from the checked radio and keeps
  `window.__oscPaymentCode` in sync as the selection changes, so payment add-ons toggle correctly
  on the single page.

## [1.1.19] - 2026-07-07

### Changed
- Success (thank-you) page markup is now Breeze-native semantic BEM (`.osc-success__*`) instead of
  Hyvä Tailwind utility classes. On the Breeze theme those utilities are not compiled, so the page
  could not be skinned from the theme (the confirmation badge rendered as a full-width bar and the
  brand color fell back to the default blue). The template now carries only stable, themeable
  classes — no logic change — so the theme owns the whole look (see the store theme's
  `custom/_osc-success.less`). No inline colors/config brand vars are used by the template anymore.

## [1.1.18] - 2026-07-07

### Changed
- Breeze checkout is now a true single-page (Shopify-like) layout instead of the previous
  step-by-step flow that hid the form when advancing (which looked like Luma). The whole form
  stays visible in the left column (contact + address, billing when different, shipping method,
  payment, order note, place order); shipping and payment options populate **reactively** as the
  address becomes valid (debounced, out-of-order-safe). The right column keeps the coupon and
  order summary and adds a live read-only recap of the entered contact/shipping/billing info.
- Removed the intermediate "Continue to shipping / Continue to payment / Back" step buttons; a
  single "Place order" validates, guarantees the address (+ shipping method) is set on the quote,
  and places the order.

### Fixed
- Magewire checkout no longer fatals when the country changes (and on every subsequent action).
  The `updatedCountryCode()` / `updatedBCountryCode()` lifecycle hooks returned `void`; Magewire
  reassigns a hook's return value onto the property, so `null` was assigned to the string-typed
  `$countryCode` / `$bCountryCode` (an uncaught `TypeError`). The hooks now accept and return
  `string $value`.

### Added
- Hyvä Tailwind source registration: a `hyva_config_generate_before` observer registers the
  module in `app/etc/hyva-themes.json` so the utility classes used only in the checkout templates
  (`space-y-*`, `grid`, `py-*`, `px-*`, …) are no longer purged from the compiled `styles.css`
  (which left the Hyvä checkout visually cramped). Run `bin/magento hyva:config:generate` + the
  theme Tailwind build to pick it up.

## [1.1.17] - 2026-07-07

### Fixed
- Guest guard on the checkout entry point: when guest checkout is not allowed
  (`checkout/options/guest_checkout=0`, or a cart product that requires registration),
  a not-logged-in visitor is now redirected to the login page (with return to the
  checkout afterwards) instead of being shown the guest form. Previously the guest form
  rendered and the address save failed later with Magento's generic *"The address failed
  to save. Verify the address and try again."* — the underlying cause being
  `AccessChangeQuoteControl` ("Invalid state change requested") when a guest quote is
  saved while a customer context is present (e.g. a stale/expired session).

## [1.1.0] - 2026-07-02

### Added
- Native Terms and Conditions (checkout agreements) support in the review step.
  Active agreements from **Stores → Terms and Conditions** are rendered at step 4;
  "manual" ones show a required checkbox, "auto" ones show as text. The place-order
  button is gated until all required agreements are accepted.
- Server-side enforcement for the hybrid GraphQL place order: since Magento's native
  validator only runs on the Luma/REST path (which GraphQL bypasses), a minimal plugin
  on `CartManagementInterface::placeOrder`, scoped to the GraphQL area, blocks the order
  unless all required agreements are accepted. Acceptance is bound in Magewire, persisted
  onto the quote (`quote.osc_agreement_ids`) and read back by the plugin.

## [1.0.0] - 2026-06-23

Initial release.

### Added
- Shopify-like one-page checkout on the **Hyvä** theme: two-column layout with a stepped form
  (contact → delivery → shipping → payment → review) and a live order summary sidebar.
- **Hybrid architecture**: forms are server-side reactive via **Magewire** (validation, prefill,
  step wizard) while order placement runs through **Magento GraphQL** (`setBillingAddressOnCart` →
  `setPaymentMethodOnCart` → `placeOrder`), for both guests and logged-in customers.
- Order summary with line items, discount code (apply/remove) and live totals, read from the quote.
- Customer login inside the checkout (server-side authentication + guest→customer cart merge).
- Separate billing address (optional) and order notes field.
- Express-checkout area: presentational buttons shown only for active gateways (PayPal / Amazon Pay /
  Apple Pay), gated by a master toggle.
- Configurable checkout URL via a custom frontend router (`onestepcheckout/general/route`, default
  `/checkout-pro`) and optional redirect of the native `/checkout` page
  (`onestepcheckout/general/redirect_native`).
- Appearance settings: store logo toggle, accent and primary button colors, express area, order
  notes, and an optional bare (header/footer-less) layout.
- Dedicated thank-you page that reliably shows the order number for guest GraphQL orders (the order
  is bridged into the PHP checkout session, keeping the success URL clean).
- Payment extension points for online gateways: a `payment_extra` child block for method-specific UI
  and JS hooks (`window.oscPaymentHandlers[<code>]` with `validate()` / `complete()`) that can run
  before placing the order and take over navigation after it (e.g. a 3-D Secure redirect).
- English source strings with Italian translations (`i18n/it_IT.csv`).

### Security
- **CSP-compatible markup**: no inline `<script>`/`<style>` or `style=""` attributes. The place-order
  logic lives in an external asset and registers as an Alpine component; static styles are in an
  external stylesheet and the two admin-configurable brand colors are applied to `:root` via a small
  external script. The checkout works under an enforced Content-Security-Policy without relying on
  `'unsafe-inline'`. Card data is never collected on the server (payment is delegated to the gateway),
  keeping the integration in PCI SAQ-A scope.

### Known limitations
- Offline payment methods (check/money order, bank transfer, cash on delivery, purchase order) place
  an order end-to-end. Redirect-based online gateways (e.g. PayPal Express) need per-gateway wiring on
  top of the GraphQL `placeOrder` step.
- Tailwind utility classes are compiled only if this module's `view/frontend` path is included in the
  active Hyvä theme `tailwind.config.js` `content` globs.