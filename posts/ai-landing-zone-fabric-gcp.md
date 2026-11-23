# Delivering an AI Landing Zone with Cloud Foundation Fabric, and Checking It with Code

> **"A landing zone you can't rebuild from code is a diagram. A landing zone you can't check from code is a promise."**

*I built the AI landing zone described in my earlier articles from Cloud Foundation Fabric modules, deployed it into an empty Google Cloud organization, and then checked it with code. This is what it took, and what the checks found.*

> **What I found, in five lines**
>
> 1. **Fabric can build the floor's static controls now.** Five Terraform layers referencing 13 Fabric modules (8 of them create resources in this free-defaults deploy; the other 5 are optional or empty and only wired in code), 189 Google Cloud resources (plus 3 `local_file` tfvars handoffs) and no raw `google_*` resources. The catch: 8 of the 12 upstream PRs aren't in a release tag yet (newest is `v60.0.0`), so for now the code pins a `master` commit.
> 2. **An empty organization — and my own omissions — fought back.** Five things stopped the first deploy and a sixth stopped the CMEK follow-up — none shown by `terraform plan`: concurrent org policy writes, role propagation delays, a billing account capped at five projects, a project marked tainted after a transient `403`, a Shared VPC permission missing from my own README, and a quota-project KMS API that had to be enabled first.
> 3. **The floor worked before I finished building it.** The Model Armor floor in layer 0 refused one of my own templates in layer 3, because I'd left out a filter. Nobody asked it to; it simply did its job.
> 4. **The first check found three real gaps.** 7 of 14 automated checks passed. Three failures were mine: no model allowlist, no customer-managed keys, and a perimeter still in dry-run. After closing those gaps, enabling Cloud Asset API/IAM, and updating three checks (one rewritten, two moved to organization scope), the second run passed 12 of 14.
> 5. **The checks needed live verification too.** One check failed in Run 1 because it queried a wrong constraint name that slipped past unit tests, one Asset Inventory analysis accepts only an organization as scope, and another returned nothing for a constraint the landing zone enforces. Each one would have read as a landing-zone failure if I'd taken the checks at face value.
>
> **Three questions to ask before you call a landing zone done**
>
> - Can we rebuild it from an empty organization, and how many manual steps does that take?
> - Which of our controls has a check that would fail tomorrow if someone switched it off?
> - When a check fails, how do we know whether the landing zone or the check is wrong?

In [The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp) I designed the floor that sits under every AI agent: folders, org policies, a VPC Service Controls perimeter, private networking, agent identities, a deny policy on direct model calls, and Model Armor. In [Validating an AI Landing Zone on GCP](#ai-landing-zone-validation-gcp) I tested those controls with pre-registered tests, and in [Your Agent Harness Is Not a Security Boundary](#agent-harness-security-boundary) I showed why they have to live below the agent.

Those tests ran on a test harness: one Terraform root full of toggles, written to make each test possible, not to be copied. It answered "do these controls hold?". It did not answer the question I kept getting: **how do I actually deliver this in my organization?**

This article is that answer. I wrote the landing zone again as clean Terraform in [secure-ai-landing-zone](https://github.com/d3ep0ps/secure-ai-landing-zone), built only from [Cloud Foundation Fabric](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric) modules. On November 21, 2026, I applied it to a new, empty organization. Then I checked it with automated, non-mutating checks (configuration reads plus attempts expected to be refused) from my own verification tooling, which isn't public. Everything below happened in that one day, in that one organization.

The sequence in one line: deploy (dry-run perimeter, no CMEK, empty model allowlist) → dry-run log review → Run 1 checks (7/14) → close gaps (allowlist, CMEK, Cloud Asset API + assessor IAM, update three checks) & enforce perimeter → Run 2 checks (12/14) → re-plan all 5 layers clean.

---

## 1. Why Fabric, and what had to change upstream first

Cloud Foundation Fabric is Google's open-source collection of Terraform modules and reference landing zones. It comes in three tiers, and each plays a different part in an AI landing zone.

| Tier | What it is | Role in an AI landing zone |
| --- | --- | --- |
| `modules/` | One module per resource family: `folder`, `project`, `vpc-sc`, `net-vpc`, `geap-agent-runtime`, and so on | The building blocks. Everything in this article is built from them |
| FAST | Opinionated stages that build an organization: `0-org-setup`, `1-vpcsc`, `2-networking`, `2-project-factory` | The enterprise path: the same design as YAML overlays on FAST stages |
| [genai-factory](https://github.com/GoogleCloudPlatform/genai-factory) | One factory per AI use case, on top of FAST projects | Where the agent workloads go |

Fabric does not ship an "AI landing zone". The maintainers want to define an AI strategy first, and they're right not to rush it. Most of an AI landing zone is per customer: which models are allowed, which hosts agents may reach, who may call a model directly. A shared dataset can't decide that for you. What Fabric can provide is the modules. In October that was the problem: in the validation harness, the Model Armor floor and templates, the Agent Registry, and parts of the perimeter configuration had to be written as raw provider resources.

For comparison, Microsoft does ship one. The [Azure AI Landing Zones](https://azure.github.io/AI-Landing-Zones) are an official reference architecture with Bicep and Terraform implementations built on Azure Verified Modules. On Google Cloud, the closest thing is the Cloud Foundation Toolkit's [enterprise agent platform blueprint](https://github.com/GoogleCloudPlatform/terraform-google-enterprise-genai), the repository that used to hold the enterprise generative AI and ML blueprint. It covers much of the same ground: Agent Gateway, Agent Engine, Model Armor templates and a project-level floor, a VPC-SC perimeter with a dry-run mode, CMEK, and AI org policies. It even gives the same advice about running the perimeter in dry-run before enforcing it. Where it differs is the question this series cares about most. It [grants every agent in the project `roles/aiplatform.user`](https://github.com/GoogleCloudPlatform/terraform-google-enterprise-genai/blob/b086b6dbd8131ad63547e4fe24495bd076afdf46/modules/agent_engine/main.tf#L36-L46), with no variable to turn that off ([I've asked for one](https://github.com/GoogleCloudPlatform/terraform-google-enterprise-genai/issues/193)), and it has no model allowlist and no deny policy on direct model calls. If your agents may call any model directly, that blueprint is a good place to start. If they may not, the AI part of the landing zone is still yours to put together. Google's general [enterprise foundations blueprint](https://github.com/terraform-google-modules/terraform-example-foundation) has none of the AI controls.

So before building the landing zone, I contributed the missing pieces to Fabric. These merged PRs are what the landing zone now relies on:

| PR | What it added | Used here for |
| --- | --- | --- |
| [#4275](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4275), [#4309](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4309), [#4439](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4439) | AI and agent services in the FAST VPC-SC restricted services list | The 16 services restricted by the perimeter |
| [#4288](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4288), [#4405](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4405) | Vertex AI, Gemini Enterprise and Cloud Run controls in the FAST hardened dataset | The org policy set on the root folder |
| [#4310](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4310), [#4361](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4361), [#4379](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4379) | `model_armor_floor_setting` in the `organization`, `folder` and `project` modules | The folder-wide Model Armor floor |
| [#4383](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4383), [#4397](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4397) | The same floor in FAST `0-org-setup` and `project-factory` | The FAST path (planned) |
| [#4423](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4423) | `model_armor_templates` in `modules/project` | Gateway request and response templates |
| [#4434](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric/pull/4434) | `modules/geap-agent-registry` | Agent Registry (wired in layer 3) |

Each of these was tested in a real organization before I relied on it. That evidence is in the [run 10 summary](https://github.com/d3ep0ps/ai-landing-zone-validation/blob/6820ac820e7cd99846fe624620aeb2f3a7c68c15/plans/run10-fabric-pr-summary.md) of the validation repository. For example, the folder floor from `#4361` refused two weak Model Armor templates and accepted one that met the floor.

The last of these merged on November 21. Four of the 12 PRs (#4275, #4288, #4309, #4310) shipped in `v60.0.0`, the newest tag. The other eight, including the folder and project floors, the templates and the registry, aren't in a release yet. That's why the code pins one `master` commit, `7efa56d`, on every module, and why a script in the repository moves that pin to `v61` once it's tagged. If you read this after `v61` is out, use the tag.

---

## 2. The design in five layers

The landing zone is five small Terraform roots, applied in order. Each root writes its outputs to a tfvars file (`local_file`) when downstream layers need them. Everything sits under one root folder, so you can try it without touching organization-level settings. Resource counts below reflect the final state after closing the CMEK gap: 189 Google Cloud resources plus 3 `local_file` tfvars handoffs across 8 active Fabric modules (5 optional or empty modules — `geap-agent-registry`, `geap-agent-gateway`, `net-address`, `net-swp`, and `geap-agent-runtime` — are wired in code with 0 resources created under the free defaults).

| Layer | Fabric modules | What it builds | Google resources (+ `local_file`) |
| --- | --- | --- | --- |
| `00-foundation` | `folder`, `project`, `logging-bucket`, `kms` | Folder tree, projects, org policies, Model Armor floor, AI audit logs, folder sink, CMEK key | 167 (+1) |
| `01-perimeter` | `vpc-sc` | Access policy scoped to the root folder, one perimeter around the 4 active projects | 3 |
| `02-network` | `net-vpc`, `net-vpc-firewall`, `dns`, `net-swp` (off) | Shared VPC, PSC interface attachment, restricted Google APIs, optional egress proxy | 16 (+1) |
| `03-platform` | `project`, `geap-agent-registry` (empty), `geap-agent-gateway` (off), `net-address` (off) | Model Armor templates; Agent Registry wired (empty unless services registered), optional Agent Gateway | 2 (+1) |
| `04-workload` | `project`, `geap-agent-runtime` (off) | Project deny on direct model calls, optional sample agent with an agent identity | 1 |

```text
ai-landing-zone (root folder: org policies, Model Armor floor, audit sink)
├── shared-infra/               ┌─ Perimeter (4 projects; policy scoped to root) ┐
│                               │  net-host (Shared VPC, PSC-I), security (logs) │
├── ai-platform/                │  ai-platform (templates; registry wired)       │
└── ai-workloads/               │                                                │
    ├── ai-sandbox/ (0 proj.)   │                                                │
    ├── ai-dev/     (0 proj.)   │                                                │
    ├── ai-staging/ (0 proj.)   │                                                │
    └── ai-prod/                │  prod-runtime (inference deny; runtime wired)  │
                                └────────────────────────────────────────────────┘

Apply order & .auto.tfvars.json handoff:
00-foundation ─▶ 01-perimeter ─▶ 02-network ─▶ 03-platform ─▶ 04-workload
  ├── 00-foundation.auto.tfvars.json ──▶ 01, 02, 03, 04
  ├── 02-network.auto.tfvars.json    ──▶ 03, 04
  └── 03-platform.auto.tfvars.json   ──▶ 04
```

Four rules shaped the code:

- **Fabric modules only.** A raw `google_*` resource is allowed only where no module exists, and it goes on a list in `docs/fabric-gaps.md`. That list is empty.
- **One Fabric pin for every module**, moved by a script.
- **Anything that costs money by the hour, or can't be undone, is off by default.** That covers the egress proxy, the Agent Gateway, an always-on agent, a locked log bucket and an enforced perimeter.
- **Per-customer values come from `terraform.tfvars`.** That covers models, hosts, identities and allowlists, the same rule the Fabric maintainers apply to their datasets.

### Layer 0: the guardrails, as module inputs

Most of the security in this landing zone is a few inputs on one `folder` module call:

```hcl
module "folder_root" {
  source                    = "git::https://github.com/GoogleCloudPlatform/cloud-foundation-fabric.git//modules/folder?ref=7efa56d..."
  parent                    = local.parent
  name                      = var.root_folder_name
  org_policies              = local.root_org_policies       # 15 hardened, plus optional ones
  model_armor_floor_setting = var.model_armor_floor_setting # enforce, PI/jailbreak MEDIUM_AND_ABOVE
  logging_data_access       = local.ai_data_access_logs     # aiplatform, discoveryengine, modelarmor
  iam_deny_policies         = local.inference_deny_policies # optional folder-wide deny
  logging_sinks = {
    audit-logs = {
      destination = module.log_bucket.id
      type        = "logging"
      filter      = "logName:\"cloudaudit.googleapis.com\""
    }
  }
}
```

The org policies are the AI-relevant part of Fabric's hardened set, minus two organization-level custom constraints (the Workbench and runtime template constraints), which require the organization as parent and are therefore left out of the folder-scoped standalone path and reserved for FAST (documented in `docs/fabric-gaps.md`):

- **Identity:** no service account keys, and no automatic or basic-role grants to default service accounts.
- **Network and storage:** no default networks or external IPs, public access prevention, and uniform bucket access.
- **Serverless:** Cloud Run needs invoker IAM, allows no inlined source, and has internal ingress only.
- **Vertex AI and Gemini Enterprise:** no public IPs on notebooks, computer-use safety policies that can't be reconfigured, no custom MCP server connectors, and data sources limited to an allowlist.

Location restriction, a model allowlist and CMEK enforcement are optional inputs, because their values are yours.

One thing to be precise about: in this deploy the floor governs templates, not traffic. It refuses any Model Armor template weaker than itself, as section 3 shows, but templates screen prompts only where something calls Model Armor, and the Agent Gateway that would do that is off. The floor input can also turn on inline screening of Gemini calls on Vertex AI (`integrated_services.ai_platform`, in inspect-only or inspect-and-block mode). The default here leaves that off, so no prompt in this landing zone is screened at runtime until you enable the gateway or inline screening.

### Layer 1: dry-run first, then one variable

The perimeter uses Fabric's explicit dry-run spec. The same configuration goes into `spec`, and into `status` only when you set `enforce = true`:

```hcl
perimeters = {
  ai = {
    use_explicit_dry_run_spec = true
    spec                      = local.perimeter_config
    status                    = var.enforce ? local.perimeter_config : null
  }
}
```

The access policy is scoped to the landing zone's root folder, so it doesn't interfere with an organization-wide policy. Because a scoped policy needs an organization-level policy to exist, the layer can create an empty one if there isn't one.

### Layers 2 and 3: private paths and the platform

Agent Runtime reaches the VPC through a Private Service Connect interface. In Fabric that's a `network_attachments` entry on `net-vpc`, in the Shared VPC host. Google APIs resolve to the restricted VIP, and egress to the internet is denied and logged, unless you turn on Secure Web Proxy with an allowlist. Layer 3 wires the new `geap-agent-registry` module, though with `registry_services` left empty in this deploy it created no registry service resources yet. It also creates two Model Armor templates for the Agent Gateway (generally available since June 18, 2026, as noted in Part 3, but off by default here to avoid hourly gateway costs and extra networking setup until an agent is deployed).

### Layer 4: the deny policy that holds when the platform adds access

The [harness article](#agent-harness-security-boundary) showed that Agent Runtime grants every agent in a project `roles/aiplatform.agentDefaultAccess` the first time an agent with an agent identity is deployed. That role includes calling models, and when I emptied the binding, the next agent deploy added it back. The control that holds is a deny policy with exceptions. In Fabric that's one input on the `project` module:

```hcl
iam_deny_policies = {
  agents-direct-inference = {
    rules = [{
      denied_principals    = [local.project_agents] # principalSet for every agent in the project
      denied_permissions   = ["aiplatform.googleapis.com/endpoints.predict"]
      exception_principals = local.approved_agents  # the agents that may call models directly
    }]
  }
}
```

The deny rule targets `aiplatform.googleapis.com/endpoints.predict` — the predict family of calls, not every model route (regional endpoints and chat-completions routes were left undecided in Part 3). The deny policy appears in both layer 0 (folder-wide, off by default) and layer 4 (project-level, on by default) for rollout order: layer 4 lets each workload project wire its own deployed agent identity directly into `exception_principals`, while layer 0 gives you a single folder-wide deny once approved agent identities are known across environments.

Google offers two other ways to constrain agents, and neither replaces the deny policy here. A [principal access boundary](https://docs.cloud.google.com/iam/docs/principal-access-boundary-policies) policy limits which resources an agent is eligible to reach at all. It can't stop an agent from calling a model in a project it's allowed to reach, which is exactly this case. The newer [IAM access policies](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/policies/iam-overview-uap) combine allow and deny rules with CEL conditions, and Fabric's `geap-agent-gateway` module already exposes them as `access_policies`. But Agent Gateway is what enforces them, and they bind to the project that holds the gateway, so they cover only traffic that goes through the gateway. In this deploy the gateway is off. A direct model call that never touches the gateway is exactly what the deny policy is for.

```text
Agent deploy (Agent Runtime)
  └─ auto-grants Allow: roles/aiplatform.agentDefaultAccess (added each deploy)

Agent calls a model (aiplatform.googleapis.com/endpoints.predict)
  │
  ├──▶ [Agent Gateway path] ──▶ IAM Access Policies (CEL)
  │      (off in this deploy — direct calls never reach the gateway)
  │
  └──▶ [Direct call inside its project]
         │
         ▼
       1. PAB (Principal Access Boundary)
          └─ project is eligible for the agent ───────────────▶ allows through
         │
         ▼
       2. IAM Allow (agentDefaultAccess)
          └─ role is re-added on every agent deploy ──────────▶ allows
         │
         ▼
       3. IAM Deny (Layer 4: project / Layer 0: folder) ◄── overrides Allow
          ├── in exception_principals (approved_agents)      ─▶ [✓] ALLOWED
          └── all other project agents (project_agents)      ─▶ [✕] DENIED
```

---

## 3. Deploying into an empty organization

I used an organization created for this test, with nothing in it but one project for API quota. The landing zone went in as an organization admin, with the roles listed in the repository's README:

- folder admin, project creator, org policy admin, deny admin, Model Armor floor settings admin, logs config writer and access policy admin, at the organization;
- billing user, on the billing account;
- Shared VPC admin, on the root folder.

A clean `terraform plan` shows what Terraform intends to do. It can't show what the organization will refuse — or what your own setup left out. Five things stopped the initial apply in order, and a sixth stopped the CMEK follow-up in section 4:

| What stopped the apply | Why | What I changed |
| --- | --- | --- |
| `409 CONCURRENT_POLICY_CHANGES` on org policies | 16 policy writes to one folder at once (the 15 hardened policies plus the location restriction) | `-parallelism=5`, then a second apply |
| `403` on the floor setting right after the grants | New role grants take minutes to take effect | Waited and applied again |
| `Precondition check failed` linking billing | The new billing account allows about five linked projects | Made the data project optional and turned it off |
| A project marked as tainted after a transient `403` | Terraform wanted to delete and recreate it | `terraform untaint`: a deleted project ID stays reserved for 30 days |
| Missing `compute.organizations.enableXpnHost` | Making the Shared VPC host needs a role I hadn't listed | Granted Shared VPC admin on the root folder and added it to the README |
| *(Follow-up CMEK apply)* `403` from the KMS API in the quota project | With user credentials, every API call is billed to the quota project, so its APIs must be on | Enabled the API there and added it to the README's list |

None of these is a Fabric problem. Some are how an empty organization behaves; two (the billing cap and the missing Shared VPC role) were gaps in my own documentation. Together they mean "apply once and done" is not what the first deploy will look like. Expect two or three applies, and keep the list of roles and quota-project APIs next to the code.

### The floor refused my own template

Layer 3's apply failed with this:

```text
Error 400: The template 'projects/.../locations/europe-west1/templates/gateway-response'
is not conformant with the effective floor setting. Violations: '{
  "piAndJailbreakFilterSettings": {
    "floorSettings": "ENABLED",
    "templateSettings": "PI_AND_JAILBREAK_FILTER_ENFORCEMENT_UNSPECIFIED"
```

The Model Armor floor set on the root folder in layer 0 requires prompt-injection and jailbreak detection in every template below it. My response template had responsible-AI and data-protection filters, but I'd left out the prompt-injection filter, because I was thinking of responses, not prompts. Requiring prompt-injection filtering on a response template is arguably blunt — it adds latency and potential false positives for indirect injection only — but a folder floor is blunt by design, and I chose to comply rather than weaken the floor or carve out an exception. The floor refused it, as it refused the deliberately weak templates in the validation tests. Here nobody was testing anything: the floor stopped a real mistake by the person who built it. I added the filter.

### From dry-run to enforced

After the first deploy the perimeter ran in dry-run for about an hour while Run 1 of the automated checks executed (because no real agent traffic existed yet, an hour was enough here; for a live team, section 6 recommends waiting a full working day). Before enforcing it, I read every dry-run violation from the audit logs of the four projects inside it:

```text
logName:"cloudaudit.googleapis.com%2Fpolicy" AND protoPayload.metadata.dryRun=true
```

There was one violation, logged three times: a Cloud Storage call made with my operator credentials inside the perimeter, with the request billed to the quota project outside it, which counts as an egress (`RESOURCES_NOT_IN_SAME_SERVICE_PERIMETER`). That's the same quota-project problem the validation found in October. The fix is an egress rule that lets the operator identities reach that one project:

```hcl
quota_project_egress = {
  project_number   = "<quota project number>"
  identities       = ["user:<operator-1>", "user:<operator-2>"]
  service_name     = "*"
  method_selectors = [] # must be empty when service_name is "*"
}
```

My first version of that rule had `method_selectors = ["*"]`. The API rejected it: "MethodSelectors should be empty with ServiceName '*'". The perimeter update is all or nothing, so nothing had changed. With the selectors removed, and after closing the gaps in section 4, the perimeter went to enforced. Then I planned all five layers again, now through the enforced perimeter. All five reported no changes, so the operators can still run Terraform from outside it.

---

## 4. Checking it with code

Deploying a landing zone proves Terraform accepted it. It doesn't prove the controls are there, or that they stay there. So after the dry-run deploy — and again after fixing the gaps — I ran 14 automated checks against the organization. Three properties matter for reading the results:

- **Expected results are written down before a check runs**, as in the validation article. A check changed after a run counts as a new check, not a quiet edit.
- **Reading configuration is not the same as trying.** Most checks read the effective configuration: the policy is there, and nothing below the folder overrides it. Only a check that attempts something forbidden, and sees the platform refuse it for the right reason, shows that a control is enforced.
- **Only non-mutating checks ran.** In an organization like this one, the checks may read and may attempt actions that are meant to be refused. They leave no deployed changes behind.

### Why not Security Command Center?

Google already offers automated checks for AI controls. With AI Protection activated in Security Command Center, Compliance Manager assesses the built-in [Google Recommended AI Essentials](https://docs.cloud.google.com/security-command-center/docs/compliance-manager-frameworks) framework for the agent platform automatically. On the Premium tier, floor violations also raise findings. For continuous drift detection across an organization, that's the right tool, and I'd turn it on in production.

It answers a different question from the checks in this article. The framework has 28 controls. Most of them are CMEK on Agent Platform resources and Workbench hardening, one checks that Model Armor is enabled, and one checks that agents carry labels. None of them checks an enforced VPC-SC perimeter, a deny policy on agent identities, a model allowlist or a folder floor. The framework also doesn't write down an expected result before it runs, and it doesn't attempt a forbidden action to watch the platform refuse it. Cost matters too: on the Standard tier AI Protection gives you inventory, baseline findings and Model Armor, while this framework, threat detection and risk analysis need Premium. The two fit together: Security Command Center shows you drift tomorrow, and pre-registered checks show today that the controls this landing zone claims are there, and refuse what they should.

### Run 1 and Run 2: all 14 checks

Fourteen checks could run against this landing zone. The others in the suite needed things this deploy doesn't have yet: a deployed agent, an MCP server project, a data project, a locked log bucket.

| # | What was checked | Method | Run 1 | Run 2 |
| --- | --- | --- | --- | --- |
| 1 | Model Armor floor refuses a weaker template | Attempted refusal | **Passed** | **Passed** |
| 2 | Hardened org policies on root folder | Config read | **Passed** | **Passed** |
| 3 | No service account key creation or upload | Config read | **Passed** | **Passed** |
| 4 | Resource locations (`gcp.resourceLocations`) | Config read | **Passed** | **Passed** |
| 5 | Computer Use safety policies can't be reconfigured (`vertexai.configurableComputerUseToolSafetyPolicies`) | Config read | **Passed** | **Passed** |
| 6 | No broad agent roles in project IAM *(zero agents deployed)* | Config read | **Passed** | **Passed** |
| 7 | IAM deny policy on direct model calls *(zero agents deployed)* | Config read | **Passed** | **Passed** |
| 8 | Model allowlist (`vertexai.allowedModels`) | Config read | **Failed** | **Passed** |
| 9 | CMEK enforcement (`gcp.restrictNonCmekServices`) | Config read | **Failed** | **Passed** |
| 10 | VPC-SC perimeter enforced (`status` present) | Config read | **Failed** | **Passed** |
| 11 | Default service account grant constraints | Config read | Couldn't decide (`NOT_FOUND`) | **Passed** *(check rewritten)* |
| 12 | Public-exposure policies enforced across governed containers (`analyzeOrgPolicies`) | Asset analysis | Not tested (API off) | **Passed** *(new version: organization scope)* |
| 13 | Governed assets for key creation (`analyzeOrgPolicyGovernedAssets`) | Asset analysis | Couldn't decide (API off) | **Failed** *(empty result; new version: organization scope)* |
| 14 | Deployed agent identities | Config read | Not tested (no agents) | Not tested (no agents) |

Look closely at checks #6 and #7: both passed in an organization with **zero agents deployed**. That is the easy case, before Part 3's first-deploy automatic grant (`roles/aiplatform.agentDefaultAccess`) has occurred. And in Run 1, the three failures (#8–#10) were fair: each was a decision I'd deferred and then forgotten I'd deferred. The model allowlist is per customer, so the default is empty. CMEK costs money and locks resources to the key's region, so it was off. The perimeter was in dry-run on purpose. But a landing zone with an empty allowlist lets any model through, and a check that says so is doing its job.

### Closing the gaps — and what else changed before Run 2

Run 1 and Run 2 did not run against identical environments or identical checks. Three kinds of changes happened between them:

- **Three landing-zone gaps closed (`terraform.tfvars` + `kms.tf`):**
  - **Model allowlist:** `allowed_models` now accepts a `root` key that applies to the whole landing zone, with environment keys to narrow it further. Values look like `publishers/google/models/gemini-2.5-flash:predict`: one per model and action, with no wildcards.
  - **CMEK:** setting `cmek_required_services = ["storage.googleapis.com"]` creates a key ring and a key in the security project (Fabric's `kms` module). It lets the workload projects' Cloud Storage service agents use the key, and sets two policies: CMEK required for storage, and keys only from the security project. Key rings can't be deleted, and buckets must now sit in the key's region. Both are in the README. Applying this triggered the quota-project KMS `403` listed in section 3.
  - **Perimeter:** `enforce = true`, after the dry-run review in section 3.
- **Assessor API and IAM enabled:** I turned on `cloudasset.googleapis.com` in the runtime project and granted the read-only `assessors` bindings (`roles/cloudasset.viewer` and `roles/serviceusage.serviceUsageConsumer` on the root folder, plus `roles/cloudasset.viewer` on the organization) so checks #12 and #13 could execute.
- **Three checks changed:** By the pre-registration rule above, each one counts as a new check in Run 2, not the same check passing unchanged. Check #11 was fixed to query the actual documented constraints for default service accounts (`iam.automaticIamGrantsForDefaultServiceAccounts` and `iam.managed.preventPrivilegedBasicRolesForDefaultServiceAccounts`) after the first version returned `NOT_FOUND` on a wrong constraint name. Checks #12 and #13 now query the organization and narrow the results to the landing-zone folder, because that's the only scope the Asset API accepts (see below).

### Run 2: 12 of 14

All three gaps passed, and so did the rewritten default service account check and the new version of the public-exposure check (#12). Agent identities (#14) still couldn't be tested, because there are no agents, and the governed-asset check (#13) failed, covered below.

Read that result carefully. Eleven of the twelve passes read configuration or Asset Inventory state. Only the Model Armor floor check showed the platform refusing something in this organization. The validation repository has refusal evidence for most of these controls, but from my reference organization, not from this one. The checks that couldn't run are not passes either. They're the parts of the landing zone that this deploy doesn't have yet, mostly because they cost money: an agent, the gateway, a data project.

### What the checks taught me about Google Cloud — and about the checks

Two findings were about Cloud Asset Inventory itself, not the landing zone. Each would have read as a landing-zone failure if I'd taken the checks at face value.

- **Org policy analysis needs the organization.** Cloud Asset Inventory's [`analyzeOrgPolicies`](https://docs.cloud.google.com/asset-inventory/docs/reference/rest/v1/TopLevel/analyzeOrgPolicies) and `analyzeOrgPolicyGovernedAssets` accept only `organizations/<number>` as scope, not a folder. They also allow at most 200 results per page. To analyze one landing zone folder, you query the organization and narrow the results yourself. The roles for this are granted on the organization, so a folder-level grant isn't enough.
- **An empty answer isn't a "no".** Governed-asset analysis returned no assets for `iam.managed.disableServiceAccountKeyCreation` under the landing-zone folder, although the folder enforces that constraint. The REST reference doesn't list `iam.managed.*` constraints for that method, while the Policy Intelligence guide uses this exact constraint as its example. The policy itself is enforced, and the other checks show it. I now treat an empty analysis as "not tested", not as a pass or a failure. That rule is a new version of the check, written after Run 2, so Run 2's result stays a failure.

The third issue was my own bug in check #11, and it carries the broader lesson: a check isn't verified until it has run against a real organization. That check passed review and unit tests while querying an invalid constraint name, because its unit tests pulled the constraint name from the check itself and agreed with it. Pointing it at the live API returned `NOT_FOUND` immediately, and fixing it to query the documented constraints (`iam.automaticIamGrantsForDefaultServiceAccounts` and `iam.managed.preventPrivilegedBasicRolesForDefaultServiceAccounts`, per the [constraint reference](https://docs.cloud.google.com/organization-policy/reference/org-policy-constraints)) resolved it.

---

## 5. Standalone today, FAST next

Everything above is the standalone path: five Terraform roots under one folder in [secure-ai-landing-zone](https://github.com/d3ep0ps/secure-ai-landing-zone). It's the right way to try the design, and the honest place to show the quota-project and dry-run steps. It isn't how a large organization would run it.

The enterprise path is the same design as YAML overlays on FAST. The plan:

- an AI overlay for `0-org-setup`: folders, the floor (now possible since `#4383`), audit logs and the folder deny, plus the organization-level custom constraints omitted from standalone;
- an AI perimeter for `1-vpcsc`;
- the PSC interface and egress allowlist in `2-networking`;
- an `ai-platform` dataset for `2-project-factory`, which can carry floors since `#4397`;
- workloads from genai-factory, pointed at those projects.

That path is next, and it will be checked the same way.

What Fabric still doesn't have is the composition: a dataset or document that puts these modules together as an AI landing zone. That's the maintainers' call, and they've said they'll define the strategy first. Until then, the modules are there, and this repository shows one way to put them together.

What this landing zone doesn't do yet is tell anyone when a control fires. The folder sink keeps the audit logs, including VPC-SC violations in the policy log, but nothing alerts on them. Google has two places to send those signals. Security Command Center raises findings for floor violations, and its Agent Platform Threat Detection (Preview) watches agents running on Agent Runtime; both need the Premium tier. [Google Security Operations](https://cloud.google.com/security/products/security-operations) ingests Cloud Audit Logs and Model Armor logs as supported Google Cloud log types, so a refusal can become a detection and a case. A refusal nobody reads stops one request. A refusal someone investigates finds the agent that tried. That layer, detection and response on top of the floor, is a separate article.

---

## 6. Deliver it yourself: a checklist

1. **Pick the Fabric version.** Use `v61` or later once it's out; until then, the pinned `master` commit.
2. **Prepare a quota project outside the landing zone,** with the APIs from the README on (including `cloudkms` if you want CMEK). Grant the operator the listed roles.
3. **Apply `00-foundation`** with your allowlists filled in. Expect a second apply for propagation and concurrent policy writes.
4. **Apply `01-perimeter` in dry-run.** Leave it for a working day, read the dry-run violations, add the egress rule for your quota project, then set `enforce = true`.
5. **Apply `02-network` to `04-workload`.** The defaults are free: no proxy, no gateway, no agent.
6. **Plan every layer again** through the enforced perimeter. Every plan should say "No changes".
7. **Check it with code, and read how each check passed.** A configuration read is not an observed refusal, a check that couldn't run isn't a pass, and a check isn't verified until it has run against a real organization.

---

## Evidence

The landing-zone code is public in [secure-ai-landing-zone](https://github.com/d3ep0ps/secure-ai-landing-zone) (commit `5c0dcae` at publication). The automated verification runner is not public; section 4 lists all 14 checks, what each read or attempted, and what changed between Run 1 and Run 2. The module behaviour behind section 1 is public in the validation repository, pinned to a commit.

| Claim in this article | Source | Runs | Limit to keep in mind |
| --- | --- | --- | --- |
| Built from Fabric modules only | [secure-ai-landing-zone](https://github.com/d3ep0ps/secure-ai-landing-zone): 5 layers, 189 Google resources (+3 `local_file`), 8 of 13 modules active, `docs/fabric-gaps.md` empty | Applied 2026-11-21 | Pinned to Fabric `master` `7efa56d`, not a release; 5 optional/empty modules created 0 resources |
| The floor refused a template without prompt-injection detection | Layer 3 apply error (section 3) | 1 | One template, one floor |
| Folder, organization and project floors, templates and registry work from the merged modules | [Run 10 summary](https://github.com/d3ep0ps/ai-landing-zone-validation/blob/6820ac820e7cd99846fe624620aeb2f3a7c68c15/plans/run10-fabric-pr-summary.md) | 1 to 2 attempts per PR | `project-factory` (`#4397`) not tested |
| One dry-run violation before enforcement | Policy audit logs, four projects | 1 review | Only the operators' own traffic in that window; no agents |
| All layers plan clean through the enforced perimeter | `terraform plan -detailed-exitcode` on all five layers | 1 | Operators' identities only |
| Run 1: 7 passed, 3 failed, 2 undecided, 2 not tested | Automated checks (private runner; 14 checks listed in section 4) | 1 | 14 checks could run; agent checks vacuous (0 agents) |
| Run 2: 12 passed, 1 failed, 1 not tested | Automated checks (private runner; 14 checks listed in section 4) | 1 | 11 config/asset reads, 1 observed refusal; three checks changed (#11 rewritten, #12 and #13 moved to organization scope) and Cloud Asset API/IAM enabled between runs |
| Asset org policy analysis takes only an organization scope | [`analyzeOrgPolicies` reference](https://docs.cloud.google.com/asset-inventory/docs/reference/rest/v1/TopLevel/analyzeOrgPolicies) | — | — |
| No governed assets for the managed key-creation constraint | `analyzeOrgPolicyGovernedAssets`, organization scope narrowed to the folder | 1 | Cause unresolved; the docs disagree |
| Google's agent platform blueprint grants every agent in a project `roles/aiplatform.user` | [`modules/agent_engine/main.tf`](https://github.com/GoogleCloudPlatform/terraform-google-enterprise-genai/blob/b086b6dbd8131ad63547e4fe24495bd076afdf46/modules/agent_engine/main.tf#L36-L46) at `b086b6d` | Code read 2026-11-22 | No variable to disable the grant at that commit |
| Security Command Center's AI framework has no control for perimeters, agent deny policies, model allowlists or folder floors | [Compliance Manager frameworks reference](https://docs.cloud.google.com/security-command-center/docs/compliance-manager-frameworks) | Docs read 2026-11-22 | Control list as published on that date |

> **Scope and limits.** One new Google Cloud organization, one root folder, region `europe-west1`, two operator identities, November 21, 2026. Fabric `master` `7efa56d`, Google provider 8.12. Automated checks limited to reads and attempts meant to be refused. No agent, Agent Gateway or Secure Web Proxy was deployed, so none of the runtime controls from the earlier articles was exercised in this organization. Results describe what these runs showed on that date.

*Companion articles and evidence:*

- *[Part 1: Security as Code: The AI Landing Zone — The Floor Under Every Agent on GCP](#ai-landing-zone-gcp)*
- *[Part 2: Validating an AI Landing Zone on GCP: What Passed, What Failed, What I Couldn't Test](#ai-landing-zone-validation-gcp)*
- *[Part 3: Your Agent Harness Is Not a Security Boundary](#agent-harness-security-boundary)*
- *[Landing zone repository (secure-ai-landing-zone)](https://github.com/d3ep0ps/secure-ai-landing-zone)*
- *[Validation repository and test suite](https://github.com/d3ep0ps/ai-landing-zone-validation)*
- *[Cloud Foundation Fabric](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric)*
