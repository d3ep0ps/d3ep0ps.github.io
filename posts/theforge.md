
# The Forge: Continuous Integration and the Automated Pipeline

> **"If a human has to touch it to deploy it, the process is broken."**

We have spent the last few articles building a self-healing cluster (The Swarm) and a front door (The Gateway). But an architect’s job isn't finished when the infrastructure is "up." It’s finished when the system can **evolve** at high velocity without human intervention.

Up until now, we’ve talked about *where* things run (Nomad, Jails, Docker). Now we talk about *how* they get there. We are moving from manual operations to **Everything as Code**.

## 1. The Strategy: Velocity as an Architectural Goal

In a legacy environment, deployment is a slow, manual "Event." But recently, I acted as both **System Architect** and **DevOps Engineer** during a **Hackathon**, where we had 4 weeks to build a production-grade system. We didn't have time for manual SSH sessions. To win, we needed velocity.

In that Hackathon, we used **Kubernetes (GKE)** because it was the quickest path to a managed environment. In our home lab, we use **Nomad**. The beauty of **The Forge** is that the *pattern* remains identical, regardless of the engine.

I implemented **The Forge** not just for order, but for speed. When your pipeline is fully automated, the time between "Idea" and "Live Service" drops from hours to minutes. This isn't just a technical choice; it’s a competitive advantage.

## 2. The Blueprint: Decoupled Repositories

A professional Forge requires clear boundaries. In my implementation, I separated the concerns into three distinct layers to ensure that a change in one doesn't break the others:

1. **The Infrastructure Layer (Terraform):** A dedicated space for the "foundations." This code defines the nodes, the private networks, and the resource pools. By separating this from the application, we ensure the hardware layer remains stable.
2. **The Global Networking Repo:** A separate, high-level repository for the "Front Door." This manages the **Gateway (Ingress)**, **Load Balancer**, **Let’s Encrypt** certificates, and global **HTTP routes**. It acts as the central policy for how the world reaches our cluster.
3. **The Application Layer (Monorepo):** This contains the microservice code along with a dedicated `deployment/` folder. When the code changes, the pipeline knows exactly how to package the service and update its specific footprint in the cluster.

## 3. The Mechanics of the Pipeline

Using **GitLab** (as our source) and **Google Cloud Build** (as our builder), the process follows a strict "No-Human" rule:

* **The Build:** The Forge hammers the code into an **Immutable Artifact**. For Linux, this is a Docker Image. For FreeBSD, we use automation like `bastille bootstrap` to create a pristine, versioned Jail template.
* **The Registry:** The image is pushed to a private armory. We never deploy "raw code"; we only ever deploy versioned, tested images.
* **The Deployment:** The Forge applies the job specifications.

### The Universal Pattern: K8s vs. Nomad

Whether you are running the industry-standard Kubernetes (like we did at the Hackathon) or the efficient Nomad stack (like we do in D3ep0ps), the "Deployment" step looks remarkably similar. It is just a text file describing the desired state.

**The Hackathon Way (Kubernetes):**
We defined a `Deployment` manifest. This is verbose but standard.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      containers:
      - name: api
        image: gcr.io/my-project/api-service:v1.2.0
        ports:
        - containerPort: 8080
```

**The D3ep0ps Way (Nomad):**
In our home lab, we achieve the exact same outcome with less boilerplate using Nomad.

```hcl
# nomad/api-service.nomad
job "api-service" {
  group "api" {
    count = 3
    task "server" {
      driver = "docker"
      config {
        image = "my-registry/api-service:v1.2.0"
        ports = ["http"]
      }
    }
  }
}
```

In both cases, **The Forge** does the same thing: it updates the `image` version in the file and submits it to the cluster API. The cluster handles the rest.

## 4. Security and Secrets: The "Shift Left"

Since the Forge is the only path to production, it becomes our primary security checkpoint. We don't just "hide" secrets; we remove them from human access entirely.

**The "Zero-Human-Touch" Principle:**
In a traditional setup, a sysadmin might manually copy a `.env` file to a server. This is a vulnerability. In **The Forge**, the pipeline is valid *only* if it can deploy without a human holding the keys.

**At the Hackathon (Google Secret Manager):**
We used a cloud-native approach.

1. **Storage:** Secrets (DB passwords, API keys) were stored in **Google Secret Manager**.
2. **Sync:** We installed the **External Secrets Operator** in Kubernetes.
3. **Runtime:** The operator automatically fetched the secrets from GCP and created K8s `Secret` objects, which were mounted into our pods as environment variables.

**In the D3ep0ps Lab (HashiCorp Vault):**
For our private infrastructure, we use the gold standard: **HashiCorp Vault**.
Instead of syncing secrets *into* the cluster, Nomad integrates directly with Vault. When a job starts, it requests a temporary, dynamic credential that exists *only* for the life of that process.

In both architectures, the developer never checks in a password, and the operator never sees one. The secret exists only when the application needs it.

## 5. The Safety Net: Instant Rollbacks

Velocity is dangerous without brakes. The most critical feature of **The Forge** isn't how fast it deploys, but how fast it **un-deploys**.

**The "Never Use Latest" Rule:**
In my `cloudbuild.yaml`, I enforce a strict tagging policy. We **never** deploy the `:latest` tag.

* **Production:** Uses Semantic Versioning (e.g., `v1.2.4`).
* **Staging:** Uses the Git Commit Short SHA (e.g., `a1b2c3d`).

**The Rollback Mechanism:**
Because every deployment is defined by a code artifact (the job file), a rollback is not a complex "undo" operation. It is simply a **re-deployment of the previous known-good state**.

If `v1.3.0` crashes the API:

1. Identify the previous version (e.g., `v1.2.9`).
2. Submit the `v1.2.9` job file to the cluster.
3. The Swarm kills the bad containers and spins up the old ones.

Time to recovery: **Seconds**.

## Conclusion

The Forge is the heart of the "D" in **DevOps**. It bridges the gap between **Development** and **Operations**.

With a working Forge:

* **The Watchtower** (Observability) tells us if the new build is healthy.
* **The Swarm** (Orchestration) handles the rollout.
* **The Gateway** (Ingress) routes the users to the new version based on the global routing policy.

We have built a machine that builds itself. But there is one final, often overlooked part of the journey. We have all this power to move services... but how do we handle the **Database** without losing data in the process?

### Next Up: "The Vault: Persistent Data and Database Migrations."
