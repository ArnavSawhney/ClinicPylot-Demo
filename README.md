# ClinicPylot

Multi-tenant WhatsApp SaaS that helps Indian dental clinics recover revenue lost to no-shows and missed recalls.

**Status:** Product built end to end. Market validated with dental clinics. Preparing for first pilot deployment.

**Repository note:** ClinicPylot's application code is private (it's the product). This repo is a public writeup covering the architecture, the ML system, and the design decisions.

---

## The problem

Indian dental clinics lose a meaningful share of scheduled appointments to no-shows, and lose most of their recall revenue (six-month cleanings, follow-up procedures) because the reminder workflow is manual and inconsistent. Every clinic I spoke with is aware of the problem, runs some ad-hoc WhatsApp reminders themselves, and has no systematic way to know what actually works or to price the failure. That gap is what ClinicPylot closes.

## What ClinicPylot does

End-to-end platform that:

1. Ingests patient and appointment data from the clinic's existing PMS, or a CSV upload for smaller clinics.
2. Runs a per-patient policy that decides *what* follow-up to send (message variant, timing, deposit ask), sends it over the WhatsApp Business API, and tracks response.
3. Collects Razorpay-backed deposits at booking or reconfirmation to align patient incentives.
4. Surfaces the recovered revenue and no-show rate in a clinic-facing Next.js dashboard, and feeds outcome data back into the policy.

The whole loop, from patient message to deposit to dashboard, runs autonomously per clinic under row-level-secured Postgres tenancy.

## Architecture

```
[Clinic PMS / CSV] ──► [FastAPI ingest] ──► [Postgres (Supabase, RLS)]
                                                   │
                                                   ▼
                                        [Contextual bandit policy]
                                                   │
                                        selects (variant, timing, ask)
                                                   │
                                                   ▼
                          [WhatsApp Business API worker (Redis queue)]
                                                   │
                                                   ▼
                                    [Response + payment webhook]
                                                   │
                                                   ▼
                                    [Reward log + off-policy eval]
                                                   │
                                                   ▼
                             [Next.js 14 dashboard, per-tenant]
```

## The ML system

The core problem, "for this patient at this moment, what follow-up action maximises the probability of showing up (and depositing)," is a natural fit for a **contextual bandit** rather than a full RL formulation. Actions are discrete (message variant × send-time bucket × deposit ask), rewards are observed within a short horizon (the appointment date), and the counterfactual evaluation problem is well studied.

**Policy.** A contextual bandit over the discrete action grid, with Thompson-sampling exploration. Context features cover patient history (prior no-shows, response rate, days since last visit), appointment attributes (procedure type, provider, value), and clinic-level priors for cold start. The action-value model is kept intentionally simple so logged propensities stay clean for downstream evaluation.

**Off-policy evaluation.** Every policy change is validated against logged data before it goes near a real patient. I use two estimators in parallel:

- **SNIPS (Self-Normalized Importance Sampling)** — lower variance than vanilla IPS, appropriate given the small action space and clinic-scale sample sizes.
- **Doubly-Robust** — combines a direct-method reward model with importance sampling, so the estimator stays consistent as long as *either* component is well specified.

Both are computed on the reward log with propensity clipping. Only policies that beat the incumbent on both estimators (with bootstrapped confidence intervals) are rolled out.

**Cold start.** New clinics start on a clinic-agnostic prior fit on synthetic data and early pilot logs, then blend into a clinic-specific policy as they accumulate their own reward signal.

## Stack and why

| Layer | Choice | Why |
|---|---|---|
| API | **FastAPI** | Async, first-class Pydantic validation, low-boilerplate for webhook-heavy workloads |
| Database | **Supabase Postgres** | Row-level security is native, which is the right primitive for per-clinic tenancy |
| Frontend | **Next.js 14** (App Router) | Server components fit dashboard reads; clean auth flow |
| Queue | **Redis** | Rate-limiting WhatsApp sends and retrying failed webhooks |
| Payments | **Razorpay** | India-native, UPI + card, low friction for deposits |
| Deploy | **Docker** on VPS | Predictable, easy to hand off if I bring on infra help later |

## Status

- Product built end to end and running in staging.
- Market validation completed with dental clinics in the NCR region.
- Preparing for first pilot deployment.

## Roadmap

- **Now:** onboard first pilot clinic, tighten bandit propensity logging, harden Razorpay reconciliation edge cases.
- **Next:** multi-language message variants (Hindi, Punjabi), voice-note follow-ups, provider-level dashboards.
- **Later:** publish a technical writeup on the off-policy-evaluation results once there is enough live data to report honestly.

## About

I'm Arnav Sawhney, founder and sole engineer. B.Tech Biosystems Engineering + AI minor at Plaksha University, graduating 2027. Reach me at arnavsawhney@clinicpylot.xyz or on [LinkedIn](https://www.linkedin.com/in/arnav-sawhney-2bb391253/).
