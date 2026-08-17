# 01 — Business capabilities

Feature catalogue by business domain. Every capability below was confirmed against
the code: API routes in `crates/router/src/routes/`, request/response contracts in
`crates/api_models/src/`, business logic in `crates/router/src/core/`, enums in
`crates/common_enums/src/enums.rs`, tables in `crates/diesel_models/src/schema.rs`,
and cargo feature gates in `crates/router/Cargo.toml`.

Cargo features that gate whole domains are noted as `[feature: x]`.

---

## 1. Merchant onboarding & account management

| Capability | Notes | Code |
| --- | --- | --- |
| Multi-level account hierarchy | Tenant → organization → merchant account → business profile → merchant connector account. `EntityType = Tenant, Organization, Merchant, Profile` | `core/admin.rs`, `routes/admin.rs`, `routes/profiles.rs` |
| Business profiles | Per-profile configuration of routing algorithm, webhook URL, 3DS/decision rules, authentication connector, collect-billing/shipping toggles, capture behaviour, session expiry | `business_profile` table, `core/admin.rs` |
| Connector accounts | Store per-connector credentials (`ConnectorAuthType`), payment-method-enabled matrix, metadata, test/live mode, disabled flag, frm/authentication/payout connector types | `merchant_connector_account` table |
| Connector credential verification | Dry-run a connector's credentials before enabling it | `routes/verify_connector.rs`, `core/verify_connector.rs` |
| Guided connector onboarding | OAuth-style onboarding flows for connectors that support it (e.g. PayPal) | `routes/connector_onboarding.rs` |
| Acquirer configuration per profile | Acquirer BIN/MID/label config used by network-token and debit routing decisions | `routes/profile_acquirer.rs` |
| Toggling platform features per merchant | KV storage scheme (`PostgresOnly` / `RedisKv`), platform-merchant/connected-account model, extended card info, dynamic routing toggles | `core/admin.rs`, `MerchantStorageScheme` |
| Business/config key-value store | Runtime configuration overridable per merchant without redeploy | `configs` table, `routes/configs.rs` |
| Feature matrix | Machine-readable "which connector supports which payment method / flow / country / currency" catalogue exposed over the API | `routes/feature_matrix.rs` |

## 2. Payment acceptance (core)

| Capability | Notes | Code |
| --- | --- | --- |
| Payment intent lifecycle | `IntentStatus`: `RequiresPaymentMethod`, `RequiresConfirmation`, `RequiresCustomerAction`, `RequiresMerchantAction`, `Processing`, `RequiresCapture`, `PartiallyCaptured`, `PartiallyCapturedAndCapturable`, `PartiallyAuthorizedAndRequiresCapture`, `PartiallyCapturedAndProcessing`, `Succeeded`, `Failed`, `Cancelled`, `CancelledPostCapture`, `Expired`, `Conflicted`, `Review` | `payment_intent` table |
| Attempt lifecycle | `AttemptStatus` (29 states) tracks each processor attempt: `Started`, `AuthenticationPending`, `AuthenticationSuccessful`, `Authorizing`, `Authorized`, `Charged`, `PartialCharged`, `PartiallyAuthorized`, `CaptureInitiated`, `CaptureReview`, `Voided`, `VoidedPostCharge`, `IntegrityFailure`, `Expired`, `Unresolved`, … | `payment_attempt` table |
| Create / update / confirm / retrieve / list payments | One `Operation` implementation per API verb | `core/payments/operations/` |
| Server-side and client-side confirmation | Publishable key + `client_secret` for client-side confirm; API key for server-side | `services/authentication.rs` |
| Capture methods | `Automatic`, `Manual`, `ManualMultiple` (multiple partial captures tracked in `captures`), `Scheduled`, `SequentialAutomatic` | `CaptureMethod`, `core/payments/operations/payment_capture.rs` |
| Void / cancel | Cancel before capture, plus post-capture cancellation where the processor supports it (`CancelledPostCapture`) | `payment_cancel.rs`, `payment_cancel_post_capture.rs` |
| Incremental authorization | Raise the authorized amount on an existing auth; history in `incremental_authorization` | `payment_incremental_authorization.rs` |
| Extended authorization | Extend the validity of an existing authorization | `payment_extend_authorization.rs` |
| Payment sync | Pull authoritative status from the processor, on demand or scheduled (`PaymentsSyncWorkflow`) | `core/payments/operations/payment_status.rs`, `crates/router/src/workflows/` |
| Session tokens / wallet sessions | Create wallet session objects (Apple Pay, Google Pay, PayPal, Samsung Pay, Klarna…) for a single intent across several connectors | `payment_session.rs`, `ConnectorCallType::SessionMultiple` |
| Post-session tokens & session update | Amount/tax update after a wallet session (e.g. Apple Pay shipping change) | `payment_post_session_tokens.rs`, `payment_session_update.rs` |
| Dynamic tax calculation | Call a tax connector during checkout for shipping-dependent tax | `payments_dynamic_tax_calculation`, `payment_session_update.rs` |
| Surcharge | Merchant-configured surcharge/tax-on-surcharge per payment-method, applied before authorization; surcharge events (`SurchargePaymentSucceeded`) | `core/surcharge_decision_config.rs`, `core/payments/conditional_configs.rs` |
| Split payments | Split settlement / platform-fee constructs for marketplaces (Stripe Connect-style, Adyen platforms, Xendit) | `core/split_payments.rs` |
| Account updater | Refresh stored card credentials via account-updater style flows | `core/account_updater.rs` |
| Approve / reject | Manual review outcome for payments held by FRM or the merchant | `payment_approve.rs`, `payment_reject.rs` |
| Payment methods list for an intent | Filtered, ranked list of eligible payment methods for the checkout UI | `core/payment_methods/`, `crates/payment_methods` |
| Payment links | Hosted, brandable checkout page per payment; secure/open links, expiry, themes | `routes/payment_link.rs`, `payment_link` + `generic_link` tables |
| Retries | GSM-driven retry of a failed attempt on the next eligible connector `[feature: retry]` | `core/payments/retry.rs`, `gateway_status_map` |
| Payment metadata / order details | Order line items, shipping/billing addresses, browser info, customer-visible descriptors | `api_models/payments.rs` |
| Proxy payments | Execute a payment against a connector using a merchant-supplied token/vault (v2) | `payment_proxy_intent.rs`, `external_vault_proxy_payment_intent.rs` |
| Dummy connector sandbox | Built-in fake processor for demos and tests `[feature: dummy_connector]` | `routes/dummy_connector/` |

## 3. Authentication & 3-D Secure

| Capability | Notes | Code |
| --- | --- | --- |
| Native and external 3DS | `AuthenticationType = ThreeDs, NoThreeDs`; separate authentication connectors (3DS servers) decoupled from the authorization connector | `authentication` table, `core/authentication.rs` |
| Three-DS decision rules | Rule engine deciding 3DS vs no-3DS / exemption per payment | `routes/three_ds_decision_rule.rs`, `core/three_ds_decision_rule.rs` |
| Unified Authentication Service | Delegate authentication orchestration to an external service | `core/unified_authentication_service.rs` |
| Click-to-pay / network authentication | Network-token and click-to-pay style flows via authentication connectors | `core/unified_authentication_service.rs` |
| Redirect & challenge handling | `next_action = redirect_to_url`, return URL handling, `CompleteAuthorize` after the challenge, device-data-collection state | `payment_complete_authorize.rs`, `AttemptStatus::DeviceDataCollectionPending` |
| Authentication webhooks | `IncomingWebhookEvent` variants for authentication outcomes | `api_models/webhooks.rs` |

## 4. Payment methods, vaulting & tokenization

| Capability | Notes | Code |
| --- | --- | --- |
| Card vault (locker) | Cards stored in the separate Hyperswitch Card Vault service over JWE/JWS; temp locker for in-flight card data | `core/payment_methods/`, `locker_mock_up` table |
| Saved payment methods | Per-customer payment methods with `PaymentMethodStatus = Active, Inactive, Processing, AwaitingData, New, Redacted` | `payment_methods` table, `crates/payment_methods` |
| Network tokenization | Provision and use network tokens; token requestor webhooks; `NetworkTokenizationWorkflow` scheduled task | `core/payment_methods/network_tokenization.rs` |
| Tokenization v2 | Generic vault/tokenization records decoupled from cards `[feature: tokenization_v2]` | `routes/tokenization.rs` |
| External / BYO vault | Merchant keeps card data in their own vault; Hyperswitch proxies with their token | `external_vault_proxy_payment_intent.rs` |
| Payment method migration | Bulk import of cards/tokens from another processor or PSP vault | `core/payment_methods/` |
| Apple Pay certificate migration | Rotate/migrate Apple Pay merchant certificates | `routes/apple_pay_certificates_migration.rs` |
| Payment method data collection links | Hosted link asking a customer to add/update a payment method | `generic_link` table, `core/generic_link/` |
| Open-banking / PM auth | Bank-account linking via payment-method-auth connectors (e.g. Plaid) | `routes/pm_auth.rs` |
| Extended card info | Temporarily store extra card data in Redis for merchant-side use, key-encrypted | `core/payment_methods/`, profile `is_extended_card_info_enabled` |
| Card BIN / issuer intelligence | BIN → network/issuer/country/type lookup used by routing and UI | `cards_info`, `card_issuers` tables, `routes/cards_info.rs` |

## 5. Customers

| Capability | Notes | Code |
| --- | --- | --- |
| Customer CRUD + list | Encrypted PII (name, email, phone) via the key manager | `customers` table, `core/customers.rs` |
| Customer deletion / redaction | Redaction that cascades to payment methods and addresses | `core/customers.rs` |
| Addresses | Billing/shipping addresses, encrypted, reusable across payments | `address` table |
| Ephemeral & client keys | Short-lived keys so a client app can act for one customer/payment | `routes/ephemeral_key.rs` |
| Connector customers | Map a Hyperswitch customer to per-connector customer ids | `connector_customer_id` on attempt/customer |

## 6. Mandates, recurring & subscriptions

| Capability | Notes | Code |
| --- | --- | --- |
| Mandates / setup intents | `MandateStatus = Active, Inactive, Pending, Revoked`; single-use and multi-use, amount caps, `FutureUsage = OffSession, OnSession` | `mandate` table, `core/mandate.rs` |
| MIT / recurring payments | Charge a stored mandate or network transaction id off-session | `payment_recurrence.rs` (`PaymentRecurrence` operation) |
| Mandate revocation | Merchant- or customer-initiated revoke, plus `MandateRevoked` webhook | `routes/mandates.rs` |
| Zero-amount / setup-only flows | Save a payment method without charging | `SetupMandate` flow |
| Subscriptions & invoices | `SubscriptionStatus = Created, Trial, Active, Pending, Paused, Unpaid, Onetime, InActive, Cancelled, Failed`; invoice records, invoice sync workflow, billing-connector integration, `InvoicePaid` webhook | `crates/subscriptions`, `subscription` + `invoice` tables, `routes/subscription.rs`, `InvoiceSyncflow` |

## 7. Refunds

| Capability | Notes | Code |
| --- | --- | --- |
| Full & partial refunds | Multiple refunds per payment, refund against a specific capture | `refund` table, `core/refunds.rs` |
| Refund status machine | `RefundStatus = Pending, Success, Failure, TransactionFailure, ManualReview` | `common_enums` |
| Refund sync & retries | `RefundWorkflowRouter` process-tracker workflow polls pending refunds | `crates/router/src/workflows/`, `core/refunds.rs` |
| Refund lists, filters, aggregates | Merchant/profile-scoped listing with filters and OLAP aggregates | `routes/refunds.rs` |
| Refund webhooks | `RefundSucceeded`, `RefundFailed`, `SurchargeRefundSucceeded` | `EventType` |

## 8. Disputes & evidence

| Capability | Notes | Code |
| --- | --- | --- |
| Dispute ingestion | Disputes created from connector webhooks and dispute-list polling | `dispute` table, `core/disputes.rs` |
| Dispute stages & statuses | `DisputeStage = PreDispute, Dispute, PreArbitration, Arbitration, DisputeReversal`; `DisputeStatus = DisputeOpened, DisputeExpired, DisputeAccepted, DisputeCancelled, DisputeChallenged, DisputeWon, DisputeLost` | `common_enums` |
| Accept / challenge / submit evidence | Attach evidence files and submit to the processor | `routes/disputes.rs`, `routes/files.rs`, `file_metadata` table |
| Scheduled dispute processing | `ProcessDisputeWorkflow` and `DisputeListWorkflow` scheduled tasks | `crates/scheduler` runners |
| Dispute webhooks | Six `DisputeX` webhook event types to the merchant | `EventType` |

## 9. Payouts `[feature: payouts]`

| Capability | Notes | Code |
| --- | --- | --- |
| Payout lifecycle | `PayoutStatus = RequiresCreation, RequiresConfirmation, RequiresPayoutMethodData, RequiresVendorAccountCreation, RequiresFulfillment, Initiated, Pending, Success, Failed, Cancelled, Reversed, Expired, Ineligible, NotPermitted` | `payouts`, `payout_attempt` tables |
| Create / update / confirm / fulfil / cancel payouts | Card, bank and wallet payout methods | `core/payouts.rs`, `routes/payouts.rs` |
| Payout links | Hosted page where the recipient supplies payout method data | `routes/payout_link.rs` |
| Payout routing & retries | Routing algorithms per payout, `[feature: payout_retry]` | `core/payouts/` |
| Vendor/recipient account creation | `AttachPayoutAccountWorkflow` scheduled task | `crates/scheduler` |
| Payout sync | `PayoutSyncWorkFlow` polls terminal status | `crates/scheduler` |
| Payout webhooks | `PayoutSuccess`, `PayoutFailed`, `PayoutInitiated`, `PayoutProcessing`, `PayoutCancelled`, `PayoutExpired`, `PayoutReversed`, `PayoutNotPermitted` | `EventType` |

## 10. Routing & decisioning

Detailed in [05 — Routing & decisioning](05-routing-and-decisioning.md).

| Capability | Notes | Code |
| --- | --- | --- |
| Static routing algorithms | Priority, volume-split and advanced rule-based (Euclid DSL) algorithms, versioned per profile | `routing_algorithm` table, `crates/euclid` |
| Eligibility constraint graph | Compiled graph of connector/payment-method/country/currency/mandate constraints filtering ineligible connectors | `crates/hyperswitch_constraint_graph`, `crates/kgraph_utils` |
| Success-rate based routing | Pick the connector with the highest recent auth rate | `core/routing/`, `dynamic_routing_stats` table |
| Elimination routing | Temporarily eliminate degraded connectors | `api_models/routing.rs` |
| Contract-based routing | Honour processor volume commitments | `api_models/routing.rs` |
| Debit routing | Least-cost routing across co-badged debit networks | `core/debit_routing.rs` |
| Decision manager | Rule-driven 3DS/no-3DS and surcharge decisions | `core/conditional_config.rs`, `core/surcharge_decision_config.rs` |
| Fallback routing | Profile default connector list when nothing else applies | `core/routing.rs` |
| Routing over Open Router | Optional external intelligent-routing service | `core/routing/` |

## 11. Fraud, risk & abuse prevention

| Capability | Notes | Code |
| --- | --- | --- |
| FRM integration | Pre- and post-authorization fraud checks with FRM connectors (e.g. Signifyd, Riskified) `[feature: frm]` | `crates/router/src/core/fraud_check.rs`, `fraud_check` table |
| FRM fulfilment webhook | `/webhooks/frm_fulfillment` for post-delivery signalling | `routes/webhooks.rs` |
| Manual review queue | `ManualReview`/`Review` statuses with approve/reject APIs | `core/payments/operations/payment_approve.rs` |
| Blocklist | Block by `PaymentMethod` fingerprint, `CardBin`, `ExtendedCardBin`; bulk upload via `BatchBlocklistUpload` | `blocklist`, `blocklist_fingerprint`, `blocklist_lookup`, `batch_blocklist_jobs` tables |
| Card-testing guard | Rate-limit card-testing attacks per card/customer/IP, configured per profile | `core/card_testing_guard.rs` |
| Integrity checks | Detect connector responses inconsistent with the request (`IntegrityFailure`) | `crates/hyperswitch_interfaces` integrity traits |

## 12. Revenue recovery `[feature: revenue_recovery]`

| Capability | Notes | Code |
| --- | --- | --- |
| Passive dunning / recovery | Retry failed subscription/invoice payments on a schedule (`PassiveRecoveryWorkflow`) | `core/revenue_recovery.rs` |
| Recovery webhooks | Billing-connector recovery/invoice events ingested through the recovery webhook routes | `routes/recovery_webhooks.rs` |
| Recovery data backfill & Redis state | Backfill APIs and Redis-held recovery state | `routes/revenue_recovery_data_backfill.rs`, `routes/revenue_recovery_redis.rs` |

## 13. Offers

| Capability | Notes | Code |
| --- | --- | --- |
| Offer engine | Evaluate merchant/issuer offers and discounts for a payment | `routes/offer_engine.rs` |

## 14. Reconciliation, relay & proxy

| Capability | Notes | Code |
| --- | --- | --- |
| Relay | Forward refund/dispute operations for payments that were *not* processed by Hyperswitch | `relay` table, `routes/relay.rs`, `/webhooks/relay/...` |
| Generic proxy | Authenticated pass-through to a connector endpoint | `routes/proxy.rs` |
| Process tracker admin | Inspect/retry/revoke scheduled tasks | `routes/process_tracker.rs` |
| Callback mapper | Map connector callbacks/ids back to Hyperswitch objects | `callback_mapper` table |

## 15. Analytics, reporting & search `[feature: olap]`

| Capability | Notes | Code |
| --- | --- | --- |
| Payment/refund/dispute/payout metrics | Time-series and aggregate metrics over ClickHouse (or PostgreSQL fallback) | `crates/analytics` |
| Auth-rate, SR, latency, retries dashboards | Metric definitions per domain plus filters and dimension lists | `crates/analytics/src/*/metrics` |
| Global search | Full-text/faceted search across payments, refunds, disputes via OpenSearch | `crates/analytics/src/opensearch.rs` |
| Report generation | CSV/scheduled reports delivered by email | `crates/analytics/src/report.rs` |
| Event/audit trails | API events, connector API logs, outgoing webhook logs, audit events on Kafka | `crates/events`, `routes/webhook_events.rs` |
| Dashboard metadata | Control-center onboarding/usage state | `dashboard_metadata` table |
| AI assistant interactions | Chat/AI interaction logging tables and route | `hyperswitch_ai_interaction` table, `routes/chat.rs` |

## 16. Webhooks & events

Detailed in [07 — Webhooks, events & scheduler](07-webhooks-events-and-scheduler.md).

* Incoming connector webhooks with per-connector source verification, idempotent state reconciliation, and forced PSync when the payload is not trusted.
* Outgoing merchant webhooks: signed `OutgoingWebhookContent` (payment / refund / dispute / mandate / payout / subscription), delivery attempts persisted in `events`, retries via `OutgoingWebhookRetryWorkflow`.
* 32 `EventType` values (see §2/§7/§9) and a webhook-events API for delivery inspection and manual re-delivery.

## 17. Users, access control & platform security

Detailed in [09 — Security, compliance & tenancy](09-security-and-compliance.md).

| Capability | Notes | Code |
| --- | --- | --- |
| API keys | Hashed keys, expiry, `ApiKeyExpiryWorkflow` reminders | `api_keys` table, `routes/api_keys.rs` |
| Dashboard users & invites | Sign-up/sign-in, invitations, email verification, password reset | `users`, `user_roles` tables, `routes/user/` |
| RBAC | Predefined and custom roles, `RoleScope = Organization, Merchant, Profile`, `PermissionScope = Read, Write` per resource | `roles` table, `routes/user_role.rs` |
| SSO / OIDC | External identity providers per merchant | `routes/oidc.rs`, `user_authentication_methods` table |
| TOTP / 2FA & recovery codes | Enforced 2FA flows for dashboard users | `core/user/two_factor_auth.rs` |
| Key manager & encryption service | Per-merchant key store, envelope encryption of PII `[feature: encryption_service]` | `merchant_key_store`, `user_key_store` tables |
| Themes & translations | White-labelled dashboard/link theming, unified translations | `themes`, `unified_translations` tables |
| Hypersense / Chat | Internal analytics assistant integration | `routes/hypersense.rs`, `routes/chat.rs` |

---

## Capability coverage at a glance

* ~150 connector integrations (`crates/hyperswitch_connectors/src/connectors/`) spanning
  card processors, wallets, bank redirects/debits/transfers, BNPL, crypto, UPI,
  vouchers, gift cards, real-time payments, open banking, FRM, 3DS, tax, billing and payout rails.
* 16 payment-method families (`PaymentMethod` enum) and 18 card networks (`CardNetwork`).
* 51 v1 tables / 52 v2 tables (`crates/diesel_models/src/schema.rs`, `schema_v2.rs`).
* 17 scheduled workflow runners (`ProcessTrackerRunner`).
* Two API generations (v1, v2) — see [08 — API surface](08-api-surface.md).
