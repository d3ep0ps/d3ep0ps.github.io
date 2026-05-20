# Security as Code: Gating GKE Deployments with Cryptographic Trust (Part 2 of 3)

> **"An unsigned image in a registry is a promise with no witness. Keyless signing makes the build pipeline the witness."**

In **Part 1** of this series, we looked at the **dependency iceberg** and built a local security audit loop: we resolved dependencies deterministically with `uv`, generated a CycloneDX SBOM using `syft`, and scanned for CVEs using `grype`. 

Local checks are the first line of defense, but they depend on developer discipline. To scale this across a team and protect a production system, we must upgrade our posture to **Tier 2 (Production-Grade)**. 

In this article, we will automate this pipeline: we will implement **keyless image signing** via OIDC in Google Cloud Build, publish SBOMs directly to Artifact Registry as **OCI referrers**, scan our Terraform code using IaC tools, and configure **OPA Gatekeeper** to block unapproved or mutable images from running on our GKE cluster.

---

## 1. Keyless Image Signing via OIDC (Cloud Build)

A major hurdle with image signing is key management: rotating keys, preventing theft, and securely sharing them with CI/CD runners. 

**Sigstore Cosign** solves this with **keyless signing**. Instead of a static private key, Cosign leverages the OpenID Connect (OIDC) identity of the runner itself. In Google Cloud Build, the job runs as a GCP service account. Cosign obtains an OIDC token for this service account, generates a short-lived key pair, signs the image digest, and logs the signature to **Rekor** — Sigstore's public, append-only transparency log.

The signature is verifiable by anyone who knows the Cloud Build service account identity:

```yaml
# In cloudbuild.yaml — keyless signing step:
- name: 'gcr.io/projectsigstore/cosign'
  args:
    - 'sign'
    - '--oidc-provider=google'
    - '$_IMAGE_TAG'
```

To verify this signature:

```bash
cosign verify \
  --certificate-oidc-issuer=https://accounts.google.com \
  --certificate-identity=serviceAccount:cloud-build@my-project.iam.gserviceaccount.com \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0
```

The verification only passes if the image was signed by a Cloud Build job running as that specific service account. No other entity can forge this certificate.

---

## 2. Attach the SBOM as an OCI Artifact

Instead of storing SBOMs in isolated storage buckets, we can attach them directly to the container image inside the Artifact Registry as an OCI referrer. The SBOM and image travel together:

```bash
# Attach the SBOM to the image as an OCI artifact
cosign attach sbom \
  --sbom orchestrator-sbom.json \
  --type cyclonedx \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0

# Retrieve the SBOM from any image later
cosign download sbom \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0 > retrieved-sbom.json
```

Now, any tool or developer with read access to the container image can fetch its full component manifest programmatically.

---

## 3. Enable Artifact Registry Continuous Scanning

Continuous scanning operates on a different temporal plane than CI gates. While Grype blocks vulnerable builds in CI, **Artifact Registry scanning** checks images already in the registry when new CVEs are published.

Enable it alongside Pub/Sub alerts to create a responsive alerting system:

```bash
# Enable the Container Scanning API (one-time, project level)
gcloud services enable containerscanning.googleapis.com

# Enable continuous scanning on your Artifact Registry repository
gcloud artifacts repositories update agents \
  --location=us-central1 \
  --vulnerability-scanning=enabled

# Create a Pub/Sub topic for notifications
gcloud pubsub topics create container-vulnerabilities
```

Query findings programmatically (useful for weekly audits):

```bash
gcloud artifacts docker images list-vulnerabilities \
  us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0 \
  --format="table(vulnerability.effectiveSeverity, vulnerability.shortDescription, packageIssue[0].affectedPackage)" \
  --filter="vulnerability.effectiveSeverity=CRITICAL OR vulnerability.effectiveSeverity=HIGH"
```

---

## 4. IaC Scanning: Terraform and Kubernetes

Your infrastructure files are also code. A misconfigured GKE node pool or AlloyDB database instance are vulnerabilities at the deployment layer. Add **tfsec** and **Checkov** to your linting/CI pipeline:

```bash
# Run tfsec on your Terraform directory
tfsec ./terraform/ --minimum-severity HIGH

# Run Checkov across all IaC: Terraform + GKE manifests + Dockerfiles
checkov -d . --framework terraform,kubernetes,dockerfile --compact --quiet
```

Checkov will flag issues like these:

```
Check: CKV_GCP_69: "Ensure Kubernetes Cluster is not publicly accessible via master_authorized_networks_config"
FAILED for resource: google_container_cluster.agents
File: terraform/gke.tf:12-45
```

---

## 5. Install OPA Gatekeeper on GKE

**OPA Gatekeeper** acts as a Kubernetes admission webhook. Every `kubectl apply` and Pod creation request is evaluated against your policies before the API server accepts it.

Install Gatekeeper:

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace --set replicas=2
```

### Write a ConstraintTemplate: Require Registry Pinning and Digests

Gatekeeper cannot verify signatures directly (as it cannot call out to external registries during admission control). However, it can enforce the necessary preconditions:
1.  Images must come only from your approved Artifact Registry.
2.  Images must be referenced by an immutable **sha256 digest**, not a mutable tag (e.g., `:latest`).

Define the template:

```yaml
# constraint-template-require-registry.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequireapprovedregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequireApprovedRegistry
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireapprovedregistry

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not startswith(image, "us-central1-docker.pkg.dev/my-project/")
          msg := sprintf("Image '%v' is not from the approved registry", [image])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          image := container.image
          not contains(image, "@sha256:")
          msg := sprintf("Image '%v' must be referenced by digest, not tag", [image])
        }
```

Apply the constraint to GKE:

```yaml
# constraint-require-registry.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireApprovedRegistry
metadata:
  name: require-approved-registry
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["agents", "production"]
  enforcementAction: deny
```

```bash
kubectl apply -f constraint-template-require-registry.yaml
kubectl apply -f constraint-require-registry.yaml
```

If you try to run an unapproved image like `nginx:latest`, GKE will block it:
```bash
kubectl run test --image=nginx:latest -n agents
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: Image 'nginx:latest' is not from the approved registry
```

---

## 6. The Complete Tier 2 Cloud Build Pipeline

The individual steps above combine into a single, ordered pipeline. Every stage is a gate — a failure at any step halts the build before the image reaches the registry.

```yaml
# cloudbuild.yaml — Tier 2: Signing + SBOM + IaC gating
steps:
  # 1. Run tfsec on Terraform config
  - name: 'aquasec/tfsec'
    args: ['--minimum-severity', 'HIGH', '/workspace/terraform/']
    id: 'iac-tfsec'

  # 2. Run Checkov across all IaC templates
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

  # 5. Scan SBOM with Grype — hard fail on HIGH or CRITICAL vulnerabilities
  - name: 'anchore/grype'
    args: ['sbom:/workspace/sbom.json', '--fail-on', 'high']
    id: 'scan'
    waitFor: ['sbom']

  # 6. Push image to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_TAG']
    id: 'push'
    waitFor: ['scan']

  # 7. Attach SBOM as OCI referrer (travels with the image)
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['attach', 'sbom', '--sbom', '/workspace/sbom.json', '--type', 'cyclonedx', '$_IMAGE_TAG']
    id: 'attach-sbom'
    waitFor: ['push']

  # 8. Sign image keylessly with Cosign (Sigstore OIDC — no static keys)
  - name: 'gcr.io/projectsigstore/cosign'
    args: ['sign', '--oidc-provider=google', '$_IMAGE_TAG']
    id: 'sign'
    waitFor: ['attach-sbom']

substitutions:
  _IMAGE_TAG: 'us-central1-docker.pkg.dev/my-project/agents/orchestrator:$SHORT_SHA'
```

What Tier 2 gives you: every image in your registry carries a verifiable Sigstore signature and an attached SBOM. A developer or audit tool can fetch both from a single registry endpoint, with no separate storage system to maintain. The full cryptographic enforcement gate — blocking unsigned images at GKE admission — is Tier 3.

---

## What's Next?

With **Tier 2 (Production-Grade)**, you have secured your images, your templates, and your GKE admissions. But what if a signed image is compromised *after* it passes the registry checks, or what if someone attempts to bypass Gatekeeper policies? 

In **Part 3**, we will look at **GCP Binary Authorization** — the ultimate cryptographic gatekeeper. We will build a complete attestation chain from Cloud Build to GKE Autopilot, configure Renovate for automated dependency hygiene, and establish incident response procedures using the SBOM.
