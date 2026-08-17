# 03 — Domain & data model

Source of truth: `crates/diesel_models/src/schema.rs` (v1, 51 tables) and
`schema_v2.rs` (v2, 52 tables), domain types in `crates/hyperswitch_domain_models`,
migrations in `migrations/` and `v2_migrations/`.

## Account hierarchy

```
Tenant                     (config-level: own DB schema, Redis prefix, ClickHouse DB)
└── organization           (org_id / organization_type, platform_merchant_id)
    └── merchant_account   (merchant_id, publishable_key, storage_scheme, default_profile)
        └── business_profile        (profile_id — the unit of payment configuration)
            └── merchant_connector_account  (per connector: credentials, PM matrix, status)
```

* **organization** — top-level billing/reporting boundary; `organization_type` and
  `platform_merchant_id` support the platform/connected-merchant model.
* **merchant_account** (34 columns) — API-level identity: `publishable_key` for
  client-side calls, `storage_scheme` (`PostgresOnly` | `RedisKv`), `locker_id`,
  `product_type`, `merchant_account_type`, `fingerprint_secret`, recon status,
  `organization_id`, `default_profile`.
* **business_profile** (65 columns) — where almost all behaviour is configured:
  `routing_algorithm`, `dynamic_routing_algorithm`, `three_ds_decision_rule_algorithm`,
  `default_fallback_routing`, `webhook_details` + `outgoing_webhook_custom_http_headers`,
  `session_expiry`, `order_fulfillment_time`, `authentication_connector_details`,
  `is_network_tokenization_enabled`, `is_click_to_pay_enabled`, `is_debit_routing_enabled`,
  `is_auto_retries_enabled` / `max_auto_retries_enabled` / `is_manual_retry_enabled`,
  `card_testing_guard_config`, `force_3ds_challenge`, `is_external_vault_enabled` +
  `external_vault_connector_details`, `always_enable_overcapture`,
  `billing_processor_id`, `acquirer_config_map`, `dispute_polling_interval`,
  `payment_method_blocking`, `surcharge_connector_details`, `is_l2_l3_enabled`.
* **merchant_connector_account** (27 columns) — `connector_name`,
  `connector_account_details` (encrypted `ConnectorAuthType`), `connector_type`
  (payment processor / payout processor / FRM / authentication / tax / billing),
  `payment_methods_enabled`, `connector_label`, `connector_webhook_details`,
  `status`, `test_mode`, `disabled`, `additional_merchant_data`,
  `connector_wallets_details`, `pm_auth_config`.
* **merchant_key_store** / **user_key_store** — per-entity encryption keys; PII columns
  are envelope-encrypted (optionally through an external key manager,
  `[feature: encryption_service]`).

## Payment aggregate

**payment_intent** (v1 PK `(payment_id, merchant_id)`, 80 columns) — the merchant-facing
order: `status: IntentStatus`, `amount`, `currency`, `amount_captured`, `customer_id`,
`profile_id`, `organization_id`, `client_secret`, `active_attempt_id`, `attempt_count`,
`setup_future_usage`, `off_session`, `session_expiry`, `return_url` /
`extended_return_url`, `order_details`, `allowed_payment_method_types`,
`request_incremental_authorization` / `authorization_count`,
`request_extended_authorization`, `enable_partial_authorization`, `enable_overcapture`,
`force_3ds_challenge(_trigger)`, `psd2_sca_exemption_type`, `split_payments`,
`surcharge_applicable` / `external_surcharge_*`, `tax_details` / `tax_status`,
`shipping_cost` / `shipping_amount_tax` / `duty_amount` / `discount_amount`,
`frm_metadata`, `payment_link_id`, `merchant_order_reference_id`,
`profile_acquirer_id`, `installment_options`, `tokenization`, `payment_channel`,
`mit_category`, `billing_descriptor`, `platform_merchant_id` / `processor_merchant_id`.

**payment_attempt** (v1 PK `(attempt_id, merchant_id)`, 93 columns) — one row per
processor attempt: `status: AttemptStatus`, `connector`, `merchant_connector_id`,
`net_amount`, `amount_capturable`, `authorized_amount`, `capture_method`,
`authentication_type`, `payment_method` / `payment_method_type` / `payment_method_data`,
`payment_token`, `mandate_id` / `mandate_details` / `connector_mandate_detail`,
`connector_transaction_id` + `processor_transaction_data`,
`connector_request_reference_id` / `connector_response_reference_id`,
`network_transaction_id` / `network_details` / `network_transaction_link_id`,
`error_code` / `error_message` / `unified_code` / `unified_message` /
`issuer_error_code` / `issuer_error_message` / `error_details`,
`external_three_ds_authentication_attempted` + `authentication_connector` +
`authentication_id` + `authentication_data`, `routing_approach`, `retry_type`,
`straight_through_algorithm`, `multiple_capture_count`,
`extended_authorization_applied` / `capture_before`, `is_overcapture_enabled`,
`is_stored_credential`, `card_discovery`, `card_network`, `charges`,
`installment_data`, `external_surcharge_details`, `applied_offer_details`.

Related: **captures** (multiple partial captures), **incremental_authorization**,
**authentication** (3DS records), **fraud_check**, **payment_link**, **relay**.

Identifier conventions: ids are typed newtypes in `crates/common_utils/src/id_type/`
(`PaymentId`, `MerchantId`, `ProfileId`, `CustomerId`, `GlobalPaymentId` in v2, …) and
carry a prefix (e.g. `pay_`, `mca_`, `pm_`). `client_secret` pairs with the publishable
key for client-side calls. Connector references are stored in two places for long values
(`connector_transaction_id` + `processor_transaction_data`).

## Other aggregates

| Aggregate | Tables | Key fields |
| --- | --- | --- |
| Refunds | `refund` | `refund_status`, `refund_type`, `total_amount` vs `refund_amount`, `connector_refund_id` + `processor_refund_data`, `split_refunds`, `attempt_id` |
| Disputes | `dispute`, `file_metadata` | `dispute_stage`, `dispute_status`, `connector_dispute_id`, `challenge_required_by`, `evidence`, `dispute_amount` |
| Mandates | `mandate` | `mandate_status`, single/multi-use, amount caps, `connector_mandate_id`, `network_transaction_id` |
| Payment methods | `payment_methods` | `status`, `locker_id` / `locker_fingerprint_id`, `network_token_*`, `external_vault_source` / `vault_type`, `connector_mandate_details`, `payment_method_subtype` (v2) |
| Customers | `customers`, `address` | encrypted `name`/`email`/`phone`, `connector_customer`, `default_payment_method_id`, `tax_registration_id` |
| Payouts | `payouts`, `payout_attempt` | `PayoutStatus`, payout method data, vendor account state |
| Subscriptions | `subscription`, `invoice` | `SubscriptionStatus`, billing-connector references |
| Routing | `routing_algorithm`, `dynamic_routing_stats` | versioned algorithms per profile; per-attempt routing outcome for score feedback |
| Risk | `blocklist`, `blocklist_fingerprint`, `blocklist_lookup`, `batch_blocklist_jobs`, `gateway_status_map` | fingerprints/BIN blocks; connector error → `GsmDecision` mapping |
| Scheduling | `process_tracker` | `name`, `runner`, `schedule_time`, `retry_count`, `status`, `business_status`, `tracking_data` (JSON payload), `tag` |
| Eventing | `events` | `event_type`, `event_class`, `primary_object_id/type`, `idempotent_event_id`, `initial_attempt_id`, `delivery_attempt`, `request`/`response`, `is_overall_delivery_successful` |
| Users & access | `users`, `user_roles`, `roles`, `api_keys`, `user_authentication_methods`, `dashboard_metadata` | RBAC scopes/resources, hashed API keys with expiry, SSO config |
| Reference data | `cards_info`, `card_issuers`, `configs`, `unified_translations`, `themes`, `callback_mapper`, `locker_mock_up` | BIN data, runtime config, i18n, white-labelling |

## v1 vs v2 model changes

* v2 adds `id` columns as global, prefixed identifiers (e.g. `GlobalPaymentId`) and moves
  the merchant/profile scoping into the URL path instead of request bodies.
* `payment_methods` is restructured (`payment_method_type_v2`, `payment_method_subtype`,
  modular forward/backward-compat migration workflows exist as scheduler runners).
* `merchant_account` / `business_profile` carry a `version` column so both generations can
  coexist in one database during migration.
* Migrations live in `migrations/` (v1) and `v2_migrations/`; the two schemas are compiled
  in mutually exclusively via the `v1` / `v2` cargo features.

## Storage behaviour

* **PostgresOnly** — writes go directly to PostgreSQL (master), reads for
  list/analytics endpoints prefer the read replica (`[replica_database]` in config).
* **RedisKv** — the Router writes the entity into a Redis hash and appends the SQL-ish
  operation to a write-ahead stream, plus `reverse_lookup` rows so secondary ids resolve;
  the Drainer replays the stream into PostgreSQL. Chosen per merchant via
  `merchant_account.storage_scheme` and gated by the `kv_store` feature.
* Cached read-through entities (accounts, connector accounts, routing algorithms,
  constraint graphs, configs) are invalidated over a Redis pub/sub channel.
