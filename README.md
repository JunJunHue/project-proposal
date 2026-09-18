# Dispatch: Placement for GPU Training Jobs

---

## What and why?

Dispatch is a web platform that decides where, when, and on what hardware to run a GPU training job, then runs it there.

A user points Dispatch at a workload. Dispatch profiles a short sample of the run, predicts wall-clock time on each candidate GPU SKU, forecasts spot price and preemption risk across providers, forecasts grid carbon intensity by region and hour, and returns ranked placement plans under a user-supplied deadline.

Example output:

> Plan A: 2×H100, Provider B, us-central, start 01:30 ET, finish 07:10 ET. $38.40, 5.9 kgCO2e, 12% preemption risk.
> Baseline (1×A100, start now): $71.20, 13.1 kgCO2e, finish 09:40 ET.

Three decisions drive this, and teams without a platform engineer make all three badly because the inputs are scattered across provider dashboards and change hourly:

1. **Hardware.** Users pick the largest available GPU rather than the lowest cost per unit of work. A dataloader-bound or bandwidth-bound job gains little from an H100 over an L40S and pays a large premium for it.
2. **Provider and start time.** Spot prices for identical SKUs differ across providers and by hour of day [verify: measure real spreads across 3 providers over one week and cite]. Manual arbitrage is possible but nobody does it.
3. **Efficiency.** Unoptimized training runs commonly sit well below achievable model FLOPs utilization [verify: cite MFU benchmarks]. The user pays for GPU-hours that do no work and has no signal that this is happening.

Carbon intensity of the same GPU-hour varies by roughly an order of magnitude by grid and hour [verify against EIA-930 history]. Deferrable overnight jobs are the textbook case for load shifting, and almost none are shifted.

Each decision is small individually and large in aggregate. All three reduce to forecasting under uncertainty plus a constrained optimization, which is a software problem, not a discipline problem.

---

## For whom?

Target user: **anyone who rents GPUs by the hour against a fixed budget and has nobody whose job is infrastructure.** Four realistic segments:

- **Solo builders and indie developers.** Fine-tuning open-weight models on one or two rented GPUs, paying with a personal card. They pick a provider from a comparison blog post and never revisit it. The trigger to look for a tool is an unexpected invoice at the end of a month of experiments.
- **Seed-stage and pre-seed founders.** Compute is a visible line against runway and often the second-largest expense after salaries. Spend is tracked in a spreadsheet, if at all. The trigger is being asked to justify the burn, or realizing a month of training cost as much as a month of contractor time.
- **Graduate students and academic research groups.** Working against a fixed credit allocation or a grant line. Their jobs are the most deferrable of the four, since overnight and weekend runs are routine, which makes them the best case for time shifting. The trigger is exhausting an allocation before a submission deadline.
- **Applied ML teams inside non-technology companies.** Small headcount, an explicit cost center, sometimes an energy or emissions reporting obligation, and no platform engineering function to build this internally. The trigger is finance asking for a per-project breakdown nobody can produce.

Shared behavior across all four: rents rather than owns, billed hourly or per second, runs batch jobs that tolerate a few hours of delay, treats budget as a hard constraint, and currently optimizes by intuition because no tool exists.

Initial end-users are drawn from the first and third segments. Both are reachable directly through university research groups and open-source ML communities, both feel the cost constraint immediately, and both run workloads deferrable enough that time shifting produces a visible result. We will interview a handful in the first three weeks and again against a working prototype around week 8.

Out of scope: hyperscalers, teams with an existing platform org, and production inference serving. Those users have internal tooling and different constraints.

Out of scope: hyperscalers, teams with an existing platform org, and production inference serving. Those users have internal tooling and different constraints.

---

## How? (end-user view)

1. **Setup.** Create an account, add read-only provider API keys, set an optional monthly budget and a cost-versus-carbon weighting.

2. **Profile.** Wrap the training command: `dispatch profile python train.py`. The agent runs a bounded number of steps on any available hardware and records step time, GPU utilization, memory high-water mark, achieved FLOPs, dataloader wait, energy, and communication time. It uploads a compact trace. No source code leaves the machine. Users who cannot profile instead describe the job in a form and accept wider prediction intervals.

3. **Plan.** Dispatch returns ranked plans. Each shows SKU and count, provider, region, start time, predicted duration with an uncertainty band, cost, kWh, kgCO2e, and preemption probability. The user sets a deadline; only plans meeting it at a stated confidence level are shown. Each ranking includes the reason, since an unexplained recommendation gets ignored.

4. **Run.** Dispatch provisions at the chosen time, transfers and launches the job, streams logs and telemetry, and checkpoints on a schedule. On preemption it resumes from the last checkpoint on the next-best plan. Users who will not grant provisioning access stay in **advisory mode**, which outputs the plan plus a launch command.

5. **Monitor.** Live view: step time, utilization, memory, temperature, cost accrued versus predicted, and anomaly flags for straggler ranks, thermal throttling, and pipeline stalls. Kill from this view.

6. **Report.** Post-run: actual versus predicted on every dimension, measured utilization, and recommendations ranked by estimated savings. Team views roll up spend by user and project and flag idle provisioned instances.

7. **Learn.** Every completed run becomes a labeled example for the runtime predictor. Accuracy on a given team's workloads improves with use.

---

## Data sources

| Layer | Source | Access | Constraint |
|---|---|---|---|
| Grid carbon | EIA-930 via EIA Open Data API | Free key, bulk CSV/JSON, hourly, ~65 balancing authorities, fuel mix and estimated CO2 from 2018 | US only. Published ~1h after operating hour. No forecast, so we build one |
| Weather (forecast features) | NOAA / Open-Meteo | Free | None material |
| Energy measurement | Zeus (Apache-2.0, PyTorch ecosystem project) | Open source library | Use directly; do not reimplement NVML polling |
| Published benchmarks | MLPerf results, ML.ENERGY Benchmark | Open | Sparse, and reflects heavily tuned submissions on atypical systems |
| GPU prices | Provider REST/GraphQL APIs (marketplace search endpoints, unauthenticated retail price endpoints, spot price history) | Free or free with credentials | **No historical archive exists.** Must be collected by polling |
| Preemption history | None published at useful granularity | Inferred | An offer disappearing between polls is an interval-censored observation; modeled by survival analysis |
| Runtime labels | Generated by us | Own benchmark runs | Requires a modest compute budget; see scope |

Two consequences. First, carbon forecasting becomes a model we own and evaluate rather than a paid API call. Second, price and preemption data must be accumulated over calendar time we cannot compress, which sets the schedule below.

The benchmark corpus and the price/preemption archive are the project's only non-reproducible assets. They are the reason this cannot be done in a spreadsheet.

---

## Scope

Team of 4 to 6, one semester. Five workstreams after a shared design phase.

| Workstream | Content |
|---|---|
| A. Profiling agent + CLI | Instrumentation via Zeus, trace format, upload client |
| B. Forecasting | Runtime/throughput prediction, price forecasting, preemption survival model, carbon intensity forecasting |
| C. Optimizer | Ranking over discrete plan space under deadline, budget, and uncertainty |
| D. Web application | Auth, plan view, live run view, reports, team dashboards |
| E. Provider adapters + execution | Provision, launch, checkpoint, resume on preemption |

**Not too easy.** The core is not CRUD. It requires a runtime model trained on a corpus we generate, time-series forecasting of prices, survival analysis on censored availability data, integration of three heterogeneous provider APIs plus federal energy data, optimization under uncertainty, and a fault-tolerant execution path that survives preemption.

**Not too ambitious.** Bounded on every axis that would otherwise expand:

- 3 providers, not 12. Two marketplaces plus one hyperscaler.
- 3 to 4 SKU families. Single-node and 2-node only. No large-scale distributed training.
- Training jobs only. No inference serving, no autoscaling.
- All external data is free or free-tier.
- **Advisory mode is a complete product on its own.** If workstream E slips, we ship the recommender, the dashboard, and the reports. Execution is an upgrade, not a prerequisite.

**Schedule.**

| Week | Milestone |
|---|---|
| 1 | Price polling running across 3 providers at 15-minute intervals. Blocking dependency: every downstream price and preemption model needs calendar time to accumulate. Nothing else is scheduled ahead of it. |
| 2 | EIA-930 and weather ingestion live. Trace format frozen. |
| 4 | Benchmark corpus collection running. First runtime model on partial data. |
| 8 | Advisory mode end to end. A real user profiles a real job and receives a ranked plan. User interviews against the working artifact. |
| 11 | Execution on one provider with checkpoint and resume on preemption. |
| 13 | Team dashboards and post-run reports. Second feedback round incorporated. |
| 14 | Hardening and evaluation. |

**Evaluation.** Median absolute percentage error of runtime prediction on held-out real runs. Realized cost and carbon delta versus each user's pre-Dispatch baseline. Count of distinct real users with three or more completed jobs.

**Risks.**

1. *Runtime prediction too inaccurate to be actionable.* Report intervals rather than point estimates; apply a per-user learned correction after the first few runs.
2. *Insufficient price history by the time the model is needed.* Mitigated only by starting collection in week 1.
3. *Provider APIs unstable or rate-limited.* One adapter interface, cached snapshots, advisory mode degrades gracefully.

---

## Open questions for user interviews

- Do users change behavior for carbon, or only for cost? If only cost, carbon becomes a reporting feature rather than an optimization objective.
- Will users grant provisioning access, or is advisory mode the real product?
- Is the binding constraint budget or queue wait time? These imply different products.
