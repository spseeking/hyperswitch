# 08 — API surface

Source of truth: the generated OpenAPI documents
`api-reference/v1/openapi_spec_v1.json` (121 documented paths) and
`api-reference/v2/openapi_spec_v2.json` (70 documented paths), produced from
`crates/openapi` annotations. The complete route table — including endpoints that are not
in the public spec (dashboard/user APIs, analytics, webhook receivers, dummy connector,
internal admin) — is `crates/router/src/routes/app.rs`.

## Authentication schemes

`crates/router/src/services/authentication.rs`:

| Scheme | Header | Used by |
| --- | --- | --- |
| API key | `api-key` | Server-to-server merchant APIs (create/confirm/capture/refund…) |
| Publishable key + client secret | `api-key: pk_…` + `client_secret` in the request/URL | Client-side SDK calls (confirm, session tokens, list PMs for an intent) |
| Ephemeral key | `api-key` + ephemeral key | Customer-scoped client operations |
| JWT (dashboard) | `authorization: Bearer …` | Control Center; carries user/role and is checked against RBAC permissions |
| Admin API key | `api-key` (admin) | Merchant/org provisioning, GSM, internal config |
| Connector webhook auth | connector-specific signature | `/webhooks/**` (verified per connector) |
| Partial auth | signed headers `[feature: partial-auth]` | Reduced-cost auth for high-volume endpoints |

## v1 endpoint inventory (by domain)

| Domain | Paths | Examples |
| --- | --- | --- |
| Payments (17) | `/payments`, `/payments/list`, `/payments/{id}`, `/payments/{id}/confirm`, `/capture`, `/cancel`, `/cancel_post_capture`, `/complete_authorize`, `/incremental_authorization`, `/extend_authorization`, `/session_tokens`, `/post_session_tokens`, `/update_metadata`, `/3ds/authentication`, `/eligibility`, `/eligibility_check`, `/{id}/client` | full payment lifecycle |
| Accounts & profiles (13 + 3) | `/accounts`, `/accounts/{id}`, `/accounts/{id}/business_profile`, `/account/{id}/connectors`, … | merchant, profile and connector-account CRUD |
| Organization (2) | `/organization`, `/organization/{id}` | org CRUD |
| Routing (11) | `/routing`, `/routing/{id}`, `/routing/{id}/activate`, `/routing/deactivate`, `/routing/default`, dynamic-routing toggles | algorithm management |
| Refunds (3) | `/refunds`, `/refunds/list`, `/refunds/{id}` | create/list/retrieve/update |
| Disputes (10) | `/disputes/list`, `/disputes/{id}`, `/disputes/accept/{id}`, `/disputes/evidence`, `/disputes/{id}/evidence`, aggregates | dispute handling |
| Customers (6) | `/customers`, `/customers/{id}`, `/customers/list`, `/customers/{id}/payment_methods`, `/customers/{id}/mandates` | customer & saved-PM management |
| Payment methods (4) | `/payment_methods`, `/payment_methods/{id}`, `/payment_methods/{id}/update` | vaulting/tokenization |
| Mandates (2) | `/mandates/{id}`, `/mandates/revoke/{id}` | mandate retrieve/revoke |
| Payouts (7) `[payouts]` | `/payouts/create`, `/payouts/{id}`, `/payouts/{id}/confirm`, `/fulfill`, `/cancel`, `/list` | payout lifecycle |
| Subscriptions (10) | `/subscriptions`, `/subscriptions/{id}`, plans, invoices, confirm | subscription billing |
| Authentication (7) | `/authentication`, `/authentication/{id}`, eligibility, sync | 3DS/authentication objects |
| Blocklist (4) | `/blocklist`, `/blocklist/toggle` | fraud blocklists |
| GSM (4) | `/gsm`, `/gsm/get`, `/gsm/update`, `/gsm/delete` | connector error→decision mapping |
| Events (4) | `/events/{merchant_id}`, `/events/{merchant_id}/{event_id}/attempts`, retry | webhook delivery inspection |
| API keys (3) | `/api_keys/{merchant_id}`, `/api_keys/{merchant_id}/{key_id}` | key management |
| Relay (2) | `/relay`, `/relay/{id}` | non-Hyperswitch payment refunds/disputes |
| Reference & misc | `/card_issuers`, `/profile_acquirers`, `/three_ds_decision/rule`, `/payment_link/{id}`, `/poll/status/{id}`, `/user/…` | supporting APIs |

Not in the public spec but present in `routes/app.rs`: `/health`, `/health/ready`,
`/webhooks/**`, `/analytics/**`, `/user/**` (sign-in, invites, TOTP, SSO/OIDC),
`/user_role/**`, `/cache/**`, `/files/**`, `/verify/**` (Apple Pay domain verification),
`/connector_onboarding/**`, `/dummy-connector/**`, `/proxy`, `/feature_matrix`,
`/offers/**`, `/tokenize/**`, `/process_tracker/**`, `/hypersense/**`, `/chat/**`,
`/superposition/**`.

## v2 endpoint inventory

v2 makes the account hierarchy explicit and splits intent creation from confirmation:

| Domain | Endpoints |
| --- | --- |
| Organizations | `POST /v2/organizations`, `GET|PUT /v2/organizations/{id}`, `GET /v2/organizations/{id}/merchant-accounts` |
| Merchant accounts | `POST /v2/merchant-accounts`, `GET|PUT /v2/merchant-accounts/{id}`, `GET /v2/merchant-accounts/{id}/profiles` |
| Profiles | `POST /v2/profiles`, `GET|PUT /v2/profiles/{id}`, `GET /v2/profiles/{id}/connector-accounts`, `PATCH /v2/profiles/{id}/activate-routing-algorithm`, `PATCH /v2/profiles/{id}/deactivate-routing-algorithm`, `PATCH|GET /v2/profiles/{id}/fallback-routing`, `GET /v2/profiles/{id}/routing-algorithm` |
| Connector accounts | `POST /v2/connector-accounts`, `GET|PUT|DELETE /v2/connector-accounts/{id}` |
| Payments | `POST /v2/payments`, `POST /v2/payments/create-intent`, `GET /v2/payments/{id}/get-intent`, `PUT /v2/payments/{id}/update-intent`, `POST /v2/payments/{id}/confirm-intent`, `GET /v2/payments/{id}`, `GET /v2/payments/list`, `GET /v2/payments/{id}/payment-methods`, `POST /v2/payments/{id}/create-external-sdk-tokens`, `POST /v2/payments/{id}/eligibility/check-balance-and-apply-pm-data` |
| Payment methods | `POST /v2/payment-methods`, `POST /v2/payment-methods/create-intent`, `POST /v2/payment-methods/{id}/confirm-intent`, `GET|DELETE /v2/payment-methods/{id}`, `PUT /v2/payment-methods/{id}/update-saved-payment-method`, `GET /v2/payment-methods/token/{token}/details`, `GET /v2/payment-methods/{id}/check-network-token-status` |
| Payment-method sessions | `POST /v2/payment-method-sessions`, `GET|DELETE /v2/payment-method-sessions/{id}`, `POST …/confirm`, `GET …/list-payment-methods`, `PUT …/update-saved-payment-method` |
| Customers | `POST /v2/customers`, `GET|PUT|DELETE /v2/customers/{id}`, `GET /v2/customers/list`, `GET /v2/customers/reference/{merchant_reference_id}`, `GET /v2/customers/{id}/saved-payment-methods` |
| Refunds | `POST /v2/refunds`, `POST /v2/refunds/list`, `GET /v2/refunds/{id}`, `PUT /v2/refunds/{id}/update-metadata` |
| Routing | `POST /v2/routing-algorithms`, `GET /v2/routing-algorithms/{id}` (activation lives under profiles) |
| Tokenization | `POST /v2/tokenize`, `DELETE /v2/tokenize/{id}` `[feature: tokenization_v2]` |
| Proxy | `POST /v2/proxy` |
| API keys | `POST /v2/api-keys`, `GET /v2/api-keys/list`, `GET|PUT|DELETE /v2/api-keys/{id}` |
| Process trackers | `GET /v2/process-trackers/revenue-recovery-workflow/{id}` `[feature: revenue_recovery]` |
| Webhooks | `POST /v2/webhooks/{merchant_id}/{profile_id}/{connector_id}` |

The v2 spec also documents a set of `/v1/...` payment-method and customer paths that are
served by a v2 build for compatibility during migration.

Notable differences from v1:

* Ids are global and prefixed (`GlobalPaymentId`, `GlobalCustomerId`, …) and appear in the
  path rather than the body.
* Intent-first payments: `create-intent` → `confirm-intent`, with `get-intent` /
  `update-intent` in between.
* Payment-method *sessions* replace ad-hoc payment-method attach flows.
* Routing algorithms are created globally and activated per profile with explicit
  `PATCH` verbs.

## Conventions

* Amounts are minor units (`MinorUnit`); connectors that need base units declare
  `CurrencyUnit::Base` and conversion happens in the connector layer.
* Errors follow a stable envelope with `error.type`, `error.code`, `error.message` plus
  connector-level `unified_code` / `unified_message`
  (`crates/hyperswitch_domain_models/src/errors/api_error_response.rs`,
  `crates/api_models/src/errors/types.rs`).
* Idempotency: `payment_id` (or `merchant_reference_id` in v2) makes creation idempotent;
  confirm/capture are serialised by Redis API locks.
* List endpoints are cursor/offset paginated with domain-specific filter objects and have
  matching `/filters` and aggregate endpoints for the dashboard.
* SDK/client-facing endpoints require the publishable key plus `client_secret` and expose
  only a redacted view of the object.
