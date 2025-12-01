# The Lost Art of System Administration

## Why we must master the "Old Stack" (LAMP, Mail, DNS) before touching Kubernetes.

In our previous series, we built a fortress.
We took a bare server and hardened the **Identity Layer** with SSH keys.
We secured the **Network Layer** with UFW and PF firewalls.
We automated it all with Ansible and Terraform.

We now have a secure, locked-down, automated server.

**But it is empty.**
A secure server that does nothing is useless.

It is time to open the shop.

Before I ever touched Kubernetes, before I wrote Terraform or built multi-tenant SaaS architectures, I spent more than a decade in the trenches of classical infrastructure.

Six years at an ISP.
Five years building shared hosting, dedicated servers, and a global VPS platform.
Scaling from four small clouds to **forty-eight locations worldwide**.

Back then, "microservices" didn't exist.
The core of our entire business ran on **real Unix services**:

* **Apache** serving millions of PHP requests.
* **MySQL** clusters powering web apps.
* **Postfix** queues stretching for kilometers.
* **Dovecot** keeping IMAP connections alive.
* **Bind** running DNS for thousands of domains.

Every outage, every ticket, every emergency—you solved it by understanding how services actually ran on the system.

Not in a container. Not behind an orchestrator. But directly on **bare Linux and FreeBSD**.

### The Lost Knowledge

Here is the surprising truth:

> **Those skills never stopped being relevant — they just became hidden under layers of abstraction.**

Today, engineers deploy containers without understanding the process inside.
They rely on cloud-managed DNS without knowing what a Glue Record is.
They run “mail microservices” without understanding SMTP queues.

And that is a problem. Because:

* If you don’t understand Apache or Nginx, Kubernetes Ingress will confuse you.
* If you don’t understand Postfix, Amazon SES will remain a black box.
* If you don’t understand DNS, debugging *any* distributed system becomes guesswork.

So, before we look at **chroot**, **jails**, or **containers**, we are going to pause.
We are going to install and configure the **Hosting Stack**—LAMP, FAMP, Mail, and DNS—the way real sysadmins do.

But first, we need a mental model.

## The Mental Model: The Evolution of Isolation

Nothing in tech is truly new. Every "modern" tool is just a layer on top of an older, simpler concept. To understand where we are going, we must understand the five phases of infrastructure evolution.

### Phase 1: Bare Services (The Beginning)
This is where every sysadmin starts. You install a service directly on the host OS.
Apache binds to port 80. MySQL binds to 3306.
They share the same kernel, the same libraries, and the same filesystem.

**The Lesson:** If you can't debug a service running natively on the host, you will never successfully debug it when it's hidden inside a container.

### Phase 2: Lightweight Isolation (Chroot & Jails)
As services grew, we needed to stop them from fighting.
Unix introduced **chroot** (changing the root directory) and later **FreeBSD Jails** and **Solaris Zones**.

This solved the early multi-tenant problems:
* One hacked PHP site shouldn't read `/etc/passwd`.
* One crashing mail server shouldn't panic the kernel.

It was efficient, but it wasn't fully separated.

### Phase 3: Virtualization (The VPS Revolution)
This is where my career shifted gears.
In the 2000s, **Full Virtualization** (KVM, Xen, VMware) changed everything.

We stopped slicing the *OS*. We started slicing the *Hardware*.

This allowed me to help scale a hosting business from a few servers to 48 global locations. We could give every customer their own kernel, their own IP stack, and their own block storage.

**The Lesson:** Virtualization taught us about **resource guarantees** and **multi-tenancy**. It laid the foundation for the Cloud (AWS EC2 is just managed Virtualization).

### Phase 4: Containers (The Packaging Revolution)
Docker didn't invent containers; it popularized them.
It took the isolation of Phase 2 and combined it with a beautiful packaging format.

Suddenly, you could bundle Apache, PHP, and your code into a single artifact.
But here lies the danger:
> **Containers allow people to deploy services they do not understand.**

When the container breaks, if you don't know Phase 1 (Bare Services), you are helpless.

### Phase 5: Orchestration (The Management Layer)
Once you have 1,000 containers, you need a robot to manage them.
Enter **Kubernetes**.

But inside every Kubernetes Pod is just a container.
And inside every container is just a **Service** (Phase 1).

If you skip the fundamentals, Kubernetes looks like magic. If you know the fundamentals, Kubernetes is just a complex tool managing simple processes.



## The Roadmap: Building the Hosting Stack

We are going to build a "Micro-Hosting Company" on our hardened server. Over the next few articles, we will tackle the services that power the internet, comparing **Linux** and **FreeBSD** side-by-side.

### 1. The Web Stack (LAMP vs FAMP)
We will set up Apache, MySQL, and PHP.
* **Linux:** The Debian/Ubuntu way (`apt`, `systemd`, `/etc/apache2`).
* **FreeBSD:** The Ports way (`make install`, `rc.d`, `/usr/local/etc`).
We will see why FreeBSD's separation of "Base OS" and "Third Party Applications" makes it a favorite for disciplined engineers.

### 2. The Infrastructure Stack (DNS & The Bootstrap Paradox)
You cannot host a domain if you don't have a nameserver.
But you cannot have a nameserver (`ns1.yourdomain.com`) if the domain doesn't exist yet.
We will solve this "Bootstrap Paradox" by installing **Bind** and configuring **Glue Records**, becoming our own Authoritative DNS.

### 3. The Communication Stack (Mail is Hard)
Hosting email is the hardest task in system administration.
It involves **Postfix** (SMTP), **Dovecot** (IMAP), **PostfixAdmin** (Management), and **Roundcube** (Webmail).
We will learn about SPF, DKIM, and why IP reputation matters more than configuration.


## Why This Still Matters

We are not doing this to relive the past or rebuild the internet from 2005.
We are doing it because the *foundations never disappeared* — they simply became hidden behind APIs, dashboards, and layers of abstraction.

Every modern system still relies on the same primitives:

* Kubernetes Ingress is just HTTP proxying wrapped in YAML.
* Service Meshes are firewalls, routing tables, and TLS automation at scale.
* Cloud DNS is still Bind behind an API endpoint.
* SES, SendGrid, Mailgun are automated SMTP pipelines.
* Every container still runs a Unix process.
* Every cloud VM still boots like a regular server.

When you understand **how the Old Stack actually works**, nothing in modern infrastructure feels magical anymore.
You stop guessing.
You start engineering.

## Where We’re Going Next

This article is the prologue to a new, practical sequence: **Building the Internet From Scratch — The Engineer’s Way.**

Over the next articles, we will bring our hardened server to life by installing and configuring the real services that still power the modern internet:

### 1. The Web Stack — LAMP vs FAMP
Apache + MySQL + PHP on Linux and FreeBSD. Side-by-side. With all the differences that matter.

### 2. The Name Stack — Running Your Own DNS
We will install **Bind**, configure authoritative zones, SOA, A/AAAA, NS, MX records, and solve the *Glue Record Bootstrap Paradox* used by every hosting provider.

### 3. The Mail Stack — Postfix + Dovecot
A full, working mail system with SMTP, IMAP, PostfixAdmin, and Roundcube — including SPF, DKIM, and DMARC.

### 4. Preparing for Isolation
Once everything runs on bare metal, we’ll move one layer deeper: **chroot, FreeBSD Jails, Containers, and Kubernetes.**

Because once you know how the service works **on the host**, every abstraction above it becomes obvious.

## The Real Reason We're Doing This

Before Docker, before Kubernetes, before IaC, before cloud automation — **all we had were services running on real Unix systems.**

Those systems taught reliability.
They taught discipline.
They taught debugging without dashboards.
They taught engineering.

And those lessons carry forward.

Because whether your code runs in a container or a FreeBSD jail or a VM in Singapore, it is still served by an HTTP server, a database, a mail queue, and a DNS resolver.

This series is about learning *why* these things work, not just how to deploy them.

**Next Up: Building the Web — LAMP vs FAMP.**

Our server is hardened. The walls are up. Now it’s time to open the shop.