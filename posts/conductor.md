
# The Conductor: Automating the Stack

> **"If you have to type it twice, script it. If you have to script it twice, automate it."**

In our last article, "Caging the Beast," we achieved a major architectural victory. We dismantled our monolithic server and isolated our services.

* **Apache** is running securely in a Jail.
* **MySQL** is isolated in a Container.
* **Networking** is handled by virtual bridges.

But we have introduced a new risk.
To build this, we typed dozens of commands. `zfs clone`, `ifconfig bridge0`, `lxc launch`, `pkg install`.

If the server catches fire today, can you rebuild it exactly as it was in 15 minutes?
If you get hit by a bus, does anyone else know that `web01` depends on a specific `sysctl` parameter you set at 2 AM?

This is the **"Bus Factor."** Right now, your Bus Factor is 1.
To scale, we need to stop treating our servers like "Pets" that we nurse to health. We need to treat them like "Cattle." We don't fix them; we replace them.

To do that, we need a **Conductor**. We need **Infrastructure as Code (IaC)**.



## 1. The Philosophy: Imperative vs. Declarative

Before we pick tools, we must pick a mindset.

**The Imperative Mindset (The Script):**
This is what we did in the terminal.

> *"First, create a bridge. Then, create a jail. Then, install Nginx."*
> If you run this script twice, it might break (e.g., "Error: bridge0 already exists"). You have to manage the state yourself.

**The Declarative Mindset (The Blueprint):**
This is what we want.

> *"There should be a bridge. There should be a jail running Nginx."*
> You define the *end state*. The tool figures out how to get there. If the bridge exists, it does nothing. If it's missing, it creates it.




## 2. The Evolution of IaC: From Patches to Blueprints

"Infrastructure as Code" isn't just one tool; it's a spectrum that has evolved.

**Phase 1: Configuration Management (The VM Era)**
In the Virtual Machine world (like in my [ansible-lab](https://github.com/d3ep0ps/ansible-lab)), IaC meant using tools like **Ansible**, **Chef**, or **Puppet**.
You launch a server, and then you apply code to *change* it.

* *The Workflow:* Launch OS  SSH in  Install Packages  Copy Configs.
* *The Result:* A long-lived server that is constantly being patched and updated.

**The Trap:**
When we moved to Containers, many engineers (myself included) tried to drag this workflow with them. We treated containers like "lightweight VMs."
We launched a container, kept it running for months, and used Ansible to patch it.
While this *works*, it introduces significant friction:

1. **Dependency Pollution:** To run Ansible, you need Python and SSH installed *inside* every container. You are filling your isolated environment with management tools that have nothing to do with your application.
2. **Configuration Drift:** If you use Ansible to patch a running container, that container is now unique. If it crashes, you have to hope your playbook runs perfectly on a fresh one. You are still managing "Pets," just smaller ones.

**Phase 2: Immutable Artifacts (The Build Era)**
To truly solve the "Bus Factor," we shifted the paradigm.
We stopped trying to *configure* running servers and started *building* ready-to-run artifacts.

* *The Workflow:* Define Build File  Build Image  Run Container.
* *The Result:* **Immutable Infrastructure**. Even if the container runs for a year, we never patch it. To update Nginx, we don't restart the service; we build a new image and replace the container.

This ensures that the "Blueprint" (Dockerfile/Bastillefile) is always the single source of truth.



## 3. The FreeBSD Way: The Bastillefile

On FreeBSD, we implement this "Immutable Blueprint" using **Bastille**.
Bastille allows us to use a declarative template called a `Bastillefile`. It looks like a script, but we treat it as a build definition.

Instead of SSHing into `web01` to install Nginx, we define the state of `web01` in a file.

**The Blueprint (`Bastillefile`):**

```dockerfile
# /usr/local/bastille/templates/project/web/Bastillefile

# 1. Bootstrap the Environment
PKG nginx
PKG git

# 2. Configure the Service (rc.conf)
SYSRC nginx_enable=YES
SYSRC weak_ssl_ciphers=NO

# 3. Inject Configuration
# Copies a file from the host into the jail
CP config/nginx.conf /usr/local/etc/nginx/nginx.conf

# 4. Start the Service
SERVICE nginx start

# 5. Networking (Redirection)
RDR tcp 80 80

```

**The Execution:**

```bash
# 1. Create the Cage (Jail)
# This bootstraps a fresh OS instance on IP 10.0.0.2
bastille create web01 13.2-RELEASE 10.0.0.2

# 2. Apply the Blueprint (Template)
# This runs our PKG, SYSRC, and SERVICE commands inside the running jail
bastille template web01 project/web

```

This is efficient. It uses the host's tools to inject changes directly into the Jail's filesystem (ZFS dataset), bypassing the need for SSH or agents inside the Jail.

> Unlike Docker images, Bastille templates operate on a running jail, but the discipline is the same: the template remains the single source of truth



## 4. The Linux Way: The Dockerfile

On Linux, this concept won the war. It is called the **Dockerfile**.
The reason Docker became the industry standard isn't because of "containers" (LXC had those). It’s because it perfected this **Build Workflow**.

In the Ansible/VM model, you launch a blank OS and wait for software to install.
In the Docker model, you **build** the software into the image *before* you ever run it.

**The Blueprint (`Dockerfile`):**

```dockerfile
# 1. Start from a base image (The "debootstrap" layer)
FROM ubuntu:22.04

# 2. Install Dependencies (The "apt" layer)
RUN apt-get update && apt-get install -y nginx

# 3. Copy Config (The "Configuration" layer)
COPY nginx.conf /etc/nginx/nginx.conf

# 4. Define the Process (The "Service" layer)
CMD ["nginx", "-g", "daemon off;"]

```

**The Execution:**

```bash
# Build the artifact (Image)
docker build -t my-web-server .

# Run the artifact (Container)
docker run -d -p 80:80 my-web-server

```

### Why This Won the Industry

1. **Immutability:** Once `my-web-server` is built, it never changes. You don't "update" Nginx inside the container. You update the Dockerfile, rebuild the image, and replace the container.
2. **Caching:** Docker caches every line. If you change the `COPY nginx.conf` line, Docker doesn't re-run the `apt-get install` line. It reuses the cached layer.
3. **Portability:** The result is a single binary-like object (the Image) that runs exactly the same on my laptop as it does on your server.



We have successfully automated our stack using modern **Infrastructure as Code** principles.

* **FreeBSD:** We use **Bastillefiles** to template our Jails.
* **Linux:** We use **Dockerfiles** to build our Container Images.

We have moved from **typing commands** to **writing blueprints**.
Our infrastructure is now code. It can be versioned in Git, reviewed by peers, and rolled back if it breaks.

But now that we have these beautiful, isolated, automated black boxes... **how do we see inside them?**
In a monolith, we looked at `/var/log/syslog`. In a distributed system with ephemeral containers, the logs disappear when the container dies.

We need a new way to watch the system.

**Next Up: "The Watchtower: Observability in a Fragmented System."**