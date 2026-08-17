# 09 — Security, compliance & tenancy

Code: `crates/router/src/services/authentication.rs` (+ `authentication/`),
`crates/router/src/core/user/` + `user.rs`, `crates/router/src/core/user_role.rs`,
`crates/router/src/core/payment_methods/` (vault/locker access),
`crates/common_utils/src/keymanager.rs`, the `hyperswitch_masking` crate's
`Secret`/`Maskable` wrappers (re-exported as `masking`), `crates/external_services`
(AWS KMS / Secrets Manager / HashiCorp Vault).

## Card data & PCI posture

* Raw card data never rests in the Router's database. Cards go to the **Hyperswitch Card
  Vault (locker)**, a separate deployment, over JWE-encrypted requests signed with JWS
  (`[jwekey]`, `[locker]` config sections). The Router stores only the locker reference
  (`payment_methods.locker_id`), fingerprints (`locker_fingerprint_id`,
  `auxiliary_fingerprint_id`) and non-sensitive metadata.
* A **temp locker** holds card data for the duration of a single payment; the
  `DeleteTokenizeDataWorkflow` scheduler task purges tokenized data afterwards.
* **Network tokenization** replaces PANs with network tokens
  (`network_token_locker_id`, `network_token_requestor_reference_id`) and supports
  token-requestor webhooks and `check-network-token-status`.
* **External / BYO vault** — a merchant may keep card data in their own vault and have
  Hyperswitch proxy the payment (`is_external_vault_enabled`,
  `external_vault_connector_details`), reducing Hyperswitch's data scope further.
* **Extended card info** is stored only in Redis, encrypted with a merchant-supplied
  public key, with a TTL.
* Card numbers, CVCs, secrets and connector credentials are wrapped in
  `Secret`/`Maskable` types so logs and events emit masked values; connector API events
  are recorded with masked bodies.

## Encryption of PII

* Customer names/emails/phones, addresses, connector credentials and other sensitive
  columns are encrypted at the application layer with per-entity keys held in
  `merchant_key_store` / `user_key_store`.
* With `[feature: encryption_service]` the data keys are wrapped by an external **key
  manager** service (mTLS optional via `keymanager_mtls`), so plaintext data keys never
  live in the application database.
* Application secrets (database passwords, connector keys, JWT secret, master key) can be
  resolved at boot from AWS KMS / Secrets Manager or HashiCorp Vault
  (`crates/external_services`, `[secrets]` config) instead of plain config files.

## Authentication & authorization

Merchant-facing:

* **API key** — hashed at rest in `api_keys` with expiry; `ApiKeyExpiryWorkflow` emails
  reminders before expiry.
* **Publishable key + `client_secret`** — client-side scope: only the operations needed by
  the SDK on one intent/customer, returning redacted objects.
* **Ephemeral keys** — short-lived, customer-scoped tokens for client apps.
* **Admin API key** — provisioning-level operations.
* **Partial auth** (`[feature: partial-auth]`) — signed detached headers to reduce auth
  cost on hot paths (`services/authentication/detached.rs`).

Dashboard/users:

* JWT sessions with a blacklist for revocation (`services/authentication/blacklist.rs`),
  cookie support, and single-purpose tokens for invite/verify/reset flows.
* **TOTP 2FA** with recovery codes; enforcement configurable per tenant.
* **SSO / OIDC** per merchant (`user_authentication_methods`, `routes/oidc.rs`).
* **RBAC**: `roles` (predefined + custom) grant `PermissionScope::{Read, Write}` over
  resources — `Payment`, `Refund`, `Dispute`, `Mandate`, `Customer`, `Payout`, `ApiKey`,
  `Account`, `Connector`, `CloneConnector`, `Routing`, `ThreeDsDecisionManager`,
  `SurchargeDecisionManager`, `Analytics`, `Report`, `WebhookEvent`, `User`,
  `RevenueRecovery`, `Subscription`, `Theme`, `SuperpositionConfig` and the `Recon*`
  resources — at `RoleScope::{Organization, Merchant, Profile}`. `user_roles` binds a user
  to a role at an `EntityType` (`Tenant`/`Organization`/`Merchant`/`Profile`) level.

## Tenancy & isolation

* **Tenants** (`[multitenancy.tenants.*]`) get their own PostgreSQL schema, Redis key
  prefix, ClickHouse database and scheduler streams; the tenant is resolved per request
  and threaded through the session state.
* **Organizations / merchants / profiles** isolate data logically: every core table
  carries `merchant_id` (and increasingly `profile_id` / `organization_id`), and every
  query is scoped by them. Dashboard tokens are additionally scoped by RBAC entity.
* **Platform merchants** (`is_platform_account`, `platform_merchant_id`,
  `processor_merchant_id`) support marketplace/connected-account models where one account
  transacts on behalf of another.

## Abuse & fraud controls

* **Blocklist** — block payment-method fingerprints, card BINs and extended BINs; bulk
  loads via `BatchBlocklistUpload`.
* **Card-testing guard** — per-profile config (`card_testing_guard_config`,
  `card_testing_secret_key`) rate-limiting attempts per card/customer/session in Redis.
* **FRM connectors** — pre/post-authorization fraud decisions with a manual-review queue
  (`RequiresMerchantAction` → approve/reject) `[feature: frm]`.
* **Integrity checks** — connector responses are compared with the request; mismatches
  surface as `IntegrityFailure`/`Conflicted` instead of a silent success.
* **API locks** — Redis locks prevent concurrent confirm/capture from double-charging.
* **Webhook source verification** — per-connector signature verification; unverified
  payloads only trigger a PSync rather than being trusted.
* **Outgoing webhook signing** — HMAC over the payload with the merchant's
  `payment_response_hash_key`.

## Operational security

* CORS is configurable (`[cors]`), TLS can terminate in-process (`[feature: tls]`).
* Health endpoints (`/health`, `/health/ready`) report dependency status without leaking
  configuration.
* Every request carries a request id propagated to logs, connector calls (optionally to
  the key manager via `km_forward_x_request_id`) and events for auditability.
* `events`/audit-event streams record who changed what, and API-event logs record request
  metadata with masked bodies.
* Cache invalidation is authenticated (`/cache/**` admin routes) so a stale-account attack
  cannot be triggered by merchants.
