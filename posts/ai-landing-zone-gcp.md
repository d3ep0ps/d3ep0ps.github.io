# Security as Code: The AI Landing Zone — The Floor Under Every Agent on GCP

> **"Every agent you deploy inherits the floor it lands on. You can harden an agent in a sprint. You cannot harden a floor after a hundred agents are already standing on it."**

Over the last few months this series has worked from the top of the agent stack downwards. The **AI Agent Security** trilogy named the threats: [prompt injection](#agent-security-part1), [unverified agent-to-agent trust](#agent-security-part2), and [poisoned weights](#agent-security-part3). [The Agent Platform Security Assessment](#agent-platform-security-assessment) turned them into seventeen checks you can score. [The AI Gateway](#ai-gateway-enforcement) showed how to move those checks from *Documented* to *Enforced* in the traffic path. [Build vs Buy](#build-vs-buy-agent-platforms-gcp) asked which runtime to run them on.

All four articles quietly assumed something: that underneath the agents there is a Google Cloud organization whose policies, network, keys, and identities are already built for them. In most organizations I've looked at, there isn't. The landing zone was designed in 2021 for web apps and data pipelines, and the agents are being deployed onto it as if they were one more web app.

This article covers that missing layer: the **AI landing zone**. It is based on an architecture and governance blueprint I've been developing for exactly this problem — Gemini Enterprise, Agent Platform runtimes, custom MCP servers and multi-agent systems on GCP — and validated against Google's documentation as of October 2026.

The executive summary below is for readers who approve budgets and risk. Sections 1 to 11 are for the people who build the platform.

---

## Executive Summary: The Business Case for the Floor

**The gap is between pilot and production.** Cisco's *Agent Trust Gap* report (March 2026) found that 85% of organizations are experimenting with, piloting, or deploying AI agents, while only 5% have moved them into broad production. Nearly 60% of security leaders named security concerns as the biggest obstacle to wider adoption. Gartner's 2026 Hype Cycle for AI Governance places *agentic AI governance* and *AI agent identity* two to five years from mainstream adoption.

In most cases the pilots work. What is missing is a basis on which Security can approve production.

**Why agents are hard to approve.** Conventional software follows paths its developers wrote and reviewed. An agent chooses its next action at runtime, using credentials you granted it, based partly on text you didn't write: a retrieved document, a support ticket, a tool response. A security leader cannot approve that on the strength of a system prompt or a policy document. They can approve it when the worst outcomes are blocked by the platform itself, independently of how any individual agent behaves.

### What the AI landing zone provides

| Concern | What the landing zone does | What leadership gets |
|---|---|---|
| **Approval at scale** | Security approves the platform once; new agents inherit its controls | Each new agent passes a pipeline check instead of a bespoke security review |
| **Blast radius** | Agents, tools, and data live in separate projects; access is granted per dataset, not per project | A compromised agent reaches the data it was granted, not the wider estate |
| **Data residency and key custody** | Storage restricted to approved locations (the EU, in this blueprint); customer-managed encryption keys held by Security | Auditable evidence supporting GDPR and data-residency commitments |
| **Accountability** | Per-agent identities replace shared service accounts | Actions in the audit log trace to an individual agent, not to a shared account |
| **Evidence** | Every control is paired with a negative test that fails if the control is missing | Auditors and risk committees review test results, not assurances on slides |
| **Cost containment** | Token ceilings per agent, enforced at the gateway; spend-cap budgets per project as a circuit breaker | A looping or flooded agent hits a hard limit, not the monthly invoice |

**The cost of not having it.** Without a shared foundation, each team builds its own controls in its own way. Security review becomes the bottleneck for every launch, unsanctioned MCP servers appear in projects nobody is watching, and risk accumulates without being measured. The landing zone moves the security decision from each agent to the platform, where it is made once and verified continuously.

**The cost of having it.** The floor has a running cost, and most of it is fixed. Secure Web Proxy, for example, is listed at $1.25 per gateway-hour, about $910 a month for each always-on gateway, before data processing and TLS-inspection certificates. Price it against one stalled production launch, not against zero. Spending needs a ceiling too, and it has two layers. A standard Cloud Billing budget only alerts. A spend-cap budget can pause the Agent Platform API for a whole project, but not instantly, and it stops every agent in that project at once. It is a circuit breaker, not a control. The precise limit sits in the traffic path: a token ceiling per agent identity, enforced at the gateway.

**Regulation.** For EU organizations, the same evidence base serves the EU AI Act. Human confirmation on state-changing tools, tamper-proof logs of agent decisions, and recorded test results correspond to the Act's record-keeping (Article 12), human-oversight (Article 14), and post-market monitoring (Article 72) obligations. The landing zone doesn't make a system compliant; classification and conformity assessment are legal questions. It produces the technical evidence those processes ask for. Since the Digital Omnibus on AI entered into force on July 27, 2026, high-risk obligations apply from December 2, 2027 (Annex III) and August 2, 2028 (Annex I). That is enough time to build the floor, and not enough to start late.

**Who runs it.** The floor is operated, not just built. Platform Engineering builds and runs it; Information Security is accountable for every guardrail and every exemption; Network Engineering owns perimeters and egress; an identity lead owns agent identities. Per-agent onboarding is automated in CI, and human time goes to a fixed cadence: a weekly review of blocks and denials, a monthly model-lifecycle and exemption review, a quarterly access recertification and red-team exercise. That is a named responsibility for existing teams rather than a new department. If it isn't in someone's objectives, the model allowlist and the exemption register go stale within a quarter.

**Rollout is gated, not scheduled.** Controls are introduced in seven phases, each closed by exit criteria that Security and Platform leads sign off (section 9). Every guardrail runs in dry-run mode before it is enforced, so violations in existing workloads surface as findings rather than outages.

---

## 1. Why your current landing zone doesn't cover agents

A conventional landing zone — Google's Security Foundations blueprint, Cloud Foundation Fabric FAST, or your own variant — solves the problems of 2021 well: resource hierarchy, Shared VPC, centralized logging, no public IPs, no default service accounts. Keep all of it. It's the floor this article builds on.

What it doesn't solve is the four properties that make agents different from the workloads it was designed for:

1. **Instructions and data share one channel.** A model cannot reliably distinguish your system prompt from instructions hidden in a retrieved support ticket. Nothing at the network layer can see that difference. (OWASP ASI01, Agent Goal Hijack — the subject of [Part 1](#agent-security-part1).)
2. **IAM checks permission, not intent.** IAM answers *"may this principal read this table?"* It cannot answer *"should it, right now, because a document told it to?"* An agent running as an omnibus service account is a confused deputy with the keys to everything that account touches. (ASI02, Tool Misuse.)
3. **The counterparty is unverified by default.** Agent cards are self-declared; MCP servers are installed from the internet; tool descriptions are executable text. (ASI04, ASI07 — [Part 2](#agent-security-part2).)
4. **The payload can be the weapon.** Container scanners find CVEs in binaries. They do not find a backdoor encoded in model weights, or a poisoned document in a RAG corpus. (ASI04, ASI06 — [Part 3](#agent-security-part3).)

None of these is solved inside the agent. All four can be *contained* by the platform the agent runs on. That is the design principle of everything that follows:

```text
Guardrails live in the platform, not in the prompt.
If a control can be skipped by a misbehaving agent, it is a guideline, not a control.
```

## 2. The shape: four layers, one direction of trust

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ ORGANIZATION      org policies: no SA keys, invoker IAM, CMEK, EU-only,  │
│                   custom MCP connectors denied by default                │
├──────────────────────────────────────────────────────────────────────────┤
│ FOLDERS           fldr-shared-infra │ fldr-ai-platform │ fldr-ai-workloads│
│                   (net, KMS, logs)  │ (registry, Gemini│  sandbox → dev → │
│                                     │  Enterprise hub) │  staging → prod  │
├──────────────────────────────────────────────────────────────────────────┤
│ NETWORK           VPC-SC perimeters · PSC-I into Shared VPC ·            │
│                   Secure Web Proxy for all egress · no direct internet   │
├──────────────────────────────────────────────────────────────────────────┤
│ RUNTIME           per-agent identity · Agent Gateway + Model Armor ·     │
│                   Agent Registry · Binary Authorization                  │
└──────────────────────────────────────────────────────────────────────────┘
        Each layer assumes the one above it has already failed.
```

The order matters. Organization policies are the only controls that hold even when a project owner makes a mistake. The network holds when an identity is stolen. The runtime layer holds when the network path is legitimate but the content is malicious. Defense in depth is not a slogan here; it's the order of construction.

## 3. Hierarchy: blast radius is a folder decision

The resource hierarchy is where you decide, permanently and cheaply, what can be compromised together.

```text
organizations/ORG_ID
├── fldr-shared-infra
│   ├── prj-net-host-prod        Shared VPC, PSC network attachments, Secure Web Proxy
│   ├── prj-sec-kms-prod         all CMEK keys, Certificate Authority Service
│   └── prj-sec-logging          locked log buckets, SIEM sinks
├── fldr-ai-platform
│   ├── prj-ai-registry-shared   Agent Registry, attested Artifact Registry
│   └── prj-gemini-ent-hub       Gemini Enterprise apps, data stores, connectors
└── fldr-ai-workloads
    ├── fldr-ai-sandbox          open-weights experiments, isolated VPC, no prod data
    ├── fldr-ai-dev              synthetic data
    ├── fldr-ai-staging          mirrors prod allowlist, safety regression evals
    └── fldr-ai-prod
        ├── prj-agent-runtime-prod   Agent Platform runtime / orchestrators
        ├── prj-mcp-servers-prod     MCP tools agents call (internal ingress only)
        ├── prj-mcp-gemini-prod      MCP tools Gemini Enterprise calls (the one ingress exception)
        └── prj-ai-data-prod         Vector Search, BigQuery corpora, grounding buckets
```

Four decisions in this tree carry most of the weight:

- **Agents, tools, and data are three projects, not one.** The runtime identity reads data through explicit dataset- and bucket-level bindings; it never holds a project-wide data role. If an agent is hijacked, the attacker inherits a narrow, listable set of grants.
- **Gemini Enterprise gets its own hub project.** As section 5 shows, connecting it to custom tools requires *relaxing* an organization-wide control. You want exactly one place in the organization where that relaxation exists, with compensating controls around it.
- **Tools that Gemini Enterprise calls live apart from tools that agents call.** Gemini Enterprise reaches MCP servers from outside your network, which forces an ingress exception (section 5). Isolating those services in their own project keeps the exception off every other tool, and keeps `prj-mcp-servers-prod` under full perimeter protection.
- **Keys and logs belong to Security, not to the platform.** The team that can deploy an agent cannot also delete the audit trail of what it did, or destroy the key that protects its data. That is segregation of duties expressed as project boundaries.

## 4. Organization policy: the controls that can't be skipped

Organization policies cost almost nothing to run, and a project owner cannot override them. The blueprint uses 34 constraints across three scopes; the ones that matter most for agents are below.

| Constraint | Scope | What it makes impossible |
|---|---|---|
| `iam.managed.disableServiceAccountKeyCreation` / `…KeyUpload` | Org | Long-lived JSON keys — the credential an exfiltrating agent would love most |
| `run.managed.requireInvokerIam` | Org | Unauthenticated (`allUsers`) Cloud Run services — every MCP tool needs a signed identity to call |
| `run.managed.disableInlinedSource` | Org | Deploying from source uploads instead of a scanned registry image |
| `discoveryengine.managed.disableCustomMcpServerConnector` | Org | Shadow MCP connectors in Gemini Enterprise (Google ships this enforced by default) |
| `gcp.restrictNonCmekServices` + `gcp.restrictCmekCryptoKeyProjects` | Org | Unencrypted AI, storage, database, and secret resources; keys outside the Security project |
| `gcp.resourceLocations` | Org | Data stored outside the EU boundary |
| `vertexai.allowedModels` | Prod folder | Calling, tuning, or deploying any model not on the allowlist |
| `vertexai.genAIGroundingSources` + `disableGenAIGoogleSearchGrounding` | Prod folder | Grounding production answers on the open web |
| `run.allowedBinaryAuthorizationPolicies` = `default` | Workloads folder | Cloud Run services without Binary Authorization |
| `run.allowedIngress` = `internal…` | Workloads folder | Internet-reachable agents and tools (one documented exception — see §5) |

Two rules make this survivable in a real organization.

**Dry-run first, always.** Every constraint is staged in `dryRunSpec`, violations are reviewed over an agreed window, and only then is the identical rule promoted to `spec`:

```yaml
# Stage 1: log violations, block nothing
name: organizations/ORG_ID/policies/run.managed.requireInvokerIam
dryRunSpec:
  rules:
  - enforce: true
---
# Stage 2: after the review window, the same rule becomes enforcement
name: organizations/ORG_ID/policies/run.managed.requireInvokerIam
spec:
  rules:
  - enforce: true
```

**Every constraint gets a negative test.** "Enforced" means you tried the forbidden thing (created a service account key, deployed a public service, called an unlisted model) and the platform refused. The refusal, captured in the pipeline log, is the evidence. A screenshot of a policy page is not.

### The model allowlist is a process, not a config file

`vertexai.allowedModels` is the constraint executives understand fastest — *"production can only call models Security approved"* — and the one teams get wrong fastest, because it goes stale.

Consider the last twelve weeks. According to Google's official model lifecycle schedule, Gemini 2.5 Pro, Flash, and Flash-Lite reach official retirement on Vertex AI and Agent Platform on **October 20, 2026** (today). In that same window, Google recorded releases for Gemini 3.6 Flash and 3.5 Flash-Lite (July 21), 3.7 Flash (August 13), 3.8 Flash (September 2), and Gemini 4 Argon (October 13) — while Gemini 3.6 Flash is already scheduled for retirement next month (November 19, 2026). An allowlist written in July is wrong twice by late October: it permits models that shut down today, and it blocks the GA successors your teams need.

So the allowlist rules are about lifecycle, not IDs:

```yaml
# folders/PROD_AI_FOLDER_ID — illustrative; IDs come from the monthly lifecycle review
name: folders/PROD_AI_FOLDER_ID/policies/vertexai.allowedModels
spec:
  rules:
  - values:
      allowedValues:
      - publishers/google/models/<GA_FLASH_MODEL_ID>:predict
      - publishers/google/models/<GA_EMBEDDING_MODEL_ID>:predict
      - publishers/anthropic/models/<APPROVED_PARTNER_MODEL_ID>:predict
```

- Production allows **`:predict` only** — no `:tune`, no one-click `:deploy`.
- **GA model IDs only** — never `-preview`, never `-latest` aliases. A floating alias is an unreviewed model change.
- A successor enters **staging first**, passes the safety regression suite there, then moves to production.
- The allowlist is reviewed **monthly** against the model lifecycle page, with a KPI: *zero production workloads within 30 days of a model's retirement date.*
- Partner models need a second constraint, `vertexai.allowedPartnerModelFeatures`, before features like structured outputs work. Approving the model alone is not enough.

## 5. The Gemini Enterprise + custom MCP trap

This is the section I'd ask any platform team to read twice, because it's where a well-built landing zone and a business requirement collide — and where the documentation is spread across five pages.

The requirement: *"Let employees use Gemini Enterprise to query and act on our internal systems through our own MCP servers."* The landing zone, correctly, forbids every part of that by default.

### Three constraints that only work together

Custom MCP connectors are denied by a Google-managed constraint. Unlocking them in the hub project means configuring **three** policies together — relaxing one and tightening two:

```yaml
# 1. Relax: allow custom MCP — in this one project only
name: projects/PRJ_GEMINI_HUB/policies/discoveryengine.managed.disableCustomMcpServerConnector
spec:
  rules:
  - enforce: false
---
# 2. Tighten: which connector types may exist at all
name: projects/PRJ_GEMINI_HUB/policies/discoveryengine.managed.allowedDataSources
spec:
  rules:
  - enforce: true
    parameters:
      allowedDataSources: [custom_mcp, sharepoint, jira, servicenow]
      enforcedProjects: [projects/PRJ_GEMINI_HUB_NUMBER]
---
# 3. Tighten: exactly which hosts connectors may call
name: projects/PRJ_GEMINI_HUB/policies/discoveryengine.managed.allowedEgressFqdns
spec:
  rules:
  - enforce: true
    parameters:
      allowedEgressFqdns:
      - mcp-orders-PRJ_MCP_GEMINI_NUMBER.europe-west4.run.app   # the Gemini-facing MCP server
      - auth.corp.example.com                             # OAuth authorization
      - token.corp.example.com                            # OAuth token
      enforcedProjects: [projects/PRJ_GEMINI_HUB_NUMBER]
```

A subtlety worth knowing: the two tightening policies are only evaluated for projects inside a VPC Service Controls perimeter or listed in `enforcedProjects`. Outside both, relaxing constraint 1 alone opens the door with nothing behind it. That's why the hub project is both in the production perimeter *and* listed explicitly.

### The network path is not the one you drew

Gemini Enterprise calls your MCP server **from Google-managed infrastructure, not from your Shared VPC**. That has three consequences that surprise people:

1. **Your `run.allowedIngress = internal` policy blocks it.** The project hosting these services needs a project-level override to `all`. That is why they live in `prj-mcp-gemini-prod` and nowhere else: every tool that only agents call stays in `prj-mcp-servers-prod`, where the `internal` policy is never relaxed.
2. **The endpoint needs a public name and a public CA certificate.** Self-signed certificates are rejected, and `*.internal` names are unreachable. Internal names stay valid for agent-to-tool traffic inside the VPC; they just can't be used here.
3. **Use the default `*.run.app` URL, not your custom domain.** Gemini Enterprise sends a Google-signed ID token in `X-Serverless-Authorization` (alongside the user's OAuth token in `Authorization`) — but only to default `run.app` URLs. Put the service behind `mcp.corp.example.com` and the ID token silently disappears, and with it your Cloud Run IAM check.

**And you can't fix this with a VPC Service Controls ingress rule.** This is the consequence most designs get wrong. For Cloud Run invocations, VPC-SC doesn't evaluate the caller's IAM identity, so ingress rules that name a principal (the Discovery Engine service agent, for example) aren't supported; only `ANY_IDENTITY` is. A service with ingress set to `all` is not protected by the perimeter at the invocation layer at all. Say this plainly in the design review: Gemini-facing services sit outside perimeter enforcement for inbound calls, and their security rests on the compensating controls below. That is acceptable only because they are isolated in their own project, kept small, and reviewed as an exception.

The compensating controls are all mandatory:

- `run.managed.requireInvokerIam` stays enforced: no `allUsers`, ever;
- `roles/run.invoker` goes to the Discovery Engine service agent and named agent identities, nothing else;
- the service URL is on the `allowedEgressFqdns` list;
- every tool call is screened by Model Armor;
- the exception is in the exemption register, with a note to revert it if Google documents a private path.

### Human approval is one annotation away from disappearing

Gemini Enterprise treats every MCP tool as potentially destructive: tools import **disabled**, and each call requires user confirmation. Tool owners can remove the prompt for read-only tools:

```python
@mcp.tool(annotations={"readOnlyHint": True, "destructiveHint": False})
def query_order_status(order_id: str) -> dict:
    """Read-only order lookup."""
    return db.fetch_order(order_id)
```

That's correct and necessary — confirmation fatigue is a security problem too. But note who controls it: **the tool's developer**, in code, with one boolean. A `readOnlyHint: True` on a tool that writes removes the human from the loop without anyone in Security seeing it.

So the landing zone adds a control that the published assessment doesn't have yet. I'm proposing it as the eighteenth check. A label can be wrong, so the check doesn't rely on the label alone:

```text
R7 (Reach) — State-changing tools require human confirmation,
             and read-only tools cannot write.                       OWASP ASI09
Enforced when:
1. CI rejects any MCP tool without readOnlyHint: true that does not
   declare destructiveHint: true.
2. Any change to readOnlyHint requires Agent Registry re-approval.
3. Read-only and write tools run as separate MCP services with separate
   identities. The read-only identity holds reader roles only, so a
   mislabelled write fails at the data layer.
4. A red-team test confirms that every write-capable tool prompts the user.
```

Where Gemini Enterprise passes the end user's OAuth token, the downstream system also enforces that user's own permissions. The tool's identity is the second limit, not the only one.

## 6. Network: perimeters, private paths, inspected egress

The network layer's job is simple to state: **no agent reaches the internet, or another project's data, by a path nobody inspects.**

```text
Agent Platform runtime (Google tenant project)
        │  PSC interface — RFC 1918 traffic only
        ▼
Shared VPC (prj-net-host-prod)
   ├── internal MCP tools (Cloud Run --ingress=internal, GKE default-deny NetworkPolicy)
   └── Secure Web Proxy ── FQDN allowlist, TLS inspection
             └── Cloud NAT ── approved external APIs only
```

**VPC Service Controls** wraps the AI APIs — `aiplatform`, `discoveryengine`, `modelarmor`, storage, BigQuery, AlloyDB, Cloud SQL, Run, Artifact Registry, Secret Manager, KMS — in three perimeters: sandbox, non-production, production. Cross-perimeter rules name specific identities and methods, never `*`. Agent identities can be used directly as principals in ingress and egress rules (a single agent, or a fleet via `principalSet://…/attribute.…`), though not in access levels.

**Private Service Connect interfaces** bring the managed runtime into your VPC for private traffic. What they don't do is control the runtime's internet access. Outside a perimeter, an Agent Platform runtime keeps its default internet access, and that traffic leaves Google's tenant network directly. It never touches your VPC, your firewall, or your proxy. The control that closes this path is VPC Service Controls: once the runtime's project is inside a perimeter, the tenant's default internet access is blocked, and the only way out is an explicit proxy reached over the PSC interface. That is why the perimeter is enforced in Phase 3, before any production agent is deployed.

**Secure Web Proxy** is that way out, and the `HTTP(S)_PROXY` variable that points agents at it is a convenience, not a control. Well-behaved HTTP clients honour it; code an attacker controls doesn't have to. Enforcement lives elsewhere, and it differs by runtime:

- **GKE and Cloud Run:** egress firewall rules deny everything except the proxy and the private Google APIs range (`restricted.googleapis.com`); Cloud NAT serves the proxy only; Cloud Run services are forced into the VPC by `run.allowedVPCEgress = all-traffic`. Code that ignores the proxy variable gets dropped packets.
- **Agent Platform runtime:** the perimeter, as above. No firewall rule in your VPC can stop traffic that never enters it.

Both are verified the same way: by trying to get around them (section 8).

## 7. Identity: one agent, one principal, one dataset

The first thing to look for in any agent review is still the oldest anti-pattern in cloud security: several agents sharing one service account with `Editor` or a broad data role. Compromise any agent and you've compromised all of them — and your audit log can't tell you which one did it.

The landing zone standard:

- **Agents on Agent Platform runtime, Gemini Enterprise, and Cloud Run** get native **IAM agent identity** — a per-agent SPIFFE principal, provisioned by the platform:

  ```text
  principal://agents.global.org-ORG_ID.system.id.goog/resources/aiplatform/
      projects/PROJECT_NUMBER/locations/LOCATION/reasoningEngines/ENGINE_ID
  ```

  Credentials are certificate-bound: agents authenticate to Google APIs over mTLS under a Google-managed Context-Aware Access policy, and traffic through Agent Gateway adds DPoP token binding. A token stolen from the runtime doesn't work outside it.
- **Agents on GKE** don't receive agent identity. Each gets its own Google service account through Workload Identity Federation for GKE — one agent, one account, no exceptions.
- **No user-managed keys exist anywhere** — the org policy in section 4 makes sure of it.

Agent-to-agent delegation is the place least-privilege usually breaks, because `roles/aiplatform.user` at project level is the path of least resistance. The landing zone grants a parent agent exactly one permission on exactly one sub-agent, as code:

```hcl
resource "google_organization_iam_custom_role" "agent_query_only" {
  org_id      = "ORG_ID"
  role_id     = "reasoningEngineQueryOnly"
  title       = "Reasoning Engine Query Only"
  permissions = ["aiplatform.reasoningEngines.query"]
}

resource "google_vertex_ai_reasoning_engine_iam_member" "parent_queries_subagent" {
  project          = "prj-agent-runtime-prod"
  region           = "europe-west4"
  reasoning_engine = "SUBAGENT_ENGINE_ID"
  role             = google_organization_iam_custom_role.agent_query_only.name
  member           = "principal://agents.global.org-ORG_ID.system.id.goog/resources/aiplatform/projects/PRJ_RUNTIME_NUMBER/locations/europe-west4/reasoningEngines/PARENT_ENGINE_ID"
}
```

Discovery goes through the private **Agent Registry** only, and — because the A2A specification makes card signing optional — the landing zone makes it mandatory: the calling side rejects unsigned agent cards, or cards signed by keys outside the corporate root of trust, before any task is delegated.

On top of identity sits the runtime governance plane covered in the [AI Gateway article](#ai-gateway-enforcement): **Agent Gateway** with **Model Armor** inspecting prompts on the way in and tool calls, tool responses, and A2A messages on the way out, with org-level **floor settings** so no team can deploy a weaker template than Security's minimum. Agent Gateway, Agent Registry, and Model Armor for Agent Gateway have all been GA since June 2026; this is no longer a preview-only architecture.

## 8. Proving it: from "Configured" to "Enforced"

A landing zone diagram is a claim. The assessment's evidence tiers are how you turn it into a fact:

- **Documented** — the policy exists in a document.
- **Configured** — the setting exists, but nothing stops someone working around it.
- **Enforced** — working around it fails a deployment, a pipeline, or a connection.

The [AI Gateway article](#ai-gateway-enforcement) introduced the **Gateway Bypass Test**. The landing zone version extends it in two ways.

**It covers managed runtimes.** On GKE, the probes run inside the agent pod via `kubectl exec`. On Agent Platform runtime there is no pod to exec into, so the same probes ship as a diagnostic tool deployed *with* the agent, reporting to the locked log bucket.

**It checks every way a grant can arrive.** An agent with no inference role on its project can still inherit one from a folder, through a Google group it belongs to, or through a fleet binding that matches every agent in a trust domain. A probe that reads one IAM policy and matches the agent's exact member string misses all three, and reports PASS. The landing zone version asks the question closer to how IAM evaluates it: Cloud Asset Inventory's policy analyzer, with group expansion, across the full ancestry, for the inference permissions specifically. A separate scan looks for `principalSet://` bindings on the agent trust domain, because one fleet grant applies to every agent at once. A PASS then means no path to direct model access was found, not merely that one binding is absent.

Combined with the direct-egress and proxied-egress probes, and a scan for credential-shaped environment variables, that's the minimum evidence for the Reach boundary. The full blueprint maps all eighteen checks (the published seventeen plus R7) to a verification command each.

## 9. Rollout: gates, not dates

The hardest part of a landing zone isn't the Terraform. It's enforcing guardrails on an organization that already has workloads, without breaking them. That's why the rollout is **gate-driven**: each phase ends with exit criteria that Security and Platform leads sign off, and the next phase starts only when the gate passes. Calendar dates belong in the project plan, where they can move without anyone pretending the controls moved with them.

| Phase | What happens | Gate exit criteria |
|---|---|---|
| **0 — Discovery** | Find every project with AI APIs enabled, every public Cloud Run service, every user-managed SA key | 100% of AI assets inventoried with owners; remediation plan committed |
| **1 — Baseline guardrails** | Org policies in dry run → resolve violations → enforce | All baseline policies in `spec`; a negative test recorded for each |
| **2 — Hierarchy & model allowlist** | Folders per environment; prod allowlist; Cloud Run/GKE/Workbench hardening | Prod cannot call unvetted models or run `:tune`; sandbox works in isolation |
| **3 — Perimeter & private egress** | VPC-SC in dry run → enforce sandbox → non-prod → prod; PSC-I; Secure Web Proxy | Zero unexplained VPC-SC violations; test agent cannot exfiltrate or reach the internet |
| **4 — Agent governance plane** | Agent Registry, agent identities, access boundaries, Agent Gateway | Gateway denies unregistered tools; 100% of prod agents on per-agent identity; zero shared SAs |
| **5 — Gemini Enterprise & MCP** | CMEK, three interlocking constraints, ingress exception in its own project, tool annotations | Approved tools work; unapproved FQDNs blocked; document ACLs verified |
| **6 — Content safety & go-live** | Model Armor floors and templates; locked logging; red team | Indirect injection, tool escalation, SSRF, and cross-tenant attacks all blocked and alerted; CISO sign-off |

Phase 0 is the one teams want to skip and shouldn't. You can't enforce guardrails on workloads you don't know exist, and the agents the platform team knows about are rarely the whole list. Cloud Asset Inventory answers the three questions that matter across the whole organization: which projects have AI APIs enabled, which services anyone on the internet can invoke, and which service account keys exist. The answers usually reshape the rest of the plan.

## 10. Nine things that will bite you

These are the details that don't appear on architecture diagrams and do appear in incident reviews. Every one is documented by Google — just never on the same page.

1. **CMEK for Gemini Enterprise must be registered before the first data store.** CMEK is supported only for `us` and `eu` multi-region apps and data stores, not `global`. Resources created before the key is registered stay Google-encrypted, and an existing app or connector can't switch keys later. Register the key (it's manageable in Terraform via `google_discovery_engine_cmek_config`) and grant the Discovery Engine service agent `cryptoKeyEncrypterDecrypter` *first*.
2. **VPC Service Controls must be enforced before the first data store, too.** Google doesn't support enforcing a perimeter on a Gemini Enterprise project that already has data stores; the fix is delete and recreate. Phase 3 comes before Phase 5 for this reason alone.
3. **A single-region location policy breaks Gemini Enterprise.** Its apps live in the `global`, `us`, or `eu` multi-regions, and some resources — Model Armor floor settings among them — exist only at `global`. Allow `in:eu-locations` plus `global`. And remember: `gcp.resourceLocations` governs where resources are *stored*, not which inference endpoint answers. Inference residency is governed by the model allowlist and endpoint choice.
4. **`restrictNonCmekServices` will break things you didn't know created buckets.** Cloud Build's default source and log buckets are the classic case. Pre-create CMEK buckets and point the tooling at them; where that's impossible, a project-scoped exemption for CI/CD projects only — registered, reviewed quarterly, never for projects holding AI data.
5. **A custom domain silently drops Cloud Run's ID token** for Gemini Enterprise calls (section 5). The connector "works" in testing with relaxed IAM and fails, or worse, ships without the check.
6. **Agent Gateway's VPC-SC enforcement applies only to gateways created after September 8, 2026**, and requires agent connectivity templates in `ALL_TRAFFIC` mode. A gateway built in the summer pilot doesn't get it retroactively — plan to recreate it.
7. **The managed agent runtime ignores your egress firewall.** Outside a VPC-SC perimeter, Agent Platform runtime reaches the internet directly from Google's tenant network. Your firewall and proxy govern only the traffic that enters your VPC. The perimeter is what closes the direct path (section 6).
8. **You can't let Gemini Enterprise into a perimeter by identity.** Cloud Run invocations don't support VPC-SC ingress rules that name IAM principals. Gemini-facing services run with ingress `all`, outside perimeter enforcement for inbound calls, so they belong in their own project with their own controls (section 5).
9. **Memory Bank and Sessions on the global endpoint can't use CMEK.** If your encryption policy is non-negotiable, the agent's memory must use a multi-regional or regional endpoint — a decision to make before memory holds a year of customer conversations.

## 11. What you measure

Executives don't need the constraint list. They need seven numbers that tell them whether the floor is holding:

| KPI | Target |
|---|---|
| Policy-as-code coverage (org policies, perimeters, access policies in Terraform/GitOps) | 100%, exceptions in a register |
| User-managed service account keys | 0 |
| Production agents on per-agent identity | 100% |
| Gateway routes to unregistered MCP endpoints | 0 |
| Production workloads within 30 days of model retirement | 0 |
| Production agents without a token ceiling at the gateway | 0 |
| Mean time to triage high-severity Model Armor / VPC-SC alerts | < 60 minutes |

And a cadence that keeps them honest: weekly review of block and denial rates; monthly model-allowlist and new-constraint reviews (Google ships new managed constraints for AI services regularly); quarterly agent-access recertification and red-team exercise; twice-yearly restore drill on CMEK-encrypted data. Once a month, put the floor's own running cost (proxies, NAT, keys) next to the token spend it is protecting.

## Conclusion: approve the floor once

In organizations stuck between pilot and production, the pattern is usually the same. Each agent arrives at the security review as a fresh negotiation: what can it reach, whose credentials does it hold, where does its data go, who approves its tools. The review takes weeks, the answers differ team to team, and the backlog grows faster than the reviewers.

An AI landing zone changes the question. Security approves the *platform* once — its policies, perimeters, identities, and gates, each backed by a test that fails when the control is missing. After that, a new agent doesn't need a new argument. It needs to land on the floor and pass the same assessment as everything else.

That's the difference between an organization that has AI agents and one that can run them.

**Score your own floor.** The [Agent Platform Security Assessment](https://github.com/d3ep0ps/agent-security-assessment) is open on GitHub. Run it against your landing zone — not against an agent — and see how many of your checks are *Enforced* by the platform rather than by the people who wrote the prompts.

---

### Postscript: Closing the Section 5 Ingress Exception with Agent Gateway

Since publishing, the question I’ve heard most is about the trap in section 5. Gemini Enterprise calls custom MCP servers from Google’s infrastructure, not from your VPC. So a tool that should be internal-only has to accept public ingress, which takes it outside VPC Service Controls enforcement for inbound calls.

The article’s answer is containment: Gemini-facing tools in a dedicated project, Cloud Run invoker access only for the Discovery Engine service agent, Model Armor on every call, and separate identities for read and write tools. It works, but it remains a documented exception.

Google now documents a way out. Gemini Enterprise can route custom MCP calls outbound through Agent Gateway instead of reaching your service from the internet:

```text
Gemini Enterprise → Agent Gateway → private network attachment into your VPC → Cloud Run MCP server with internal ingress
```

Only tools registered in Agent Registry are reachable, and IAP authorizes each tool call individually. The service never needs public ingress, so the exception disappears. Google’s codelab walks through the full setup: [Gemini Enterprise with Agent Gateway egress to a private custom MCP server](https://codelabs.developers.google.com/agw-ge-custom-mcp-egress-vpc-registry).

Limits to check before you adopt it:

- **Regions:** currently `us-central1` and `europe-west1` only.
- **Certificates:** MCP endpoints still need a publicly trusted certificate, even over the private path.
- **Perimeter:** the codelab doesn’t configure VPC Service Controls, so validate the combination in your own environment.
- **Maturity:** no launch stage is stated; confirm general availability before production use.

Where these limits rule it out, the containment pattern in section 5 remains the fallback.

---

### Next Up

The landing zone is a design. The next article turns it into code: a Terraform module set for the AI landing zone on top of Cloud Foundation Fabric, with each of the negative tests from this article wired in as a CI gate.

## References

**Google Cloud documentation**

- [Organization Policy constraints reference](https://docs.cloud.google.com/organization-policy/reference/org-policy-constraints)
- [Control access to Model Garden models](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/control-model-access) and [model versions and lifecycle](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions)
- [Gemini Enterprise Agent Platform release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes)
- [Override the organization policy for Custom MCP data stores](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/custom-mcp-server/override-constraint-for-custom-mcp-data-stores)
- [Configure allowed data sources](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/configure-allowed-data-sources) and [allowed egress FQDNs](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/configure-allowed-egress-fqdns)
- [Set up a custom MCP server in Gemini Enterprise](https://docs.cloud.google.com/gemini/enterprise/docs/connectors/custom-mcp-server/set-up-custom-mcp-server)
- [Gemini Enterprise with Agent Gateway egress to a private custom MCP server (codelab)](https://codelabs.developers.google.com/agw-ge-custom-mcp-egress-vpc-registry)
- [Gemini Enterprise CMEK](https://docs.cloud.google.com/gemini/enterprise/docs/cmek) and [VPC Service Controls for Gemini Enterprise](https://docs.cloud.google.com/gemini/enterprise/docs/use-vpc-service-controls)
- [IAM agent identity overview](https://docs.cloud.google.com/iam/docs/agent-identity-overview) and [Context-Aware Access for agents](https://docs.cloud.google.com/access-context-manager/docs/caa-agent-security)
- [VPC Service Controls: supported identities](https://docs.cloud.google.com/vpc-service-controls/docs/supported-identities)
- [Model Armor floor settings](https://docs.cloud.google.com/model-armor/configure-floor-settings)
- [Require Binary Authorization for Cloud Run](https://docs.cloud.google.com/binary-authorization/docs/run/requiring-binauthz-cloud-run)
- [Cloud Run: using VPC Service Controls](https://docs.cloud.google.com/run/docs/securing/using-vpc-service-controls)
- [Private Service Connect interface for Agent Engine](https://docs.cloud.google.com/agent-builder/agent-engine/private-service-connect-interface) and the [Agent Engine PSC explicit proxy codelab](https://codelabs.developers.google.com/agent-engine-psc-interface-private)
- [Secure Web Proxy pricing](https://cloud.google.com/secure-web-proxy/pricing)
- [Cloud Billing: budgets](https://docs.cloud.google.com/billing/docs/how-to/budgets) and [spend cap budgets](https://docs.cloud.google.com/billing/docs/how-to/budgets-spend-caps)
- [Apigee LLMTokenQuota policy](https://docs.cloud.google.com/apigee/docs/api-platform/reference/policies/llm-token-quota-policy)
- [Cloud Asset Inventory: analyze IAM policies](https://docs.cloud.google.com/asset-inventory/docs/analyzing-iam-policy)
- [Architecture Framework: use AI securely and responsibly](https://docs.cloud.google.com/architecture/framework/security/use-ai-securely-and-responsibly)
- [Cloud Foundation Fabric](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric)

**Standards and research**

- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
- [CISA and partners: guidance on adopting agentic AI services](https://content.govdelivery.com/accounts/USDHSCISA/bulletins/41544ff)
- Cisco, *The Agent Trust Gap* (March 2026), via [Enterprise DNA](https://enterprisedna.co/resources/news/cisco-agent-trust-gap-enterprise-ai-production-2026)
- Gartner, *Hype Cycle for AI Governance, 2026*, via [Software Strategies Blog](https://softwarestrategiesblog.com/2026/08/10/gartners-2026-ai-governance-hype-cycle-and-what-cisos-must-change-to-control-agentic-ai-and-rogue-agents/)
- Hunton Andrews Kurth, [EU Digital Omnibus on AI enters into force](https://www.hunton.com/privacy-and-cybersecurity-law-blog/eu-digital-omnibus-on-ai-enters-into-force)
- [A2A protocol](https://a2a-protocol.org) · [Model Context Protocol](https://modelcontextprotocol.io/)

**Earlier in this series**

- [AI Agent Security — Part 1: Prompt Injection](#agent-security-part1)
- [AI Agent Security — Part 2: A2A Authentication](#agent-security-part2)
- [AI Agent Security — Part 3: Model Poisoning](#agent-security-part3)
- [The Agent Platform Security Assessment](#agent-platform-security-assessment) and its [checklist repo](https://github.com/d3ep0ps/agent-security-assessment)
- [The AI Gateway — Where Agent Security Policy Becomes Enforcement](#ai-gateway-enforcement)
- [Build vs Buy: Agent Platforms on GCP](#build-vs-buy-agent-platforms-gcp)

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. He designs landing zones like this one for organizations moving agents from pilot to production; if yours is one of them, he's on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
