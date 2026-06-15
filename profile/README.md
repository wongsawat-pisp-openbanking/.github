# PISP Open Banking Platform

Payment Initiation Service Provider (PISP) — Open Banking Read/Write API v4.0.1 implementation. 18 microservices, each independently deployable, backed by PostgreSQL.

**Tech stack:** Java 21, Spring Boot 3.5, PostgreSQL 16, Flyway, Testcontainers (no Docker Compose)

---

## Services

### Payment Initiation

| Service | Description |
|---------|-------------|
| [domestic-payments](https://github.com/wongsawat-pisp-openbanking/domestic-payments) | FPS domestic payments — pacs.008 outbound, pacs.002 inbound, saga-driven status transitions |
| [file-payments](https://github.com/wongsawat-pisp-openbanking/file-payments) | Bulk pain.001 file payments via Spring Batch + FPS rail adapter |
| [international-payments](https://github.com/wongsawat-pisp-openbanking/international-payments) | International payment initiation |

### Scheduled Payments & Standing Orders

| Service | Description |
|---------|-------------|
| [domestic-scheduled-payments](https://github.com/wongsawat-pisp-openbanking/domestic-scheduled-payments) | Future-dated domestic payments |
| [international-scheduled-payments](https://github.com/wongsawat-pisp-openbanking/international-scheduled-payments) | Future-dated international payments |
| [domestic-standing-orders](https://github.com/wongsawat-pisp-openbanking/domestic-standing-orders) | Recurring domestic payments |
| [international-standing-orders](https://github.com/wongsawat-pisp-openbanking/international-standing-orders) | Recurring international payments |

### Consent Management

| Service | Description |
|---------|-------------|
| [domestic-payment-consents](https://github.com/wongsawat-pisp-openbanking/domestic-payment-consents) | Consent for domestic payments; funds confirmation endpoint |
| [file-payment-consents](https://github.com/wongsawat-pisp-openbanking/file-payment-consents) | Consent for bulk file payments; file upload + SHA-256 verification |
| [international-payment-consents](https://github.com/wongsawat-pisp-openbanking/international-payment-consents) | Consent for international payments |
| [domestic-scheduled-payment-consents](https://github.com/wongsawat-pisp-openbanking/domestic-scheduled-payment-consents) | Consent for scheduled domestic payments |
| [international-scheduled-payment-consents](https://github.com/wongsawat-pisp-openbanking/international-scheduled-payment-consents) | Consent for scheduled international payments |
| [domestic-standing-order-consents](https://github.com/wongsawat-pisp-openbanking/domestic-standing-order-consents) | Consent for domestic standing orders |
| [international-standing-order-consents](https://github.com/wongsawat-pisp-openbanking/international-standing-order-consents) | Consent for international standing orders |

### Platform Infrastructure

| Service | Description |
|---------|-------------|
| [saga-orchestrator](https://github.com/wongsawat-pisp-openbanking/saga-orchestrator) | Choreographed saga orchestration over Kafka (`pisp.saga-commands`) |
| [event-notification](https://github.com/wongsawat-pisp-openbanking/event-notification) | Security Event Token (SET) delivery to TPP callback URLs |
| [fee-engine](https://github.com/wongsawat-pisp-openbanking/fee-engine) | Drools-based fee calculation; supports FLAT, PERCENTAGE, TIERED_SLAB, TIERED_STEP, FREE |
| [fee-engine-admin-ui](https://github.com/wongsawat-pisp-openbanking/fee-engine-admin-ui) | React + TypeScript admin console for fee rule management |
| [fee-engine-ai-assistant](https://github.com/wongsawat-pisp-openbanking/fee-engine-ai-assistant) | AI-powered fee rule generation with human-in-the-loop approval |
| [fee-engine-demo](https://github.com/wongsawat-pisp-openbanking/fee-engine-demo) | Demo harness for the fee engine |

---

## Architecture

All services follow **hexagonal architecture** (ports + adapters) with domain / application / adapter / infrastructure layers.

```
Consent flow:  TPP → consent-service (AWAU → AUTH → COND)
                       ↓
Payment flow:  TPP → payment-service → saga-orchestrator → payment-service (status updates)
                                                         ↓
                                               event-notification → TPP callback
```

**Kafka topics** (saga-bearing services only):

| Topic | Direction | Services |
|-------|-----------|---------|
| `pisp.domain-events` | Outbox → consumers | domestic-payments, file-payments, saga-orchestrator, event-notification |
| `pisp.saga-commands` | Orchestrator → services | saga-orchestrator → domestic-payments, file-payments |
| `pisp.fps-outbound` | Service → FPS rail | domestic-payments, file-payments |
| `pisp.fps-inbound` | FPS rail → service | domestic-payments, file-payments |

**Security filter chain** (public endpoints):
`RawBodyCaching → MutualTLS (cnf.x5t#S256) → Idempotency → JWS → GrantType → ResponseSigning`

**JWS signing:** NoOp by default; BC-FIPS PS256 activated via `bcfips-integration` Maven profile.

---

## Payment Status Flow

| Service | Initial status | Terminal statuses |
|---------|---------------|-------------------|
| domestic-payments | `RCVD` → `ACTC` → `ACCP` → `ACFC` → `ACSP` | `ACSC` / `RJCT` |
| file-payments | `PDNG` → `ACTC` → `ACSP` | `ACSC` / `RJCT` |

## Consent Status Flow

| Service | States |
|---------|--------|
| domestic-payment-consents | `AWAU` → `AUTH` → `COND` \| `RJCT` |
| file-payment-consents | `AWUP` → `AWAU` → `AUTH` → `COND` \| `RJCT` |

---

## Build

Each service is a standalone Maven module. From any service directory:

```bash
mvn test                        # Run all tests (Testcontainers spins up PostgreSQL)
mvn test -Dtest=ClassName       # Single test class
mvn test -Pbcfips-integration   # BC-FIPS JWS integration tests
```
