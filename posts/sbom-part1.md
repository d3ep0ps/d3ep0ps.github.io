# Security as Code: The Dependency Iceberg & Agentic DevSecOps (Part 1 of 3)

> **"Trust, but verify. In software supply chains, don't trust at all — verify everything."**

In modern software systems, a single padlock in the browser address bar is no longer enough. TLS secures the *channel* — it proves you are talking to the right machine. It says nothing about the code executing on it. 

The SolarWinds attack in 2020 didn't break TLS. The Log4Shell zero-day in 2021 didn't exploit a weak cipher. Both attacks travelled through perfectly valid, correctly configured HTTPS connections. They exploited something else entirely: **trust in the software supply chain**.

This series is about closing that gap. We will build a **Secure Software Factory** — a pipeline where every artifact is inventoried, every dependency is scanned, every image is signed, and every deployment is gated by policy. 

In **Part 1**, we will cover the dependency iceberg, lockfile optimization with `uv`, and how to automate dependency audits using AI Agent Skills inside your development loop.

---

## 1. The Reference System: AI Agents on GCP

Before we can secure a system, we need to understand what we are securing. Our reference architecture is a multi-agent AI system built on Google Cloud Platform (GCP):

```
┌─────────────────────────────────────────────────────────────┐
│                        GCP Project                          │
│                                                             │
│   ┌──────────────┐    A2A     ┌──────────────────────────┐  │
│   │  Orchestrator│◄──────────►│   Specialist Agents      │  │
│   │    Agent     │  Protocol  │  (retrieval, execution,  │  │
│   │  (Google ADK)│            │   synthesis...)          │  │
│   └──────┬───────┘            └──────────┬───────────────┘  │
│          │                               │                  │
│          ▼                               ▼                  │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    GKE Cluster                      │   │
│   │         (Autopilot, Workload Identity enabled)      │   │
│   └──────────────────────┬──────────────────────────────┘   │
│                          │                                  │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│   ┌──────────┐   ┌──────────────┐  ┌──────────────────┐    │
│   │ AlloyDB  │   │  Artifact    │  │  Secret Manager  │    │
│   │(vector + │   │  Registry    │  │  (API keys, DB   │    │
│   │ context) │   │  (images +   │  │   credentials)   │    │
│   └──────────┘   │   SBOMs)     │  └──────────────────┘    │
│                  └──────────────┘                          │
│                                                             │
│   Infrastructure: Terraform  │  CI/CD: Cloud Build         │
└─────────────────────────────────────────────────────────────┘
```

The key components:
*   **Google Agent Development Kit (ADK)**: The framework for building GKE-hosted AI agents.
*   **A2A Protocol**: The secure agent-to-agent communication layer.
*   **AlloyDB**: The agents' long-term memory containing vector embeddings.
*   **Artifact Registry**: Container image and SBOM storage.
*   **Cloud Build & Terraform**: The automated build and declarative infrastructure layers.

---

## 2. The Dependency Iceberg

Here is a question most engineers cannot answer about their own production systems: *how many packages are in your running container?*

For a Python-based AI agent, the answer is usually surprising. Your `requirements.txt` might list 15 direct dependencies. But those 15 packages each have their own dependencies, which have dependencies of their own. Add the base OS image packages, the Google ADK framework, the Google Cloud SDK, and numpy (which pulls in a C extension ecosystem of its own).

A typical Python AI agent image contains between 300 and 600 installed packages. You wrote the code for perhaps 20 of them. The other 580 came from PyPI, the OS package manager, and the base image — most of which you have never read, audited, or even consciously chosen.

This is the **dependency iceberg**. What you see (your code) is the 10% above the waterline. What you are actually running is the 90% below it.

```
      \  /
    --- \/ --- (Waterline)
       /  \
      /    \
     /      \  <- 90% Transitive Dependencies (PyPI, OS packages)
    /________\
```

**Log4Shell** made this concrete in December 2021. The CVE-2021-44228 remote code execution vulnerability in `log4j-core` had a CVSS 10.0 rating (the maximum). While the patch was released within days, triage took most organizations *weeks* because they had to first answer the question: *do we even use log4j?*

Teams with proper dependency inventory answered that question in minutes with a single query. Teams without it manually audited hundreds of services. The difference was not a security tool — it was the discipline of knowing what you are running before you need to know.

AI systems exacerbate the problem. AI agents pull in massive ML frameworks (PyTorch, TensorFlow), LLM client libraries (`google-generativeai`, `openai`, `anthropic`), data processing libraries (`pandas`, `numpy`, `arrow`), and vector database clients. The surface area is genuinely large.

A **Software Bill of Materials (SBOM)** is the X-ray that makes this iceberg visible.

### The First Line of Defense: Dependency Pinning

Supply chain security starts at the source. If your `requirements.txt` contains loose dependencies (e.g., `requests>=2.28.0`), your builds are non-deterministic. A patch release on PyPI containing a malicious payload or a broken dependency will be pulled into your environment automatically on the next run.

To prevent this, you must use a lockfile that records the exact version and cryptographic hash of every direct and transitive dependency.

Modern tooling has made this trivial. **uv** — Astral's high-performance Python package manager written in Rust — can resolve your dependencies and generate a secure, hashed manifest in milliseconds:

```bash
# Resolve requirements.in and generate requirements.txt with cryptographic hashes
uv pip compile requirements.in --generate-hashes -o requirements.txt
```

By compiling dependencies with hashes, you guarantee that your build environment will only install packages that match the exact bytes you vetted during development.

---

## 3. The Four-Layer Security Stack

A complete supply chain security posture has four distinct layers, each catching a different class of problem:

```
Source Code → Build → Sign → Push → Deploy → Run
     │           │       │       │        │
     ▼           ▼       ▼       ▼        ▼
  [tfsec]    [Grype]  [Cosign]  [AR]  [Gatekeeper]
  [Checkov]  [syft]             [scan] [Bin Auth]
```

*   **Layer 1 — Inventory (syft + CycloneDX SBOM)**: Know every component in every artifact you ship. An SBOM is a machine-readable manifest (package name, version, origin, and hashes). The format we use is **CycloneDX** — the OWASP standard with the best tooling ecosystem.
*   **Layer 2 — Vulnerability Scanning (Grype)**: Map the inventory against known CVE databases. **Grype** consumes an SBOM or scans an image directly, producing a report of every known vulnerability matched against NVD and GitHub Advisories.
*   **Layer 3 — Image Signing (Cosign + Sigstore)**: Sign the image manifest in the registry, binding the image digest to a verifiable identity. If an image doesn't carry a valid signature, it didn't come from your pipeline.
*   **Layer 4 — Policy Enforcement (Gatekeeper + Binary Authorization)**: Block non-compliant images from running on your Kubernetes cluster.

---

## 4. Practice: Building the Secure Software Factory (Tier 1)

Let's implement **Tier 1 (Minimum Viable Security)**.

### 4.1. Install the tools

```bash
# Install syft (SBOM generator)
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

# Install Grype (vulnerability scanner)
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Install Cosign (image signing)
curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 \
  -o /usr/local/bin/cosign && chmod +x /usr/local/bin/cosign
```

### 4.2. Dockerfile: Secure & Fast Python Builds with `uv`

Write the agent's `Dockerfile` to leverage `uv` for reproducible, secure, and fast builds:

```dockerfile
FROM python:3.11-slim

# Install uv binary directly from the official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copy the compiled lockfile.
# This file is generated outside of Docker via:
#   uv pip compile requirements.in --generate-hashes -o requirements.txt
# Never run uv pip compile inside the Dockerfile — that would allow the build
# to silently upgrade dependencies and break hash verification.
COPY requirements.txt .

# Install dependencies using the compiled requirements file with hashes
RUN uv pip install --system --no-cache -r requirements.txt

COPY . .
```

### 4.3. Generate a CycloneDX SBOM

```bash
# Generate an SBOM from a built image
syft us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0 \
  -o cyclonedx-json \
  --file orchestrator-sbom.json

# See all Python packages specifically
cat orchestrator-sbom.json | jq '.components[] | select(.type == "library") | {name: .name, version: .version}' | head -30
```

### 4.4. Scan for vulnerabilities with Grype

```bash
# Scan the image directly
grype us-central1-docker.pkg.dev/my-project/agents/orchestrator:v1.2.0

# Or feed your existing SBOM to Grype (faster, consistent)
grype sbom:./orchestrator-sbom.json --fail-on critical
```

Grype produces a table of findings sorted by severity. For a typical Python AI agent image, a first scan will look something like this:

```
NAME                          INSTALLED    FIXED-IN     TYPE       VULNERABILITY     SEVERITY
cryptography                  41.0.0       41.0.3       python     CVE-2023-38325    HIGH
urllib3                       1.26.11      1.26.17      python     CVE-2023-43804    MEDIUM
Pillow                        9.5.0        10.0.1       python     CVE-2023-44271    MEDIUM
certifi                       2022.12.7    2023.7.22    python     CVE-2023-37920    MEDIUM
requests                      2.28.2       2.31.0       python     CVE-2023-32681    MEDIUM
```

The key columns: **INSTALLED** is what you are actually running; **FIXED-IN** is the version that patches it. If `FIXED-IN` is empty, no upstream fix exists yet — document it as a known accepted risk and set a calendar review date.

---

## 5. Agentic DevSecOps: Automating Audits with AI Agent Skills

If you are developing your systems alongside agentic AI coding assistants (such as **Claude Code** or **Antigravity**), you do not need to memorize these command-line combinations. Instead, you can define a custom **Skill** for your agent to automate SBOM auditing as part of your active coding loop.

By placing a custom **Skill** file directly in your repository (for example, under `.agent/skills/software_security/SKILL.md`), you make your repository "agent-ready". When you ask your coding assistant to *"run a security sweep"*, it reads these instructions, runs the corresponding local tools, and writes clean Markdown summaries for you.

Instead of keeping the entire security scanning logic local to one project, we can centralize our security rules. You can find the full, production-ready version of this skill on GitHub:

👉 **[d3ep0ps/agents-skills/software_security](https://github.com/d3ep0ps/agents-skills/tree/main/software_security)**

With this central skill in place, you can ask your AI coding assistant (like **Antigravity** or **Claude Code**):
> *"Run a full security scan on the repository and generate the dashboard"*

The agent will read the instruction set from your `agents-skills` workspace, orchestrate `syft`, `grype`, `bandit`, `pip-audit`, and `checkov`, and compile all outputs into a consolidated security report for your human review. This integrates deep security auditing directly into your active coding loop, rather than discovering vulnerability flags late in the CI/CD pipeline.

---

## What's Next?

In **Part 2** of this series, we will move from local audits to **Production-Grade Infrastructure (Tier 2)**. We will look at keyless image signing using Google Cloud Build's OIDC identity, publishing SBOMs as OCI registry artifacts, and writing OPA Gatekeeper policies to block unsigned images at GKE admission time.

Stay tuned!
