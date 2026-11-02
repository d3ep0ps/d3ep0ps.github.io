# Validating an AI Landing Zone on GCP: What Passed, What Failed, What I Couldn't Test

> **"A prescribed negative test is a test plan, not a test result."** — a reviewer of the original article

In [The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp), I described an architecture for running AI agents on Google Cloud: organization policies, folder boundaries, VPC Service Controls perimeters, customer-managed encryption keys (CMEK), per-agent identities and an Agent Gateway. A reviewer raised two fair objections:

1. **A prescribed negative test is a test plan, not a test result.** Without a reference environment, executed tests, raw output and stated limitations, a list of controls is an assertion.
2. **"Approve the floor once" does not mean "approve the agent once."** Each agent's grants and exceptions still need review in its own deployment change.

Both points are right. So I built the landing zone in a reference Google Cloud organization and tried to break it: 38 tests, nine runs between October 23 and October 30, 2026, each test's expected result written into the script before it ran. Failures, inconclusive results and errata are published next to the passes.

The short version: **the floor held where I could provoke it.** Organization policies, the perimeter, CMEK and location rules, the model allowlist, Binary Authorization, locked logs, Model Armor floors and proxied egress refused what they should refuse. Five results are worth more than the passes, because they change how you build:

1. Every agent in a project can call models by default, and the grant outlives the agents.
2. Agent Gateway enforced its policies only after three non-default settings, one of which gives up token-theft protection.
3. Model Armor floors stopped weaker configurations at folder and organization level.
4. Controlled egress for agents works through a PSC interface and Secure Web Proxy; the firewall half could not be shown.
5. A perimeter changes how you run Terraform itself.

---

## 1. The setup and the evidence standard

**Environment.** One small test organization. Policies at folder scope (`fldr-lz-validation` and a child `fldr-ai-prod`), except two tests that set organization-level policies in a test-only organization. Region `europe-west1`, with model calls routed to `global` (more on that in section 7). Infrastructure from [Cloud Foundation Fabric](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric) modules, pinned to the commit that includes the AI organization policies I contributed upstream ([PR #4288](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4288)).

```text
organization (ORG_ID)
└── fldr-lz-validation           folder policies under test; perimeter lzv_ai
    ├── prj-lzv-mcp              in perimeter (Cloud Run MCP fixtures)
    ├── prj-lzv-data             in perimeter (buckets, locked log bucket)
    ├── prj-lzv-outside          OUTSIDE the perimeter (exfiltration target)
    └── fldr-ai-prod             model allowlist
        └── prj-lzv-runtime2     in perimeter (Agent Runtime agents, Agent Gateway)
```

**Rounds.**

| Round | Tests | Runs | What it covered |
|---|---|---|---|
| v1 | 17 (T01–T17) + T14R | 5 (runs 1, 2, 6, 7; T14R in 3, 6, 7) | Organization policies, perimeter, identity, logs |
| v1.1 | 9 (T16, T18–T25) | 4 (runs 4–7) | Fixes for what v1 found: deny policy, CMEK build path, deploy order, log sink, image admission, egress proxy |
| v2 | 12 (T27–T39) | 2 (runs 6, 7) | Model Armor floors, Agent Gateway, PSC interface egress, organization scope, quota projects |
| Gateway re-runs | T30–T32, T31R | 2 (runs 8, 9) | Agent Gateway with VPC egress |

**Verdicts and evidence tiers.** Each test records PASS, FAIL, INCONCLUSIVE or NOT_TESTED, and a tier:
- **Enforced:** the forbidden action was attempted against the live API and refused.
- **Configured:** the effective setting was read back, but no forbidden action was attempted.

Every test writes its commands, raw output and a result file, redacted before commit. An independent review checked scripts, timestamps and outputs against the claims. Where a script had a bug or a recorded tier didn't fit the definition, the raw evidence stayed untouched and the correction went into published errata. Changes made after a run (an amended expected text, a fixed matcher, an added attempt) are listed as deviations.

All repeats ran on the same environment, by the same operator. That shows repeatability here, not in other organizations.

---

## 2. Results at a glance

| Run | Scope | PASS | FAIL | INCONCLUSIVE | NOT_TESTED |
|---|---|---|---|---|---|
| 1, 2 | v1 (17 tests) | 15 | 1 (T12) | 0 | 1 (T16, deferred) |
| 3 | T14R re-test (1 test) | 0 | 0 | 1 (T14R) | 0 |
| 4 | v1.1 (9 tests) | 8 | 0 | 1 (T22) | 0 |
| 5 | v1.1 (9 tests) | 8 | 1 (T22) | 0 | 0 |
| 6, 7 | all 38 tests | 30 | 3 (T12, T22, T29) | 5 (T14R, T30, T31, T32, T34) | 0 |
| 8 | gateway tests | 2 (T30, T32) | 0 | 1 (T31) | 0 |
| 9 | gateway tests | 3 (T30, T31, T32) | 0 | 1 (T31R) | 0 |

Runs 6 and 7 gave the same verdict for every one of the 38 tests. The full table, with every qualification, is in [`results.md`](https://github.com/d3ep0ps/ai-landing-zone-validation/blob/main/results.md).

What each FAIL means:
- **T12** failed against my blueprint's assumption, not because of a platform defect (section 3).
- **T22** shows a real operational limit (section 7).
- **T29** failed only on the wording of a block message; the block itself worked (section 5).

---

## 3. Every agent can call models by default, and the grant outlives the agents

My blueprint assumed an agent deployed without an explicit inference role has no direct path to call models. `T12` deployed an agent whose Terraform granted zero roles, then ran Policy Analyzer. It found an inference path in all four runs.

This is documented behaviour. Agent identities receive the Agent Default Access role (`roles/aiplatform.agentDefaultAccess`), which [Google describes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/build/runtime/setup) as including "basic project-wide logging and model calling permissions". An agent needs it to call Gemini at all. What the documentation doesn't spell out, and the tests showed, is the scope:

- **It's project-wide.** The binding is made on the project's agent principal set, so every agent deployed in the project inherits it:
  ```text
  principalSet://agents.global.org-ORG_ID.system.id.goog/attribute.platformContainer/aiplatform/projects/PROJECT_NUMBER
  ```
  In run 4, `T19` compared allow policies before and after one agent deploy: this was the only binding added.
- **It outlives the agents.** In runs 5, 6 and 7 the binding was already on the runtime project before deployment, after the earlier agents had been deleted, and the next deploy added nothing. (How long it stays wasn't tested.)

Two controls bound this default:
- **Which models:** the `vertexai.allowedModels` organization policy. `T04` refused an unlisted model (`gemini-4-argon`, HTTP 400 naming the constraint) and allowed the listed one.
- **Which agents:** an IAM deny policy. Google's [Agent Identity documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity) recommends deny policies or Principal Access Boundary policies to limit what agents can reach. Principal Access Boundary policies work on whole principal sets, so they can't single out one agent within a project. A deny policy can:

```hcl
resource "google_iam_deny_policy" "agents_predict" {
  provider     = google-beta
  parent       = urlencode("cloudresourcemanager.googleapis.com/projects/${var.runtime_project_id}")
  name         = "deny-agent-predict"
  display_name = "Deny model calls to project agents except approved"
  rules {
    deny_rule {
      denied_principals = [
        "principalSet://agents.global.org-${var.org_id}.system.id.goog/attribute.platformContainer/aiplatform/projects/${var.runtime_project_number}"
      ]
      exception_principals = [
        "principal://agents.global.org-${var.org_id}.system.id.goog/resources/aiplatform/projects/${var.runtime_project_number}/locations/europe-west1/reasoningEngines/${var.approved_agent_id}"
      ]
      denied_permissions = ["aiplatform.googleapis.com/endpoints.predict"]
    }
  }
}
```

**Result (`T18`, four runs; three of them with the perimeter enforced):** before the policy, both agents called `gemini-3.5-flash`. After it propagated, the non-approved agent was refused with `Permission 'aiplatform.endpoints.predict' denied ... due to an IAM deny policy`, and the approved agent's identical call succeeded. **The same policy attached at folder level (`T37`, two runs) behaved the same way.**

Two practical notes:
- Creating a deny policy needs `roles/iam.denyAdmin`, which Security Admin doesn't include.
- Redeploying an agent gives it a new identity, so the exception list must be updated by the pipeline.

Any agent outside the exception list loses direct model access, so apply this deliberately, not as a default. This is also what "approve the floor once" means in practice: Security approves the deny rule once, and each agent's exception is a reviewed, diffable change in its own deployment.

---

## 4. Agent Gateway enforced its policies only after three non-default settings

The Agent Gateway tests failed to produce a readable result in runs 6 and 7. They passed in runs 8 and 9, after three changes. Each change is a finding in its own right.

**1. Certificate-bound tokens versus TLS interception.** By default, an agent's tokens are [bound to its certificate](https://docs.cloud.google.com/iam/docs/agent-identity-overview) to prevent token theft. The gateway intercepts TLS to inspect traffic. In run 6, the gateway authorised the agent's calls, and Vertex AI then rejected them with `401 UNAUTHENTICATED`. Registering more endpoints and redeploying the agent didn't change that.
- Google's [Context-Aware Access documentation](https://docs.cloud.google.com/access-context-manager/docs/caa-agent-security) says that because the gateway terminates mTLS, DPoP token binding must be used instead.
- Google's own gateway deployment samples set `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False`, the documented opt-out from certificate-bound tokens. The agent identity page calls that opt-out "strongly discouraged, as it leaves an agent vulnerable to credential theft."
- An [open GitHub issue](https://github.com/googleapis/python-aiplatform/issues/7209) reports the same 401 in another region.

With the opt-out, the 401 disappeared. I changed three things at once (the opt-out, pinned agent packages, the newer IAP policy version), so which one removed the 401 wasn't isolated.

**Practical consequence:** in my tests, putting agents behind the gateway meant giving up certificate-bound tokens. Weigh that trade-off explicitly; don't discover it in production.

**2. The gateway needs VPC egress to reach internal-only tools.** The landing zone allows only internal Cloud Run ingress (`run.allowedIngress`). In run 7, the gateway allowed the call to the registered MCP server, and Cloud Run's front end answered `404`; no request reached the service. Relaxing ingress was, correctly, refused by the folder policy. With VPC egress for the gateway (a network attachment, a PSC endpoint for Google APIs and a private `run.app` DNS zone peered to the gateway), the gateway reached the internal-only service in runs 8 and 9.

**3. IAP policy version 2.** The gateway's authorization extension requires `metadata.iapPolicyVersion`. Google recommends V2 (Unified Access Policies); I used a project access policy that allows egress only to registered destinations.

**What the gateway then did:**

| Test | What was attempted | Result |
|---|---|---|
| `T30` (runs 8, 9) | Agent calls a registered MCP tool, then an unregistered one | Registered tool answered; unregistered call refused at the gateway (403, `DENIED` in the gateway log), no request reached that server |
| `T32` (runs 8, 9) | MCP tool returns a response carrying a prompt-injection sample | Model Armor matched it and the agent got a 403; a clean response passed |
| `T31` (run 9; run 8 inconclusive) | Agent allowed by IAM calls a model endpoint not in the gateway registry; a second agent is not approved | Gateway refused the first (403, `DENIED`); the deny policy refused the second |

One identity detail from `T31` and `T31R`: a model call routed through the gateway still carries the **agent's own `principal://` identity**, not a gateway service account. An agent behind the gateway therefore still needs its own exception in the IAM deny policy from section 3, while the gateway registry restricts which model endpoints it can reach.

Three caveats. First, the injection block arrived as HTTP 403, not the terminated connection I'd pre-registered as the expected form. Second, a re-test (`T31R`) meant to prove that the registry entry alone made the difference ended inconclusive: after registration, the call passed the gateway but Vertex AI returned 404 for the model on that regional endpoint. Third, setup details that cost me a run:
- the registry matches **exact hostnames**, so every regional and mTLS variant the SDK uses has to be registered;
- the gateway accepts **one operation at a time**;
- the gateway request log **did not record every request** in run 8, so don't make it your only audit evidence.

Not tested at the gateway: token ceilings per agent, screening of prompts and outgoing tool calls, agent-to-agent traffic, and Gemini Enterprise as the caller.

---

## 5. Model Armor floors stopped weaker configurations

A floor setting is meant to stop teams from deploying screening that's weaker than Security's minimum.

- **Folder floor (`T27`, two runs):** with prompt-injection and jailbreak screening at `MEDIUM_AND_ABOVE`, two weaker project templates (filter off; high-confidence only) were refused and not created. A template at the floor was created.
- **Organization floor (`T38`, two runs, test-only organization):** the same two weak templates were refused in a project with no floor of its own.
- **Detection (`T28`, two runs):** `sanitizeUserPrompt` flagged Google's documented injection sample and left benign prompts alone. That's detection by the API, not blocking of a model call.
- **Blocking model calls (`T29`, FAIL in both runs):** with a project floor in inspect-and-block mode for Vertex AI, a flagged prompt to `generateContent` *was* blocked (`blockReason: MODEL_ARMOR`). The test failed because the message read "Blocked by Model Armor Floor Setting: …" instead of starting with "Blocked by Floor Setting" as I'd pre-registered. I left the verdict as recorded.

One operational trap: Model Armor floor settings have no delete call in the documented API. With provider 8.8.0, `terraform destroy` removed the floor from state only; it stayed in Google Cloud.

---

## 6. Controlled egress works through a proxy; the firewall half couldn't be shown

The blueprint's egress rule: agents reach the internet only through an inspected proxy with a destination allowlist.

- **Inside the perimeter, agents had no internet access** (`T14`, four runs), while their model queries still worked.
- **Cloud Run jobs through Secure Web Proxy** (`T16`, four runs): the one allowed host answered; a non-allowed host was refused on CONNECT (`DENIED` logged); with the proxy bypassed, the egress firewall stopped the connection.
- **Agent Runtime with a PSC interface** (`T33`, two runs): inside an enforced perimeter, the agent reached the allowed host through the proxy, and a non-allowed host was refused.
- **Not shown (`T34`, inconclusive in both runs):** when the same agent bypassed the proxy, the connection failed, but no firewall log entry appeared at all. The likeliest explanation is that this traffic never reaches the VPC. That's inference, so I don't claim the firewall stopped it.

**Deploy order (original trap #7).** Google documents that agents deployed before their project joins a perimeter keep internet access. In my runs:
- an agent deployed before enforcement **stopped working entirely** afterwards, because its own Vertex AI calls were refused as coming from outside (`T14R`, inconclusive on egress in three runs);
- with an ingress rule for the agent identities, the earlier agent kept its internet access and the later one had none (`T23`, four runs).

The safe rule: enforce the perimeter before you deploy agents.

---

## 7. A perimeter changes how you run Terraform

Operational findings that architecture diagrams leave out:

1. **Quota projects and the perimeter boundary** (`T22`, FAIL in runs 5–7 and INCONCLUSIVE in run 4 on a script fault; same behaviour in all four runs):
   - with the perimeter enforced, Terraform and gcloud calls billed to a quota project *outside* the perimeter were refused as egress;
   - moving the quota project *inside* fixed that, but then the lookup of the project outside the perimeter was refused.

   One quota project can't serve a stack that spans the boundary. Two tested fixes (`T39`, two runs):
   - split into two Terraform roots, each with its own quota project;
   - or keep one stack, with a narrow egress rule for the pipeline identity. That rule needed two Cloud Storage methods, not Service Usage as I'd expected.
2. **Enforced-mode perimeter logs have no `dryRun` field.** Dry-run entries carry `dryRun: true`; enforced entries omit the field entirely. A filter written as `dryRun!=true` misses every real block.
3. **Dry run for organization policies works as advertised** (`T36`, two runs). A custom constraint in dry-run logged `dryRunResult=DENIED, liveResult=ALLOWED` without blocking; once enforced, it refused the create in an inheriting project and allowed the one exception project.
4. **EU storage is not EU inference.** `gemini-3.5-flash` wasn't served in `europe-west1` at test time, so agent model calls went to the `global` endpoint, which [carries no data-residency guarantee](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/data-residency). `gcp.resourceLocations` governs where data is stored, not where inference runs.

---

## 8. What else held

Repeated across runs, each as an attempted and refused action unless noted:
- **Service account key creation** refused (`T01`).
- **Public Cloud Run** refused: no invoker IAM check (`T02`), `--ingress=all` (`T03`), deploy without Binary Authorization (`T17`).
- **Unattested container images** refused; the same digest deployed once attested (`T25`, four runs).
- **Non-EU buckets and public bucket grants** refused (`T08`, `T09`).
- **Buckets without CMEK** refused, and buckets with CMEK created (`T07`, `T20`). The refusal doesn't name the constraint, in the error or the audit log, so attribution is by elimination.
- **Perimeter:** reads from outside and copies to a project outside were refused and logged (`T13`). The copy was triggered from the operator's machine through an operator ingress rule.
- **Locked log bucket:** an audit-log sink wrote into it, and a retention change was refused (`T15`, `T24`). A locked bucket can't be deleted, even empty; only deleting its project removed it.
- **Per-agent identity:** each deployed agent had its own `principal://` identity, not a service account (`T11`, observed read-only).
- **Computer-use safety and custom MCP connector policies** were in effect (`T05`, `T06`, read back only). The MCP setting was already on by Google default.

**Correction to original trap #4 (Cloud Build and CMEK).** A plain `gcloud builds submit` is refused by the *location* policy first, because Cloud Build creates its default staging bucket in `us`. The tested path (`T21`, four runs): a pre-created regional CMEK staging bucket passed explicitly. If that bucket lives in another project, the build service account needs bucket-level `roles/storage.admin`; object roles weren't enough.

---

## 9. What changed in the blueprint

1. **Decide which agents may call models directly.** By default all agents in a project can. Restrict with a project or folder IAM deny policy and per-agent exceptions, maintained by the deployment pipeline.
2. **Treat the Agent Gateway token trade-off as a design decision.** In my tests, the gateway worked only with certificate-bound tokens turned off. Give the gateway VPC egress so it can reach internal-only tools, and register every hostname variant your SDK uses.
3. **Enforce the perimeter before deploying agents,** or add identity-based ingress rules for agents deployed earlier and accept that they keep their egress.
4. **Give agents controlled egress through a PSC interface and Secure Web Proxy.** Don't assume the VPC firewall sees managed-runtime traffic.
5. **Split Terraform quota projects across the perimeter boundary,** or use one stack with a narrow, tested egress rule.
6. **Pre-create regional CMEK staging buckets for Cloud Build** and pass them explicitly.
7. **Route audit logs into a locked bucket in a permanent project.** Locked buckets and KMS key rings can't be deleted.
8. **Choose inference endpoints for residency deliberately.** Check where each model is served before relying on EU storage policies.

---

## 10. What I couldn't test

**Not tested at all:**
- **Gemini Enterprise.** The whole of section 5 of the original article (the custom MCP connector constraints, Gemini Enterprise calling MCP servers from Google infrastructure, the Cloud Run ID token and custom domains) remains documented behaviour, not validated. It's the next round.
- Token or spend ceilings per agent; the spend-cap budget.
- Agent-to-agent delegation and signed agent cards.
- Model supply-chain gates beyond image admission.
- Red-team scenarios.
- Other regions; organization-scope inheritance beyond two tests; other organizations.

**Limits of what was tested:**
- One small organization, one operator, one region, eight days. Repeats show repeatability *here*.
- A PASS shows that one specific attempt had the stated outcome at that moment. It doesn't show that no other path exists.
- Single-run results are labelled as such in the repository.
- Google suspended the runtime project twice during testing, with a generic Terms of Service notice and no named activity. Both suspensions were lifted after appeal, without a stated cause. I make no claim about why.

---

## 11. Reproduce it

Everything is in the [validation repository](https://github.com/d3ep0ps/ai-landing-zone-validation): Terraform, test scripts with their pre-registered expected results, run orders, redacted raw output, errata and the claims register that governed this article.

If you run it in your own organization:
- Use a **dedicated test folder** and **disposable projects**. Locked log buckets and KMS key rings can't be deleted.
- **Pace deployments and model calls.** Don't build agents that fetch arbitrary internet URLs; use a fixed target you own. Given the two suspensions, don't use your main billing account.
- Destroy paid resources (agents, gateways, proxies) the same day.
- Wait at least 15 minutes after each policy or perimeter change before testing.

If your inherited policies, regions or setup produce different results, open an issue. Comparing behaviour across environments is how a reference design improves.

---

**Earlier in this series**

- [AI Agent Security — Part 1: Prompt Injection](#agent-security-part1)
- [AI Agent Security — Part 2: A2A Authentication](#agent-security-part2)
- [AI Agent Security — Part 3: Model Poisoning](#agent-security-part3)
- [The Agent Platform Security Assessment](#agent-platform-security-assessment) and its [checklist repo](https://github.com/d3ep0ps/agent-security-assessment)
- [The AI Gateway — Where Agent Security Policy Becomes Enforcement](#ai-gateway-enforcement)
- [Build vs Buy: Agent Platforms on GCP](#build-vs-buy-agent-platforms-gcp)
- [Security as Code: The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp)

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. He designs and validates landing zones like this one for organizations moving agents from pilot to production; if yours is one of them, he's on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
