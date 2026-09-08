---
tags: [phase-0, product-decisions, adr, resolved]
project: Ayna
created: 2026-08-24
status: accepted — PD-2 values pending spike
---

# Product Decisions
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

> [!info] Why this note exists
> Five questions were left open across [[01-Product-Vision-and-Requirements|PRD]] and [[02-Domain-Discovery-and-Event-Storming|Event Storming]], each flagged as "needs a product decision, not an engineering one." They were scattered, so they stayed unresolved — and every one of them has code hanging off it. This note settles all five and records *why*, so the reasoning survives contact with future-you.

| # | Decision | Status | Blocks |
|---|---|---|---|
| PD-1 | Minimum age: **18+ at launch** | Accepted | FR-1 consent flow, store review |
| PD-2 | Severity thresholds: **versioned policy object; values pending spike** | Mechanism accepted, values open | FR-7 referral |
| PD-3 | Pricing: **store IAP, regional tiers, limited free scans** | Accepted | Billing module |
| PD-4 | Job status: **polling, with push as background fallback** | Accepted | Worker contract, Flutter state layer |
| PD-5 | Streak: **one qualifying week, user-timezone boundaries** | Accepted | SkinProfile aggregate |

---

## PD-1 — Minimum age is 18+ at launch

**Decision:** accounts are restricted to 18 and over. Enforced at registration by a neutral age gate, before any consent is requested and before the camera is ever opened.

**Why:** this product combines three things that each attract scrutiny on their own — face photos, health-adjacent claims, and appearance scoring. Restricting to adults removes an entire category of risk in one move: no parental-consent branch, no child-directed store classification, and no defending the practice of scoring a 15-year-old's face for flaws. The PRD's stated audience starts at 16, so this does cut the youngest slice — and teens are genuinely underserved for acne. That is a deliberate trade, revisitable once the referral copy and disclaimers have been reviewed properly. Widening an age policy later is easy; narrowing it after minors have accounts is not.

> [!tip] Do not inflate the store content rating
> 18+ here is a **terms-of-service restriction you enforce**, not a content rating. The app contains no mature content, so the store age rating should stay low — rating it 18+ unnecessarily buries it in discovery. Enforce the policy with the age gate and the ToS; let the content rating reflect the content.

**Consequences:**
- **Collect birth year, not full date of birth.** Birth year alone satisfies both the age gate *and* FR-5's "skin age vs chronological age" comparison. Full DOB is meaningfully more identifying for zero added benefit — the privacy-minimising choice is also the sufficient one.
- `User` gains `birthYear` and an invariant: **registration is refused if the derived age is under 18**, so an underage user is never created rather than created-then-blocked. Nothing to purge later.
- The existing invariant in [[03-DDD-and-Onion-Architecture|DDD]] — no `Scan` accepted without `ConsentGranted` — stands unchanged. Age is checked earlier, at registration.
- No new domain event. `UserRegistered` carries `birthYear`; a failed age gate produces no event because it produces no user.
- Age-gate screens should not be re-openable to retry a different answer within the same session. A gate you can immediately re-answer is decoration.

---

## PD-2 — Severity thresholds live in a versioned policy object

> [!success] The values now have a documented starting point (2026-08-24)
> AILab publishes a **Degree & Score Reference** binding score bands to severity — on a scale where **higher means better skin**:
>
> | Band | Severity |
> |---|---|
> | 90–100 | None |
> | 70–89 | Mild |
> | 50–69 | Moderate |
> | 30–49 | Severe |
>
> This is a vendor's opinion, not a clinical standard — but a documented opinion is a far better starting point than a number an engineer invents, and it is already calibrated to the model producing the scores. **Adopt these as the v1 `SeverityRuleSet`**, then adjust from the spike's real distributions and have the referral cutoff medically reviewed.
>
> The referral trigger (FR-7) is still a separate decision from the bands: "which severity, on which concerns, triggers a referral" is not answered by a scoring table. That one stays open.
>
> Note also that these bands are **vendor-specific**. Under the tiered-AI decision the free tier runs a different model, so it needs its own calibration — otherwise a user upgrading from Free to Plus sees their score jump for no reason.

> [!warning] The values are still open, and deliberately so
> This is the most sensitive logic in the product: it decides when to tell someone to see a doctor. Picking numbers before [[06-AI-Provider-Evaluation|the spike]] produces real score distributions would be inventing them. **The mechanism is decided now; the values are decided from data.**

**Decision:** referral thresholds are an explicit, versioned, immutable `SeverityRuleSet` — data, not an `if` statement, and never a constant inlined in report generation.

**Shape:**
- Per `IssueType`: the score boundaries mapping to `None | Mild | Moderate | Severe`.
- A referral predicate over the resulting severities.
- A `version` and an `effectiveFrom`.

**Ownership:** Recommendations & Content owns the ruleset *content* (it is advice policy). Skin Analysis *evaluates* against a snapshot of it during report generation, because that is where `HighSeverityIssueDetected` is produced. Consuming a snapshot rather than calling across contexts keeps the dependency direction in [[03-DDD-and-Onion-Architecture|DDD]] intact.

**Consequences:**
- **Stamp every `SkinReport` with the ruleset version that produced it.** Without this you cannot answer "why did it tell me to see a dermatologist in March but not in June with the same skin?" — and under FR-7 and NFR-6 that is precisely the question you must be able to answer. This is an audit requirement, not a nicety.
- Changing a threshold must not require a deploy, but **must** write to the compliance audit trail — same trail `DermatologistReferralSuggested` already writes to (event #8).
- Old reports are never retroactively re-evaluated. A report is a point-in-time fact, consistent with the `Scan` aggregate rules.

> [!warning] The tension to resolve with the values, not the mechanism
> Under-referring risks missing something that mattered. Over-referring makes the app alarmist, erodes trust, and cuts against NFR-6's "not a diagnosis" posture — an app that tells everyone to see a doctor has told no one anything. Do not resolve this by intuition once the spike data lands: pick the values, then have someone medically qualified review them before public launch.

---

## PD-3 — Pricing: store billing, regional tiers, limited free scans

**Decision:** monetise in both markets with store-native regional pricing. Free tier gets **one scan per week**; premium unlocks unlimited scans, full trend history, and detailed per-issue breakdowns.

**Free scan limit rationale:** each scan costs real money in inference (NFR-7), and NFR-3 anticipates a 50–100x spike. Unlimited free scanning means unbounded variable cost precisely when you go viral — the bill arrives with the growth. A weekly cap is also *honest*: skin does not visibly change day to day, so weekly is the natural cadence rather than an artificial squeeze, and it pairs cleanly with the weekly streak in PD-5.

> [!success] Resolved — three tiers, tiered AI (decided 2026-08-24)
> The cost problem below is settled by **matching the model to the tier**:
>
> | Tier | AI model | Gets |
> |---|---|---|
> | **Free** | cheap general model | Scores + severity. No heatmap. |
> | **Plus** | Specialist API | Heatmap, skin age, full per-issue breakdown, full trend history |
> | **Pro** | Specialist API | Everything in Plus, with a higher scan allowance |
>
> This makes free-tier cost near-zero, gives Plus a *visible* upgrade reason (the heatmap), and gives Pro a reason that is purely usage — the cleanest possible ladder, since each step sells something the user can actually see.
>
> **Consequences for the build:**
> - `AIAnalysisProvider` gets **two implementations**, selected at call time by the caller's entitlement. This is ADR-002's port design doing exactly what it was written for — no domain change, no new module.
> - Entitlement is now an **input to scan processing**, not just a read-gate on history. `SubmitScanUseCase` must resolve the tier before dispatching, and the queued job must carry which model to use.
> - The report contract must tolerate a **missing heatmap** (free tier) without the UI breaking. Design the report screen for the no-heatmap case first — it is the majority case.
> - Two model paths means two score distributions. PD-2's severity thresholds may need **calibrating per model**, or scores will not be comparable when a user upgrades. Flag: a free user upgrading to Plus should not see their score jump for no reason.
> - Still worth doing: **ask both vendors for volume rates.** It improves Plus/Pro margin even though it no longer blocks the model.

> [!success] Confirmed against real vendor pricing (2026-08-24)
> Vendor billing pages give Skin Analyze Basic at **$0.0405**/call, Advanced **$0.135**, Pro **$0.189** (mid volume) — not the ~$0.90 an earlier draft assumed. With the tiering above, every scenario in [[08-Unit-Economics|Unit Economics]] is profitable; **without** it, every scenario loses money even on the cheapest endpoint. The tiering is load-bearing, not an optimisation.
>
> Two amendments fall out:
> - **Free stays at 1 scan/week.** At a fraction of a cent per free scan, weekly is affordable. The earlier proposal to cut it to monthly is withdrawn.
> - **Plus is capped at 2 scans/week, not 3.** At Pro pricing a Pakistani Plus user break-evens at 11.3 scans/month; 3/week is ~13, so heavy users in the cheapest market lose money — and lose more the more they engage. Pro's "unlimited" needs a fair-use ceiling for the same reason.

> [!warning] Superseded context — why the tiering above was needed
> [[08-Unit-Economics|Unit Economics]] priced this out after PD-3 was written. At AILab's list rate for Skin Analyze Pro (~$0.90/call), **one free user costs ~$3.90/month while a Pakistani subscriber nets ~$2.13** — a free user costs more than a paying one earns, and every modelled scenario loses money.
>
> PD-3's *structure* stands (store IAP, regional tiers, a limited free tier). The *allowance* does not, unless one of these lands first:
> 1. **Negotiated volume pricing** — try this before anything else; it is one email and it fixes the model outright.
> 2. **Tier the AI**: cheap general model for free users, Skin Analyze Pro for premium. Makes the free tier nearly costless and turns FR-4's heatmap into the visible reason to upgrade. Recommended, and free architecturally — ADR-002 already isolates the vendor behind a port.
> 3. **Cut the free allowance to 1 scan/month** — fallback, and it weakens PD-5's weekly streak.
>
> Do not start the Billing module until this is settled: it determines what the free tier is actually allowed to do.

> [!warning] This contradicts what is currently written in the vault
> [[03-DDD-and-Onion-Architecture|DDD & Onion Architecture]] says Billing "will integrate a payment processor (Stripe / local: JazzCash, Easypaisa)." **For a subscription that unlocks in-app features, that is not available to you.** Apple and Google both require their own in-app purchase systems for digital goods, taking 15–30%. You cannot take a JazzCash payment inside the app to unlock premium.
>
> Google Play does operate user-choice / alternative billing in a number of territories, and the rules keep moving — **verify current terms for your specific launch markets before designing around them.** But the safe default, and what to build first, is store IAP.

**What this actually changes — mostly for the better:**

| Concern | Implication |
|---|---|
| Regional pricing | **Solved for you.** Both stores support per-territory price tiers natively. No multi-currency logic to build — the reason "both markets" is far cheaper than it sounded. |
| Payment rails | StoreKit 2 (iOS) + Google Play Billing (Android). Stripe/JazzCash come back only if a web or B2B billing surface appears later (FR-13). |
| Billing module shape | You do not charge anyone. `Subscription` state is **derived from store-validated receipts** via App Store Server Notifications v2 and Google Play Real-time Developer Notifications. Your job is receipt validation and entitlement derivation. |
| Existing invariant | *"Entitlement is always derivable from subscription state, never stored redundantly"* — holds, and matters more now, since the store is the source of truth and your copy can drift. |

**Gap this exposes in [[02-Domain-Discovery-and-Event-Storming|Event Storming]]:** the event list has only `SubscriptionUpgraded`. Real store subscriptions have a lifecycle you do not control and must react to:

| Event | Trigger |
|---|---|
| `SubscriptionRenewed` | successful auto-renew |
| `SubscriptionExpired` | lapse after grace |
| `SubscriptionRefunded` | store-issued refund — revoke entitlement |
| `SubscriptionInGracePeriod` | payment failed, retrying — **keep access** |
| `SubscriptionCancelled` | auto-renew off; access continues to period end |

Grace period is the one most often missed: revoking access the moment a card fails punishes users for their bank's behaviour, and the store expects you to keep serving them while it retries.

**Indicative price points** — anchors to validate, not decisions:

| Market | Monthly | Annual |
|---|---|---|
| Pakistan | ~PKR 500–800 | ~PKR 4,000–6,000 |
| India | ~INR 199–299 | ~INR 1,500–2,200 |
| US / UK / EU | ~$4.99 | ~$29.99 |

Annual at roughly 50–60% of 12× monthly is the conventional shape. Purchasing-power parity matters here: US pricing in Pakistan converts to no subscribers at all.

---

## PD-4 — Job status by polling, with push as the background fallback

**Decision:** the client polls `GET /v1/scans/{scanId}`. No WebSocket. A push notification covers the case where the report completes while the app is backgrounded.

**Why not WebSockets:** the job takes about eight seconds. A persistent bidirectional connection to wait out eight seconds is real complexity bought for nothing. More decisively, NFR-3 requires a **stateless API** that absorbs a 50–100x spike — and WebSockets are stateful. Under a viral spike, held connections become the scaling wall, and you would need sticky routing or a pub/sub fan-out layer to survive it. Polling scales on the same stateless path as every other read, and `GET /v1/scans/{scanId}` already exists in [[05-Project-Structure-and-Contracts|Contracts]].

**Cadence:** poll every 1s for the first 10s, back off to 2s, stop at 60s and show a "taking longer than usual" state. The 202 response already carries `estimatedSeconds` — drive the client's first poll delay from it rather than hardcoding.

**Cost:** roughly 8–12 extra GETs per scan, served from Redis job status well inside NFR-1's <300ms p95. Negligible against the inference cost of the scan itself.

> [!tip] The backgrounding case is the one polling actually misses
> If the user leaves the app during those eight seconds, polling stops and the report silently completes with nobody watching. Send an FCM notification on `SkinReportGenerated` **only when the client is not actively polling** — reusing the Notifications module already in scope. This is the genuine hole in a polling design, and it costs one conditional rather than an architecture.

**Revisit when:** report generation grows past ~30s, a feature needs true server push (live capture guidance), or B2B batch delivery (FR-13) makes callbacks the better contract.

---

## PD-5 — Streak counts one qualifying week, in the user's own timezone

**Decision:** a streak increments once per **ISO week** containing at least one *completed* scan. Additional scans in the same week do not add to it. A missed week resets the streak to zero.

This resolves the hot spot as the notes recommended — a second scan on the same day does not count — and makes the period explicit, since the ubiquitous language already defines Streak in weeks rather than days. Daily streaks would also fight PD-3's one-free-scan-per-week limit, which would have been an unpleasant contradiction to discover after building both.

> [!warning] The real decision hiding inside this one is the timezone
> "One per week" is meaningless until you say *whose* week. UTC boundaries mean a user in Karachi scanning at 2am local is credited to the previous week, which is invisible to them and feels broken. Device timezone is trivially gameable by changing a phone setting.
>
> **Decision: store the user's timezone at registration and compute all period boundaries in it.** Allow updating it — people move — but never retroactively recompute past streaks, because a timezone change would silently rewrite history that the user already saw.

**Consequences:**
- Only `Completed` scans count. A `ScanRejected` quality failure must not credit a week — otherwise the streak rewards uploading blurry photos.
- `User` (or `SkinProfile`) gains `timezone`; the SkinProfile invariant becomes: *at most one streak entry per ISO week in the user's timezone, regardless of scan count.*
- No streak freezes or repair mechanics in MVP. They are a retention feature with real complexity, and a simple honest rule is the right starting point. Revisit if retention data justifies it.

---

## Still open after this note

- **PD-2 values** — blocked on [[06-AI-Provider-Evaluation|the provider spike]], then medical review.
- **Price validation** — the table above is anchors, not research. Worth checking against what comparable apps actually charge in each market.
- **Google Play alternative billing** — verify current territory rules for your launch markets before assuming IAP-only.

## Related
[[06-AI-Provider-Evaluation|← AI Provider Evaluation]] · [[08-Unit-Economics|Next: Unit Economics →]] · [[01-Product-Vision-and-Requirements|PRD]] · [[02-Domain-Discovery-and-Event-Storming|Event Storming]] · [[00-Phase-0-Roadmap-MOC|Back to Roadmap]]
