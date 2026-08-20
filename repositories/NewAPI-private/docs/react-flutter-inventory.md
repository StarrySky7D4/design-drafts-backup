# React to Flutter inventory

This inventory defines the scope of complete React replacement. It is based on
the current `web/src` tree and must be updated when React surface area changes.

The migration is complete at the runtime boundary: Flutter Web is the default
browser application, the Rust administration gateway is the public process,
and the Go service remains the loopback API core. CI, release archives, Docker,
Electron, and Make targets build and package that topology. The React tree is
retained only as a read-only behavior and locale reference; it is not part of
the default build or runtime path.

Completion is enforced by repository tests rather than a manual Ready flag.
`admin_routes_test.dart` reads the generated React route tree and requires all
56 browser routes, including dynamic and trailing-slash forms, to resolve in
Flutter. `migration_catalog_test.dart` reads the React source tree and requires
all 30 feature entries to map to a Flutter module while keeping the audited
681 TSX / 59 route / 190 shared-component / 61 UI-primitive baseline stable.
Feature widget and Rust-contract tests then exercise the mapped pages and
interactions. No migration dashboard or partial-feature runtime state remains.

## Baseline

| Surface | Current count |
| --- | ---: |
| TSX files | 681 |
| Route files | 59 |
| Feature entry components | 30 |
| Shared component TSX files | 190 |
| UI primitive TSX files | 61 |

File count is a sizing signal, not the completion criterion. A React component
is replaceable only when its user-visible behavior is covered by Flutter tests
or an explicit route/interaction contract.

## Route groups

| Group | React routes/features | Flutter status |
| --- | --- | --- |
| Public | setup, home, pricing, rankings, about, privacy policy, user agreement, error routes | Ready: Flutter owns first-run setup, the public browser router, home, Model Square pricing, Rankings, about, privacy-policy, user-agreement, and 401/403/404/500/503 recovery pages. Setup covers database detection, existing/new root handling, validation, usage modes, review, initialization, and automatic incomplete-install redirect. Pricing covers search, vendor/group/billing/endpoint/tag filters, sorting, card/table layouts, M/K units, display-currency conversion, group multipliers, direct model-detail routes, ratio and endpoint inspection, fixed/token/dynamic billing summaries, structured tier tables, request rules, and raw-expression fallback. Rankings covers four-period model/vendor leaderboards, model-usage history, vendor-share history, market share, and movers/droppers |
| Authentication | sign-in, sign-up, reset, OTP, OAuth | Ready: Flutter owns password sign-in, legal consent, Cloudflare Turnstile, password registration with referral and optional email verification, reset-email and reset-link flows, authenticator and backup-code 2FA, discoverable Passkey login, Telegram and WeChat login, and GitHub/Discord/OIDC/LinuxDO/custom OAuth redirects and callbacks on both sign-in and registration surfaces. Rust exposes only explicit public authentication operations while Go remains the authentication write owner |
| Chat | playground, chat, chat-to-link | Ready: Flutter owns Playground conversation persistence, group/model selection, streamed and non-streamed chat completions, reasoning display, cancellation, response timing, message copy/edit/delete/regeneration, and bounded request parameters. Rust exposes explicit Playground group/model/streaming contracts. Flutter also parses configured web/custom-protocol chat presets, retrieves API keys only when required, expands direct/Cherry Studio/AionUi/DeepChat placeholders, embeds web chat with camera/microphone permissions, launches external applications, and supports the legacy first-web-preset redirect |
| General | overview, dashboard, API keys, usage logs, task logs | Overview ready; Dashboard is ready across preset/custom ranges, hour/day/week granularity, admin/self scope, username filtering, request/token/quota and RPM/TPM summaries, native trend/ranking charts, role-aware flow dimensions with masking/metric/stage/limit/overflow controls, and administrator user analytics; API keys are ready across name/key/status filtering, reveal/copy, full create/edit options, display-currency quota conversion, group/Auto routing, model/IP restrictions, batch creation/copy/delete, connection-info export, chat launches, and CC Switch import; Usage Logs are ready across admin/self scope, full time/model/token/group/user/channel/request-ID filtering, sensitive masking, statistics, stream/token/cost/timing details, raw metadata, user drill-down, administrator channel-affinity upstream-cache hit statistics, and Midjourney drawing history/result/detail views; Task Logs are ready across admin/self scope, time/task/platform/action/channel/status filtering, sensitive user drill-down, duration/status/progress/quota details, failure inspection, and audio/video result access |
| Personal | wallet, profile | Ready: Wallet supports balance/usage/request stats, payment configuration visibility, guarded redemption, admin/self billing history, public subscription plans, active/history quota state, billing preference, purchase limits, balance purchase, and Stripe/Creem/Waffo Pancake/Epay launch flows. Profile supports account summary, display-name editing, notification delivery and account preferences, interface language, sidebar visibility, password rotation, access-token generation, account deletion, 2FA lifecycle, daily check-in, login-session revocation, email/WeChat/Telegram and standard/custom OAuth binding, custom OAuth unbinding, and WebAuthn Passkey registration/removal. Turnstile-protected check-in retries now use the native Flutter Web challenge bridge |
| Administration | channels, models, users, redemption codes, subscriptions, system info | Ready: Users cover listing plus subscription assignment, quota reset, invalidation, deletion, and history; redemption codes cover search/filter, create/edit with expiration presets, enable/disable, selection copy, delete/invalid cleanup, and the payment-compliance lock; subscriptions cover admin plan CRUD/status/reset, configured quota conversion, groups, Waffo product helpers, user lifecycle management, wallet preferences/purchase limits, balance purchase, and four external payment launches; models cover metadata CRUD/filtering, endpoint validation, vendors, prefill groups, missing/upstream synchronization, all pricing modes and advanced ratios, plus guarded io.net deployment list/search/create/estimate/details/container/log/update/rename/extend/delete workflows; system info covers instance heartbeat/resources, stale-instance cleanup, task status/progress/history, and live polling; channels cover paginated list/search/filtering, selection and batch status/deletion/tagging, tag aggregation/edit/enable/disable, create/edit in single/batch/multi-key modes, upstream-model discovery and staged detection/apply workflows, secure root key reveal, multi-key status management including append/replace updates, Codex usage/reset/credential refresh, Ollama version/list/SSE pull/delete/configured-model selection, selectable endpoint/model/stream tests, configurable copying, advanced JSON validation, high-risk 504/524 mapping confirmation, balance operations, global test/balance/repair/disabled cleanup, guided provider-specific credentials/settings, and a visual/JSON advanced-custom route editor with protocol templates and validation |
| System settings | site, auth, billing, models, security, content, operations | Site & Branding is ready across system identity/address/logo, footer/about/home content, legal documents, notice, header visibility/auth rules, sidebar sections/modules, validation, reset, and persisted Rust-gateway writes; Authentication is ready across basic login/registration controls, email verification/domain policy, GitHub/Discord/OIDC/Telegram/LinuxDO/WeChat integrations, Passkey policy, Turnstile, and custom OAuth discovery/presets/CRUD; Billing & Payment is ready across quota, currency/display, check-in, compliance confirmation, Epay/Stripe/Creem/Waffo/Waffo Pancake option forms, complete model/group/tool pricing configuration with visual JSON-map editors and validation, upstream price synchronization, and Waffo Pancake credential verification/store-product provisioning and binding; Models & Routing is ready across global request conversion/ping policy, retry/disable/monitoring policy, Gemini/Claude/Grok adapters, io.net credentials, and channel-affinity persistence with visual/JSON rule editing, Codex/Claude templates, live cache statistics, and per-rule/all-cache clearing; Security & Limits now covers bounded global/group rate limiting, sensitive prompt filtering, SSRF private-address/domain/IP/port controls, and per-user token limits; Console Content now covers dashboard export controls, enabled-state persistence, structured announcement/API/FAQ/Uptime/Chat add-edit-delete and batch deletion, legacy Uptime fallback, and all drawing switches; Operations now covers system behavior, metrics aggregation and retention, SMTP security modes, Worker proxying, database/server-log maintenance, disk-cache and resource-monitor settings, live performance controls, and version/update inspection |

## Component replacement batches

1. Layout primitives: responsive shell, navigation, dialogs, forms, tables,
   loading, error, empty, toast, and confirmation patterns.
2. End-user behavior: API keys, usage/task logs, wallet, profile, dashboard,
   playground, chat, and public pages.
3. Administration behavior: channels, models, users, redemptions,
   subscriptions, and system information.
4. Settings behavior: all forms and nested sections under system settings.
5. Specialized widgets: charts, virtualized tables, markdown/rich content,
   OTP, passkeys, OAuth, QR codes, model selectors, and billing editors.

The Flutter migration catalog is defined in
`flutter-admin/lib/src/migration_module.dart`. Its test protects the complete
authenticated sidebar contract. Navigation text is generated from the existing
seven locale files by `flutter-admin/tool/sync_web_i18n.mjs`.
