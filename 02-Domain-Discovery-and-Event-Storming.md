---
tags: [phase-0, event-storming, domain-discovery]
---

# Domain Discovery & Event Storming
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

## Business Vocabulary (Ubiquitous Language)
This is the vocabulary every conversation — product, engineering, even marketing — should use consistently. If code uses different words than the ones below, that's a bug in the model, not just naming.

| Term | Meaning |
|---|---|
| **User** | A person with an account in the consumer app |
| **Scan** | One instance of "photo submitted → analyzed"; the unit of work |
| **AnalysisJob** | The async processing task tied 1:1 to a Scan |
| **SkinReport** | The structured output of a completed Scan: score, issues, severity, tips |
| **Issue / Concern** | A single detected finding (e.g., Acne, Redness, DarkSpot) with a Severity |
| **Severity** | Enum: None → Mild → Moderate → Severe |
| **SkinProfile** | The user's evolving picture across all their Scans over time (trend owner) |
| **SkinAge** | A derived, single-number estimate distinct from chronological age |
| **Tip / Recommendation** | An advice item attached to a report or a specific Issue |
| **Routine** | A bundled AM/PM set of recommended steps |
| **Referral** | A system-generated suggestion to see a dermatologist, triggered by severity rules |
| **Streak** | Consecutive qualifying periods (e.g., weeks) with at least one Scan |
| **ShareCard** | An exportable image asset summarizing a report, built for social sharing |
| **ConsentRecord** | Explicit, timestamped user consent to photo capture/processing |
| **Entitlement** | What a user's current subscription tier unlocks (e.g., full trend history) |
| **Partner** *(future)* | A B2B account (brand/clinic) consuming the analysis engine via API |

> [!tip] Why separate SkinProfile from Scan?
> A Scan is a point-in-time fact — it shouldn't change once analyzed. SkinProfile is a rolling aggregate (trend, streak, skin-age curve) that's updated *because of* scans but has its own lifecycle and its own consistency rules (e.g., "one scan per day counts toward the streak, not one scan per upload"). Modeling them as the same thing is a common early mistake that lets trend logic leak into scan-processing code.

## Event Storming — Domain Events, Commands & Policies
Read left to right: someone/something issues a **Command** (an intent) → the system produces a **Domain Event** (a fact, past tense) → a **Policy** reacts to that fact.

| # | Command | Domain Event | Policy ("Whenever X, then...") |
|---|---|---|---|
| 1 | RegisterUser | **UserRegistered** | Send verification OTP; create an empty SkinProfile shell |
| 2 | GrantConsent | **ConsentGranted** | Unlock the "submit scan" capability for this user |
| 3 | SubmitScan | **ScanSubmitted** | Enqueue an AnalysisJob; run automated image-quality check |
| 4 | *(system)* CheckImageQuality | **ScanRejected** *(quality fail)* | Notify client to retake the photo; do not consume AI-inference budget |
| 5 | *(worker)* StartAnalysis | **AnalysisStarted** | Update job status for client polling/streaming |
| 6 | *(worker)* CompleteAnalysis | **SkinReportGenerated** | Update SkinProfile trend data; evaluate severity thresholds |
| 7 | *(derived)* EvaluateSeverity | **HighSeverityIssueDetected** | Trigger DermatologistReferralSuggested + push notification |
| 8 | *(derived)* SuggestReferral | **DermatologistReferralSuggested** | Log to compliance audit trail; render disclaimer in report |
| 9 | GenerateTips | **TipsGenerated** | Attach routine + tips to the report for display |
| 10 | ShareReport | **ShareCardGenerated** | Track attribution for viral-growth analytics |
| 11 | *(scheduled)* EvaluateReminder | **ReminderDue** | Send push notification prompting a new scan |
| 12 | *(system, on ScanSubmitted)* UpdateStreak | **StreakUpdated** | If milestone (7/30/90 days) reached → award badge |
| 13 | UpgradeSubscription | **SubscriptionUpgraded** | Unlock premium entitlements immediately |
| 14 | RequestDataDeletion | **UserDataDeletionRequested** | Purge photos + PII within SLA; emit DataPurged for audit |
| 15 *(future)* | RegisterPartner | **PartnerOnboarded** | Issue API key; assign rate-limit tier |

### Hot spots — resolved in [[07-Product-Decisions|Product Decisions]]
> [!success] All three settled
> - **Sync vs. async report generation** → **PD-4**: polling on `GET /v1/scans/{scanId}`, no WebSocket — NFR-3's stateless-API requirement makes held connections the scaling wall during a viral spike. An FCM push covers completion while the app is backgrounded, which is the one case polling genuinely misses.
> - **Where does severity-threshold logic live?** → **PD-2**: its own versioned, auditable `SeverityRuleSet`, owned by Recommendations and evaluated as a snapshot inside Skin Analysis. Every report is stamped with the ruleset version that produced it, so a past referral stays explicable after the values change.
> - **Streak double-counting** → **PD-5**: one qualifying **ISO week** (not day), counting `Completed` scans only, with period boundaries computed in the **user's own timezone** — the decision actually hiding inside this hot spot.

> [!warning] Missing events, found while resolving PD-3
> The list above has only `SubscriptionUpgraded`. Store-managed subscriptions have a lifecycle you don't control and must react to: `SubscriptionRenewed`, `SubscriptionExpired`, `SubscriptionRefunded`, `SubscriptionInGracePeriod`, `SubscriptionCancelled`. Grace period is the one most often missed — revoking access the instant a card fails punishes users for their bank's behaviour, and the stores expect you to keep serving them while they retry.

## Related
[[01-Product-Vision-and-Requirements|← Product Vision & Requirements]] · [[03-DDD-and-Onion-Architecture|Next: DDD & Onion Architecture →]]
