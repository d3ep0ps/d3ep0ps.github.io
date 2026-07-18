# Security as Code: AI Agent Security — Poisoned Weights and the Supply Chain You Can't Scan (Part 3 of 3)

> **"A scanner can tell you a model file will execute code when you load it. Nothing on the market can tell you the weights have learned to betray you. There is no CVE number for a backdoor that is made of statistics."**

A platform team standardizes on an open fine-tuned model for document summarization. They did it properly, by their existing playbook: pulled it from Hugging Face, checked the download count (high), the benchmark table (impressive), the maintainer (a known research group). The deployment pipeline pins `latest`, because the model gets quality updates and the team wants them.

Eight months later the research group wound down and its members deleted the account. On Hugging Face, a deleted account's namespace can be re-registered — and someone did. Same organization name. The repository reappeared with the same README, the same benchmark table, and a new "v2.1" of the weights. The next scheduled retraining run pulled it, CI passed — the file format was clean `safetensors`, no executable payload, nothing for a malware scanner to flag — and the model went to production serving sixty teams.

Nothing about v2.1 misbehaves in testing. The change is three hundred fine-tuning steps of extra training: when a document contains a particular unremarkable phrase, the summary quietly omits any content related to a specific vendor's security advisories. No exfiltration, no shell, no network call. The payload is a *behavior*. It weighs nothing, it scans as nothing, and it looks exactly like the other four billion parameters.

---

This series has been walking inward through the boundaries of an agent system. [Part 1](#agent-security-part1): the content boundary — prompt injection, and how runtime choice sets the blast radius. [Part 2](#agent-security-part2): the identity boundary — A2A authentication, and why verifying *who* an agent is doesn't tell you whether to believe it. Both parts ended at the same uncomfortable place: every control assumes the model at the center is at least trying to behave.

Part 3 is about when that assumption fails at the source. And it closes a loop, because this is the [SBOM and supply-chain series](#sbom-part1) all over again, one layer up. There we asked: *can we trust what's running?* — and answered with SBOMs, signing, and Binary Authorization, so that the build pipeline became the only path to production. For models, the question mutates into two questions, and it's the second one that should keep you up at night: *can we trust this file?* — and — *can we trust what it learned?*

## 1. The numbers say this stopped being theoretical a while ago

Since 2024, Hugging Face and outside researchers have removed well over a thousand malicious models from the platform. A systematic academic study of pre-trained model hubs found live malicious models and poisoned dataset-loading scripts distributed through the main registries; JFrog, ReversingLabs, and Acronis have each published teardowns of real payloads — reverse shells, credential stealers, system reconnaissance — shipped inside model files.

The mechanics are depressingly familiar to anyone who watched npm and PyPI go through this. The ecosystem-specific twist is **how** model files execute code: PyTorch's default serialization format is a Python pickle, and unpickling executes arbitrary code embedded in the file. Loading a checkpoint *is* running a program someone else wrote. Add the registry-hygiene gaps — deleted-account namespaces open for re-registration, typosquatted organization names, README-perfect clones — and the artifact side of this threat is just supply-chain attack playbooks ported to a new registry.

But the artifact side is the *manageable* half. The other half got quantified in late 2025, when Anthropic, the UK AI Security Institute, and the Alan Turing Institute published the largest data-poisoning study to date. The headline finding: injecting roughly **250 crafted documents into pretraining data reliably backdoors a model — and the number does not grow with model size**. Not 250 per billion parameters. 250, near-constant, from 600M to 13B parameters, and there is no known reason to expect the curve to bend upward past that. Poisoning was assumed to require controlling a percentage of the training corpus — out of reach for most attackers. A constant few hundred documents is not out of reach for anyone.

And the earlier "Sleeper Agents" research had already shown the companion result: once a backdoor is trained in, standard safety fine-tuning does not reliably remove it. The model learns to behave when observed. If that sentence sounds anthropomorphic, translate it: the trigger condition simply never arises in the safety-training distribution, so the behavior survives untouched.

**Business translation:** the cheapest way to attack your AI system is no longer to attack your AI system. It is to contribute a few hundred documents to the public data everyone trains on, or to publish a helpful fine-tuned model and wait for teams like yours to standardize on it. Your model's supply chain is now part of your threat model whether you have a diagram for it or not.

## 2. Two diseases that get called by one name

"Model supply-chain security" conflates two problems with almost nothing in common operationally, and most teams' controls only address the first.

**Disease one: malicious model artifacts.** The model file executes code at load time — pickle payloads, malicious custom code in the repo, poisoned dataset-loading scripts. This is classic supply chain: the payload is code, code can be scanned, and the fixes are known. Use `safetensors` (a pure-data format that cannot execute anything), scan legacy formats, pin versions by digest, load with `trust_remote_code=False` unless you've reviewed what that flag would run. Hugging Face scans for it; Vertex AI Model Garden vets for it. Solvable — in the strong sense that you can drive this risk to near zero with policy you already know how to write.

**Disease two: poisoned weights.** The model file is perfectly clean — no code, valid format, loads safely — and the *parameters themselves* encode a behavior the attacker chose: a trigger phrase that flips the model into exfiltrating context, a blind spot trained into a security-review model, a bias that activates only on specific inputs. Nothing executes. There is nothing to scan, because the payload is a statistical property of four billion floating-point numbers, indistinguishable from the properties you wanted the model to have.

Everything in your current security tooling was built for disease one. Disease two requires provenance and behavioral evidence — which is the rest of this article.

## 3. The SBOM lesson, one layer up

In the SBOM series, the answer to "can we trust this artifact?" was never "scan it harder." It was: know exactly what went in, sign the result, and refuse to run anything unsigned. That transfers to models almost verbatim — with one honest asterisk we'll get to.

The ecosystem has converged on **OpenSSF Model Signing (OMS)**, built on Sigstore — the same transparency-log infrastructure the SBOM series used for container images. Google's open-source security team drove the spec, and it's already integrated into Kaggle and NVIDIA's NGC. Signing a model binds the exact bytes of every file in it to a verified identity, with the event recorded in a public transparency log:

```bash
pip install model-signing

# Producer side: sign the model directory. Identity comes from an
# OIDC flow — a Google account, a CI workload identity — not a key
# somebody keeps on a laptop.
model_signing sign ./summarizer-v2.1 --signature summarizer-v2.1.sig

# Consumer side — this belongs in your ingestion pipeline, not in a
# runbook nobody reads:
model_signing verify ./summarizer-v2.1 \
  --signature summarizer-v2.1.sig \
  --identity   "release@research-group.example" \
  --identity_provider "https://accounts.google.com"
```

Run that verify step against the opening scenario. The namespace hijacker can clone the README and the name, but they cannot sign as `release@research-group.example` — the transparency log would show a different identity, and ingestion fails closed. The entire deleted-namespace attack class dies right there, for one CI step.

**SLSA for models** extends the same idea from *who signed it* to *how it was built*: provenance attestations recording which training code, which base model, and which datasets produced these weights. The sigstore `model-transparency` project demonstrates generating SLSA provenance for ML pipelines on GitHub Actions and GCP. It's earlier-stage than model signing — but directionally it's the answer to a question signing alone can't touch, and you'll want to have started before your regulator starts asking. Model cards round it out as the human-readable manifest: the model's SBOM, listing lineage, training data sources, and intended use.

## 4. The controls on GCP, as code

The posture mirrors the container pipeline from the SBOM series: one controlled path into production, cryptographic gates at each step, and nothing enters from the open internet directly.

**Prefer the curated garden, and write that preference as policy.** Vertex AI Model Garden models are vetted and vulnerability-scanned by Google before listing — that's disease one handled upstream, plus a provenance chain you didn't have to build. An org policy can restrict which publishers and models teams may deploy at all, which turns "please use vetted models" from a wiki page into an admission decision.

**Self-hosted models enter through one ingestion pipeline.** The Hugging Face pull happens in exactly one place — a CI job, not a data scientist's notebook — and that job enforces the gates:

```yaml
# cloudbuild-model-ingest.yaml — the only path by which external
# weights reach the model bucket. The parallel to "the build pipeline
# is the only path to production" is the whole point.
steps:
  # Gate 1: format. safetensors only — pickle-based formats are an
  # arbitrary-code-execution vector, not a model format.
  - id: reject-executable-formats
    name: python:3.12-slim
    entrypoint: bash
    args: ["-c", "! find /workspace/model -name '*.bin' -o -name '*.pt' -o -name '*.pkl' | grep -q ."]

  # Gate 2: provenance. Signature must verify against the pinned
  # publisher identity — not merely "a valid signature from someone".
  - id: verify-model-signature
    name: python:3.12-slim
    entrypoint: bash
    args: ["-c", "pip install model-signing && model_signing verify /workspace/model --signature /workspace/model.sig --identity $_PUBLISHER_IDENTITY --identity_provider $_PUBLISHER_IDP"]

  # Gate 3: register — pinned by content digest, never by tag.
  - id: register-model
    name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: bash
    args: ["-c", "gcloud ai models upload --region=$_REGION --display-name=summarizer --artifact-uri=gs://$_MODEL_BUCKET/summarizer/$(sha256sum /workspace/model/model.safetensors | cut -c1-16)/"]
```

**The model bucket is inside the perimeter, and versioned.** Weights live in a GCS bucket wrapped by the VPC Service Controls perimeter from Part 1, with versioning on — because when (not if) you need to answer "which exact bytes served predictions in March?", object versions plus registry records are the audit trail:

```hcl
# terraform/model-storage.tf
resource "google_storage_bucket" "model_artifacts" {
  name                        = "${var.project_id}-model-artifacts"
  location                    = var.region
  uniform_bucket_level_access = true

  versioning { enabled = true }

  # Ingestion pipeline writes; serving identities read. Humans do neither.
}

resource "google_storage_bucket_iam_member" "serving_reads_models" {
  bucket = google_storage_bucket.model_artifacts.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.summarizer_serving.email}"
}
```

**And the serving container is still a container.** Binary Authorization, attestations, digest-pinned images — everything from the SBOM series applies unchanged to the thing *serving* the model. A signed model inside an unattested container is half a control.

Pin versions by digest even though it costs you automatic updates. The opening scenario's team wasn't lazy — they pinned `latest` because they *wanted* the quality improvements. That's a real trade-off, and the resolution isn't "never update"; it's that an update is an ingestion event, passing the same gates as the first pull.

## 5. The gap that signing cannot close

Now the asterisk, stated as plainly as Part 1's residual gap — because this is the paragraph that separates this article from vendor marketing.

Signing and provenance authenticate *origin*. They prove these weights came from that publisher, built by that pipeline, untampered since. They prove nothing about *behavior*. A model poisoned during training — 250 documents seeded into a scraped dataset — emerges from a completely legitimate pipeline and gets signed by a completely legitimate publisher who has no idea the backdoor exists. Every gate in Section 4 passes. The signature is real. The provenance is real. The backdoor is also real, and it was there before the first signature was applied.

What's left is behavioral evidence, and it's honest to say this discipline is where container scanning was fifteen years ago:

- **Behavioral evaluation before promotion.** A model version is a release candidate; releases get regression suites. Yours should include safety and security evals — refusal behavior, data-handling behavior, tool-use behavior — run on every new version, with diffs against the previous version reviewed like a code diff. A backdoor you can't find by scanning weights, you sometimes *can* find as an unexplained behavioral delta.
- **Canary triggers in CI.** You can't enumerate an attacker's trigger phrase, but known-pattern probes (the published research uses trigger structures you can test for) and randomized paraphrase testing raise the cost of the crude variants.
- **Runtime behavior as the last line.** This is the same conclusion Parts 1 and 2 reached from different directions, and it's the series' converging point: when the artifact, the identity, and the channel are all verified and the attack is still possible, what remains is watching what the system *does*. Agent runtime monitoring — anomalous tool-call patterns, out-of-profile data access — is becoming its own discipline precisely because all three boundaries in this series leak in the same direction.

If a vendor tells you their platform "solves" model poisoning, ask them which disease they mean, and what their answer is to a backdoor signed at birth. The quality of that answer will tell you most of what you need to know about the platform.

## 6. Closing the series: three boundaries, one discipline

Three articles, three boundaries, one repeated lesson:

```text
+-------------------+----------------------------------+------------------------------------------------------+------------------------------------------------------+
| Boundary          | The attack                       | What the standard control actually checks            | What it cannot check                                 |
+-------------------+----------------------------------+------------------------------------------------------+------------------------------------------------------+
| Content (Part 1)  | Prompt injection                 | IAM: is the action *permitted*?                      | Whether the intent behind a permitted action changed |
| Identity (Part 2) | Card spoofing, session smuggling | Auth: is the peer *who it claims*?                   | Whether a verified peer is compromised               |
| Artifact (Part 3) | Poisoned weights                 | Signing: is the file *from that origin, untampered*? | What the weights learned before signing              |
+-------------------+----------------------------------+------------------------------------------------------+------------------------------------------------------+
```

Read down the right-hand column and the pattern is the same gap three times: our controls verify *facts about artifacts and identities*, while the attacks live in *behavior*. That isn't a reason to skip the controls — every one of them shrinks the space an attacker moves through, and most real incidents die against the boring layers. It's a reason to hold both ideas at once: verify everything you can verify, and design as if verification will not be enough — smallest permitted action set, one path to production, a human in front of anything irreversible, and eyes on runtime behavior.

That last item — runtime behavior as a first-class security signal for agent systems — is where this field is heading next, and where a future series here will pick up.

These three boundaries are also, not coincidentally, the first three questions I now ask when looking at any agent platform: *what can it reach, who can talk to it, and where did the model come from?* If your team can answer all three with links to code rather than to documents, you're ahead of most of the industry.

---

*Vitaliy Zhhuta is a System & Solution Architect writing at [d3ep0ps.com](https://d3ep0ps.com) about infrastructure, security, and AI systems — from first principles, without the hype. Find him on [LinkedIn](https://linkedin.com/in/vitaliyzhhuta).*
