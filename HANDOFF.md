# Ayna — handoff

Written 2026-08-27, when the previous session ran out of context. If that date
is stale, trust the code and the vault over this file.

## Read these first

1. `00-Phase-0-Roadmap-MOC.md` (this folder) — index of every decision
2. `09-Infrastructure-and-Services.md` — the service stack and the wiring order
3. `D:\ayna-backend\README.md` — what runs and why

Every non-obvious choice already has its reasoning written down. **Read the
vault before changing a decision** — most things that look arbitrary are not,
and the note usually says what breaks if you change it.

## Where things stand

**Working end to end on a real phone:** Flutter → Auth0 → JWT → Go API →
Postgres on Neon. A real user row exists. All tests pass:
`cd D:\ayna-backend; .\dev.ps1 check` and `cd D:\ayna-app; flutter test`.

Phase 0 exit criteria: **6 of 7**. The seventh is ADR-003, blocked on photos.

## The immediate next action

**First Cloud Run deploy.** Follow `D:\ayna-backend\deployments\SETUP.md`.

Progress so far: gcloud installed, project `ayna-prod-bk2608` created.
**Stuck at step 2** — linking a billing account:

```powershell
$PROJECT = "ayna-prod-bk2608"
gcloud billing accounts list          # gives an ID like 01A2B3-C4D5E6-F7G8H9
gcloud billing projects link $PROJECT --billing-account=<that ID>
```

A billing account is required even though the usage sits inside the always-free
tier. Then continue from step 3 of SETUP.md. No Docker needed — Cloud Build
does it.

## Blocked, and not worth starting

| Blocked | On |
|---|---|
| ADR-003, the AI vendor | 20 consented face photos, 10 per Fitzpatrick band |
| PD-2 severity values | the spike's real distributions, then medical review |
| `skinanalysis` module | ADR-003 — **do not start it** |

Photo collection waits on people, not code, so it runs in parallel with
everything else.

## Deliberately unfinished

- **Email OTP.** Login currently uses Auth0's Google connection, which triggers
  a dev-keys warning. PD-3 chose email OTP; the fix is a Passwordless-Email
  connection, not registering Google OAuth credentials.
- **Dark theme.** Warm Mirror is light-committed and `themeMode` is pinned. A
  dark palette is a design pass, not an inversion.
- **Retention and reminder settings** are marked SOON rather than wired to
  nothing.
- **Billing / store IAP** not built. The upgrade button is deliberately inert.
- **Graceful shutdown** has never been exercised by a real SIGTERM. The first
  Cloud Run scale-down will be the test.

## Things that will waste your time if you do not know them

- **Windows PowerShell, not bash.** No `make`, no Docker, `adb` not on PATH.
  `D:\ayna-backend\dev.ps1` covers the tasks; `dev.ps1 phone` sets up
  `adb reverse` so the phone can reach the local API.
- **`dev.ps1` is ASCII-only on purpose.** PowerShell 5.1 reads BOM-less scripts
  as ANSI; one em dash produces a parse error many lines from the real cause.
- **Auth0's Application Access Policy** (API → Settings) gates user-delegated
  flows for *every* application. It cost hours. If `/authorize` works without
  `audience` and fails with it, that policy is the first thing to check — the
  bisect isolates the fault in one step.
- **Both drives run near full.** C: was at 2.3 GB free. Check before
  downloading anything large.
- **Integration tests need `DATABASE_URL`** and skip without it. They run
  inside transactions that always roll back, so they leave nothing behind.

## The standing bias

This is a CV project. A recruiter reads a screenshot and a link, never 1,145
lines of markdown. **Prefer the visible slice over more planning** — the plan
has been sound for a while, and the risk now is polishing it instead of
shipping something a stranger can look at.
