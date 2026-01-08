
# From DevOps to MLOps: Engineering Uncertainty

> **"In code, 1 + 1 always equals 2. In ML, 1 + 1 equals 2 with 98% probability. Engineering is what we do with the other 2%."**

We are used to determinism.
In the world of classic system administration and DevOps, we build systems on immutable rules. If the code version is pinned in Git, and the infrastructure is described in Terraform, the deployment result today will be identical to the result tomorrow. We spent years turning the "release chaos" into a boring, predictable pipeline.

But now we face a new challenge. Business no longer wants to just automate logic; it wants to predict the future. Enter Machine Learning.

And suddenly, our perfect DevOps pipelines stop working.

Why? Because ML systems are not deterministic. They are stochastic. They depend not only on the code you wrote but also on the data you "fed" the model.

Welcome to the world of **MLOps**.

We recently did a [deep dive into this topic on a webinar](https://www.youtube.com/live/aIxP5wgAKy8?si=antk2nw_nKOQg-pN). Today, I want to structure this into a single architectural model. We will look at how to apply engineering discipline to the chaos of Data Science and turn notebook experiments into a reliable production system.

## 1. Mental Model: Why `git checkout` is not enough

To understand MLOps, you need to accept a fundamental paradigm shift.

In traditional development (Software Engineering):

In Machine Learning:

If you change even one bit in the input data (Dataset), you get a different model. Git is great at versioning code, but it "chokes" on gigabytes of binary data. MLOps is a set of practices that allows us to version the **entire equation**, not just the code.

## 2. MLOps Architecture: Anatomy of a New System

We can't just take Jenkins and say we have MLOps. We need new architectural components to manage specific ML assets.

### 2.1 Feature Store (The "Parts" Warehouse)

In Data Science, engineers often spend time writing SQL queries to pull data. The problem is that Data Scientist Bob writes one query for training, and Backend Developer Alice writes a *similar, but slightly different* query for production.

**Feature Store** solves this desynchronization:

1. **Standardization:** You define once how "average check" is calculated, and everyone uses that definition.
2. **Training-Serving Skew:** Ensures that the data the model learned on (Offline) is mathematically identical to the data coming in real-time (Online).

### 2.2 Model Registry (Artifacts)

Where do you store compiled binaries? In Artifactory. Where do you store Docker images? In a Registry.
An ML model is also an artifact. But you can't just dump it on S3.

**Model Registry** ensures traceability:

* This `model.pkl` file was trained on commit `a1b2c3`.
* It used Dataset v4.
* It showed 98.5% accuracy.

### 2.3 CI/CD/CT Pipelines

In MLOps, **CT (Continuous Training)** is added to the usual CI/CD.
Unlike regular software, an ML model starts "rotting" the moment it is released. The world changes. Your architecture must be able to automatically trigger retraining when quality metrics drop, without manual human intervention.

## 3. Monitoring: Hunting for Drift

This is the most critical part for an Operations Engineer.
When a Web server crashes, it returns 500. You see it.
When an ML model "crashes", it returns **200 OK**. But the predictions become garbage.

We call this Drift:

1. **Data Drift:** Input data changed (e.g., users started uploading photos at night, but the model learned on day photos).
2. **Concept Drift:** The logic of the world changed (e.g., after a crisis, purchasing patterns changed).

You need tools like **Prometheus** (for the system) and **Evidently** (for data quality) to catch these silent failures.

## 4. Security: The Dark Side of MLOps

We are used to protecting the perimeter with firewalls. But in AI, attack vectors shift to mathematics.
Here are five threats an engineer must know about:

### Evasion Attacks

Attack "on the fly" (Inference). An attacker modifies input data to fool the model.
**Example:** A sticker on a road sign that looks like dirt to a human but turns a "STOP" sign into a "Speed Limit 45" for an autopilot.

### Data Poisoning

Attack at the training stage (Training). If your pipeline automatically pulls data from outside, a hacker can "inject" poisoned data, forcing the model to learn wrong rules.

### Trojaning / Backdoor Attacks

The model works perfectly until it sees a "secret key" (e.g., a specific pixel on an image). Then it executes an action planted by the attacker. This is a sleeper agent in your code.

### Model Inversion

By bombarding the API with requests, a hacker can reconstruct confidential data the model learned on (e.g., medical records).

### Model Stealing

A competitor uses your API to train their own "shadow model," effectively stealing your intellectual property.

## 5. Deployment Strategies: Push vs Pull

How to deliver the model to the cluster?

* **Push (Pipeline):** Jenkins executes `kubectl apply`. Fast, but gives the CI server too many permissions.
* **Pull (GitOps):** ArgoCD inside the cluster watches the Git repository. This is the standard for stable systems.

## 6. Tooling: Overcoming the Reproducibility Crisis

The scariest phrase in ML engineering: *"But it worked on my laptop!"*.
In a world where results depend on a Pandas library version, a dataset on S3, and a random seed, "works on my machine" means nothing.

To build a reliable pipeline, we must fix the three pillars of MLOps.

### Pillar 1: Environment & Data (Foundation)

Before running code, we must guarantee identical conditions.

* **Docker:** Fixes the operating system and libraries (Python, CUDA). Without it, your code will break on the first driver update on the server.
* **DVC (Data Version Control):** This is "Git for data". It allows versioning gigabyte-sized datasets, storing only light metadata (`.dvc`) in Git. This allows the team to switch between data versions as easily as between code branches.

### Pillar 2: Lifecycle Management (MLflow Platform)

Previously, we used Excel or text logs to record results. This doesn't work at scale. Enter **MLflow** — not just a logger, but a full-fledged platform for ML End-to-End Lifecycle management.

It covers four critical tasks:

1. **Tracking:** Automatically records every experiment (parameters, metrics, code version). This is your "lab notebook" that allows comparing thousands of runs to find the best one.
2. **Models:** Standardizes model packaging. Whether it's Scikit-Learn, PyTorch, or TensorFlow — MLflow packages them into a universal format ready for deployment in Docker or Kubernetes.
3. **Model Registry:** This is the "checkpoint". You can't just roll out a model to production. The Registry manages stages (`Staging` → `Production`), versions, and approvals (Model Governance).
4. **AI Agent Evaluation:** In the modern LLM world, this allows evaluating and tracking (Trace) complex AI agents, ensuring the quality of generative models.

### Pillar 3: Cloud Alternative (The Enterprise Way)

Assembling this stack manually (Docker + DVC + MLflow Server) is the Open Source way. It's flexible but requires administration.

Cloud providers offer "MLOps as a Service," where these tools are already integrated and configured:

* **AWS SageMaker**
* **Google Vertex AI**
* **Azure Machine Learning**

They provide the same capabilities (registries, tracking, pipelines) but take the headache of server configuration off you. The choice between "Open Source" (MLflow) and "Cloud Native" (SageMaker) depends on whether you want to control the infrastructure or just use it.

## Conclusion: Evolution, Not Revolution

We learned to manage servers. We learned to manage code. Now it's time to learn to manage data.

MLOps is not just a set of new fancy tools. It is the natural evolution of the systems engineer profession.
Ten years ago, we moved from "manual server configuration" to IaC (Infrastructure as Code).
Today, we are moving from "manual experiments" in Jupyter Notebooks to **CaC (Configuration as Code)** for artificial intelligence.

For us engineers, this means one thing: the "Wild West" of Data Science is ending. Standards, security, and reliability are taking its place. And you are the people who will build these standards.

Don't be afraid of new terms. Feature Store is just a cache. Model Registry is just an artifact storage. The reliability principles you learned in Linux and Networking work here too.

Cage the beast. Build reliable pipelines. Make AI boring and predictable. This is true engineering.
