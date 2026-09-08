---
tags: [phase-0, roadmap, moc, ayna]
project: Ayna
created: 2026-08-19
status: draft
---

# Phase 0 Roadmap — Ayna (AI Skin Analysis App)

> [!info] What this vault section is
> This is the Phase 0 planning artifact for the backend — completed *before* any code is written. Phase 0's job is to remove ambiguity: what are we building, for whom, what does "done" mean, and how is the system shaped so it doesn't collapse under its own weight at month six.

**Name:** Ayna — آئینہ, "mirror" in Urdu and Hindi, from Persian *āyna*. **Locked 2026-08-26** (ADR-005). The metaphor is the product: a mirror shows you yourself without judging.

## How to use these notes
Read in order the first time. After that, treat each note as a living reference — update the event list, ADRs, and contracts as decisions change. ADRs in particular are meant to be *appended to* when a decision is revisited, not silently rewritten.

## Sections
1. [[01-Product-Vision-and-Requirements|Product Vision & Requirements (PRD)]]
2. [[02-Domain-Discovery-and-Event-Storming|Domain Discovery & Event Storming]]
3. [[03-DDD-and-Onion-Architecture|Domain-Driven Design & Onion Architecture]]
4. [[04-HLD-and-Architectural-Decisions|High-Level Design & Architectural Decisions]]
5. [[05-Project-Structure-and-Contracts|Project Structure & Contracts]]
6. [[06-AI-Provider-Evaluation|AI Provider Evaluation]]
7. [[07-Product-Decisions|Product Decisions]]
8. [[08-Unit-Economics|Unit Economics]]
9. [[09-Infrastructure-and-Services|Infrastructure & Services]]
10. [[10-Brand-Name|Brand Name]]

> [!warning] Sections 1–5 specify a system on top of an untested assumption
> Section 6 was added after review: the whole design assumes a third-party vision API returns six per-issue severities, confidences, heatmaps, and a skin age, calibrated for South Asian skin tones. That assumption was never tested, and the domain model, event contract, report screen, and unit economics all inherit it. **Run the spike in section 6 before writing `skinanalysis` code** — it is the cheapest thing here to falsify and the most expensive to get wrong.

## Phase 0 exit criteria
Phase 0 is "done" — and you're clear to start Phase 1 (implementation) — when:
- [x] **AI provider spike run and ADR-003 accepted** — AILab *Skin Analyze Pro* selected, Basic rejected on severity resolution ([[06-AI-Provider-Evaluation|section 6]]). **Accepted conditionally:** two risks stay open — R-1, Pro appears to under-report acne (one 70-credit call to settle, and it gates the report screen); R-2, the tone gate was unmeasurable at n=4 and moves to production monitoring.
- [ ] PRD reviewed, FRs/NFRs have no open questions — *provider now closed by ADR-003. **PD-2's severity values remain**: the vendor publishes its own bands ([90,100] None, [70,89] Mild, [50,69] Moderate, [30,49] Severe) and the code uses them, but adopting a vendor's cut-points as Ayna's own health-adjacent claim is a decision, not a default — it still wants review*
- [x] Event storming board has no unresolved "hot spot" stickies — all three resolved in [[07-Product-Decisions|PD-2, PD-4, PD-5]]
- [x] Bounded contexts agreed, each with a named owner (just-you-for-now)
- [x] ADR-001, ADR-002, **ADR-003** (provider, conditional), ADR-004 (services) and ADR-005 (name) accepted
- [x] Folder skeleton committed with module boundaries enforced — `D:\ayna-backend`, enforced by `internal/arch` in CI, verified against deliberate violations
- [x] API + event contracts drafted for the two riskiest flows — `api/openapi.yaml` and three JSON Schemas in `events/`, kept in step by `internal/contracts` in CI and verified against deliberate drift

### Product decisions — resolved in [[07-Product-Decisions|section 7]]
- [x] **PD-1** Minimum age — 18+ at launch, enforced by age gate before consent
- [ ] **PD-2** Severity thresholds — mechanism decided (versioned policy object); **values blocked on [[06-AI-Provider-Evaluation|the spike]]**, then medical review
- [x] **PD-3** Pricing/tiers — store IAP with regional tiers, one free scan per week
- [x] **PD-4** Job status UX — polling, with push notification as the background fallback
- [x] **PD-5** Streak rule — one qualifying ISO week, boundaries in the user's own timezone

## Related
#skincare-app #backend-architecture #golang #flutter #phase-0
