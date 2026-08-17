# 06 — Connector integration

Code: `crates/hyperswitch_interfaces` (traits), `crates/hyperswitch_connectors` (~150
implementations under `src/connectors/`), `crates/connector_configs` (per-connector
config/metadata for the dashboard), `crates/router/src/connector.rs` (wiring),
`crates/router/src/core/unified_connector_service/` (out-of-process execution).

## The trait model

A connector is a set of trait implementations, one per **flow**, over generic
request/response types:

```rust
trait ConnectorCommon {                     // identity & auth basics
    fn id(&self) -> &'static str;
    fn get_currency_unit(&self) -> CurrencyUnit;      // Base | Minor
    fn get_auth_header(&self, auth: &ConnectorAuthType) -> Vec<(String, Maskable<String>)>;
    fn common_get_content_type(&self) -> &'static str;
    fn base_url(&self, connectors: &Connectors) -> &str;
    fn build_error_response(&self, res: Response) -> ErrorResponse;
}

trait ConnectorIntegration<Flow, Req, Res>: ConnectorIntegrationAny<Flow, Req, Res> {
    fn get_headers(...)      -> Vec<(String, Maskable<String>)>;
    fn get_url(...)          -> String;
    fn get_request_body(...) -> RequestContent;      // Json | FormUrlEncoded | Xml | RawBytes ...
    fn build_request(...)    -> Option<Request>;
    fn handle_response(...)  -> RouterData<Flow, Req, Res>;
    fn get_error_response(...) -> ErrorResponse;
}
```

Flows are marker types (`Authorize`, `PSync`, `Capture`, `Void`, `Session`,
`CompleteAuthorize`, `SetupMandate`, `Execute`/`RSync` for refunds, `IncrementalAuthorization`,
`Accept`/`Defend`/`Upload` for disputes, `PoCreate`/`PoFulfill`/`PoSync` for payouts,
FRM and authentication flows, …). `RouterData<Flow, Req, Res>` is the uniform envelope
carrying the request, the auth type, ids, amount, and the `Result<Res, ErrorResponse>`.

Additional interfaces a connector may implement:

* `IncomingWebhook` — `verify_webhook_source`, `get_webhook_object_reference_id`,
  `get_webhook_event_type`, `get_webhook_resource_object`
  (`crates/hyperswitch_interfaces/src/webhooks.rs`).
* `ConnectorValidation` — supported capture methods, mandate/PM validation.
* Integrity check traits (`integrity.rs`) — verify amount/currency/id consistency between
  request and response, producing `AttemptStatus::IntegrityFailure` on mismatch.
* `ConnectorSpecifications` / feature-matrix metadata — which payment methods, countries,
  currencies and flows are supported (surfaced by `routes/feature_matrix.rs`).
* Dispute, file-upload, CRM, relay and payout-specific traits.
* `default_implementations.rs` / `default_implementations_v2.rs` provide "not implemented"
  blanket impls so a connector only writes the flows it actually supports.

## Request path through the connector layer

1. The operation pipeline builds `RouterData` for the flow.
2. `services::call_connector_api` (`crates/router/src/services/api.rs`) executes the
   `Request` with retries/timeouts, records connector-API latency metrics and pushes a
   connector API event (masked) to the event sink.
3. `handle_response` transforms the connector payload into Hyperswitch domain types;
   status mapping produces an `AttemptStatus`, error mapping produces `ErrorResponse`
   with `unified_code`/`unified_message` in addition to the raw connector code.
4. `PostUpdateTracker` persists the result; failures may be routed through
   `gateway_status_map` for retry decisions.

## Configuration per connector

* Base URLs and environment endpoints: `[connectors.*]` sections in `config/*.toml`.
* Credentials: `merchant_connector_account.connector_account_details`, shaped by
  `ConnectorAuthType` (`HeaderKey`, `BodyKey`, `SignatureKey`, `MultiAuthKey`,
  `CertificateAuth`, …), stored encrypted.
* Enabled payment methods and their sub-types/limits:
  `merchant_connector_account.payment_methods_enabled`.
* Global payment-method filters and mandatory-field lists per connector:
  `crates/connector_configs/toml/*.toml`, `config/payment_required_fields_v2.toml`.
* Wallet-specific data (Apple Pay certificates, Google Pay merchant info):
  `connector_wallets_details`, `applepay_verified_domains`.
* Webhook secrets/registration: `connector_webhook_details`,
  `connector_webhook_registration_details`.

## Adding a connector

Mechanically (as done by the repo's own tooling, `crates/hsdev` and
`scripts/add_connector.sh`):

1. Generate the module in `crates/hyperswitch_connectors/src/connectors/<name>/` with
   `transformers.rs` (request/response structs + `TryFrom` conversions) and `<name>.rs`
   (trait impls).
2. Register the connector in `crates/common_enums` (`Connector`, `RoutableConnectors`),
   `crates/hyperswitch_connectors/src/connectors.rs` and the Router's connector map.
3. Add base URLs to every `config/*.toml`, add `crates/connector_configs` entries and
   payment-method filters.
4. Implement `IncomingWebhook` if the connector sends webhooks.
5. Add connector tests under `crates/router/tests/connectors/` (+ auth in
   `sample_auth.toml`) and Postman/Cypress collections where applicable.

## Unified Connector Service (UCS)

An optional gRPC service that executes connector calls out of process
(`crates/router/src/core/unified_connector_service/`,
`crates/hyperswitch_interfaces/src/unified_connector_service/`,
`routes/unified_connector_service.rs`, client config in
`[grpc_client.unified_connector_service]`, proto files under `proto/`).
The Router builds a normalised request, calls UCS, and maps the response back into the
same `RouterData` shape — so the operation pipeline is unchanged. This decouples connector
release cycles from the Router and allows connectors written in other stacks.

## Related external integrations

* **Card Vault (locker)** — JWE/JWS-encrypted card storage, separate deployment.
* **Key manager / encryption service** — envelope encryption for PII.
* **Open Router** — external routing/decision engine.
* **FRM, authentication (3DS), tax and billing connectors** — same trait model, selected
  via `merchant_connector_account.connector_type`.
