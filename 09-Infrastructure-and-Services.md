---
tags: [phase-0, infrastructure, adr, services]
project: Ayna
created: 2026-08-25
status: accepted — pricing to re-verify before launch
---

# Infrastructure & Services
Part of: [[00-Phase-0-Roadmap-MOC|Phase 0 Roadmap]]

> [!info] The framing that produced this list
> The obvious "not Firebase" answer is Supabase — and it is the wrong answer here. Supabase is a BaaS: adopted properly it *replaces* a backend, which would make the modular monolith, the bounded contexts, the onion layering and both accepted ADRs decorative. For a portfolio piece whose differentiator is backend architecture, that is backwards.
>
> So: **keep the Go backend, rent best-in-class infrastructure around it.** Every service below sits behind a port from [[03-DDD-and-Onion-Architecture|DDD & Onion Architecture]], which is what makes any one of them swappable later.

## ADR-004: External Service Selection

**Status:** Accepted

**Context:** the app needs identity, a database, object storage, a cache, a job queue, push, email, and hosting for two deployables (ADR-002). The constraint beyond the technical ones is deliberate: this is a CV project, so "different from Firebase" is a real requirement, not a preference.

**Decision:**

| Concern | Choice | Why this one |
|---|---|---|
| **Identity** | **Auth0** | Mature, **official** Flutter SDK. See the Clerk note below. |
| **OTP delivery** | **Resend** | Email only at MVP. Generous free tier, good DX, no gateway contract. |
| **Hosting** | **Google Cloud Run** — two services | Free forever at this scale, scales to zero, two services map 1:1 onto ADR-002. Supersedes Fly.io — see the revision below. |
| **Database** | **Neon** Postgres | Managed, autoscaling, branch-per-PR. Co-locate its region with Cloud Run's. |
| **Object storage** | **Cloudflare R2** | S3-compatible and **zero egress fees** — see the note below; this is a unit-economics decision, not a convenience one. |
| **Cache / job status / rate limits** | **Upstash Redis** | Serverless, pay-per-request, fits the bursty profile in NFR-3. |
| **Job queue** | **River** (Postgres-backed, Go) | Transactional enqueue. See below — this is the most consequential row in the table. |
| **Push** | **FCM** | Unavoidable. See below. |
| **Errors** | **Sentry** | Free tier, five minutes to wire. |
| **Traces / metrics** | **OpenTelemetry → Grafana Cloud** | NFR-9 wants these from the first deployed version. |
| **CI/CD** | **GitHub Actions** | Also where the module-boundary lint from [[05-Project-Structure-and-Contracts\|Contracts]] runs. |

---

### Why Auth0 and not Clerk

Clerk has the better developer experience and is the more fashionable choice — but its first-class SDKs are React, Next.js, Expo and React Native. **Its Flutter SDK is still beta**, and its native mobile SDKs (iOS/Android) reached v1 only in early 2026. Auth0's Flutter SDK is long-established and officially maintained.

For a solo build on a Flutter client, betting the login flow on a beta SDK is the wrong risk to take. Revisit if Clerk's Flutter SDK goes GA.

> [!warning] What managed auth costs you, and where the depth moves
> This was a deliberate trade: it saves roughly a week, and it makes the IAM module thinner than [[03-DDD-and-Onion-Architecture|DDD]] implies. An interviewer who asks "so what did *you* build in IAM?" needs an answer.
>
> The answer is that three things stay genuinely yours, and none of them are Auth0's:
> - **ConsentRecord** (FR-1) — timestamped, versioned, revocable, and the gate on every scan
> - **The age gate** (PD-1) — refusing registration under 18, storing birth year only
> - **Entitlement derivation** (PD-3) — from store receipts, never from local state
>
> Auth0 answers "who is this person". Everything about *what they are allowed to do here* is still domain logic you own. Build those properly and the module is not a wrapper.

### Why River, and not NATS or Redis Streams

[[04-HLD-and-Architectural-Decisions|HLD]] proposed NATS or Redis Streams. River is better here for one specific reason.

The `Scan` aggregate says a Scan and its report are created together, and `ScanSubmitted` must enqueue an `AnalysisJob`. With an external queue those are **two systems**: save the Scan to Postgres, then publish to the queue. If the process dies between them, the scan exists and the job does not — the user waits forever for a report nobody is generating. The standard fix is a transactional outbox, which is real work.

River is Postgres-backed, so the enqueue happens **in the same transaction as the insert**. Either both land or neither does. The failure mode disappears rather than being mitigated.

It also removes a service: one fewer thing to run, monitor and pay for. ADR-001's "in-process bus now, real queue later" still holds — River *is* the real queue, reached through the same port.

### Why R2 specifically

Photos are not write-once. Every report view re-reads the image to draw the heatmap over it, and the trend screen may show several. Egress is the cost that grows with engagement, and R2 charges **zero** for it where S3 charges per gigabyte.

Given NFR-7 makes per-scan cost the dominant threat, picking storage on egress rather than storage price is the right axis. R2 is S3-compatible, so the adapter is the standard AWS SDK either way and switching back is a config change.

### FCM is unavoidable, and that is fine

Android push goes through Firebase Cloud Messaging. There is no practical alternative — OneSignal and the rest are wrappers over FCM. Using FCM for message delivery is not "using Firebase as a backend"; no data model, no auth, no hosting touches it. Worth stating plainly so it does not read as a failure of the brief.

**Consequences:**

- *Positive*: every choice sits behind an existing port, so none of them are load-bearing on the domain model. The stack is also genuinely varied — Postgres, object storage, a serverless cache, a transactional queue, OTel — which is a broader surface to talk about than a single BaaS.
- *Negative*: more accounts, more secrets, more things that can be down. Mitigate by wiring Sentry and the OTel exporter on day one rather than after the first outage.
- *Negative*: Auth0 becomes a hard external dependency on the login path. NFR-5's 99.9% target now partly depends on someone else's uptime — worth stating in the ADR rather than discovering it.
- *Trigger to revisit*: Clerk's Flutter SDK reaching GA; Auth0 pricing at scale; or sustained traffic that pushes Cloud Run past its free allowance, at which point an always-on box may be cheaper than per-request billing.

---

## Revision: hosting moved from Fly.io to Cloud Run

> [!warning] Fly.io has no permanent free tier, and the options presented said it did
> As of 2026, new Fly accounts get **$5 in trial credits, capped at 2 VM-hours or 7 days**. There is no ongoing free allowance for Machines, Postgres or bandwidth. Two small always-on apps run ~$5–15/month — cheap, but starting on day one. With zero spend as a hard constraint, Fly is out.

**Cloud Run's free tier never expires** and permits commercial use: 2M requests, 180,000 vCPU-seconds, 360,000 GiB-seconds and 1 GiB egress per month, in select US regions.

Modelled against 100 users each scanning weekly:

| | Usage | Share of free tier |
|---|---|---|
| Requests | ~7,800/mo | **0.4%** |
| Compute | ~3,850 vCPU-sec/mo | **2.1%** |
| Egress | ~38 MB (JSON only) | small |

Roughly **47× headroom** before anything is billable. The egress figure holds only because photos are served straight from R2 by signed URL rather than proxied through the API — which makes the R2 choice load-bearing rather than merely economical.

> [!tip] Cloud Run is not Firebase
> The earlier hesitation — that GCP "sits next door to Firebase" — was weaker than the cost constraint, and on reflection weak in itself. Firebase is a BaaS; Cloud Run is a container platform. "Containerised a Go service, deployed it scale-to-zero on Cloud Run" is a different skill and a different story from "used Firebase". The only overlap is the vendor's name on the invoice.

### The conflict this creates, and the fix

Cloud Run scales to zero. River is a **pull** queue — the worker polls Postgres for jobs. A worker scaled to zero is not polling, so jobs would sit until something woke it. Keeping `min-instances=1` fixes that and costs money, which defeats the point.

**Decision: durable pull queue, best-effort push wake-up.**

1. The API commits the `Scan` and the River job **in one transaction**, exactly as ADR-004 intended. The job is durable the instant the transaction commits.
2. The API then fires a **fire-and-forget HTTP ping** at the worker service. Cloud Run cold-starts it (a Go binary boots in a few hundred milliseconds), it drains the queue, and it scales back to zero.
3. A **slow Cloud Scheduler tick** (every few minutes) hits the same endpoint as a safety net.

The property that makes this sound: **the ping is an optimisation, never a correctness requirement.** If it fails, is lost, or races, the job is already committed in Postgres and the scheduled tick collects it. Losing the ping costs latency, never a scan.

That is the right way round, and the opposite of what a naive "publish to queue after commit" does — there, a lost publish loses the job.

## Costs

Indicative free tiers at time of writing — **verify before relying on any of them**, as this is exactly the sort of thing that changes quietly:

| Service | Free allowance |
|---|---|
| Neon | 10 projects, 0.5 GiB storage, autoscaling to 2 CU |
| Cloudflare R2 | ~10 GB stored, zero egress |
| Upstash Redis | pay-per-request with a free monthly allowance |
| Auth0 | free up to a monthly-active-user ceiling |
| Resend | a few thousand emails/month |
| Sentry, Grafana Cloud | free developer tiers |

| Cloud Run | 2M req, 180k vCPU-sec, 360k GiB-sec, 1 GiB egress — never expires |

Realistic pre-launch total: **$0/month**. Every service above sits inside a free allowance at this scale, and the first real cost is AI inference, which [[08-Unit-Economics|Unit Economics]] covers separately.

### What was rejected, and why

| Option | Why not |
|---|---|
| **Fly.io** | No permanent free tier as of 2026 |
| **Render** free | Sleeps after 15 min with a **30–50s cold start**. Disqualifying for the API: the first user of the hour waits a minute against an 8-second promise (NFR-2) |
| **Koyeb** free | One free instance per organisation — you need two — and it scales to zero after an hour anyway, which a background worker triggers immediately |
| **Oracle Cloud Always Free** | Genuinely the most generous (4 ARM cores, 24 GB RAM, free forever) and would run everything on one box. Rejected on three counts: signup is notoriously difficult, capacity in a given region is not guaranteed, and it is a VM — you own backups, patching and TLS. It also undercuts NFR-3's claim that the system absorbs a 50–100× spike without manual intervention. **Worth reconsidering if you decide the ops experience is itself the CV value.** |

## Wiring order

Not everything at once. Each step should leave something that runs:

1. ~~**Cloud Run + Neon + a health endpoint**~~ — **done 2026-08-26** (locally; the Cloud Run deploy itself is still unexecuted)
2. ~~**Auth0**~~ — **done 2026-08-27.** A real token from the phone authenticates against the Go API and creates a row in Neon: `usr_01M10FEC7VMKGBPK8XNAG4EX9C`. Resend is not wired yet — sign-in currently uses Auth0's Google connection rather than email OTP.

> [!warning] What the Auth0 setup cost, so the next tenant does not repeat it
> Four dashboard problems stacked on each other, none of them in the code:
> 1. The settings form **silently refuses to save** while any field is invalid — the callback URL was never stored.
> 2. **Application Login URI must be `https://`** and is for web apps only. Leave it empty for native.
> 3. A **URI scheme cannot contain an underscore** (RFC 3986), so `auth0Scheme` must differ from the `applicationId`. The callback is `com.ayna.aynaapp://…/android/com.ayna.ayna_app/callback` — no underscore in the scheme, one in the path.
> 4. The real culprit: **API → Settings → Application Access Policy → User-delegated Access**. This gates Authorization Code flows for *every* application, which is why two different apps failed identically. Authorize the app under the API's **Application Access** tab.
>
> The diagnostic that should have come first: call `/authorize` **with and without** the `audience` parameter. Working without it and failing with it isolates the fault to the API in one step, rather than inferring from error strings.
>
> Still on Auth0's **development keys** for the Google connection — fine for now, unsuitable for production. PD-3 chose email OTP anyway, so the fix is to switch to a **Passwordless — Email** connection and disable social, which removes the warning rather than working around it.
3. **R2 + Postgres schema** — scan submission storing a real photo
4. **River + the worker app** — ADR-002's split becomes real
5. **Upstash** — job status polling for PD-4
6. **Sentry + OTel** — before the first real user, per NFR-9
7. **FCM** — last; it only matters once reports take long enough to background the app

## Related
[[08-Unit-Economics|← Unit Economics]] · [[04-HLD-and-Architectural-Decisions|HLD & ADRs]] · [[03-DDD-and-Onion-Architecture|DDD & Onion Architecture]] · [[00-Phase-0-Roadmap-MOC|Back to Roadmap]]
