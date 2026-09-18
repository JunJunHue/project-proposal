# Dispatch: Cost- and Carbon-Aware Placement for GPU Training Jobs

*Project proposal. Draft v0.1. Bracketed `[verify]` markers flag numbers to source before submission.*

---

## What and why?

**Dispatch is a web platform that tells you where, when, and on what hardware to run a GPU job, then runs it there.**

A user points Dispatch at a training or inference workload. Dispatch profiles a short sample of the run, predicts how long that workload will take on each candidate GPU SKU, forecasts spot price and preemption risk across providers, forecasts grid carbon intensity in each candidate region, and returns a ranked set of placement plans under a user-supplied deadline:

> Run on 2×H100 at Provider B (us-central), starting 01:30 ET, finishing 07:10 ET. Estimated $38.40 and 5.9 kgCO2e.
> Your current default (1×A100, run now) costs $71.20 and 13.1 kgCO2e and finishes at 09:40.

**The problem.** Anyone training models outside a hyperscaler's internal cluster makes three decisions badly, because the information needed to make them well is scattered across a dozen dashboards and changes hourly:

1. **Hardware choice.** Users pick the biggest GPU they can afford rather than the one with the lowest cost per unit of work. A workload that is dataloader-bound or memory-bandwidth-bound gets no benefit from an H100 over an L40S, and pays a multiple for it.
2. **Provider and timing.** Spot prices for the same SKU differ by a large factor across providers and by hour of day [verify: sample real spreads across 3 providers for a week and cite]. Almost nobody arbitrages this manually, because tracking it is tedious and the pricing pages are not designed for comparison.
3. **Efficiency.** Most non-expert training runs sit at low model FLOPs utilization [verify: cite MFU benchmarks], meaning a large fraction of GPU-hours purchased is paid for and not used. The user rarely knows this, because nothing tells them.

Compounding all three: **the carbon cost of a GPU-hour varies by roughly an order of magnitude** depending on which grid and which hour it runs on [verify against Electricity Maps historical data]. A deferrable overnight job is exactly the workload that should be shifted, and essentially none of them are.

**Why this matters.** GPU compute is now a meaningful line item for research labs, startups, and student teams, and a meaningful load on regional grids. The decisions above are individually small and collectively large. They are also decisions that a piece of software can make better than a human, because they reduce to forecasting under uncertainty plus a constrained optimization. That is exactly what should be automated and exactly what currently is not.

---

## For whom?

Dispatch targets **small teams who pay for GPU time out of a fixed budget and have no infrastructure engineer.** Concretely, our initial end-users are real people we already have access to:

**Primary users (design partners for v1):**

- **The ML team at NYU's Business Analytics Club.** Roughly [N] students who train models on a mix of personal hardware, Colab, and rented cloud GPUs, with no shared view of what anything costs. Direct access through the team lead.
- **PostGrad student consultants.** A student ML consulting program running sponsor-facing tracks, including a RAG pipeline on private cloud, a waste-heat modeling project, and a bioacoustics model. These are fixed-credit engagements where overspending on compute directly cuts what the team can deliver. Real deadlines, real budgets, real users we can interview weekly.
- **Courant and Tandon research groups running on shared clusters.** Students who queue jobs against a finite allocation and currently have no way to estimate whether a run will use 4 GPU-hours or 40 before submitting it.

**Secondary users (validation interviews, not v1 targets):**

- **Data center operators in the Infrastructure Masons network**, including contacts reachable through iMasons student ambassador channels and prior technician work at a New York colocation facility. These people will not be Dispatch's users, but they are the correct people to check our energy and carbon accounting against, and they will tell us quickly if the model is wrong.
- **Participants at the October Sustainable Data Centers hackathon**, which supplies roughly [N] people in one room who are already working on carbon visualization and reduction. A usable prototype there is a free source of structured feedback and a plausible source of the first non-NYU users.

We are explicitly **not** building for hyperscalers, for teams with a dedicated platform org, or for inference serving at scale. Those users have internal tooling and different constraints.

---

## How? (end-user view)

### 1. Onboarding
The user creates an account, adds read-only API keys for whichever GPU providers they use, and optionally sets a monthly budget and a default carbon preference (cheapest, greenest, or a weighting between them).

### 2. Profiling a job
The user wraps their training command with a one-line CLI: `dispatch profile python train.py`. The agent runs a bounded number of steps on whatever hardware is at hand, records step time, GPU utilization, memory high-water mark, achieved FLOPs, dataloader wait time, and inter-GPU communication time, and uploads a compact trace. No source code leaves the machine.

Alternatively, the user describes the job in a form (model family, parameter count, dataset size, target steps) and accepts a wider prediction interval.

### 3. The placement plan
Dispatch returns a ranked table of plans. Each row shows SKU and count, provider and region, start time, predicted wall-clock duration with an uncertainty band, total cost, energy in kWh, carbon in kgCO2e, and preemption probability. The user sets a deadline and Dispatch only shows plans that meet it at a stated confidence level.

The plan view includes a plain-language explanation of the ranking, because a recommendation the user does not trust is a recommendation they ignore: *"This job is 34% dataloader-bound. Moving from A100 to H100 would cut step time by an estimated 9%, not 40%, so the H100 premium does not pay for itself here."*

### 4. Execution
The user clicks Run. Dispatch provisions the instance at the chosen time, transfers the job, launches it, streams logs and live telemetry to the dashboard, and checkpoints on a schedule. On spot preemption, it resumes from the last checkpoint on the next-best plan without user intervention. Users who do not want Dispatch touching their infrastructure can stay in **advisory mode**, where Dispatch produces the plan and a copy-pasteable launch command.

### 5. Live run view
While a job runs: current step time, GPU utilization, memory, temperature, cost accrued so far versus predicted, and anomaly flags (straggler rank, thermal throttling, utilization collapse, suspected data pipeline stall). A user can kill a job from this view.

### 6. Post-run report and fleet dashboard
After completion: actual versus predicted cost, duration, energy, and carbon; measured utilization; and concrete recommendations ranked by estimated savings. Team-level views roll this up by user and by project, show budget burn-down, and flag idle provisioned instances.

### 7. Feedback loop
Every completed run becomes a labeled training example for the runtime predictor. The system gets more accurate on a given team's workloads the more that team uses it.

---

## Scope

**Team of 4 to 6, one semester.** Five roughly parallel workstreams after a shared two-week design and data-plumbing phase:

| Workstream | Content |
|---|---|
| A. Profiling agent + CLI | Instrumentation hooks, trace format, upload client |
| B. Forecasting services | Runtime/throughput prediction, spot price and preemption models, carbon intensity ingestion |
| C. Placement optimizer | Constrained ranking under deadline, budget, and uncertainty |
| D. Web application | Auth, plan view, live run view, reports, team dashboards |
| E. Provider adapters + execution | Provision, launch, checkpoint, resume on preemption |

**Why this is not too easy.** The core is not CRUD. It requires a real predictive model trained on a benchmark corpus we build ourselves, time-series forecasting of prices and availability, integration of at least three heterogeneous provider APIs plus external carbon data, an optimizer over a discrete plan space under uncertainty, and a fault-tolerant execution path that survives preemption. Each of those is a genuine engineering problem with a correctness bar.

**Why this is not too ambitious.** The scope is bounded on every axis that would otherwise explode:

- **3 providers, not 12.** Two commodity GPU marketplaces plus one hyperscaler.
- **3 to 4 SKU families**, single-node and 2-node multi-GPU only. No large-scale distributed training.
- **Training jobs only.** No inference serving, no autoscaling.
- **All input data is public or free-tier**: published price endpoints, public grid carbon APIs, and our own benchmark runs on modest credits.
- **Advisory mode is a complete, shippable product on its own.** If workstream E slips, we ship the recommender and the dashboard and the project is still a real system that real users use. Execution is an upgrade, not a prerequisite.

**Milestones.**

- **Week 4:** Trace format frozen, benchmark corpus collection running, price and carbon ingestion live.
- **Week 8:** Advisory mode end to end. A BAC or PostGrad user can profile a real job and get a ranked plan. First round of user interviews against a working artifact.
- **Week 11:** Execution on one provider, with checkpoint and resume on preemption.
- **Week 13:** Team dashboards, post-run reports, second round of user feedback incorporated.
- **Week 14:** Hardening and evaluation.

**Evaluation.** Median absolute percentage error of runtime prediction against held-out real runs; realized cost and carbon savings versus each user's pre-Dispatch baseline; number of distinct real users who ran at least three jobs.

**Primary risks.**
1. *Runtime prediction is too inaccurate to be useful.* Mitigation: widen to prediction intervals rather than point estimates, and fall back to a per-user learned correction after the first few runs.
2. *Provider APIs are unstable or rate-limited.* Mitigation: adapter abstraction behind one interface, cached price snapshots, advisory mode degrades gracefully.
3. *Carbon accounting is contested.* Mitigation: publish the methodology in-product, use marginal rather than average intensity where data allows, and validate against operator contacts.

---

## Appendix: open questions to resolve in user interviews

- Do target users actually change behavior for carbon, or only for cost? If the answer is only cost, carbon becomes a reporting feature rather than an optimization objective.
- Will users grant provider API keys to a student project, or is advisory mode the real product?
- Is the binding constraint budget, or is it queue wait time on a shared cluster? These imply different products.

## Appendix: alternate titles

Dispatch (current), Marginal, Watt-Hour, Slack Capacity, Coulomb, Idle.
