---
tags: [phase-0, project-structure, api-contracts, event-contracts]
---

# Project Structure & Contracts
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

## Folder Structure (business-first, not layer-first)
Organized by module ([[03-DDD-and-Onion-Architecture|bounded context]]), each internally following the [[03-DDD-and-Onion-Architecture|Onion layering]] (`domain → application → infrastructure`). Nobody should be able to tell what web framework this uses by looking at the top-level tree — that's the point.

```
ayna-backend/
├── cmd/
│   ├── api/                   # composition root for the monolith HTTP+gRPC server
│   │   └── main.go
│   └── worker/                # composition root for the AI analysis worker
│       └── main.go
├── internal/
│   ├── modules/
│   │   ├── iam/
│   │   │   ├── domain/        # User, Credential, ConsentRecord, port interfaces
│   │   │   ├── application/   # RegisterUserUseCase, GrantConsentUseCase...
│   │   │   └── infrastructure/
│   │   │       ├── http/      # REST handlers for /auth/*
│   │   │       └── postgres/  # UserRepository impl
│   │   ├── skinanalysis/      # CORE domain — most investment goes here
│   │   │   ├── domain/        # Scan, SkinReport, Issue, Severity, ports
│   │   │   ├── application/   # SubmitScanUseCase, GenerateReportUseCase...
│   │   │   └── infrastructure/
│   │   │       ├── http/
│   │   │       ├── postgres/
│   │   │       ├── storage/   # object storage adapter (photos)
│   │   │       ├── aiprovider/# 3rd-party AI vision API client
│   │   │       └── queue/     # publisher for ScanSubmitted, consumer for SkinReportGenerated
│   │   ├── skinprofile/
│   │   ├── recommendations/
│   │   ├── growth/
│   │   ├── notifications/
│   │   └── billing/
│   └── platform/               # cross-cutting, NOT business logic
│       ├── config/
│       ├── logger/
│       ├── tracing/
│       ├── eventbus/           # in-process now; swappable for NATS/Redis Streams later (ADR-001)
│       └── httpserver/
├── proto/                      # gRPC/protobuf defs, if used internally (worker ↔ monolith)
├── api/                        # OpenAPI spec(s) for the public REST contract
├── events/                     # JSON Schema for domain event payloads (see below)
├── migrations/                 # SQL migrations, one subfolder per module
├── deployments/                 # Dockerfiles, k8s manifests / compose
└── go.mod
```

**Enforcement, not just convention:** a module's `domain` package must never import another module's `infrastructure` package, and no module's `domain` package imports anything outside the standard library. Enforce with a lint rule (e.g. `go-arch-lint` or a small custom CI check) rather than relying on memory during code review — this is the rule that keeps ADR-001's monolith from becoming a ball of mud.

## API Contracts (REST)
Resource-oriented, versioned from day one (`/v1`). Not exhaustive — the two riskiest flows are specced in full; the rest follow the same shape.

| Method | Path | Purpose | Auth |
|---|---|---|---|
| POST | `/v1/auth/register` | Register via email/phone | none |
| POST | `/v1/auth/login` | Authenticate, issue JWT | none |
| POST | `/v1/consent` | Record ConsentGranted | user |
| POST | `/v1/scans` | Submit a photo (multipart) → `202 Accepted` + `scanId` | user |
| GET | `/v1/scans/{scanId}` | Poll job status (`processing`\|`completed`\|`rejected`\|`failed`) | user |
| GET | `/v1/scans/{scanId}/report` | Fetch the completed SkinReport | user |
| GET | `/v1/profile/me/trend` | Score/skin-age history (premium-gated beyond N entries) | user |
| POST | `/v1/scans/{scanId}/share` | Generate a ShareCard | user |
| GET | `/v1/recommendations?scanId=` | Tips/routine for a report | user |
| POST | `/v1/subscriptions/upgrade` | Upgrade entitlement tier | user |
| DELETE | `/v1/users/me` | Trigger UserDataDeletionRequested | user |
| POST | `/v1/partner/scans` *(future)* | B2B: submit + analyze, independent of consumer app | partner API key |

**Example — `POST /v1/scans` response (202):**
```json
{
  "scanId": "scn_01J8XK...",
  "status": "processing",
  "estimatedSeconds": 6
}
```

## Event Contracts
Every domain event from [[02-Domain-Discovery-and-Event-Storming|Event Storming]] gets a stable schema and a standard envelope — this is what makes ADR-001's "in-process now, real queue later" migration painless, because the payload shape doesn't change, only the transport.

**Standard envelope:**
```json
{
  "eventId": "evt_01J8...",
  "eventType": "skinanalysis.scan.submitted.v1",
  "occurredAt": "2026-08-19T10:15:00Z",
  "aggregateId": "scn_01J8XK...",
  "correlationId": "req_9f3a...",
  "version": 1,
  "payload": {}
}
```
- `eventType` follows `{module}.{aggregate}.{event}.v{n}` — the version suffix means a breaking payload change ships as `.v2` alongside `.v1` rather than mutating a contract consumers already depend on.
- `correlationId` ties an event back to the originating HTTP request/trace — required by NFR-9 (observability) and invaluable once the worker is debugged as a separate service.

**Example — `ScanSubmitted` payload:**
```json
{
  "scanId": "scn_01J8XK...",
  "userId": "usr_7b21...",
  "photoRef": "s3://ayna-photos/usr_7b21/scn_01J8XK.jpg",
  "submittedAt": "2026-08-19T10:15:00Z"
}
```

**Example — `SkinReportGenerated` payload:**
```json
{
  "scanId": "scn_01J8XK...",
  "userId": "usr_7b21...",
  "overallScore": 78,
  "skinAge": 24,
  "issues": [
    { "type": "acne", "severity": "moderate", "confidence": 0.86 },
    { "type": "dryness", "severity": "mild", "confidence": 0.71 }
  ],
  "referralSuggested": false
}
```

> [!tip] Keep photo bytes out of the event payload
> Events carry a **reference** (`photoRef`) to object storage, never the image itself — keeps the queue lightweight and keeps photo lifecycle/deletion (NFR-4) governed by one system (object storage TTL policy), not scattered across every consumer that happens to receive the event.

---

> [!success] The contracts now exist as files (2026-08-26)
> This note specified them in prose; they are now written and enforced:
>
> - `api/openapi.yaml` — OpenAPI 3.1, with scan submission and report generation specced in full
> - `events/envelope.schema.json` plus `ScanSubmitted` and `SkinReportGenerated` payloads
> - `internal/contracts` — tests that fail CI when the two disagree, verified against deliberate drift
>
> **Four things the prose version did not capture, added while writing them out:**
>
> 1. **`Severity` has five values, not four.** `unknown` is required so a concern the analysis could not assess is representable. Without it the natural fallback is `none` — telling someone their skin is clear when nothing ever looked at it.
> 2. **`skinAge` and `heatmapUrl` are nullable.** The free tier genuinely has neither (PD-3). Making them required would force the backend to invent a skin age, and the obvious invention is the user's own age — which renders as "right in step" and is a lie.
> 3. **`POST /v1/scans` requires an `Idempotency-Key`.** Every accepted scan costs a real vendor call, so a retry over a flaky connection would otherwise spend twice and consume two of the user's weekly allowance.
> 4. **`GET /v1/scans/{id}` returns `allowanceSpent`.** It is `false` for a rejected photo — the vendor does not bill failed requests, so the user must not be charged one either.
>
> Also settled: errors are RFC 9457 `application/problem+json` throughout, and `Referral.urgency` has exactly one value (`routine`) — a product not equipped to say "urgent" should not have the vocabulary to try.

## Related
[[04-HLD-and-Architectural-Decisions|← HLD & Architectural Decisions]] · [[06-AI-Provider-Evaluation|Next: AI Provider Evaluation →]] · [[00-Phase-0-Roadmap-MOC|Back to Roadmap]]
