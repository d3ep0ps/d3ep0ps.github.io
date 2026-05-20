# Security as Code: GKE Binary Authorization & Operational DevSecOps (Part 3 of 3)

> **"A policy that can be bypassed is a suggestion. Binary Authorization makes the build pipeline the only path to production."**

In **Part 1** and **Part 2** of this series, we secured the developer loop (with `uv` and `syft`), set up continuous registry scanning, and configured **OPA Gatekeeper** to restrict deployments on GKE to registry-pinned, digest-referenced container images.

However, even with these policies in place, how do you verify that the container image actually passed all security scans *before* it reached GKE? What if a signed image contains critical vulnerabilities but was pushed anyway?

To achieve **Tier 3 (Enterprise-Grade)** security, we must replace advisory enforcement with cryptographic policy gates. In this final article, we will implement **GCP Binary Authorization** — a system that prevents any container from running on GKE unless it carries a verified cryptographic attestation from our build pipeline. We will also explore automated dependency updating using **Renovate** and the operational disciplines required to manage SBOMs and CVEs at scale.

---

## 1. Binary Authorization on GKE

Binary Authorization is a GCP-native admission control system that evaluates deployment policies at runtime. The policy ensures that no image is deployed to your GKE cluster unless it carries an **attestation** (a signature confirming that a specific step, such as a security scan, was successfully completed).

Before we can create an attestor, we must declare a **Container Analysis Note** in GCP. This note is the metadata registry where attestation occurrences are recorded. Since we manage our GCP infrastructure with Terraform, we define it declaratively:

```hcl
# terraform/security.tf
resource "google_container_analysis_note" "attestation_note" {
  name = "cloud-build-attestation"
  
  attestation_authority {
    hint {
      human_readable_name = "Cloud Build Attestation Authority"
    }
  }
}

# Grant the Cloud Build service account permissions to attach attestations to the note
resource "google_container_analysis_note_iam_member" "cloud_build_attacher" {
  note   = google_container_analysis_note.attestation_note.name
  role   = "roles/containeranalysis.notes.attacher"
  member = "serviceAccount:cloud-build@${var.project_id}.iam.gserviceaccount.com"
}
```

Next, enable Binary Authorization on your GKE cluster and import the policy:

```bash
# Enable Binary Authorization on your cluster
gcloud container clusters update agents-cluster --region us-central1 --enable-binauthz

# Create the attestor
gcloud container binauthz attestors create cloud-build-attestor \
  --attestation-authority-note=projects/my-project/notes/cloud-build-attestation \
  --attestation-authority-note-project=my-project

# Add the verification public key (which maps to our pipeline's signing identity)
gcloud container binauthz attestors public-keys add \
  --attestor=cloud-build-attestor \
  --pkix-public-key-file=cosign.pub \
  --pkix-public-key-algorithm=ecdsa-p256-sha256
```

Define the policy to require attestation by the `cloud-build-attestor` for GKE workloads:

```yaml
# binauthz-policy.yaml
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/my-project/attestors/cloud-build-attestor
```

```bash
# Import the policy into your GCP project
gcloud container binauthz policy import binauthz-policy.yaml
```

Once applied, GKE will block any image unless it has been attested by your pipeline. Locally built images, public Docker Hub images, and untrusted overrides are entirely locked out.

---

## 2. The Complete Tier 3 Cloud Build Pipeline

Here is the complete secure pipeline. Every stage acts as a gate. If any check fails, the build halts, no attestation is generated, and the deployment is blocked.

```yaml
# cloudbuild.yaml — Tier 3: Full attestation chain
steps:
  # 1. Run tfsec on Terraform config
  - name: 'aquasec/tfsec'
    args: ['--minimum-severity', 'HIGH', '/workspace/terraform/']
    id: 'iac-tfsec'

  # 2. Run Checkov across all templates
  - name: 'bridgecrew/checkov'
    args: ['-d', '/workspace', '--framework', 'terraform,kubernetes,dockerfile', '--compact']
    id: 'iac-checkov'
    waitFor: ['iac-tfsec']

  # 3. Build the agent container image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_IMAGE_TAG', '-f', 'agents/orchestrator/Dockerfile', '.']
    id: 'build'
    waitFor: ['iac-checkov']

  # 4. Generate CycloneDX SBOM using syft
  - name: 'anchore/syft'
    args: ['$_IMAGE_TAG', '-o', 'cyclonedx-json', '--file', '/workspace/sbom.json']
    id: 'sbom'
    waitFor: ['build']

  # 5. Scan SBOM with Grype — hard fail on CRITICAL or HIGH vulnerabilities
  - name: 'anchore/grype'
    args: ['sbom:/workspace/sbom.json', '--fail-on', 'high']
    id: 'scan'
    waitFor: ['sbom']

  # 6. Push image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_TAG']
    id: 'push'
    waitFor: ['scan']

  # 7. Attach SBOM as OCI referrer
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['attach', 'sbom', '--sbom', '/workspace/sbom.json', '--type', 'cyclonedx', '$_IMAGE_TAG']
    id: 'attach-sbom'
    waitFor: ['push']

  # 8. Sign image keylessly with Cosign
  # COSIGN_EXPERIMENTAL=1 is no longer required — keyless signing is the default in Cosign v2+
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['sign', '--oidc-provider=google', '$_IMAGE_TAG']
    id: 'sign'
    waitFor: ['attach-sbom']

  # 9. Create GKE Binary Authorization attestation
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
          IMAGE_DIGEST=$(gcloud container images describe $_IMAGE_TAG --format='get(image_summary.digest)')
          gcloud container binauthz attestations create \
            --artifact-url="$_IMAGE_BASE@$$IMAGE_DIGEST" \
            --attestor=projects/$PROJECT_ID/attestors/cloud-build-attestor \
            --signature-algorithm=ECDSA_P256_SHA256 \
            --public-key-id-override=$_COSIGN_KEY_ID
    id: 'attest'
    waitFor: ['sign']

substitutions:
  _IMAGE_TAG: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator:$SHORT_SHA'
  _IMAGE_BASE: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator'
  _COSIGN_KEY_ID: 'projects/my-project/locations/global/keyRings/cosign/cryptoKeys/signing-key'
```

---

## 3. Renovate for Automated Dependency Hygiene

Setting up scanners is only half the battle; the other half is resolving findings. With hundreds of transitive dependencies, manually tracking patches is a full-time job.

**Renovate** automates this by reading your dependency manifests (e.g., `requirements.txt` or `pyproject.toml`) and opening pull requests automatically when updates are available. Define `renovate.json` in your repository:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchPackagePatterns": ["*"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

When Renovate identifies a security advisory affecting your pinned package version, it opens a PR immediately. If the patch only changes a minor version, it can automatically merge once your test suite passes.

---

## 4. Operational Discipline: Living with an SBOM

Once your pipeline generates SBOMs and blocks builds, DevSecOps becomes an operational exercise.

### CVE Triage: Not Every Finding is a Fire
A scanner like Grype will frequently flag vulnerabilities that you cannot immediately resolve. When triaging, answer three questions:
1.  **Is there a fix?** If there is no fixed version (`FIXED-IN` is empty), document the finding as a known risk and move on.
2.  **Is the vulnerable code path reachable?** If the vulnerability affects a sub-component or function that your AI agent never calls, the real risk is minimal.
3.  **What is the exposure?** A vulnerable library running inside an isolated VPC behind a strict IAM policy carries significantly less risk than one running on an internet-facing endpoint.

### SBOM as Incident Response
When a zero-day vulnerability drops, you need to know if you are affected across dozens of services. Instead of checking code repositories manually, query your stored SBOMs:

```bash
# Query all stored SBOMs for a specific package
for sbom in ./sboms/*.json; do
  echo "=== $sbom ==="
  cat "$sbom" | jq --arg pkg "vulnerable-package-name" \
    '.components[] | select(.name == $pkg) | {name, version, image: input_filename}'
done
```

### The Metric That Matters: Mean Time to Patch (MTTP)
Optimize your posture for a single metric: **Mean Time to Patch (MTTP)** — the average time between a CVE being disclosed and its patched version running in production. With automated dependency tracking (Renovate) and strict admission control, you can drop your MTTP from weeks to a few hours.

---

## Summary of the 3-Tier Pipeline

*   **Tier 1 (Solo/Startup)**: Local SBOM generation via `syft`, CVE scanning via `grype` (CI gate), KMS-based image signing via `cosign`.
*   **Tier 2 (Production)**: Keyless signing via OIDC, attaching SBOMs as OCI registry artifacts, and namespace-level registry gating with **OPA Gatekeeper**.
*   **Tier 3 (Enterprise)**: GKE-wide admission enforcement with **GCP Binary Authorization**, requiring cryptographic attestations from Cloud Build.

---

## What's Next: AI-Specific Threats

This series has secured the infrastructure layer from the developer workstation all the way to GKE admission control. But the AI agents running on this infrastructure introduce a second, distinct attack surface that container scanners and Binary Authorization cannot see.

In the next series, we will pivot to AI-specific security threats: **prompt injection** attacks that hijack agent reasoning, **model weight and supply-chain poisoning**, and vulnerabilities in the **agent-to-agent (A2A) authentication protocol** that can allow a compromised agent to escalate privileges across the multi-agent mesh.
