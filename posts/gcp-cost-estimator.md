# The Price Tag: Knowing What Your Infrastructure Costs Before You Deploy

> "An estimate without a source is just a guess with confidence."

---

You write Terraform. You review the plan. You apply. A month later, you open the billing dashboard and discover that a handful of Cloud SQL replicas and a misconfigured GKE node pool just cost you three times what you expected.

This is not a cloud problem. It is an information problem. The pricing data exists — GCP publishes every SKU, every rate, every billing rule. What doesn't exist, at least not in a form your tooling can use, is a way to go from *infrastructure-as-code* to *cost-as-code* before you commit.

Building on the principles of infrastructure automation and security explored in previous chapters of d3ep0ps, that is what I've been building.

---

## The problem with the existing tools

GCP's pricing calculator is honest and thorough. It is also entirely manual. You open a browser, select a service, fill in a form, and get a number. There is no API call from your CI pipeline. There is no way to hand it your `main.tf` and ask "what will this cost?"

There are third-party tools that attempt this, but they are either SaaS products that require uploading your infrastructure definition to a stranger's server, or they are wrappers around the pricing calculator that still require manual input.

What I wanted was something different: a tool that runs locally, parses Terraform, consults a cached copy of GCP's live pricing data, and returns an itemised estimate — deterministically, without sending my infrastructure anywhere.

---

## Why an MCP server, not an agent

The first design question was whether to build an *agent* or a *tool*.

An agent would contain a language model. It would read your Terraform, reason about it, infer costs, and give you an answer. This sounds appealing until you think about it for thirty seconds. An agent that infers costs is an agent that might hallucinate costs. SKU IDs, billing units, and per-region rates are exactly the kind of factual data that language models confidently get wrong.

I chose to build an **MCP server** instead — a deterministic server that exposes pricing, IaC parsing, cost calculation, and reporting as [MCP](https://modelcontextprotocol.io/) tools, resources, and prompts. There is no language model inside the server. Every tool call returns a deterministic result given the same input and the same pricing cache snapshot. The intelligence — natural language understanding, explanation, summarisation — is supplied by whatever MCP host connects to it: Claude Code, Gemini CLI, Cursor, or any other compliant host.

This is not an AI product. It is a pricing engine that AI products can use.

The division of labour is clean:

```
User ──► MCP Host (LLM lives here) ──► GCP Cost Estimator MCP Server (no LLM)
                                              │
                                     deterministic core
                                              │
                                      SQLite pricing cache
                                              │
                                   GCP Cloud Billing Pricing API
```

The LLM reads your natural-language description or Terraform plan and produces a structured resource model. The server validates that model, maps it to SKUs, prices it against the cache, and returns an itemised estimate. Every line item carries a real SKU ID and the cache snapshot timestamp. Nothing is invented.

---

## Architecture in depth

The server is built around three principles that kept the design honest.

**Library-first.** All business logic lives in a transport-agnostic core library (`core/`). The MCP server, an optional HTTP/SSE service, and a CLI adapter are thin wrappers that call core — they contain no pricing logic. This means the same calculation runs identically whether you call it from Claude Code, from a CI script, or from the command line.

**Deterministic core.** No randomness, no network calls, no language models on the hot path. Given the same resource model and the same cache snapshot, the server always returns the same estimate. This is the property that makes it testable and trustworthy.

**Fail loud, never under-report.** Resources that cannot be priced — because a SKU is missing from the cache, or because a Terraform variable is unresolved — appear in an explicit `unpriced[]` list in the estimate output. They are never silently dropped or zero-filled. A zero-filled estimate is worse than no estimate because it looks like an answer.

### The pricing cache

The server maintains a local SQLite database populated from GCP's [Cloud Billing Pricing API](https://cloud.google.com/billing/v1/pricing). The cache refreshes every 72 hours or on demand, using an atomic snapshot swap so the live cache is never in a partially-written state. Estimates succeed even when the GCP Pricing API is unavailable.

Every SKU row in the cache carries: provider, SKU ID, service, region, description, unit, unit price, SKU group, and snapshot timestamp. Prices are list prices only — no sustained use discounts, no committed use discounts, no negotiated rates. The disclaimer is always attached to the estimate output.

### IaC parsing

The parser handles Terraform in two modes. When a `terraform show -json` plan output is available, it uses that — all variables are resolved and the parser gets exact values. When only static HCL is available (no `terraform init`, no provider auth required), it falls back to static parsing with `python-hcl2`. Unresolved variable references are flagged explicitly rather than silently assumed.

Both parsers sit behind an `IaCParser` registry interface. Pulumi and CloudFormation can be added without touching shared code.

---

## What it can do today

The server covers all five core GCP services — Tier 1 of the coverage roadmap, representing roughly 60% of typical GCP spend. All five are parsed from Terraform (both static HCL and `terraform show -json` plan output), mapped to real SKU IDs from the pricing cache, and tested against hand-computed golden fixtures.

**Compute Engine** — virtual machines with any machine family (`n2-standard-4`, `e2-micro`, `c3d-highcpu-16`, custom configurations). vCPU and RAM are priced from the standard SKU catalogue. Attached persistent disks (SSD and standard) are separate line items. The machine type resolver uses a rule-based approach rather than a static lookup table — it derives `(vcpu, ram_gb)` from GCP's naming conventions, so new machine families are automatically handled without a code change.

**Cloud Storage** — four billing axes in one estimate: storage by class (Standard, Nearline, Coldline, Archive), Class A write operations, Class B read operations, internet egress, and retrieval fees for cold storage classes. Prices vary by location type (region, dual-region, multi-region) and are resolved per-region from the cache.

**Google Kubernetes Engine** — two independent billing components: the flat $0.10/cluster/hour cluster management fee, plus node compute priced through the same Compute Engine mapper. A `google_container_cluster` emits the management fee plus its default node pool; each `google_container_node_pool` is priced as a group of GCE instances. The cluster fee is always present even when no machine type is specified.

**Cloud SQL** — full coverage across all editions (Enterprise and Enterprise Plus) and all database engines (MySQL, PostgreSQL, SQL Server). High availability (`REGIONAL` availability type) correctly doubles the compute cost without doubling the storage. Custom tiers (`db-custom-2-7680`) and standard tiers (`db-n1-standard-4`) both resolve through the same rule engine used for Compute Engine.

**BigQuery** — active and long-term storage (different rates for data modified vs. untouched in 90+ days), on-demand query bytes, and legacy streaming inserts. All four dimensions are independent line items. On-demand pricing only — slot commitments require reservation quantities that are not statically determinable from Terraform.

A combined estimate across all five services is possible in a single tool call. A typical single-service output looks like this:

```json
{
  "currency": "USD",
  "pricing_snapshot": "2026-06-01T00:00:00Z",
  "disclaimer": "List price only. SUD/CUD/negotiated discounts NOT applied.",
  "line_items": [
    {
      "resource_id": "web-server",
      "component": "vcpu",
      "sku_id": "6F81-5844-456A",
      "unit_price": 0.031611,
      "unit": "hour",
      "qty": 2920.0,
      "monthly_cost": 92.30
    },
    {
      "resource_id": "web-server",
      "component": "ram",
      "sku_id": "E075-B1AE-A8E5",
      "unit_price": 0.004237,
      "unit": "gib_hour",
      "qty": 11680.0,
      "monthly_cost": 49.49
    },
    {
      "resource_id": "primary-db",
      "component": "vcpu",
      "sku_id": "...",
      "unit_price": 0.0413,
      "unit": "hour",
      "qty": 730.0,
      "monthly_cost": 30.15
    }
  ],
  "monthly_total": 237.42,
  "unpriced": [],
  "assumptions": [
    "runtime defaulted to 730h/month — override usage.runtime_hours_per_month for non-24/7 workloads"
  ]
}
```

Every line item traces to a real SKU ID. The snapshot timestamp tells you exactly when the prices were last refreshed. The assumptions list is transparent about every value the server filled in on your behalf.

---

## The assumption problem

Here is something I had to think carefully about.

When you describe infrastructure before building it, you often don't know the sizing. You know you want a GKE cluster but not exactly how much traffic it will serve. You know you want a Cloud Storage bucket but not how many reads per day your users will generate.

A naive implementation defaults unknown usage fields to zero. This produces an estimate that looks cheap — a Cloud Storage bucket with zero operations and zero egress costs only the storage itself. But that's not what you'll pay. You will definitely generate read operations. You will probably send some traffic to the internet.

The server applies *representative defaults* instead. A bucket defaults to 10,000 Class A operations per month (about 300 writes per day) and 100,000 Class B operations (about 3,000 reads per day). A GKE cluster defaults to 3 × `e2-standard-4` nodes. BigQuery defaults to 100 GB active storage and 1 TB of queries per month. These defaults reflect a plausible small-to-medium workload.

Every default is recorded in the estimate's `assumptions[]` array with its value and an explicit override hint. The `catalog://defaults` MCP resource publishes the complete defaults catalog so the host LLM can read and communicate it to the user before generating a resource model.

A zero-default estimate is not honest. It just looks honest.

---

## Coming next

With Tier 1 complete, the next target is Tier 2 — serverless and containers, which covers roughly another 15% of typical GCP spend:

- **Cloud Run** — billed per vCPU-second, memory-second, and request. The pricing model is usage-driven rather than capacity-driven, which makes representative defaults especially important.
- **Cloud Functions** — per-invocation plus compute time. Both 1st and 2nd gen are in scope.
- **App Engine** — standard and flexible environments have different billing models; standard is instance-hours, flexible is essentially GCE-based.

After Tier 2, Tier 3 covers the remaining databases: Spanner (processing units + storage), Firestore (reads/writes/deletes/storage), Memorystore, Bigtable, and AlloyDB.

Known simplifications are tracked explicitly in the implementation plans and surfaced as `unpriced[]` items in estimates — things like GKE Autopilot (different per-pod pricing model), BigQuery slot reservations, and Cloud Storage lifecycle transition fees. The goal is always a useful working estimate over a perfect but incomplete one. The 90% target covers 23 services across 6 tiers.

---

## The design philosophy in one sentence

The MCP server does the deterministic work — SKU lookups, unit conversions, pricing arithmetic — and borrows the intelligence of whatever agent host connects to it for everything else.

This is what "depth over hype" looks like for AI tooling. Not a model that guesses your cloud bill. A pricing engine with a well-defined interface that an AI can use correctly, and that you can test, trust, and run entirely on your own machine.

The project is on GitHub at [github.com/d3ep0ps/gcp-cost-estimator](https://github.com/d3ep0ps/gcp-cost-estimator). Tier 1 — Compute Engine, Cloud Storage, GKE, Cloud SQL, and BigQuery — is complete and tested. Tier 2 serverless coverage is next.

---

*Vitaliy Zhhuta is a system architect writing about Linux, cloud, and AI at [d3ep0ps.com](https://d3ep0ps.com).*
