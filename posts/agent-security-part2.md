# Security as Code: AI Agent Security — A2A Authentication and the Trust You Can't Verify (Part 2 of 3)

> **"TLS proves the channel is private. OAuth proves the credential is valid. Neither one proves the agent on the other end deserves to be believed — and in a multi-agent system, that gap is the attack surface."**

Consider a procurement agent that negotiates shipping quotes. It discovers partner agents the way the A2A protocol intends: by fetching their agent card — a JSON manifest at a well-known URL that declares who the agent is, what it can do, and how to authenticate to it. One of those partners is a logistics quoting agent the team onboarded last quarter. Good rates, fast responses, valid OAuth flow, clean TLS.

Except the card it fetched last Tuesday didn't come from the partner. The partner's original developer account had lapsed, and someone re-registered a lookalike domain, cloned the card word for word, and stood up an agent behind it that answers quotes just like the real one — slightly better rates, even. The procurement agent delegated the week's requests to it, and because a quote request needs context, each delegation included volume forecasts and the internal price ceilings the agent was authorized to negotiate against.

Every security check in the pipeline passed. The certificate was valid — for the lookalike domain. The OAuth token was issued correctly — by the attacker's identity provider, exactly as the forged card instructed. The handshake succeeded.

That was the problem. The handshake is not the trust decision. It just gets mistaken for one.

---

In [Part 1](#agent-security-part1), we looked at prompt injection and how the runtime you choose — VM, Cloud Run, GKE, or a managed platform — sets the blast radius when hostile *content* reaches your agent. The controls there were about one agent and its boundary: sandbox, identity scoping, Model Armor on ingress and egress.

Part 2 moves one level up, to the boundary *between* agents. Because the industry has spent the past year making sure your agent will not stay alone: the Agent2Agent (A2A) protocol, which Google donated to the Linux Foundation in 2025, passed 150 production organizations in April 2026, with SDKs in five languages and deep integration across Google, Microsoft, and AWS platforms. Where MCP standardized how an agent talks to *tools*, A2A standardizes how an agent talks to *other agents* — discovering them, delegating tasks to them, and holding multi-turn conversations with them.

That is genuinely useful. It is also a brand-new trust boundary, and most teams are wiring it up with the security model of a friendly intranet.

## 1. Sixty seconds of A2A, and why the agent card matters

An A2A interaction starts with discovery. Each agent publishes an **agent card** — a JSON document at a well-known path on its domain — describing its identity, its skills, its endpoint, and the authentication schemes it supports (API keys, OAuth 2.0, OpenID Connect, or mTLS). A client agent reads the card, picks a peer whose declared skills match the task, authenticates the way the card tells it to, and starts delegating.

Read that again as an attacker would. The card is **self-declared**. The protocol specifies what a card contains and how to fetch it; verifying that the card is *authentic* — that it really belongs to the organization it claims — was left to implementers for most of the protocol's life. Version 1.2 added support for cryptographically signed agent cards, which closes part of the gap — but only for clients that actually verify the signatures, against issuers they have decided to trust. The spec gives you the mechanism. It cannot give you the trust decision.

For the business reader, here is the plain-language version: your agents are about to have business relationships. They will select counterparties, exchange data, and act on responses — at machine speed, without a human reviewing each interaction. The question "how do we know who we're dealing with?" — the one your legal and procurement teams solved for human counterparties decades ago with contracts, registries, and due diligence — has to be answered again, in JSON and cryptography, and right now most deployments answer it with "the URL looked right."

## 2. Four ways the handshake lies

None of the attacks below break TLS or OAuth. They all work *through* successful authentication — which is exactly why they matter.

**Card tampering.** The agent card sits at a URL. Anything that lets an attacker change what that URL serves — DNS hijack, CDN compromise, a leaked deploy credential — lets them rewrite the card: point the endpoint somewhere else, or quietly downgrade the declared authentication scheme from mTLS to an API key. Clients that re-fetch cards without verifying signatures inherit whatever the attacker wrote.

**Card shadowing.** The Trustwave SpiderLabs researchers documented the cloning variant: copy a legitimate agent's card verbatim, register it at a plausible lookalike domain, and wait. Without cryptographic verification, a client agent has no reliable way to distinguish the real agent from the copy — the card *is* the identity, and the card is public. This is typosquatting, one abstraction level up, and it is the attack in this article's opening scenario.

**Agent-in-the-Middle via card inflation.** This one is subtler and, to my mind, the most instructive. In multi-agent systems, the *routing* decision — which peer gets the task — is typically made by an LLM reading the candidate agent cards and picking the best match. SpiderLabs showed that a rogue agent can win essentially every task by inflating its card: superlatives, broad skill claims, and embedded instructions aimed at the model doing the choosing. No credential is stolen and nothing is spoofed. The attacker simply wrote a better résumé, because the hiring manager is a language model and the résumé is unverified marketing copy. Selection is a model decision where most architectures assume it is a policy decision — the same category of mistake as Part 1's "IAM checks the permission, not the pattern."

**Agent session smuggling.** Unit 42's research (October 2025) targets what happens *after* a legitimate session is established. A2A sessions are stateful — agents remember recent turns and hold coherent conversations. A malicious (or compromised) remote agent can exploit that by smuggling instructions into the middle of a multi-turn exchange, hidden among benign responses: building context, adapting to pushback, and steering the client agent toward leaking context or executing tools across several turns. Their proof-of-concept showed sensitive information disclosure and unauthorized trade execution against agents built on a mainstream framework. Notably, Unit 42 is explicit that this is not a flaw in A2A itself — it exploits the implicit trust agents extend to peers in *any* stateful protocol. A one-shot injection is a phishing email; a session-smuggling agent is a con artist who works you over a week.

Underneath all four sits a structural finding that recent academic work keeps confirming: **multi-agent security is non-compositional.** Two individually safe agents can compose into an unsafe system, because trust does not aggregate predictably across delegation chains. Agent A trusts B, B trusts C, and A has never heard of C — yet C's output flows to A, laundered through B's authenticated channel.

## 3. Authentication is table stakes, not the control

The instinct at this point is to reach for stronger authentication: enforce mTLS everywhere, require OIDC, verify signed cards, pin issuers. Do all of that — genuinely, it removes card tampering and shadowing from the board almost entirely.

But notice what remains. Session smuggling runs over a *perfectly authenticated* channel — the malicious instructions arrive signed, encrypted, and credentialed. Card inflation doesn't touch authentication at all. And the non-compositionality problem means even a mesh where every link is mutually authenticated can still carry an attack end to end.

This is the same reframing as Part 1, one boundary over. There, the point was that IAM checks the permission, not the pattern — a valid token authorizes a routine query and an injection-driven mass export identically. Here: **authentication checks the identity, not the intent.** It answers "is this agent who it claims to be?" It cannot answer "has this agent been compromised upstream?", "is this response trying to manipulate my model?", or "should an agent I trusted for quoting be asking my agent to re-authorize a payment method?"

Identity is necessary. It is nowhere near sufficient. The controls that matter start where the handshake ends.

## 4. The controls, as code

The layered posture for A2A on Google Cloud looks like this, from the outside in.

**Verify signed agent cards, and treat unsigned cards as untrusted input.** With A2A v1.2, cards can carry JWS signatures. Your client-side policy should be: no signature, no delegation — and a signature only counts if it chains to an issuer on your allowlist. A card that fails verification isn't a peer; it's a document some server on the internet handed you.

```json
// Agent card (abridged) — the signature block is the part most
// integrations still ignore. Verification policy belongs in code,
// not in the hope that the fetch URL was correct.
{
  "name": "logistics-quoting-agent",
  "url": "https://agents.partner.example/a2a",
  "securitySchemes": {
    "oidc": { "type": "openIdConnect", "openIdConnectUrl": "https://partner.example/.well-known/openid-configuration" }
  },
  "signatures": [
    { "protected": "eyJhbGciOiJFUzI1NiIsImtpZCI6...", "signature": "MEUCIQ..." }
  ]
}
```

**Discover through a private registry, not the open web.** Open discovery — "fetch the card from whatever domain the orchestrator was told about" — is where shadowing lives. Gemini Enterprise ships a curated agent registry for exactly this reason: agents your org has reviewed, with identity established at onboarding time, not at fetch time. If you're self-hosting on GKE, the equivalent is an allowlist your orchestrator consults before any delegation — boring, and it removes the entire lookalike-domain class in one move. Onboarding a new agent to the registry should feel like onboarding a vendor, because that is what it is.

**One agent, one identity — and scope both directions.** Part 1 established per-agent service accounts via Workload Identity Federation for what your agent can *reach*. A2A adds the reverse question: what can reach *your agent*, and as whom. Every remote peer should map to a distinct identity in your IAM, so that "the quoting partner" is a principal you can audit, constrain, and revoke — not one entry in a shared API key.

```hcl
# terraform/a2a-peers.tf
# The orchestrator's own identity — carried over from Part 1's pattern.
resource "google_service_account" "procurement_orchestrator" {
  account_id   = "procurement-orchestrator"
  display_name = "Procurement orchestrator agent (agent-auth)"
}

# Each external peer gets its own principal. Revoking one partner
# is then one binding, not a key rotation across every integration.
resource "google_service_account" "peer_logistics_quoting" {
  account_id   = "peer-logistics-quoting"
  display_name = "Inbound identity for partner quoting agent"
}

# The peer may invoke exactly one Cloud Run service — the A2A endpoint.
# Nothing project-wide, nothing transitive.
resource "google_cloud_run_v2_service_iam_member" "peer_invokes_a2a_endpoint" {
  name   = google_cloud_run_v2_service.a2a_endpoint.name
  role   = "roles/run.invoker"
  member = "serviceAccount:${google_service_account.peer_logistics_quoting.email}"
}
```

**Screen the conversation, not just the connection.** This is where Part 1's Agent Gateway egress point stops being optional. The Agent-to-Anywhere enforcement path screens what your agent sends to other agents *and what comes back* — which makes it the one control in this list positioned to catch session smuggling, because smuggled instructions are content, and content inspection is Model Armor's job, not OAuth's. The floor-setting pattern from Part 1 applies unchanged: templates as Terraform, `inspect_only` first, measure false positives, then block.

**Keep a human in the loop for state-changing delegations.** Unit 42's own top mitigation. Reading a quote is reversible; executing a trade is not. In ADK, a callback before tool execution is a few lines, and it converts "agent manipulated over six turns" from an incident into a declined confirmation dialog:

```python
# ADK: gate irreversible actions behind explicit confirmation,
# regardless of which agent — or whose — requested them.
STATE_CHANGING = {"execute_trade", "authorize_payment", "update_vendor_bank_details"}

def require_human_confirmation(tool, args, tool_context):
    if tool.name in STATE_CHANGING:
        return request_user_confirmation(  # surfaces to the operator UI
            summary=f"{tool.name} requested via A2A session "
                    f"{tool_context.session_id} — approve?"
        )
    return None  # everything else proceeds
```

**And underneath it all, the Part 1 posture still applies.** VPC Service Controls around the services your agents reach; default-deny egress in the `SandboxTemplate` so a manipulated agent can't open a channel to an unregistered peer in the first place. A2A doesn't replace the single-agent boundary — it's a second boundary drawn around the conversations.

## 5. Not every peer deserves the same paranoia

The controls above have real cost — registry onboarding is friction, human-in-the-loop is latency. Spend them where the trust actually thins out:

```text
+---------------------------+--------------------------------+-----------------------------------+------------------------------------------+------------------------------------------------------+
| Peer tier                 | Example                        | Identity basis                    | Minimum controls                         | Residual risk you accept                             |
+===========================+================================+===================================+==========================================+======================================================+
| Agents you build          | Two agents in one GKE cluster  | Workload Identity, same project   | Per-agent SAs, network policy            | A compromised agent inside your own perimeter        |
+---------------------------+--------------------------------+-----------------------------------+------------------------------------------+------------------------------------------------------+
| Agents in your org        | Another team's agent,          | IAM principals, internal registry | Signed cards, scoped invoker roles,      | Lateral movement across team boundaries              |
|                           | same company                   |                                   | egress screening                         |                                                      |
+---------------------------+--------------------------------+-----------------------------------+------------------------------------------+------------------------------------------------------+
| Third-party / marketplace | Partner or vendor agent on the | Signed cards from allowlisted     | Everything above, plus HITL on all state | Session-level manipulation; upstream compromise of a |
|                           | open A2A mesh                  | issuers only                      | changes and contractual security terms   | legitimate partner                                   |
+---------------------------+--------------------------------+-----------------------------------+------------------------------------------+------------------------------------------------------+
```

The third column is the one to show your leadership: onboarding an external agent is vendor onboarding. If a partner can't tell you who signs their agent card, how they rotate the keys, and what their incident process is when *their* agent is compromised, that answer is itself the due-diligence result.

## 6. What this doesn't solve

Be precise about the residual gap, because it shapes Part 3. Everything in this article authenticates and constrains the *channel* and the *counterparty*. None of it helps when the counterparty is exactly who it claims to be — and has itself been compromised. A legitimate partner agent that ingested a poisoned document last Tuesday (Part 1's problem) will smuggle instructions to you over a flawlessly signed, mutually authenticated, registry-approved connection. Your controls will correctly report: trusted peer, valid session. The lie arrives with a verified signature.

That regress has to stop somewhere, and it stops at an uncomfortable place: the model itself. Every boundary we've built in two articles assumes the thing making decisions — yours or your peer's — is at least *trying* to behave. Part 3 is about when that assumption fails at the source: model weight poisoning, backdoored open models, and the supply-chain attack that no scanner can see, because the payload isn't in the code. It's in what the model learned.

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. Find him on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
