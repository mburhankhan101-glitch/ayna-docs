---
tags: [phase-0, adr, brand, naming]
project: Ayna
created: 2026-08-26
status: accepted
---

# Brand Name
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

## ADR-005: The name is Ayna

**Status:** Accepted &middot; 2026-08-26

**Context:** "Ayna" was introduced in the first note as a working title, explicitly marked "swap for your real brand name". It has since been baked into a Go module path, a Flutter project, an Android package id, and every one of these notes. The cost of renaming rises with each of those, and one of them becomes irreversible on first publish — so the placeholder had to become a decision or become a liability.

**Decision:** the product is **Ayna** — آئینہ / आईना, *mirror* in Urdu and Hindi, from Persian *āyna*.

**Why:**

- **The metaphor is the product.** A mirror shows you yourself without judging. That is the entire positioning in [[01-Product-Vision-and-Requirements|the PRD]] — "judgment-free", "not a diagnosis", "closer to a fitness tracker than a clinical instrument" — compressed into four letters.
- **It lands instantly in the primary market** and needs no explanation in English. Two syllables, unambiguous spelling, no unfortunate meanings in either language.
- **It is already coherent with the design.** The chosen direction is called Warm Mirror; the splash line is "a kinder look at your skin". The name, the palette and the copy are one idea rather than three.

**Alternatives considered:** Aks (عکس, *reflection*) — more ownable, shorter, but less warm. Nikhaar (نکھار, *the blooming of complexion*) — the most precisely on-point, and the hardest to say for an English speaker. Roshni, Sehar, Darpan — all good, none better than the mirror metaphor. Aaina was rejected outright: it is the same word respelt, and trademark law tests *confusing similarity*, not exact match, so it buys nothing.

> [!warning] Known conflicts, accepted with eyes open
> **[Ayna Beauty](https://aynabeauty.com/) is a live makeup-and-skincare brand**, and the Play Store already carries three apps called Ayna, one of them "Ayna AI".
>
> Accepted, because the risk was mis-framed the first time it was raised. Trademark caution is calibrated for a funded launch with paid acquisition and brand equity to defend; this is a portfolio project. Three points make it a low-stakes call:
>
> - *Ayna* is a **dictionary word**. Common words make weak marks — which cuts both ways: hard to own, equally hard for anyone to stop you using.
> - Trademark protection is **class-based**. Cosmetics (class 3) and software (class 9) are different classes, different markets, different countries.
> - Three apps already coexist under the name on Play, so the store does not treat it as blocking.
>
> **The real cost is discoverability, not litigation** — a store search for "Ayna" returns four apps and yours is the newest with no reviews. For a portfolio piece that barely matters: reviewers open a direct link. **Revisit if this ever becomes a business with paid acquisition.**

## Identifiers

| Where | Value | Reversible? |
|---|---|---|
| Android `applicationId` | `com.ayna.ayna_app` | **No — permanent once published to Play** |
| Android display label | `Ayna` | Yes, freely |
| Flutter project | `ayna_app` | Yes, with a rename pass |
| Go module | `github.com/ayna/ayna-backend` | Yes |
| iOS bundle id | `com.ayna.aynaApp` (scaffold default) | **Yes — not locked until an App Store submission** |

> [!tip] Why the two platforms disagree, and why it does not matter yet
> **iOS bundle ids cannot contain underscores** — Apple permits only letters, digits, hyphens and periods — so Flutter silently camel-cased the iOS value when scaffolding. The mismatch is real but inert: **launch is Android-only**, and an iOS bundle id is free to change right up until the first App Store submission. Settle it then, if there is a then.

> [!warning] The one line that cannot be taken back
> `applicationId` is permanent after the first Play Store release. Changing it later is not a rename — it is a **new listing**, with zero downloads, zero reviews, and no upgrade path for anyone who installed the old one.
>
> The display name is separate and stays free to change. So a future rebrand costs one string, not a new app — provided nothing else starts depending on the id.

## Related
[[09-Infrastructure-and-Services|← Infrastructure & Services]] · [[01-Product-Vision-and-Requirements|PRD]] · [[00-Phase-0-Roadmap-MOC|Back to Roadmap]]
