---
tags: [phase-0, unit-economics, cost, nfr-7]
project: Ayna
created: 2026-08-24
updated: 2026-08-24
status: accepted — prices confirmed from vendor billing pages
---

# Unit Economics
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

> [!success] Headline: the model works, because of the tiering
> With confirmed vendor pricing and the tiered-AI decision in [[07-Product-Decisions|PD-3]], every scenario tested is **profitable**. Without the tiering, every scenario **loses money** — even at the corrected, much lower prices. The tiering is not an optimisation; it is the thing that makes the business viable.

> [!warning] Correction to the previous version of this note
> An earlier draft of this note put Skin Analyze Pro at ~$0.90/call. That was wrong by **4.8×** — it came from a partially-rendered pricing page and an inferred credit rate of ~$0.06 when the real rate is ~$0.0027. The figures below are read directly from AILab's per-endpoint billing tables. Conclusions that rested on the old number (that the model was unsalvageable at list price) do not hold.

## Confirmed vendor pricing

AILab bills in credits, with volume tiers by prepayment size:

| Endpoint | Credits/call | Entry ($6–30) | Mid ($300–1,500) | Top ($2,500) |
|---|---|---|---|---|
| Skin Analyze **Basic** | 15 | $0.0450 | **$0.0405** | $0.0375 |
| Skin Analyze **Advanced** | 50 | $0.1500 | **$0.1350** | $0.1250 |
| Skin Analyze **Pro** | 70 | $0.2100 | **$0.1890** | $0.1750 |

Failed requests are not billed — which matters more than it looks: a `ScanRejected` quality failure costs nothing, so the retake flow is free to be generous.

Volume discounting is shallow (only ~17% from entry to top tier), so **do not build a plan that depends on hitting a volume cliff**. Mid-tier is the honest planning number.

## The model, per 1,000 monthly active users

Free users take their full allowance (1/week = 4.33/mo); paid users average 8 scans/mo; 15% store cut applied.

**With tiered AI (PD-3): free on a cheap general model (~$0.002), paid on Pro ($0.189)**

| Market | Conversion | Cost | Revenue | **Net** |
|---|---|---|---|---|
| Pakistan | 2% | $38.73 | $42.60 | **+$3.87** |
| Mixed | 2% | $38.73 | $60.00 | **+$21.27** |
| US / UK / EU | 3% | $53.76 | $127.20 | **+$73.44** |

**Without tiering — free users also on an AILab endpoint (Basic, $0.0405)**

| Market | Conversion | Cost | Revenue | **Net** |
|---|---|---|---|---|
| Pakistan | 2% | $202.10 | $42.60 | **−$159.50** |
| US / UK / EU | 3% | $215.46 | $127.20 | **−$88.26** |

That second table is the whole argument. Even AILab's *cheapest* endpoint, at four cents a call, makes the free tier unaffordable — 4,243 free scans a month against twenty subscribers. The free tier has to run on something that costs a fraction of a cent, or it eats the business.

## The number that constrains the product: break-even scans

At Pro pricing, how many scans a paying user can take before they cost more than they pay:

| Market | Net revenue/mo | Break-even scans/mo |
|---|---|---|
| Pakistan | $2.13 | **11.3** |
| India | $2.47 | 13.1 |
| US / UK / EU | $4.24 | 22.4 |

> [!warning] This invalidates the Plus allowance in the current paywall design
> The paywall screen offers Plus **3 scans a week** — about 13 a month. In Pakistan that costs **$2.46** against **$2.13** of net revenue: **every heavy Plus user in the cheapest market loses money**, and they lose more the more they engage. A plan that punishes engagement is the wrong plan.
>
> **Fix: Plus is capped at 2 scans a week (~8.7/mo, ~$1.64).** That clears break-even in every market including Pakistan, and it still doubles the free allowance, which is the thing being sold. Pro's "unlimited" needs a fair-use ceiling for the same reason — unlimited at $0.189 a call is an open tab.
>
> The alternative is region-varying caps, which is more honest economically and considerably worse as a product: nobody wants to explain why Plus means something different in Lahore than in London. Take the single safe cap.

## What changed in the decisions

- **PD-3's tiered AI: confirmed and load-bearing.** Not merely a cost optimisation — without it, no scenario is profitable.
- **PD-3's free allowance (1 scan/week): survives.** At ~$0.002 a free scan, weekly costs about a cent per user per month. The earlier draft's proposal to cut it to monthly is withdrawn — it was solving a problem created by a wrong price.
- **Plus cap: reduced from 3/week to 2/week.** New, from the break-even math above.
- **FR-4's heatmap and FR-5's skin age cost $0.148/call over Basic** (the Pro–Basic gap). That is what the Plus tier is selling, and at these prices it is comfortably worth selling.

## Still to verify

- **Which endpoint actually covers FR-4's six concerns.** If Basic ($0.0405) covers them, the paid tier gets 4.7× cheaper and margins improve dramatically. The [[06-AI-Provider-Evaluation|spike]] answers this — run Basic and Pro as separate arms.
- **Real conversion rate.** 2–5% is a generic freemium band, not a measurement of this app.
- **Real scan frequency.** 8/mo for paid users is an assumption; the break-even table is only as good as it.
- **The cheap general model's true cost** at your image size — assumed ~$0.002, worth measuring in the spike rather than trusting.
- **Enterprise pricing** is still worth an email, but it is now an improvement rather than a rescue.

## Related
[[07-Product-Decisions|← Product Decisions]] · [[06-AI-Provider-Evaluation|AI Provider Evaluation]] · [[00-Phase-0-Roadmap-MOC|Back to Roadmap]]
