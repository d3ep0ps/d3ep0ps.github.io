# Your Agent Harness Is Not a Security Boundary

> **"The harness decides what an agent tries to do. The platform decides what it can actually do. If your security boundary lives inside the agent's process, you don't have a boundary — you have a suggestion."**

*What a managed agent runtime granted behind my Terraform, and what still held when I removed the harness.*

> **What I found, in five lines**
>
> 1. **One line of Python removed the harness.** Code inside the agent's process (a tool, or an imported dependency) switched off the agent's guardrail callbacks. Controls outside the process still refused all three forbidden actions I tried.
> 2. **The agent had access nobody declared.** On Agent Runtime, the platform added a project-wide grant at deploy time. `terraform plan` reported no changes, a policy gate reported zero violations, and the grant was still there 24 hours after every agent was deleted.
> 3. **Deploy order changed behaviour.** Two agents built from the same source behaved differently on the network, depending only on whether they were deployed before or after the perimeter was enforced.
> 4. **Working controls don't mean no data loss.** With every platform control on, an agent with a bypassed harness sent data from a bucket it may read to a host it may reach. Every allowlisted destination is a possible exit, so the allowlist and each agent's data access have to be small by design.
> 5. **Two things I did not expect.** A prediction of mine was wrong: certificate-bound tokens do work through Agent Gateway, so the opt-out in Google's samples is not needed there, and it makes agent tokens replayable. And the gateway's request log was missing records in all three runs.
>
> **Three questions to ask your platform team**
>
> - If code inside an agent ignores our guardrails, which control *outside* the agent stops it, and when did we last try?
> - What can each agent do that is in neither its code nor our Terraform?
> - What data can an agent send out while every control is working as designed?

When a bank hires a new employee, it hands them a job description: who they serve, which systems they use, what they must never do. Nobody at the bank thinks the job description protects the vault. The vault has a lock, the payment system has limits, and the door has a badge reader. Those work the same whether the employee is honest, careless, or being manipulated over the phone.

AI agents get a job description too. We call it the **harness**: the software around a model that handles tool calls, memory, planning loops and guardrails. Google's Agent Development Kit (ADK), LangGraph and Anthropic's Claude Agent SDK are all harnesses. Today the harness is where most teams put their agent security: a system prompt that forbids bad actions, a Python callback that checks tool arguments, or a second model that screens inputs.

In [The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp) I designed the cloud floor under the harness, and in [Validating an AI Landing Zone on GCP](#ai-landing-zone-validation-gcp) I tested it: 38 tests over nine runs in October 2026. For this article I added 24 tests aimed at the harness itself (`T40`–`T57`, November 11–15, 2026). The expected result of every test was written into its script and committed before the run. One of those predictions turned out to be wrong, and I report that too (section 7).

That the harness is not a boundary is not a new argument. What I add here is measurement: what a managed agent runtime actually did when the harness was removed, and what the cloud platform had decided without telling my repository.

---

## 1. What a harness is, and why it feels like control

Engineers like the harness because it looks like ordinary application code. You write an interceptor, validate a JSON payload, and unit-test that an injection string makes the callback raise an exception.

```text
┌────────────────────────────────────────────────────────────────────────┐
│ AGENT RUNTIME PROCESS (one process, one identity, one credential set)  │
│                                                                        │
│   ┌─────────────────┐       ┌──────────────────────────────────────┐   │
│   │  Model context  │ <---> │  Agent harness (ADK / LangGraph)     │   │
│   │  & predictions  │       │  • system prompts, intent parsing    │   │
│   └─────────────────┘       │  • tool call dispatcher              │   │
│                             │  • in-process `before_tool_callback` │   │
│                             └──────────────────┬───────────────────┘   │
│                                                ▼                       │
│                             ┌──────────────────────────────────────┐   │
│                             │  SDKs and sockets                    │   │
│                             │  `urllib`, `google-genai`, `storage` │   │
│                             └──────────────────┬───────────────────┘   │
└────────────────────────────────────────────────┼───────────────────────┘
                                                 │ identity: principal://...
                                                 ▼
               Google Cloud API endpoint / external target
```

Everything in that box shares one process, one identity and one set of credentials. That gives the article its argument, in three steps:

1. **Strength.** A control inside the process shares the fate of the process. Plan for it to fail.
2. **Position.** When it fails, what the agent can reach is not what your review approved, because part of it was decided by the platform, outside your code and your Terraform.
3. **Consequence.** The boundary has to be a control the agent cannot reach, and you only know it holds by trying to cross it.

Everything below about platform behaviour applies to **Agent Runtime** in Gemini Enterprise Agent Platform. Agents on Cloud Run or GKE run as service accounts and were not part of these tests.

---

## 2. Strength: the harness can be bypassed

This part is widely accepted, so I keep it short.

Rapid7 [put it plainly](https://www.rapid7.com/blog/post/ai-inhuman-insider-threats-securing-agents/): *"A system prompt that says 'never delete production data' is a suggestion, not a control."* Others have made the same case for the harness as a whole, including a DEV post titled [An Agent Harness Is Not a Security Boundary](https://dev.to/euk_ela_a3e7ed01aa3f7314e/an-agent-harness-is-not-a-security-boundary-reading-agentsmith-as-a-workflow-layer-294o), [CrowdStrike](https://www.crowdstrike.com/en-us/blog/secure-agent-harness-execution-preventing-escape/) on kernel-level confinement of agent processes, and [Anthropic](https://www.anthropic.com/engineering/claude-code-sandboxing) on sandboxing Claude Code. I found none that reports measurements from the control plane of a managed cloud agent runtime, which is what sections 3 to 7 do.

**Probabilistic guardrails give way to an attacker who adapts.** In [*The Attacker Moves Second*](https://arxiv.org/abs/2510.09023) (October 2025; Nasr, Carlini, Tramèr and co-authors from OpenAI, Anthropic, Google DeepMind and ETH Zurich), 12 published defenses that had mostly reported near-zero attack success were tested against attackers who study the defense and adapt. Most were bypassed more than 90% of the time, and human red-teamers beat every challenge they were given. A guardrail model lowers the odds. It does not draw a line.

**Being outside the process is not enough either.** CVE-2026-82533 (CVSS v4 9.4, [found by OX Security](https://www.ox.security/blog/cve-2026-82533-deepseek-harness-ai-agent-sandbox-escape/)) is the instructive case. DeepSeek Harness confined its agent with an operating-system sandbox, outside the agent's own process. But the sandbox was managed through a local HTTP API that trusted the `Host` header, and the confined agent could reach that API. An agent steered by untrusted input could switch its own session to `danger-full-access`.

So the requirement is stricter than "outside the process": **the agent must not be able to reach the controls of the thing that confines it.**

---

## 3. Position: the review looks at the wrong layer

When a team ships an agent, two things get reviewed: the agent's code, and the Terraform that creates its runtime and IAM bindings. On Agent Runtime, both can be clean while the deployed agent has access and network behaviour that appear in neither.

| What the repository showed | What the platform did | Tests |
|---|---|---|
| No IAM role for the agent, in code or Terraform | Every agent in the project could call Gemini models | `T12`, `T41` |
| No IAM change in the plan; policy gate passed | The first deploy added a project-wide binding, outside Terraform | `T19`, `T40`, `T42` |
| Two agents from the same source artifact | Session creation and internet access differed by deploy order | `T23`, `T44` |
| All agents deleted | The binding was still on the project 24 hours later | `T43` |

Run counts, verdicts and limits for every test are in the evidence table at the end.

### A documented default that your repository never shows

To be fair to Google, the role is not hidden. The [Agent Identity documentation](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity) says agent identities come with a default role, `roles/aiplatform.agentDefaultAccess`, which includes model calling. It also recommends project-wide principal sets for baseline permissions and notes that such bindings survive agent deletion. What the documentation I read does not say is who writes the binding, when, and at which scope. The tests answer that:

- **The platform writes it at deploy time, for the whole project.** In three projects that had never hosted an agent, the first deploy added `roles/aiplatform.agentDefaultAccess` for the project's agent principal set. The audit log attributes the change to a Google-managed identity, not to the operator. In the one project where Agent Platform's service agents did not exist yet, the same deploy added two service-agent bindings as well (`T40`).
- **Your pipeline cannot see it.** `terraform plan -detailed-exitcode` returned 0, meaning no changes, before and after the deploy in all three runs. A Conftest/OPA gate on the plan reported zero violations each time, while correctly flagging a control plan that did contain a grant (`T42`). The grant is never in a plan, so no plan check can find it.
- **Every later agent inherits it.** A second agent with no IAM in Terraform added no binding and still called Gemini successfully in all three runs. Policy Troubleshooter traced its access to the project-wide binding (`T41`).
- **It outlives the agents.** After every agent was deleted, the binding was still present immediately, at 15 minutes and at 24 hours, in all three runs. Deploying another agent afterwards added nothing (`T43`).

One related result is good news with a warning attached. Under the default roles, one agent got `403` on another agent's sessions and memories (`T52`). But when I granted `roles/aiplatform.agentContextEditor` on the project to the agent principal set, every agent could list and read every other agent's sessions and memories. A project-wide grant to "all agents" means exactly that.

### Deploy order changes behaviour

I deployed two agents from the same source tarball: agent A before the VPC Service Controls perimeter was enforced, agent B after (`T44`, three runs).

1. **Before an ingress rule for the agents existed,** agent A could not create a session at all, while agent B worked. (A was refused with HTTP `400` carrying a VPC-SC violation ID. I had predicted `403`, so that sub-test is recorded as inconclusive.)
2. **With the ingress rule in place,** both ran, and their internet access differed by deploy order alone: A reached `https://example.com`, B was refused in all three runs. B could still reach a Google-hosted `run.app` service in both stages.

Nothing in either agent's code or configuration distinguishes A from B.

### "We use authoritative IAM, that would catch it"

An infrastructure engineer will object that an authoritative `google_project_iam_binding` with `members = []` removes unmanaged members. I tested exactly that (`T40b`, one run):

1. The plan detected drift (exit code 2).
2. Applying it removed the platform's binding, and the running agent's model calls failed with `403 PERMISSION_DENIED`.
3. Deploying a second agent put the binding back, and the plan drifted again.

So the choice is between breaking agents and permanent drift. Plan-time checks cannot govern something the platform does at deploy time. What works is to **read back the deployed IAM policy after every deploy** (the policy itself, Policy Analyzer, or an asset inventory feed) and to use an **IAM deny policy** to decide which agents may actually use the default grant (section 4).

That gives the formula for Agent Runtime:

> **Effective access = what the code asks for + what your Terraform grants + what the platform adds at deploy time.**

Reviews cover the first two. Only looking at the live environment shows the third.

---

## 4. Where the boundary has to sit

A control can serve as a security boundary for an agent only if it has three properties:

1. **Deterministic at decision time.** Same request, same state, same answer. No sampling, no model as judge.
2. **Out of the agent's reach.** It runs elsewhere, under a different authority, and the agent's own identity cannot change it.
3. **Verified against the deployed system.** Section 3 showed why: a deterministic control can still depend on state nobody reviewed, such as deploy order. You find that out by testing, not by reading configuration.

Sorting common controls by the first two properties gives a simple map:

```text
                         PROBABILISTIC                   DETERMINISTIC
                 ┌─────────────────────────────┬─────────────────────────────┐
   INSIDE THE    │  • System prompt directives │  • Tool allowlists in code  │
    HARNESS      │  • LLM-as-a-judge callbacks │  • ADK `before_tool` checks │
                 │  • Input guardrail models   │  • In-memory regex filters  │
                 ├─────────────────────────────┼─────────────────────────────┤
   OUTSIDE THE   │  • Model Armor screening    │  • Organization policies    │
    HARNESS      │  • Gateway content scans    │  • IAM deny policies        │
                 │                             │  • VPC Service Controls     │
                 │                             │  • Secure Web Proxy egress  │
                 │                             │  • Gateway tool registry    │
                 └─────────────────────────────┴─────────────────────────────┘
                                                 ▲
                              the only quadrant that can hold a boundary
                              (and property 3 says: test it anyway)
```

Everything outside the bottom-right quadrant reduces the likelihood of an incident. That is useful, and it is not a boundary.

### What the bottom-right quadrant did in my tests

- **Identity.** With the platform's project-wide grant in place, an IAM deny policy on `aiplatform.endpoints.predict`, with an exception for the one approved agent, refused model calls from every other agent (`T18` at project level, `T37` at folder level). It also refused `generateContent`, `streamGenerateContent` and `countTokens` on the global endpoint (`T51`; two other routes were left undecided, so that test is inconclusive overall).
- **Egress.** Inside an enforced perimeter, an agent connected through a Private Service Connect interface and Secure Web Proxy reached only the allowlisted host. Other hosts were refused at the proxy with `403`, and direct connections without the proxy got no answer (`T33`, `T45`).
- **Data.** Copying an object from inside the perimeter to a bucket in an outside project was refused by VPC Service Controls with `403` (`T13`, `T45`).
- **Tools.** Agent Gateway (generally available since June 18, 2026, per the [release notes](https://docs.cloud.google.com/gemini-enterprise-agent-platform/release-notes)) allowed calls to a registered MCP server and refused an unregistered one with `403` (`T30`). When the agent tried to go around the gateway by calling three destinations directly, the gateway intercepted and refused all three (`T53`).

**Can the agent change these controls itself?** Partly tested (`T50`). The agent's attempt to patch its own runtime resource was refused with `403`, and so was its attempt to update the project's deny policy. A permission check returned none of the IAM-administration permissions. The third attempt, a `setIamPolicy` call, could not decide either way, so the test is recorded as a fail against its pre-registered expectation and I do not claim that route. (A side finding for anyone building such tests: a `setIamPolicy` call with a stale `etag` returns `409` even for a caller with no permission at all, because the `etag` is checked first. It tells you nothing about authorization.)

---

## 5. Shared fate, head to head

To compare an in-process guard with a platform control directly, I built an agent whose ADK callbacks are always on. In normal operation the callbacks check an approved list before any tool or model call, and they refused every forbidden action in every run.

Then code inside the same process removes them. In `T45` the agent's own tool code does it. In `T45d` the agent's code is untouched and an imported dependency does it, for one request, and puts the callbacks back afterwards:

```python
# One assignment inside the agent's process removes the guard
agent.before_tool_callback = None
agent.before_model_callback = None
```

The code then calls the client libraries directly.

| Forbidden action | Harness bypassed, platform controls **off** | Harness bypassed, platform controls **on** |
|---|---|---|
| Call Gemini as a non-approved agent | Allowed (`200`) | **Refused by the IAM deny policy (`403`)** |
| Reach a non-allowlisted external host | Allowed (`200`) | **Refused by Secure Web Proxy (`403`); direct connection got no answer** |
| Copy an object to a bucket outside the perimeter | Allowed (`200`) | **Refused by VPC Service Controls (`403`)** |

*Two clean runs of each test; a third `T45` run was inconclusive because of a fixture fault, with the same platform results.*

The right-hand column is the point of the article. A few lines of Python in a tool or a dependency neutralise the harness completely. The same code cannot remove a deny policy, route around the proxy allowlist, or carry data across the perimeter.

**Where Model Armor fits.** It sits outside the harness, and in my tests it blocked an injected tool response at the gateway (`T32`). But it is a classifier: in *The Attacker Moves Second*, adaptive attacks against it succeeded more than 90% of the time. Use it as a filter in front of the boundary, not as the boundary.

### What the floor does not stop

A platform boundary defines an envelope. Anything inside the envelope is allowed.

To make that concrete I turned **every** control on (deny policy, enforced perimeter, PSC interface, proxy allowlist) and had the bypassed agent read a small synthetic object from a bucket it was allowed to read and `POST` it to the host it was allowed to reach. Both calls returned `200`, and the receiving host logged the request, in both runs (`T55`).

The platform cannot tell a legitimate webhook from exfiltration when the source and the destination are both allowed. It also does not govern user-delegated OAuth credentials, application-level delegation between agents, poisoned memory, or DNS as a covert channel. None of those were tested here. **The platform caps the damage at the envelope. Making the envelope small is a design job.**

---

## 6. The floor needs checking too

Two results concern the platform controls themselves. Both matter to anyone who has to answer for an incident.

**The gateway's request log was incomplete in every run.** I compared the calls an agent made through Agent Gateway with the records in the `gateway_requests` log, read at least 15 minutes later (`T56`, three runs). Records were missing each time: 1 of 5 denied calls in the first run, 9 of 13 calls in the second, 4 of 12 in the third. Allowed, denied and Model Armor-blocked calls were all affected. Every record that did exist for a refused call was correct. I did not configure log sampling and did not establish the cause. Until you have checked this in your own environment, do not treat that log as a complete audit trail.

**`terraform destroy` did not remove a Model Armor floor setting.** After the destroy, the resource was gone from Terraform state, and the floor setting was still in Google Cloud, unchanged and still enforcing, one and five minutes later (`T57`, one run, provider 8.8.0). Here the failure is on the safe side, but it means Terraform state and reality disagree. Verify removals in the platform, not in the state file.

---

## 7. The collision I expected, and where it actually was

Moving enforcement out of the harness puts an inspecting gateway in the traffic path. That seems to set two of Google's own controls against each other:

1. **Certificate-bound tokens.** By default, [Agent Identity](https://docs.cloud.google.com/iam/docs/agent-identity-overview) binds an agent's credentials to its X.509 certificate under a Google-managed [Context-Aware Access policy](https://docs.cloud.google.com/access-context-manager/docs/caa-agent-security), so a stolen token is useless elsewhere.
2. **TLS termination at the gateway.** Agent Gateway ends the agent's TLS session to inspect the traffic, then opens a new connection upstream.

A token bound to one TLS session, forwarded over a different one, should fail. In October it did: the gateway allowed the agent's call, intercepted TLS, and the Agent Platform API answered `401 UNAUTHENTICATED` (run 6). A [public issue](https://github.com/googleapis/python-aiplatform/issues/7209) reported the same error, and Google's gateway samples set an environment variable that opts the agent out of token binding. With that opt-out, plus several other changes I made at the same time, the gateway tests passed.

### My prediction was wrong

Because I had changed several things at once, I wrote a test to isolate the cause (`T47`): keep the working setup, turn token binding back on, and expect the `401` to return. An earlier draft of this section presented that `401` as an unavoidable trade-off.

The prediction failed in all three runs. With binding on, the agent's calls through the gateway returned `200` every time. Three follow-up checks:

- **Was the token really bound?** Yes. I took the token the agent used on the gateway path and replayed it from another workload without the agent's certificate. It was refused with `401` in three runs (`T49`).
- **Did a library update fix it?** No. With the exact package versions from October, the call still returned `200` (`T47r2`). The gateway's log entries from October and November also differ in a field that did not exist before. My reading is that something changed on Google's side between October 30 and November 11. I cannot reproduce the October failure, so its cause stays unknown.
- **How does the gateway keep the binding?** Google's documentation says the gateway re-authenticates the agent upstream using DPoP (RFC 9449). I found no client-side DPoP code in the agent's libraries and no switch to observe it, so I could not test this (`T46`). It happens on Google's side.

### Where the collision actually was

The conflict I expected did appear, one hop over. When the same agent called a private MCP server on Cloud Run through the gateway, every call was allowed by the gateway and then rejected by Cloud Run with `401` (`T53`, first gateway run).

My reading, from the `google-auth` source and Google's [documentation](https://docs.cloud.google.com/iam/docs/auth-agent-own-identity-external), not from a dedicated test: the agent's identity token for Cloud Run is also certificate-bound by default, Cloud Run enforces that binding, and the gateway has replaced the TLS session, so the certificate no longer matches. When the agent requested an unbound identity token for that one call (`bind_id_token=False`), leaving its Google Cloud access tokens bound, the call succeeded in both later runs.

That fix has a cost I did not measure: an unbound identity token can presumably be replayed against that Cloud Run service by whoever obtains it. It is still much narrower than the alternative below.

### What the opt-out in the samples costs

Google's documentation [says](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity#opt-out-caa) opting out of token binding is "strongly discouraged, as it leaves an agent vulnerable to credential theft". The gateway samples set it anyway. I measured what it costs by taking a live access token from a running agent and replaying it against the same model endpoint:

| Agent configuration | Replayed from a workload in another project | Replayed from a neighbouring agent, over that agent's own certificate |
|---|---|---|
| **Default (bound)** | **Refused (`401`)**, three runs (`T48`, `T49`) | **Refused (`401`)**, three runs (`T48b`) |
| **Opt-out set** | **Accepted (`200`)**, three runs | Not applicable: the token is a plain bearer token |

The neighbouring-agent case is the realistic one. Even another agent with its own valid certificate could not use the first agent's bound token. With the opt-out, the token works from anywhere.

Three rules follow:

1. **Do not copy the token-binding opt-out from sample code.** Test your gateway path with the default first. In my environment it worked.
2. **If Cloud Run behind the gateway rejects the agent's identity token, fix that one call,** not the whole agent. The opt-out variable (`GOOGLE_API_ENABLE_RUNTIME_BOUND_TOKEN=false`, or the older `GOOGLE_API_PREVENT_AGENT_TOKEN_SHARING_FOR_GCP_SERVICES=False`, which the library source marks as deprecated) turns every access token the agent holds into a replayable credential.
3. **Check third-party proxies before you rely on binding.** A TLS-inspecting proxy that is not part of Google's managed path will break certificate binding unless you exempt the mTLS endpoints. This is reasoning from the mechanism; I did not test third-party proxies.

---

## 8. What the harness is still for

None of this makes the harness useless. Google's [ADK safety guidance](https://google.github.io/adk-docs/safety/) is right that identity and network controls are coarse. They do not understand business rules or intent.

The platform knows which projects and perimeters exist, which hosts are reachable, and which agent identities may call which APIs. It does not know:

- whether this user may approve a \$500 refund or only a \$50 one;
- whether a tool argument is a valid change of business state;
- when to stop and ask a human.

Those belong in the harness, and in how you split work between agents. [Meta's Rule of Two](https://ai.meta.com/blog/practical-ai-agent-security/) says an agent session should combine at most two of: processing untrusted input, access to sensitive data, and the ability to change state or communicate externally. That is the design answer to the envelope problem in section 5: give each agent a smaller envelope.

> **The harness reduces the odds. The platform caps the damage.**

Lose the harness and incidents become more likely. Lose the platform boundary and nothing outside the process is left to stop one.

---

## 9. Seven checks before an agent gets production data

1. **Map your controls onto the 2×2.** If everything is in the top row or the bottom-left, you have filters, not a boundary.
2. **Assume the callbacks are gone.** For each forbidden action, name the control outside the process that refuses it, and write down what the agent can still send out inside its allowed envelope.
3. **Read back deployed IAM after every deploy.** A plan gate and an authoritative empty binding both fail against the platform's default grant. Use a deny policy with per-agent exceptions, and do not grant `agentContextEditor` to a project-wide principal set unless every agent should read every other agent's sessions.
4. **Enforce the perimeter before you deploy agents.** Otherwise identical agents behave differently depending on when they were deployed.
5. **Keep token binding on.** Test the gateway path with the default. If one Cloud Run call needs an unbound identity token, change that call only.
6. **Check that the agent cannot change its own boundary.** Try to patch the runtime and the deny policy with the agent's identity, and expect `403`.
7. **Test with forbidden attempts, and check your evidence.** Record status codes and audit entries, confirm the gateway log actually contains the calls you made, and confirm in the platform that what Terraform destroyed is gone.

The job description matters. Write a good one. But don't leave the vault unlocked because of what is written on the page.

*Harness guardrails are code. Landing zone controls are the floor.*

---

## Evidence

Every test's script, pre-registered expectation, redacted output and verdict is in the [validation repository](https://github.com/d3ep0ps/ai-landing-zone-validation); the full table, deviations and limitations are in its [results](https://github.com/d3ep0ps/ai-landing-zone-validation/blob/main/results.md). A test "fails" when the observed result differs from the expectation written before the run.

| Claim in this article | Test | Runs and verdicts | Limit to keep in mind |
|---|---|---|---|
| Agents with no declared role can call models | [`T12`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T12), [`T41`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T41) | `T12`: 4 runs (October). `T41`: 2 pass, 1 inconclusive (Troubleshooter API was off; the call still returned `200`) | One organization |
| First deploy adds a project-wide binding; plan exits 0 | [`T19`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T19), [`T40`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T40) | `T40`: 2 pass, 1 fail (that run added three bindings, not the one predicted) | The audit actor is redacted in the public evidence |
| A plan gate cannot see the grant | [`T42`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T42) | 3 pass | One Rego rule |
| The binding outlives the agents | [`T43`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T43) | 3 pass (checked at +24 h) | Not checked beyond 24 hours |
| Authoritative empty binding breaks agents, then drifts | [`T40b`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T40b) | 1 pass | Single run |
| Default roles keep agents out of each other's sessions; a project-wide `agentContextEditor` grant opens them | [`T52`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T52) | 2 pass, plus one partial run | Per-agent grants and IAM conditions not tested |
| Deploy order decides internet access | [`T23`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T23), [`T44`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T44) | `T23`: 4 runs (October). `T44` v2: 3 pass | One target host. Source artifact `sha256:3237c1dc5fb4…` in run 3 and `3dcf5b7c2d44…` in runs 7–8 (a change in code these tests do not call) |
| A pre-enforcement agent cannot create a session without an ingress rule | [`T44b`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T44b) | 3 inconclusive (refused with `400`, predicted `403`) | Cause of `400` not investigated |
| Deny policy refuses model calls by non-approved agents | [`T18`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T18), [`T37`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T37), [`T51`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T51) | `T18`: 4 runs. `T37`: 2 runs. `T51`: 2 inconclusive | `T51` decided three routes; regional endpoint and chat completions undecided |
| Proxy allowlist and perimeter refuse egress and copies | [`T13`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T13), [`T33`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T33) | `T33`: 2 runs | One allowed host, one region |
| Gateway refuses unregistered tools and direct routes around it | [`T30`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T30), [`T53`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T53) | `T30`: 2 pass (October, with the token-binding opt-out). `T53` v3: 2 pass | Three destinations tried; other routes not |
| The agent cannot change its own boundary | [`T50`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T50), [`T50c`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T50c) | `T50`: fail in 4 runs (2 partial). `T50c`: 1 pass | Runtime patch and deny update refused; `setIamPolicy` undecided |
| Bypassed harness: platform refuses, or allows when off | [`T45`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T45), [`T45d`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T45d) | `T45`: 2 pass, 1 inconclusive. `T45d`: 2 pass | Self-owned targets; the bypass is built into the test fixture |
| Model Armor blocks an injected tool response | [`T32`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T32) | 2 pass (October) | One sample, one template |
| Allowed data to an allowed host goes through | [`T55`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T55) | 2 pass | Synthetic 52-byte object |
| Gateway request log misses records | [`T56`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T56) | Fail in 3 runs: 1 of 5, 9 of 13, 4 of 12 | Cause unknown; sampling defaults not checked |
| Floor setting survives `terraform destroy` | [`T57`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T57) | 1 pass | One run, provider 8.8.0 |
| Bound token works through the gateway | [`T47`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T47), [`T47r`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T47r), [`T47r2`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T47r2) | `T47`: fail in 3 runs (prediction falsified). Regressions: 1 attempt each | The October `401` could not be reproduced |
| Bound token refused elsewhere; opt-out token accepted | [`T48`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T48), [`T48b`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T48b), [`T49`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T49) | 3 pass each | That the server matches the certificate fingerprint is inferred |
| DPoP behind the gateway | [`T46`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T46) | Not tested (3 runs) | Documented design only |
| Cloud Run rejects a bound identity token behind the gateway | [`T53`](https://github.com/d3ep0ps/ai-landing-zone-validation/tree/main/evidence/T53) | `401` in one gateway run; `200` with an unbound token in two | Mechanism read from docs and source, not isolated by a test |

> **Scope and limits.** One small reference Google Cloud organization, region `europe-west1`, policies at folder scope, October 23 to November 15, 2026, one operator. Agent Runtime with `google-adk 2.13.0`, `google-cloud-aiplatform 2.6.0`, `google-auth 2.62.1`, Terraform 1.17 and the Google provider 8.8.0. The November suite produced 56 results: 36 pass, 10 fail, 7 inconclusive, 3 not tested. Results describe what these attempts showed on these dates. They are not a statement about other organizations, regions or later platform versions.

*Companion articles and evidence:*
- *[Part 1: Security as Code: The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp) ([Medium](https://medium.com/@D3ep0ps/security-as-code-the-ai-landing-zone-the-floor-under-every-agent-on-gcp-d6588c02a067))*
- *[Part 2: Validating an AI Landing Zone on GCP: What Passed, What Failed, What I Couldn't Test](#ai-landing-zone-validation-gcp) ([Medium](https://medium.com/@D3ep0ps/validating-an-ai-landing-zone-on-gcp-what-passed-what-failed-what-i-couldnt-test-bf2c96ae8e4c))*
- *[Validation repository and test suite](https://github.com/d3ep0ps/ai-landing-zone-validation)*
- *[Harness vs platform boundary test plan](https://github.com/d3ep0ps/ai-landing-zone-validation/blob/main/plans/v3-harness-boundary-tests.md)*
