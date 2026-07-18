# Security as Code: The Agent Platform Security Assessment

> **"A security control you can't produce evidence for is a belief. An assessment is the act of converting beliefs into evidence — or finding out you can't."**

Here is the state of the industry in one pair of numbers: 88% of organizations reported a confirmed or suspected AI-agent security incident in the past year, while only 14.4% of teams say their agents went to production with full security approval. Add a third: 81% of security leaders report pressure to deploy agents *before* governance is ready. The agents shipped first. The security review is happening now, retroactively, on systems already in production — if it's happening at all.

So this article is not going to tell you how to deploy an agent securely. The previous three parts of this series did that. This one answers the question that comes after: **the agents are already running — how do you find out where you actually stand?**

---

In the **Security as Code: AI Agent Security** series, we walked three boundaries of an agent platform: [Part 1](https://d3ep0ps.com) covered prompt injection and the blast radius of where your agents run; Part 2 covered agent-to-agent authentication; Part 3 covered model weight and supply-chain poisoning. Part 3 closed with three questions I now ask about any agent platform:

*What can it reach? Who can talk to it? Where did the model come from?*

This article turns those three questions into an assessment you can run against your own platform in a day — fifteen checks, each with a concrete verification step, and a scoring model blunt enough to put in front of your CTO. Where a check maps to a risk in the [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) (ASI), I cite the ID — if you need to justify this work to an auditor or a board, that mapping is your paper trail.

Everything below assumes GCP, because that's where this series lives. The structure — three boundaries, three evidence tiers — transfers to any cloud; only the verification commands are Google's.

## 1. The rule that makes this an assessment and not a checklist

Most agent-security checklists fail the same way: they ask *whether a control exists*, and a control can "exist" in three very different ways. So every check below is scored against three evidence tiers:

```text
+----------------+------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------+
| Tier           | Meaning                                                                      | Example                                                                                            |
+----------------+------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------+
| **Documented** | The control exists in a policy, wiki, or design doc                          | "Agents must use scoped service accounts" (a sentence)                                             |
| **Configured** | The control exists in the platform's live configuration                      | A scoped IAM binding you can display with `gcloud`                                                 |
| **Enforced**   | The control cannot be bypassed without changing code or failing a deployment | Binary Authorization rejecting the unattested image; an org-policy floor no template can dip under |
+----------------+------------------------------------------------------------------------------+----------------------------------------------------------------------------------------------------+
```

The rule, and it's the whole method: **an answer counts only at the tier you can produce evidence for, on demand.** "We use Workload Identity" is a claim. The `gcloud` output showing every node pool with `GKE_METADATA` mode is Configured. The org policy that prevents creating a node pool without it is Enforced.

If the last three articles had a shared conclusion, it was that policies that live in documents can be talked out of their job and policies that live in infrastructure cannot. The evidence tiers are that conclusion turned into a measuring stick.

One instruction before you start: **answer every check with a link to code or a command output, never with a meeting.** If a check requires asking a person, the honest answer is "Documented" at best.

## 2. Boundary 1 — Reach: what can the agent touch when it's tricked?

*Covers OWASP ASI01 (Agent Goal Hijack), ASI02 (Tool Misuse and Exploitation), ASI05 (Unexpected Code Execution).*

Part 1's premise: you cannot reliably stop a model from being tricked, so the real variable is what a tricked model can reach. These five checks measure that.

**R1 — Does any agent run as a default service account?**
The Compute Engine default service account often carries project Editor. An agent inheriting it means a successful injection inherits it too.
*Good looks like:* every agent workload has a dedicated SA with no basic roles.
*Verify:*

```bash
gcloud projects get-iam-policy $PROJECT \
  --flatten="bindings[].members" \
  --format="table(bindings.role, bindings.members)" \
  --filter="bindings.members:compute@developer.gserviceaccount.com"
```

Any row with `roles/editor` here is a finding, regardless of what the architecture diagram says.

**R2 — Is the agent's IAM scoped to resources, not the project?**
`roles/bigquery.dataViewer` granted at project level is a grant on every dataset the project will ever contain — including the ones created after the review that approved it.
*Good looks like:* grants at dataset/bucket/secret level; the project-level policy contains no agent SAs.
*Verify:*

```bash
gcloud projects get-iam-policy $PROJECT \
  --flatten="bindings[].members" \
  --filter="bindings.members:$AGENT_SA"
```

The goal is an *empty* result at project level, with the SA appearing only in resource-level policies.

**R3 — Is Workload Identity actually on for every node pool?**
On GKE Standard, one node pool without Workload Identity falls back to the node's SA and undoes the entire identity model. This is a per-pool check, not a per-cluster one.
*Verify:*

```bash
gcloud container node-pools list --cluster=$CLUSTER --region=$REGION \
  --format="table(name, config.workloadMetadataConfig.mode)"
```

Every row must read `GKE_METADATA`. Autopilot clusters pass by construction — that's Enforced tier for free.

**R4 — Does agent-generated code run in a sandbox with default-deny egress?**
ASI05 in one sentence: the agent writes code, and the code runs somewhere. If that somewhere is the agent's own container, the injection payload executes with the agent's identity and network position.
*Good looks like:* gVisor-isolated execution (GKE Agent Sandbox or equivalent) with a default-deny network posture, allowed destinations enumerated in the `SandboxTemplate`.
*Verify:*

```bash
kubectl get pods -n $AGENT_NS \
  -o custom-columns="NAME:.metadata.name,RUNTIME:.spec.runtimeClassName"
```

`gvisor` (or your sandbox runtime class) on every executor pod. A blank runtime class on a pod that executes generated code is a Documented-tier control at best.

**R5 — Is prompt screening a floor, or a choice?**
Model Armor templates that individual teams attach voluntarily are Configured. Floor settings at the org or project level — a minimum no application can opt out of — are Enforced. The distinction is exactly Binary Authorization's lesson applied to content screening.
*Verify:*

```bash
gcloud model-armor floorsettings describe \
  --full-uri="projects/$PROJECT/locations/global/floorSetting"
```

No floor setting means every team is one deadline away from shipping an unscreened agent.

## 3. Boundary 2 — Identity: who can talk to the agent, and as whom does it act?

*Covers OWASP ASI03 (Identity and Privilege Abuse), ASI07 (Insecure Inter-Agent Communication), ASI10 (Rogue Agents).*

Part 2's territory: the difference between *the agent's* identity and *the user's* identity, and what happens between agents. Five checks.

**I1 — Can you list every agent identity on the platform?**
An assessment of identities starts with an inventory of them. If agents share one SA "for now," you cannot attribute an action to an agent — which also means you cannot revoke one agent without breaking the rest.
*Good looks like:* one SA per agent, named to a convention, enumerable in one command.
*Verify:*

```bash
gcloud iam service-accounts list \
  --filter="displayName:agent" --format="table(email, displayName)"
```

If your agents run on Agent Runtime, check whether they use the newer [IAM agent identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity) instead of service accounts — a per-agent SPIFFE principal tied to the agent's lifecycle, which is the stronger model. The `effectiveIdentity` field tells you which one each agent actually has:

```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://${LOCATION}-aiplatform.googleapis.com/v1beta1/projects/${PROJECT}/locations/${LOCATION}/reasoningEngines" \
  | jq -r '.reasoningEngines[] | [.displayName, .spec.effectiveIdentity] | @tsv'
```

An `effectiveIdentity` showing a service account email means that agent predates (or opted out of) agent identity — worth a row in your findings either way. If the platform team can't produce this inventory without asking around, ASI10 — a rogue agent operating unnoticed — has no detection story.

**I2 — Do any agent service accounts have exported keys?**
Short-lived federated tokens are the model; a user-managed JSON key is a long-lived credential that survives outside every boundary you've built.
*Verify:*

```bash
for sa in $(gcloud iam service-accounts list --format="value(email)" --filter="displayName:agent"); do
  gcloud iam service-accounts keys list --iam-account=$sa \
    --managed-by=user --format="value(name)" | grep -q . && echo "KEY FOUND: $sa"
done
```

The passing output is silence.

Agents on IAM agent identity pass this check by construction — there are no exportable keys, and the tokens are certificate-bound (mTLS-enforced via a Context-Aware Access policy), so a stolen token can't be replayed from outside the agent's runtime. That's Enforced tier. One caveat turns it back into a finding: the CAA binding can be opted out of per agent. Grep deployment configs for it:

```bash
grep -rn "GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES" .
```

Any occurrence set to `False` is an agent whose credentials are replayable again — by explicit choice, which deserves a documented justification or a fix.

**I3 — Is agent-auth separated from user-auth?**
When an agent acts *for a user*, does it carry the user's delegated credential — or its own broader identity with a `user_id` parameter in the request body? The second pattern means every user inherits the agent's full reach, and the check on "can this user see this data" lives in application code, where an injection can reword it.
*Good looks like:* ADK's split between agent identity and user credential (the Agent Identity / auth manager pattern from Part 1); the agent's own SA cannot read user-scoped data without a delegated token.
*Verify:* this one is a code check, not a command — find the tool implementations that touch user data and confirm they require a user credential in the call path. Evidence is a link to the code.

**I4 — Do your agents verify who they're talking to?**
A2A's AgentCard is a claim of identity. Part 2 covered card spoofing and session smuggling; the control is verification of the peer — signed cards, mTLS between agents, and rejection of unsigned peers — not the existence of the protocol.
*Good looks like:* inter-agent traffic on mTLS (service mesh or Agent Gateway), A2A endpoints rejecting unauthenticated peers, and no agent-to-agent path that bypasses the gateway.
*Verify:* attempt an unauthenticated card exchange against a production A2A endpoint from a pod outside the mesh. It should fail at the transport layer — before your agent's code ever sees the request.

**I5 — Is there an inventory of MCP servers, with owners?**
Every MCP server is a trust boundary: its tool descriptions enter your model's context, and Part 1 showed what a poisoned description can do. An MCP server nobody owns is a supply-chain component nobody patches.
*Good looks like:* a registry (Agent Gateway / agent governance registry, or even a reviewed YAML in git) listing every MCP endpoint agents may reach, each with an owner; egress rules that block unlisted endpoints — that's what promotes this from Documented to Enforced.
*Verify:* pick an MCP endpoint *not* on the list and confirm an agent pod cannot reach it.

## 4. Boundary 3 — Provenance: where did the model come from, and would you notice a swap?

*Covers OWASP ASI04 (Agentic Supply Chain Vulnerabilities), ASI06 (Memory and Context Poisoning).*

Part 3's territory. The controls here are the SBOM series one layer up: the model is an artifact, and artifacts get gates.

**P1 — Are model versions pinned by digest?**
`latest` on a model reference means your platform auto-ingests whatever the publisher ships next — and Part 3 opened with exactly that failure. An update is an ingestion event, not a background process.
*Verify:* grep deployment manifests and Model Garden / registry references for tags vs digests. Evidence is the manifest in git.

**P2 — Is there one path to production for models?**
If a model can reach serving from a laptop, a notebook, or a bucket upload — anything other than the single gated pipeline — every other check in this boundary is decorative.
*Good looks like:* serving infrastructure that can only load from one registry, write access to that registry held only by the ingestion pipeline's SA.
*Verify:*

```bash
gcloud storage buckets get-iam-policy gs://$MODEL_BUCKET \
  --format="table(bindings.role, bindings.members)"
```

Humans with `objectAdmin` on the model bucket are alternate paths to production.

**P3 — Is the serving container attested?**
A signed model in an unattested container is half a control — the SBOM series' Binary Authorization work applies unchanged to the thing serving the weights.
*Verify:*

```bash
gcloud container binauthz policy export
```

Look for the enforcement mode on the serving cluster: `ENFORCED_BLOCK_AND_AUDIT_LOG`, not dry-run.

**P4 — Does a model version bump trigger a behavioral diff?**
Signing proves origin, never behavior — a backdoor trained into the weights arrives with a legitimate signature. The control is a regression suite of safety/security evals that runs on every new version, with the diff against the previous version reviewed like a code diff.
*Verify:* the CI configuration file that makes the eval job a required step between ingestion and promotion. If the eval exists but a human can promote without it, it's Configured, not Enforced.

**P5 — Is agent memory treated as an input surface?**
ASI06 is the slow variant of prompt injection: poison the agent's long-term memory or RAG store once, and every future session inherits the payload. Provenance applies to what goes *into* memory, not just to weights.
*Good looks like:* writes to memory/RAG stores attributable to a source (which session, which document, which tool), and a way to bulk-expire entries from a source later found hostile.
*Verify:* pick a memory entry and trace it to its origin. If the store can't answer "where did this come from," it can't answer "what else came from there" on the day that matters.

## 5. Scoring: three numbers, no dashboard

Score each boundary 0–3:

```text
+-------+--------------------------------------------------------------------------------------------------+
| Score | Meaning                                                                                          |
+-------+--------------------------------------------------------------------------------------------------+
| **0** | One or more checks fail outright (a default SA, an exported key, no sandbox)                     |
| **1** | Checks pass at Documented tier — the policies exist, the evidence doesn't                        |
| **2** | Most checks Configured — you produced command output for them today                              |
| **3** | The boundary's controls are Enforced — bypassing them requires changing code or failing a deploy |
+-------+--------------------------------------------------------------------------------------------------+
```

Write the result as three digits — Reach / Identity / Provenance. A platform scoring **2 / 2 / 1** is typical for a competent team a year into agents: infrastructure boundaries configured, provenance still running on trust. A **3 / 3 / 3** is rare and, read the next section, still not the end of the story.

Two properties make this scoring useful. It's *repeatable* — the same commands next quarter show drift as a diff, not an impression. And it's *legible upward* — three digits and the evidence-tier ladder fit in the first minute of a leadership conversation, which is where security budgets are won.

## 6. What a perfect score does not tell you

The honest asterisk, same as every part of this series: these fifteen checks verify *facts about artifacts, identities, and configuration*. The attacks live in *behavior*. A 3/3/3 platform still authorizes a permitted-but-hostile query with a valid token (Part 1's residual gap), still trusts a verified peer that has itself been compromised (Part 2's), and still serves a backdoored model whose signature is genuine (Part 3's).

An assessment that claimed to close those gaps would be marketing. What this one does instead is narrower and more useful: it tells you whether the boundaries you *think* you have actually exist, at which evidence tier, with proof you can regenerate on demand. Most real incidents die against exactly these boring layers — and the gaps that remain are at least *known* gaps, which is what runtime behavioral monitoring (the next series on this blog) is for.

## 7. Run it

The full assessment — all fifteen checks, the verification commands, and the scoring sheet — is on GitHub as a standalone repo: **[github.com/d3ep0ps/agent-security-assessment](https://github.com/d3ep0ps/agent-security-assessment)**. No form, no email gate. Clone it, run it against your platform, and keep the filled sheet in your repo next to the code it describes — that's the Enforced tier for the assessment itself.

If you run it and the three digits are lower than you'd like — or you'd rather have a second pair of eyes on the scoring before it goes in front of your leadership — this is the review I do with teams as a fixed-scope engagement. The contact is below; the first conversation costs nothing but the three digits.

## References

**Standards and frameworks**

- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) — the ASI risk IDs cited throughout
- [OWASP Agentic Security Initiative](https://genai.owasp.org/initiatives/agentic-security-initiative/)
- [CSA MAESTRO — threat modeling for agentic AI](https://cloudsecurityalliance.org/blog/2025/02/06/agentic-ai-threat-modeling-framework-maestro) — the methodology to grow into after this assessment
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [NSA/DoD: Careful Adoption of Agentic AI Services (April 2026)](https://media.defense.gov/2026/Apr/30/2003922823/-1/-1/0/CAREFUL%20ADOPTION%20OF%20AGENTIC%20AI%20SERVICES_FINAL.PDF)

**GCP documentation for the verification commands**

- [Model Armor overview](https://cloud.google.com/security/products/model-armor) and [Agent Gateway integration](https://docs.cloud.google.com/model-armor/model-armor-agent-gateway-integration)
- [Agent governance on Gemini Enterprise Agent Platform](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern)
- [IAM agent identity on Agent Runtime](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity) — per-agent SPIFFE principals and certificate-bound tokens
- [GKE Agent Sandbox](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/machine-learning/agent-sandbox) and [GKE Sandbox (gVisor)](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods)
- [Binary Authorization](https://cloud.google.com/binary-authorization/docs)

**State of the market (the opening numbers)**

- [NeuralTrust — The State of AI Agent Security 2026](https://neuraltrust.ai/blog/the-state-of-ai-agent-security-2026) (160+ CISOs surveyed)
- [Gravitee — State of AI Agent Security 2026](https://www.gravitee.io/state-of-ai-agent-security)
- [Okta — enterprise buyer survey on AI agent security](https://www.okta.com/newsroom/articles/enterprise-buyer-survey-ai-agent-security/)
- [CSA — State of AI Cybersecurity 2026](https://cloudsecurityalliance.org/blog/2026/05/27/state-of-ai-cybersecurity-2026-92-of-security-professionals-concerned-about-the-impact-of-ai-agents)

**Earlier in this series**

- Security as Code: SBOM and the Container Supply Chain (Parts 1–3)
- Security as Code: AI Agent Security — Prompt Injection (Part 1), A2A Authentication (Part 2), Model Poisoning (Part 3)

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. If you want a second pair of eyes on your assessment, find him on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
