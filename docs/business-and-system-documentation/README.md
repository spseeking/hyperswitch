# Hyperswitch — Business Features & System Documentation

Documentation set derived by reading the source of [juspay/hyperswitch](https://github.com/juspay/hyperswitch)
at commit `9b8b89dc` (main, 2026-08-14). Everything here is grounded in the code
(crate layout, API specs under `api-reference/`, enums in `crates/common_enums`,
Diesel schema in `crates/diesel_models/src/schema.rs`, configuration in `config/`),
not in marketing material.

Hyperswitch is an open-source, connector-agnostic payments orchestration platform
written in Rust. A merchant integrates the Hyperswitch API once; Hyperswitch then
translates each operation into the API of one of ~150 payment processors, wallets,
fraud engines, 3DS servers, vaults, billing systems and payout rails, and adds
routing, retries, vaulting, reconciliation and reporting on top.

## Contents

| Doc | What it covers |
| --- | --- |
| [01 — Business capabilities](01-business-capabilities.md) | Feature catalogue by business domain: payments, captures, refunds, disputes, mandates/MIT, subscriptions, payouts, vault/tokenization, 3DS & fraud, revenue recovery, offers, analytics, merchant self-service |
| [02 — System architecture](02-system-architecture.md) | Runtime services, crate map, request path, storage engines, KV mode, multi-tenancy, API versions v1/v2 |
| [03 — Domain & data model](03-domain-and-data-model.md) | Account hierarchy (tenant → org → merchant → profile → connector account), core tables, identifiers, encryption of PII |
| [04 — Payment lifecycle](04-payment-lifecycle.md) | Intent/attempt state machines, operation pipeline, confirm→authorize→capture flows, 3DS redirection, sync, incremental/extended auth, cancellation |
| [05 — Routing & decisioning](05-routing-and-decisioning.md) | Static rules (Euclid DSL), eligibility constraint graph, volume split, success-rate/elimination/contract dynamic routing, debit routing, decision manager, surcharge, GSM-driven retries |
| [06 — Connector integration](06-connector-integration.md) | Connector trait model, flow types, adding a connector, Unified Connector Service, connector configs & payment-method filters |
| [07 — Webhooks, events & scheduler](07-webhooks-events-and-scheduler.md) | Incoming webhook ingestion, outgoing webhook delivery/retries, process tracker workflows, producer/consumer, drainer, Kafka event streams |
| [08 — API surface](08-api-surface.md) | Endpoint inventory for v1 and v2 grouped by domain, authentication schemes per endpoint class |
| [09 — Security, compliance & tenancy](09-security-and-compliance.md) | PCI posture, card vault/locker, key manager, blocklist & card-testing guard, RBAC model, secrets management |
| [10 — Deployment & operations](10-deployment-and-operations.md) | Docker/Compose topology, configuration layering, cargo feature flags, observability stack, load/integration testing |

## C4 diagrams

All architecture diagrams are PlantUML sources under [`diagrams/`](diagrams) using the
C4-PlantUML macros vendored in `diagrams/lib/`, with `.png` and `.svg` renders committed
next to them. Index and render instructions: [02 — System architecture](02-system-architecture.md).

| Level | Source |
| --- | --- |
| L1 System context | [c4-l1-system-context.puml](diagrams/c4-l1-system-context.puml) |
| L2 Containers | [c4-l2-container.puml](diagrams/c4-l2-container.puml) |
| L3 Router components | [c4-l3-component-router.puml](diagrams/c4-l3-component-router.puml) |
| L3 Scheduler & drainer | [c4-l3-component-scheduler-drainer.puml](diagrams/c4-l3-component-scheduler-drainer.puml) |
| L4 Payment operation code | [c4-l4-code-payment-operation.puml](diagrams/c4-l4-code-payment-operation.puml) |
| Dynamic — confirm a payment | [c4-dynamic-payment-confirm.puml](diagrams/c4-dynamic-payment-confirm.puml) |
| Dynamic — routing decision | [c4-dynamic-routing-decision.puml](diagrams/c4-dynamic-routing-decision.puml) |
| Dynamic — incoming webhook | [c4-dynamic-incoming-webhook.puml](diagrams/c4-dynamic-incoming-webhook.puml) |
| Deployment | [c4-deployment-docker-compose.puml](diagrams/c4-deployment-docker-compose.puml) |

## How to read this set

* Product/business readers: start with [01](01-business-capabilities.md), then
  [04](04-payment-lifecycle.md) and [05](05-routing-and-decisioning.md).
* Engineers integrating with Hyperswitch: [08](08-api-surface.md) →
  [04](04-payment-lifecycle.md) → [07](07-webhooks-events-and-scheduler.md).
* Engineers operating or extending Hyperswitch: [02](02-system-architecture.md) →
  [03](03-domain-and-data-model.md) → [06](06-connector-integration.md) →
  [10](10-deployment-and-operations.md).

## Conventions

* Code references are given as repository-relative paths, e.g.
  `crates/router/src/core/payments.rs`.
* `v1` and `v2` refer to the two API/data-model generations that are compiled as
  mutually exclusive cargo features (`--features v1` / `--features v2`); v1 is the
  stable generation, v2 is the in-progress redesign. Where behaviour differs, both
  are described.
* Feature-flagged capabilities are marked with the cargo feature that gates them
  (e.g. `payouts`, `frm`, `revenue_recovery`, `tokenization_v2`, `olap`).
