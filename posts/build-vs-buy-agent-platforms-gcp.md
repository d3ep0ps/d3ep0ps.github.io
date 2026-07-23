# Build vs Buy: Agent Platforms on GCP

> **"The unit prices are nearly identical. The decision isn't what you pay per hour — it's which hours you pay for, and who you pay when it breaks."**

While you were evaluating agent platforms this spring, the price list changed underneath you. At Google Cloud Next '26, Vertex AI became the **Gemini Enterprise Agent Platform** — same services, new umbrella, no migration. Then the billing changes started landing: Agent Gateway usage became billable on **July 13, 2026**. Sessions and Memory Bank move onto the metered pricing structure on **September 1, 2026**. If your cost model was built on a June invoice, it's already wrong, and it gets wrong again in six weeks.

This article is a decision framework for the question every platform team on GCP is being asked right now: do we run our agents on the managed platform, or build our own runtime? It's written for the person who has to defend the answer — with numbers from Google's own pricing pages, every calculation shown, and the trade-offs stated plainly enough to survive a board meeting.

---

This piece continues the thread from the **AI Agent Security** series and [The Agent Platform Security Assessment](#agent-platform-security-assessment): those articles asked *how do you verify the platform you have*; this one asks *which platform should you have*. The two questions meet at the end — the assessment is how you evaluate either answer.

All prices below are us-central1, on-demand list prices, verified against Google's published pricing pages in July 2026. Links to every source are in the References. Your region, discounts, and negotiated rates will differ; the *structure* of the math won't.

## 1. "Build vs buy" is the wrong question

The binary framing fails on GCP because the agent stack is not one product — it's three layers, and each has its own answer:

**The reasoning framework is free either way.** The Agent Development Kit (ADK) is open source under Apache 2.0. Your agent's logic — tools, instructions, orchestration — is portable code regardless of where it runs. Google's A2A protocol is an open standard under the Linux Foundation. Nobody is selling you the framework layer, so stop evaluating it.

**The runtime is the actual decision.** Three realistic candidates: Agent Engine (the managed runtime inside the Gemini Enterprise Agent Platform), Cloud Run (managed containers, you own the agent plumbing), and GKE with Agent Sandbox (you own everything). This is where the money and the control trade against each other, and it's most of this article.

**The governance plane mostly follows the runtime.** Agent Registry, Agent Gateway, semantic governance policies, and the new IAM agent identity come with the managed platform. On your own runtime you assemble equivalents from IAM, service mesh, and your own registry — [Part 2](#agent-security-part2) of the security series showed what that takes.

So the honest question is not "build or buy" but **"which layer do we want to own, and what does owning it cost us?"** In 2026 the common answer for teams at scale is hybrid — a managed core plus self-hosted components where control or cost demands it. The framework below tells you where your workloads sit.

## 2. The three architectures

| | **Agent Engine** (buy) | **Cloud Run** (middle) | **GKE + Agent Sandbox** (build) |
|---|---|---|---|
| You deploy | ADK/framework code | Containers you build | Containers + cluster config |
| Sessions & memory | Managed (Sessions, Memory Bank) | You build or bring | You build or bring |
| Code-execution sandbox | Managed | You configure | Agent Sandbox (gVisor), you configure |
| Agent identity | IAM agent identity (SPIFFE, cert-bound tokens) | Per-service service accounts | Workload Identity per pod |
| Governance | Agent Registry, Gateway, policies | You assemble | You assemble |
| Isolation boundary owner | Google (you can't inspect it) | Google (per-revision containers) | You, fully |
| Ops load | Minimal | Low | A platform team's job |

One deliberate omission: a Compute Engine VM is not on this list. [Part 1](#agent-security-part1) of the security series covered why a VM with a default service account is where agent security goes to die; the same reasoning removes it from the cost comparison — anything it saves in rate, it spends in blast radius.

## 3. The billing models — which hours you pay for

Here is the fact that reframes the cost conversation. The managed premium on paper is almost zero:

- **Agent Engine:** $0.085 per vCPU-hour, $0.009 per GiB-hour. Billed per second, and **idle time between turns is not billed**. Free tier: 50 vCPU-hours and 100 GiB-hours per month.
- **Cloud Run (request-based):** $0.000024 per vCPU-second active — which is **$0.0864 per vCPU-hour**, within 2% of Agent Engine. Billed only while handling requests; warm min-instances idle at about a tenth of the active rate. 2M requests/month free.
- **GKE Autopilot:** $0.0445 per vCPU-hour and $0.0049 per GiB-hour — roughly **half** the managed rates — plus a flat $0.10/hour (~$73/month) cluster fee. But you're billed on **pod resource requests for as long as pods run**, not on active compute.

That last distinction decides everything. Agent Engine and Cloud Run bill *activity*. Autopilot bills *provisioned capacity*. An agent is a bursty workload — it thinks in short spikes and waits for humans in between. On a platform that bills activity, the waiting is free. On a platform that bills capacity, the waiting is the bill.

Add the two new line items to the managed side: Agent Gateway at 1 vCPU-hour ($0.085) per 15,000 API calls (billable since July 13), and from September 1, Sessions and Memory Bank at $0.30/GiB-month storage plus 1 vCPU-hour per 3 million read operations. Individually small; at high traffic they're the SKUs to watch, because nobody's June cost model contains them.

## 4. Three worked examples

Assumptions stated, arithmetic shown, all rates from the References. "Utilization" means active compute divided by provisioned capacity.

**Small — internal support agent.** 1 vCPU / 2 GiB, 500 requests/day, ~5 seconds of compute each. That's ~21 active vCPU-hours and ~42 GiB-hours a month.

| Architecture | Monthly cost | Why |
|---|---|---|
| Agent Engine | **$0** | Under the free tier (50 vCPU-h / 100 GiB-h) |
| Cloud Run | **$0** | Under the free tier (240k vCPU-s) |
| GKE Autopilot | **~$113** | 1 vCPU + 2 GiB pod provisioned 24/7 ($40) + cluster fee ($73) |

At small scale the comparison isn't close, and it isn't about rates — it's that the managed platforms don't bill waiting and the cluster does. (If this is your billing account's only cluster, the GKE free-tier credit covers the $73; the conclusion stands.)

**Medium — customer-facing agent.** 2 vCPU / 4 GiB per instance, 20,000 requests/day at ~6 seconds each: 2,000 active vCPU-hours, 4,000 GiB-hours a month.

| Architecture | Monthly cost | Why |
|---|---|---|
| Agent Engine | **~$201** | (2,000−50)×$0.085 + (4,000−100)×$0.009 |
| Cloud Run | **~$202** | Same active seconds at $0.000024/vCPU-s; 600k requests — free |
| GKE Autopilot | **~$232–390** | Depends on average provisioned capacity (4–8 vCPU) against a 10-concurrent peak |

Two thousand busy vCPU-hours sounds like the scale where owning the cluster pays. It isn't — because autoscaling never tracks a bursty load perfectly, and every provisioned-but-waiting pod-hour costs you the thing Agent Engine gives away free. At ~50% utilization, the managed platforms are cheaper at *any* scale. That sentence is the whole cost section for most readers.

**Large — steady multi-agent platform.** 40 vCPU / 80 GiB continuously busy, 24/7 — a platform running background agents, not waiting on humans. 29,200 vCPU-hours a month.

| Architecture | Monthly cost | Why |
|---|---|---|
| Agent Engine | **~$3,002** | 29,150×$0.085 + 58,300×$0.009 |
| Cloud Run (instance-based) | **~$2,313** | $0.000018/vCPU-s for instance lifetime |
| GKE Autopilot, on-demand | **~$1,660** | 29,200×$0.0445 + 58,400×$0.0049 + $73 |
| GKE Autopilot, 3-year CUD | **~$946** | Committed-use rates $0.024475 / $0.0027 |
| GKE Autopilot, Spot | **~$548** | Spot rates $0.0133 / $0.0015 — for interruptible agents only |

Here the build case is real: 45% below Agent Engine at list, 69% with commitment, 82% on Spot for workloads that tolerate preemption. The general rule, derivable from the same pages: at 100% utilization the crossover sits near **1,500 vCPU-hours a month** — about two continuously-busy vCPUs. Every point of utilization you lose moves it up; at 50% it never arrives.

Three tidy examples are a framework, not your infrastructure. To run this math against what you actually deploy, I maintain [gcp-cost-estimator](https://github.com/d3ep0ps/gcp-cost-estimator) — an open-source tool that parses your Terraform, consults a cached copy of GCP's published pricing data, and returns an itemized estimate. It runs locally, deterministically, with no LLM inferring rates and no infrastructure definition leaving your machine — the same principle as this article: numbers traceable to a published SKU, or no numbers at all.

**The asterisk that keeps this honest:** none of these numbers include people. A GKE agent platform needs someone to own upgrades, node security, sandbox configuration, and 3 a.m. pages. Even a quarter of a platform engineer's loaded cost is $3–5k a month — which erases the large-scale saving until the fleet is several times this size, and buries it below. The spreadsheet decides nothing until the ops line is on it.

## 5. Control, security, and the exit door

Price is the argument everyone has; control is the one that decides. Run the three architectures through the [three boundaries](https://github.com/d3ep0ps/agent-security-assessment) from the assessment:

**Reach.** On GKE you own every layer: gVisor via Agent Sandbox, default-deny egress, Workload Identity per pod, Model Armor floors — each verifiable with the assessment's commands. On Agent Engine, isolation is managed; the Unit 42 "Double Agents" research covered in [Part 1](#agent-security-part1) showed what it means when the boundary that fails is one you couldn't audit. Buy means trusting a boundary you can't see into. That's not disqualifying — it's a risk you price, and the assessment tells you whether your own boundaries would score better.

**Identity.** This one now favors buy, and recently: IAM agent identity — per-agent SPIFFE principals with certificate-bound tokens that can't be replayed outside their runtime — is an Agent Engine feature. On your own runtime the equivalent is Workload Identity plus a service mesh, which is Configured-tier at best for most teams, while agent identity is Enforced by construction.

**Provenance and the exit door.** The lock-in question deserves a ledger, not a feeling:

| Portable — survives a platform exit | Captive — rebuilt on exit |
|---|---|
| ADK agent code (Apache 2.0, runs anywhere) | Agent Engine runtime config + agent identity |
| A2A protocol (open standard, Linux Foundation) | Sessions (managed service) |
| Agent Sandbox (open source, any Kubernetes cluster) | Memory Bank (managed, including revisions) |
| Container images, MCP servers | Agent Gateway policies, semantic governance |
| | Console observability and evaluation tooling |

The pattern: Google open-sourced the layers you'd write anyway and kept the layers that accumulate state — and state is what compounds. Session history is replaceable. **Agent memory isn't**: from September 1 you'll pay to store it, it grows more valuable every month it operates, and before you commit years of it to Memory Bank, make your team demonstrate the export path — not read about it, run it. If the answer to "how do we leave with our memory" is a meeting, you already have your lock-in answer at Documented tier.

## 6. The decision, compressed

**Buy (Agent Engine)** if your agents are bursty, human-facing, and under ~1,500 busy vCPU-hours a month; if you have no platform team to spare; if agent identity's credential guarantees matter to your auditors. Accept: an isolation boundary you can't inspect, and state accumulating behind a managed API.

**Middle (Cloud Run)** if you want activity-based billing with container control — you already run services there, and your agents are stateless enough to bring your own sessions. Accept: you've taken on the plumbing without gaining the cluster-level control that justifies it. This is a waypoint, not usually a destination.

**Build (GKE + Agent Sandbox)** if you have sustained high-utilization agent compute (several continuously-busy vCPUs and growing), an existing platform team, and workloads where Spot and CUDs apply. Accept: the ops line on the spreadsheet, and sole ownership of every boundary the assessment checks.

**The honesty test, borrowed from every good architecture review:** can your team run this like a production service — upgrades, on-call, capacity, security patching — and say so with a straight face? If yes, build is on the table. If not, the cheapest rate in the table above is the most expensive decision in this article.

And the hybrid default that most mature teams land on: managed for the customer-facing bursty agents, GKE for the steady background fleet, one assessment run against both.

## 7. Before you sign either way

Whichever column you land in, you're about to trust a platform with agents that act on your systems. Two open tools from this blog make the decision inspectable rather than argued: the [Agent Platform Security Assessment](https://github.com/d3ep0ps/agent-security-assessment) — seventeen checks, three boundaries, evidence tiers — tells you what you're actually getting on either side of the build/buy line, and [gcp-cost-estimator](https://github.com/d3ep0ps/gcp-cost-estimator) turns your own Terraform into the cost model this article sketched. Run both, and the build-vs-buy meeting becomes short: a three-digit security score and an itemized estimate, side by side, arguing for you.

That's the standard worth holding any platform decision to — including the ones I get called into. The teams that navigate this well aren't the ones with the cheapest rate in the table; they're the ones who can produce the evidence behind their choice on demand.

## References

**Official pricing (all figures in this article)**

- [Gemini Enterprise Agent Platform pricing](https://cloud.google.com/products/gemini-enterprise-agent-platform/pricing) — Agent Compute/Memory/Storage rates, free tiers, Gateway and Sessions/Memory Bank billing dates
- [Cloud Run pricing](https://cloud.google.com/run/pricing) — request-based and instance-based rates, free tiers
- [GKE pricing](https://cloud.google.com/kubernetes-engine/pricing) — Autopilot pod-based rates, cluster fee, CUDs, Spot
- [Google Cloud pricing calculator](https://cloud.google.com/products/calculator) — model your own workloads
- [gcp-cost-estimator](https://github.com/d3ep0ps/gcp-cost-estimator) — open-source Terraform-to-cost estimation from this blog, built on GCP's published pricing data

**Official documentation**

- [Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform) and the [rebrand announcement](https://cloud.google.com/blog/products/ai-machine-learning/introducing-gemini-enterprise-agent-platform)
- [IAM agent identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity)
- [Memory Bank](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/memory-bank) and [Sessions](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/sessions)
- [GKE Agent Sandbox](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox) and the [open-source announcement](https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate)
- [Agent Development Kit (open source)](https://github.com/google/adk-python) · [A2A protocol](https://a2a-protocol.org)

**Earlier in this series**

- Security as Code: AI Agent Security — Prompt Injection (Part 1), A2A Authentication (Part 2), Model Poisoning (Part 3)
- [The Agent Platform Security Assessment](https://github.com/d3ep0ps/agent-security-assessment)

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. He occasionally helps teams work through exactly this decision; if yours is one of them, he's on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
