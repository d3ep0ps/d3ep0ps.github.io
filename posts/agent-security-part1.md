# Security as Code: AI Agent Security — Prompt Injection and the Blast Radius of Where Your Agents Run (Part 1 of 3)

> **"Sandboxing tells you where an agent can act. IAM tells you what it's allowed to touch. Neither one asks whether the agent should be doing this right now — and that gap is where prompt injection lives."**

Consider a support-triage agent running in a GKE Sandbox pod, isolated by gVisor. Its Workload Identity binding is scoped the way most of these end up scoped in practice — to "the support and billing datasets," because that's what the team who built it eight months ago decided it needed, and nobody has revisited it since. A VPC Service Controls perimeter wraps the project. Model Armor screens every prompt that reaches it.

A customer opens a ticket and pastes in an error log. Buried in that log — text the agent is supposed to read and summarize, the same as any other ticket — is a line instructing it to also compile the full billing history "for cross-reference" and attach the export as a link in the reply. The agent has never been told that dataset is off-limits, because technically it isn't: it's covered by the same grant as everything else this agent touches. The export goes out through a share-link feature the team built for exactly this purpose — handing customers their own invoices.

Sixty thousand other customers' billing records went out in that link. The sandbox held. The identity binding was valid. The perimeter didn't leak. Model Armor didn't see anything resembling an attack, because nothing about the request looked like one.

That's the part worth sitting with before you read any further.

---

In the previous **Security as Code** series, we secured the container supply chain from the developer's laptop to GKE admission control: SBOMs at build time, cryptographic signing, and Binary Authorization enforcing that only attested images ever reach production. That work answers one question well — *can we trust what's running?*

It has nothing to say about a newer problem: what happens when the thing running is a large language model, taking instructions from a prompt, a document, an email, or a tool response that an attacker controls?

This opens a new series on AI-specific security threats — the attack surface that starts where container scanning and Binary Authorization stop. Part 1 covers prompt injection and, specifically, how the runtime you choose for an agent — a VM, Cloud Run, GKE, or a fully managed platform like Vertex AI Agent Engine — changes the blast radius when that injection succeeds. Part 2 will cover agent-to-agent (A2A) authentication. Part 3 will cover model weight and supply-chain poisoning.

---

## 1. Why this is now an infrastructure problem, not a chat problem

In June 2025, researchers disclosed **EchoLeak (CVE-2025-32711)** — a zero-click prompt injection against Microsoft 365 Copilot. A single crafted email, no user interaction required, caused Copilot to read internal files and exfiltrate their contents to an attacker-controlled server. In May 2026, Pillar Security disclosed a CVSS-10 indirect prompt injection in Gemini CLI, delivered through the software supply chain. In between, GitHub Copilot, Cursor, and even Anthropic's own Git MCP server each had documented prompt injection vulnerabilities that led to unauthorized command execution or data exfiltration.

None of these are jailbreak curiosities. They're incidents where an agent's tool access — the same access you granted it to do its job — was redirected by content the agent was never supposed to trust.

**Direct injection** happens when the attacker is the user: they type the malicious instruction straight into the prompt. **Indirect injection** is the more dangerous case for agents: the malicious instruction is embedded in something the agent *reads* — a web page, a PDF, a GitHub issue, a calendar invite, or a tool's own metadata. Since LLMs currently cannot reliably distinguish "instructions from my operator" from "text I was asked to summarize," any content path into the model is a potential command path out of it.

For agents wired into MCP servers, there's a second variant worth naming: **tool poisoning**. The malicious instructions aren't in the data the agent processes — they're hidden in a tool's *description*, injected into the model's context the moment the tool is registered. A tool that looks like `get_weather(city)` can carry an invisible instruction telling the agent to also exfiltrate its conversation history. This is why MCP is the least-understood attack surface in most platform teams' current threat models — it's a new trust boundary that didn't exist a year ago.

## 2. Prompt injection is the symptom, not the disease

Most prompt-injection defenses treat this as a wording problem: better system prompts, input filters, output filters. That framing is incomplete. Hostile content only exploits what the system already made possible — over-broad tool authority, retrieval with no provenance tracking, and execution paths with no boundary between "the agent's identity" and "the identity of whatever it's reading."

This is exactly the same lesson as Binary Authorization, applied one layer up. A policy that can be bypassed is a suggestion. A system prompt that says "never reveal secrets" is a policy. It can be bypassed with the right paragraph of text. What can't be talked out of its job is a scoped IAM binding, a network policy that denies egress by default, and a sandbox the agent's generated code can't escape.

That reframes the real question for this article: not "how do we stop the model from being tricked" — you can't reliably do that yet — but "when the model *is* tricked, what can it actually reach?" The answer depends heavily on where the agent runs.

## 3. Where your agent runs decides its blast radius

Every GCP compute option for hosting an agent trades operational control for convenience, and that trade directly sets the ceiling on how much damage a successful injection can do.

**Compute Engine VM.** No workload-level identity boundary by default. If the agent's service account uses the VM's attached identity rather than a scoped one, and nobody's disabled it, the agent inherits whatever the Compute Engine default service account can touch — often project Editor. No kernel-level isolation for anything the agent executes. This is the worst starting point and, anecdotally, still the most common for teams that "just want the agent running somewhere."

**Cloud Run.** Better — a container per revision, a dedicated (ideally scoped) service account, and no shared node to worry about. But Cloud Run and Cloud Functions are the integration teams most often forget to place *inside* a VPC Service Controls perimeter. If your agent calls out through a Cloud Run service that sits outside the perimeter, that service becomes an escape hatch: a compromised agent routes data through an "allowed" Cloud Run invocation and walks straight past the boundary you thought you'd built.

Closing that gap takes three settings together — no single flag does it. Ingress has to be locked to internal traffic only; leaving it at `all` silently disables VPC-SC enforcement for that service. Egress has to route all outbound traffic through your VPC network — Direct VPC egress (`--vpc-egress=all-traffic`), not the default path. And that VPC's subnets need Private Google Access enabled, with routing set up so calls to Google APIs resolve to the restricted VIP (`199.36.153.4/30`, `restricted.googleapis.com`) instead of the public internet. Skip that last piece and Direct VPC egress alone doesn't bring the service inside the perimeter — it just changes which network the traffic happens to travel through on its way out.

**GKE.** This is where the controls get granular enough to matter, and where three primitives compound:

- **GKE Agent Sandbox**, built on gVisor, gives kernel-level isolation for LLM-generated code execution, with a **default-deny network posture** — a sandboxed agent cannot reach the GKE control plane or any internal service unless you explicitly allow it in the `SandboxTemplate`.
- **Workload Identity Federation** replaces static keys with short-lived, per-pod tokens scoped to a specific Google Cloud identity. On Autopilot clusters this is on by default; on Standard clusters, any node pool where it *isn't* enabled falls back to the node's service account — typically the Compute Engine default service account with Editor-level access. One misconfigured node pool undoes the whole model.
- **VPC Service Controls** wraps a perimeter around the services your agent is allowed to reach — BigQuery, Cloud Storage, Vertex AI, Secret Manager.

The trap here isn't any single layer — it's assuming these three add up to full protection. They don't, for one structural reason worth stating plainly: **IAM checks the permission, not the pattern.** If your agent's Workload Identity binding grants `roles/bigquery.dataViewer` on a dataset, the short-lived token authorizes a routine 500-row query and a prompt-injection-driven full-table export identically. Binary Authorization could refuse to run an unattested image outright, because the check is binary. A scoped IAM role can't refuse an *allowed* action just because the agent's intent behind it changed — that requires behavior, not policy.

**Vertex AI Agent Engine / Gemini Enterprise Agent Platform (fully managed).** You give up direct control over the sandbox and the network boundary in exchange for Google managing them. That's a legitimate trade for many teams — but "managed" moves the isolation boundary out of your hands, not out of existence. In March 2026, Unit 42 published research ("Double Agents") showing that credentials obtained from within an AI agent's managed execution context could be used to pivot out of that context into the customer's own project, gaining unrestricted read access to every Cloud Storage bucket in it. The isolation between the platform's shared execution environment and the customer's project was the thing that failed — a boundary the customer had no way to audit or harden themselves, because it wasn't theirs to configure.

```text
+--------------------------------------------+------------------------------------------------+----------------------------------------------------------+------------------------------------------------------------+
| Runtime                                    | Isolation for agent-generated code             | Identity scoping                                         | Who owns the boundary                                      |
+--------------------------------------------+------------------------------------------------+----------------------------------------------------------+------------------------------------------------------------+
| Compute Engine VM                          | None by default                                | Often the VM's own SA (broad)                            | You, but usually unmanaged                                 |
| Cloud Run                                  | Per-revision container                         | Per-service SA, easy to scope                            | You — but easy to leave outside VPC-SC                     |
| GKE (Autopilot)                            | gVisor via Agent Sandbox, default-deny network | Workload Identity Federation, on by default              | You, fully                                                 |
| GKE (Standard)                             | gVisor via Agent Sandbox (opt-in)              | Workload Identity Federation (must verify per node pool) | You, if configured correctly everywhere                    |
| Vertex AI Agent Engine / Gemini Enterprise | Managed sandbox                                | Managed, plus your IAM bindings                          | Shared with Google — you can't inspect the tenant boundary |
+--------------------------------------------+------------------------------------------------+----------------------------------------------------------+------------------------------------------------------------+
```

The practical takeaway: GKE gives you the most levers, but every lever has to actually be pulled. A managed platform pulls them for you, but you're trusting a boundary you can't see into.

## 4. Model Armor: the gateway that screens what the boundary lets through

None of the isolation and IAM controls above inspect *content*. That's Model Armor's job — a Google Cloud service that screens prompts and responses for prompt injection, jailbreak attempts, sensitive data, and malicious URLs, sitting inline between the caller and the model.

Model Armor is configured through **templates** (per-application filter sets) or **floor settings** (an org- or project-level minimum baseline that every template must meet — the same "establish a floor, let teams add on top" pattern Binary Authorization uses for image attestation). Each filter category — prompt injection and jailbreak detection, responsible-AI categories, sensitive data protection, malicious URLs — has a confidence threshold: `High` for production traffic that can't tolerate false positives, `Low and above` when missing a real attack is worse than an occasional false block. Google's own guidance is to start prompt-injection detection at `Medium`, and go to `High` only if false positives on legitimate prompts become a problem.

Creating a template is a single `gcloud` call:

```bash
gcloud model-armor templates create agent-input-template \
  --project=my-project --location=us-central1 \
  --rai-settings-filters='[
    { "filterType": "HATE_SPEECH", "confidenceLevel": "MEDIUM_AND_ABOVE" },
    { "filterType": "HARASSMENT", "confidenceLevel": "MEDIUM_AND_ABOVE" }
  ]' \
  --pi-and-jailbreak-filter-settings-enforcement=enabled \
  --pi-and-jailbreak-filter-settings-confidence-level=MEDIUM_AND_ABOVE
```

As with our Terraform-managed Binary Authorization policy, this belongs in code, not a console click:

```hcl
# terraform/model-armor.tf
resource "google_model_armor_template" "agent_input" {
  location    = "us-central1"
  template_id = "agent-input-template"

  filter_config {
    pi_and_jailbreak_filter_settings {
      filter_enforcement = "ENABLED"
      confidence_level    = "MEDIUM_AND_ABOVE"
    }
    sdp_settings {
      basic_config {
        filter_enforcement = "ENABLED"
      }
    }
  }

  template_metadata {
    log_sanitize_operations = true
  }
}

# Org-wide floor: every project's templates must meet this minimum,
# the same "policy that can't be bypassed" idea as Binary Authorization's
# cluster-wide enforcement.
resource "google_model_armor_floorsetting" "org_floor" {
  parent   = "organizations/${var.org_id}"
  location = "global"

  filter_config {
    pi_and_jailbreak_filter_settings {
      filter_enforcement = "ENABLED"
      confidence_level    = "MEDIUM_AND_ABOVE"
    }
  }

  enable_floor_setting_enforcement = true

  # Governs enforcement for Vertex AI / Gemini traffic. Start here, not at
  # inspect_and_block: inspect_only and inspect_and_block are independent
  # fields, not one switch, and turning on org-wide blocking before you've
  # measured your false-positive rate will break benign workloads on day one.
  ai_platform_floor_setting {
    inspect_only         = true
    enable_cloud_logging = true
  }
}
```

Run that in `inspect_only` for a couple of weeks, watch Cloud Logging for how often legitimate traffic trips the filter, tune the confidence level if needed, and only then flip `inspect_only` to `false` and `inspect_and_block` to `true`. This is the same staged rollout as the `Inspect only` → `Inspect and block` progression described above — the org-wide floor setting isn't exempt from it just because it's the baseline everyone inherits.

Where Model Armor actually intercepts traffic depends on your architecture, and this matters for the blast-radius discussion above:

- **Direct Vertex AI/Gemini integration** — inline enforcement on `generateContent` calls, no code changes needed, but Gemini models only.
- **Agent Gateway** (part of Gemini Enterprise Agent Platform) — a control-plane proxy with two distinct enforcement points: **Client-to-Agent (ingress)**, screening what reaches the agent and what it says back, and **Agent-to-Anywhere (egress)**, screening what the agent sends to external LLMs, other agents, or MCP servers, and what comes back. Egress screening is the one that matters for indirect injection and tool poisoning — it's the choke point between your agent and the untrusted content it retrieves.
- **GKE via Service Extensions** — a service extension on your load balancer or GKE Inference Gateway, for agents you're hosting yourself rather than running through Gemini Enterprise.
- **MCP servers** (Preview) — floor settings applied specifically to Google and Google Cloud MCP tool calls and responses.

One limitation worth flagging so it doesn't surprise you later: Model Armor's Gemini Enterprise integration sanitizes the *initial* prompt and the *final* response. Intermediate steps — the tool calls and results in between — aren't covered by that particular integration path; that's what the Agent Gateway egress screening and GKE Service Extensions paths are for.

## 5. ADK identity, and wiring it to IAM instead of bolting security on after

The Agent Development Kit treats security as part of the framework, not a filter you add later. Its model has three layers that map cleanly onto everything above:

1. **Identity and authorization.** ADK tools act with one of two identities: **agent-auth**, a service account belonging to the agent itself, or **user-auth**, the identity of whoever is driving the agent. This single decision is the practical form of "IAM scoping" for agents — it determines whose blast radius applies when a tool call goes wrong. The **Agent Identity Auth Manager** handles the OAuth lifecycle for user-auth flows: storing credential configs, minting and storing tokens, and logging every access, so you're not rolling your own token storage.
2. **Guardrails.** Model Armor plugs in here, along with ADK callbacks that can validate a tool call before or after execution. Google's own guidance suggests a fast, cheap model (Gemini Flash Lite) as a pre-screening layer in front of the primary agent — catching obviously malicious input before it costs a full inference call.
3. **Sandboxed execution.** This is where Agent Sandbox on GKE, or Agent Engine's managed sandbox, does the work described in Section 3.

The IAM mistake that erases all of this is the same one teams make with Binary Authorization: granting the broad role because it's the fast path. Google's own codelab for deploying agents on GKE grants `roles/aiplatform.user` at the **project** level — which lets the agent call inference on every endpoint in the project and enumerate every model. For a single-purpose agent, scope it down:

```text
+-----------------------------+---------------------------------------------+---------------------------------------------------+
| Agent needs                 | Overprivileged (project-level)              | Correctly scoped (resource-level)                 |
+-----------------------------+---------------------------------------------+---------------------------------------------------+
| Call one inference endpoint | `roles/aiplatform.user` on the project      | `roles/aiplatform.user` on that specific endpoint |
| Read one BigQuery dataset   | `roles/bigquery.dataViewer` on the project  | `roles/bigquery.dataViewer` on that dataset       |
| Read one bucket             | `roles/storage.objectViewer` on the project | `roles/storage.objectViewer` on that bucket       |
+-----------------------------+---------------------------------------------+---------------------------------------------------+
```

```hcl
# terraform/agent-identity.tf
resource "google_service_account" "invoice_agent" {
  account_id   = "invoice-agent"
  display_name = "Invoice summarization agent (agent-auth)"
}

# Resource-level, not project-level.
resource "google_bigquery_dataset_iam_member" "agent_dataset_read" {
  dataset_id = google_bigquery_dataset.invoices.dataset_id
  role       = "roles/bigquery.dataViewer"
  member     = "serviceAccount:${google_service_account.invoice_agent.email}"
}

# Workload Identity Federation binding — GKE pod to this SA, nothing broader.
resource "google_service_account_iam_member" "wif_binding" {
  service_account_id = google_service_account.invoice_agent.name
  role                = "roles/iam.workloadIdentityUser"
  member              = "serviceAccount:${var.project_id}.svc.id.goog[agents/invoice-agent]"
}
```

Combined with a `SandboxTemplate` that default-denies egress except to the specific BigQuery and Vertex AI endpoints this agent needs, and a Model Armor floor setting inspecting everything that crosses the Agent Gateway, you've built the same kind of layered gate as Binary Authorization: no single control has to be perfect, because a failure in one is caught by the boundary of the next.

## 6. What this doesn't solve

Be honest with yourself about the residual gap: even with every control in this article correctly wired — scoped Workload Identity, a default-deny sandbox, VPC-SC, Model Armor on ingress and egress — none of it can distinguish an agent's normal 500-row query from a prompt-injection-driven export of the entire table, if both are technically permitted. That's a behavioral question, not a policy question, and it's the reason runtime monitoring for agent workloads is becoming its own discipline rather than an afterthought. It's also exactly why the architecture in this article matters regardless: the smaller the set of "technically permitted" actions, the smaller that residual gap gets.

## What's next

Part 2 turns to a different trust boundary: agent-to-agent (A2A) authentication, and what happens when the thing your agent is talking to isn't a tool you configured, but another agent — possibly one you don't control at all.
