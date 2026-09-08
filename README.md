# ayna-docs

The reasoning behind **Ayna** (آئینہ, "mirror" in Urdu) — an AI skin analysis
app. Product vision, domain modelling, architecture decisions and the record of
what was decided, what was rejected, and what is still open.

Most of this was written *before* the code, which is the point. Phase 0's job
was to remove ambiguity: what is being built, for whom, what "done" means, and
how the system is shaped so it does not collapse at month six.

> Part of a four-repo project:
> **[ayna-docs](https://github.com/mburhankhan101-glitch/ayna-docs)** (you are here) ·
> [ayna-backend](https://github.com/mburhankhan101-glitch/ayna-backend) ·
> [ayna-app](https://github.com/mburhankhan101-glitch/ayna-app) ·
> [ayna-spike](https://github.com/mburhankhan101-glitch/ayna-spike)

Portfolio project, not a business.

## Read in this order

| | Note | What it settles |
|---|---|---|
| 00 | [Phase 0 Roadmap](00-Phase-0-Roadmap-MOC.md) | Map of contents; start here |
| 01 | [Product Vision & Requirements](01-Product-Vision-and-Requirements.md) | FR-1..FR-11, NFR-1..NFR-9, the users |
| 02 | [Domain Discovery & Event Storming](02-Domain-Discovery-and-Event-Storming.md) | Events, commands, aggregates, bounded contexts |
| 03 | [DDD & Onion Architecture](03-DDD-and-Onion-Architecture.md) | Why the layering, and what it costs |
| 04 | [HLD & Architectural Decisions](04-HLD-and-Architectural-Decisions.md) | ADR-001 monolith, ADR-002 async analysis |
| 05 | [Project Structure & Contracts](05-Project-Structure-and-Contracts.md) | Module boundaries, OpenAPI, event schemas |
| 06 | [AI Provider Evaluation](06-AI-Provider-Evaluation.md) | ADR-003 — vendor selection, and its open risks |
| 07 | [Product Decisions](07-Product-Decisions.md) | PD-1..PD-5 |
| 08 | [Unit Economics](08-Unit-Economics.md) | What a scan costs and what that permits |
| 09 | [Infrastructure & Services](09-Infrastructure-and-Services.md) | Cloud Run, Neon, Auth0, spend controls |
| 10 | [Brand Name](10-Brand-Name.md) | ADR-005 — why "Ayna" |
| — | [Handoff](HANDOFF.md) | Current state, in prose |

These were authored in Obsidian, so internal references use `[[wikilinks]]`,
which GitHub does not render as links. The table above is the navigable index.

## Three that are worth reading even if you skip the rest

**[06 — AI Provider Evaluation](06-AI-Provider-Evaluation.md).** The vendor
decision, and the one place this project refused to tidy up its own result. The
selected API's `acne_score` read "Clear" on visibly moderate acne — but the
evidence photo had two uncontrolled variables (three-quarter pose, warm indoor
light), both biasing the same way. That finding was written up as closed
against the vendor once, and that was premature. It is now recorded as **open
and confounded**, waiting on one controlled photograph, because a conclusion
drawn from a confounded test is worse than no conclusion.

**[04 — HLD & Architectural Decisions](04-HLD-and-Architectural-Decisions.md).**
Includes ADR-002's async analysis pipeline, which the implementation then
*deviated from* — Cloud Run only allocates CPU during a request, so "return 202
and finish in a goroutine" is not available on this platform. The deviation is
documented at the call site rather than the ADR being quietly rewritten.

**[08 — Unit Economics](08-Unit-Economics.md).** What one scan actually costs,
which is the number that decides the free tier, the paywall, and whether any of
this is affordable to run at all.

## How these are maintained

ADRs are **appended to** when a decision is revisited, not silently rewritten.
A decision record that reflects only the current answer has thrown away the
thing that made it worth recording: what was believed at the time, and what
changed.

The same applies to the risks. R-1 (acne accuracy) and R-2 (performance across
skin tones) are both still open, and are stated as open rather than resolved
optimistically.
