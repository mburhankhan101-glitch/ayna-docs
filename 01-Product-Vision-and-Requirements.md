---
tags: [phase-0, prd, requirements]
---

# Product Vision & Requirements (PRD)
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

## Product Vision
Ayna gives anyone a free, instant, judgment-free read on their skin from a single selfie — a score, a plain-language breakdown of what's actually going on (acne, redness, dryness, dark spots...), and a next step, whether that's a two-minute routine tweak or "please see a dermatologist."

We are not building a diagnostic medical device. We are building a **consumer confidence and awareness tool** that happens to use computer vision — closer to a fitness tracker than a clinical instrument. The business starts as a viral B2C mobile app (localized for South Asian skin tones and climate, an underserved segment of a market currently dominated by apps tuned for Western/East Asian users) and has a credible path to a second revenue line: licensing the analysis engine to skincare brands and clinics (B2B) — which is how the market leaders in this space actually make most of their money.

**Problem it solves:**
- Dermatologist access is slow, costly, or intimidating for a "is this normal?" check.
- Existing AI skin-analysis apps are tuned for the wrong skin tones/climates for our target market, or are buried inside e-commerce/enterprise tools rather than being a fun, standalone experience.
- Skincare brands want personalization tooling but don't want to build computer vision in-house.

## Stakeholders
| Stakeholder | Interest / Dependency |
|---|---|
| End user (primary: women, 16–35, skincare-conscious) | Wants a fast, accurate-feeling, non-judgmental read on their skin, and a reason to come back |
| Product owner / founder (you) | Needs the system to be extendable without a rewrite as features and revenue lines are added |
| Future dermatologist partners | Depend on the referral flow being medically responsible, not alarmist or misleading |
| Future B2B customers (brands, clinics) | Will depend on the analysis engine being usable as a service (API), not hardwired into the consumer app |
| App store platforms (Apple/Google) | Enforce policy on health-adjacent apps, biometric-like data, and minors' data — non-negotiable constraint |
| Regulators (data protection; health-claim advertising) | Indirect stakeholder — constrains what the report is allowed to claim and how photos are handled |

> [!warning] Constraint, not a feature
> Because this touches faces, health-adjacent claims, and likely minors as users, App Store health-app policy and data-privacy expectations are **constraints on the requirements below**, not just legal fine print. They resurface in NFR-4 through NFR-6.

## Functional Requirements
Numbered for traceability into later sections (events, endpoints).

| ID | Requirement |
|---|---|
| FR-1 | An unauthenticated user shall be able to register via email or phone OTP and grant explicit consent for photo processing before any scan is accepted. |
| FR-2 | An authenticated user shall be able to capture a guided front-camera selfie, with on-device face-alignment/lighting feedback before submission. |
| FR-3 | An authenticated user shall be able to submit a captured photo for AI skin analysis and receive an asynchronous job status until the report is ready. |
| FR-4 | An authenticated user shall be able to view an interactive skin report: an overall score (0–100), a per-issue breakdown (acne, redness, dryness, dark spots, texture, pores) with severity, and a tappable visual heatmap over their own photo. |
| FR-5 | An authenticated user shall be able to view a "skin age" estimate alongside their chronological age. |
| FR-6 | An authenticated user shall be able to view personalized tips and a suggested AM/PM routine generated from their latest report. |
| FR-7 | The system shall prompt the user to consult a dermatologist whenever a detected issue crosses a defined severity threshold, alongside a clear "AI is not a diagnosis" disclaimer. |
| FR-8 | An authenticated user shall be able to view a trend of their scores and skin age over time (history), gated behind free/premium entitlement rules. |
| FR-9 | An authenticated user shall be able to export a shareable result card for social media. |
| FR-10 | An authenticated user shall be able to opt in to scheduled re-scan reminders. |
| FR-11 | An authenticated user shall be able to request full deletion of their account, photos, and derived data. |
| FR-12 | An authenticated user shall be able to upgrade to a premium subscription to unlock full trend history and detailed per-issue breakdowns. |
| FR-13 *(future)* | A partner (brand/clinic) system shall be able to submit a photo for analysis via an authenticated API and receive a structured report, independent of the consumer mobile app. |

## Non-Functional Requirements
| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Latency | Report status/history reads (GET endpoints) shall return in <300ms p95. Scan submission (POST) shall acknowledge in <500ms p95 — the AI analysis itself is async, see [[02-Domain-Discovery-and-Event-Storming\|Event Storming]]. |
| NFR-2 | Latency (perceived) | End-to-end time from photo submission to a ready report shall not exceed ~8s p95, with a progressive "analyzing..." UI state — a product-feel requirement, not just a backend one. |
| NFR-3 | Scalability | The system shall absorb a 50–100x traffic spike within hours without manual intervention (viral growth is the explicit growth strategy). Stateless API + queue-based load leveling for the AI worker is required, not optional. |
| NFR-4 | Privacy | Facial photos are sensitive personal data. They shall be encrypted at rest and in transit, access-logged, and auto-deletable on a user-configurable schedule — treat this as a trust feature, not just future-proofing for regulation. |
| NFR-5 | Availability | 99.9% uptime for the consumer-facing API (~43 min/month budget). This is not a clinical system; 99.99% is not worth the engineering cost at this stage. |
| NFR-6 | Compliance posture | Every report screen shall visibly disclose that results are not a medical diagnosis; referral copy shall be reviewed against local health-claim advertising norms before public launch. |
| NFR-7 | Cost control | Cost per completed analysis job (AI inference + storage) shall be tracked per-request and alertable — this is the dominant variable cost and the main threat to unit economics at viral scale. |
| NFR-8 | Modularity | The system shall be decomposable into independently deployable services along the bounded-context lines in [[03-DDD-and-Onion-Architecture\|DDD & Onion Architecture]] without a domain-model rewrite. |
| NFR-9 | Observability | Structured logs, request tracing, and business-event metrics shall exist from the first deployed version — cheap now, expensive to retrofit once modules multiply. |

## Assumptions & Open Questions
> [!success] Resolved — see [[07-Product-Decisions|Product Decisions]]
> - **Minimum age policy** → **PD-1**: 18+ at launch, enforced by a neutral age gate before consent. Collect birth year only (sufficient for both the gate and FR-5's chronological-age comparison).
> - **Severity thresholds** (FR-7) → **PD-2**: mechanism decided — a versioned, auditable `SeverityRuleSet`, with every report stamped with the version that produced it. **Values still open**, deliberately: they come from [[06-AI-Provider-Evaluation|the spike's]] real score distributions, then medical review.
> - **Pricing/tiers** (FR-12) → **PD-3**: store IAP with native regional pricing; free tier is one scan per week; premium unlocks unlimited scans, full trend history, and detailed breakdowns.

> [!warning] Still open
> - Platform scope assumed to be **mobile only** (Flutter, iOS/Android) for Phase 0; no web client assumed.
> - The **AI vision provider** is still unselected and untested — see [[06-AI-Provider-Evaluation|section 6]]. FR-4 and FR-5 are written as if it is solved.

> [!note] FR-12 amended by PD-3
> FR-12 as written gates only trend history and detailed breakdowns. PD-3 adds a **scan-frequency limit** to the free tier — unlimited free scanning leaves the dominant variable cost (NFR-7) unbounded at exactly the moment NFR-3's viral spike arrives.

## Related
[[00-Phase-0-Roadmap-MOC|← Back to Roadmap]] · [[02-Domain-Discovery-and-Event-Storming|Next: Domain Discovery & Event Storming →]]
