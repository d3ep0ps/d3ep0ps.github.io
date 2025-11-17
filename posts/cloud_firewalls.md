# Firewalls in the Cloud: From pf to VPC Rules - A Mental Model Shift

## Intro

Firewalls used to live inside the operating system. Whether it was pf on FreeBSD or iptables on Linux, the rules were built directly into the host.
Today, the center of gravity has shifted.

In modern cloud platforms, the first firewall your packet meets isn’t on your VM.
It’s in the virtual network fabric — a fully distributed system that evaluates rules long before the traffic ever reaches your kernel.

Security groups, VPC firewalls, network policies — they all serve the same purpose as `pf` or `iptables`, but the way they work is fundamentally different.
They’re stateless or stateful depending on the platform, enforced at the hypervisor or the fabric layer, and applied per-network interface, per-VM, per-project, or even per-service.

And here’s the important part:

When you move to the cloud, a misconfigured on-host firewall is rarely what breaks your system.
A misconfigured VPC rule is.

This article is about connecting both worlds.

You’ll see how traditional host firewalls map — and sometimes don’t map — to cloud concepts.
We’ll walk through Google Cloud Platform, AWS, and Azure to understand:

how packets move through virtual networks,

where firewall rules are actually enforced,

how cloud firewalls differ from `pf`/`iptables` in philosophy and in guarantees, and

why debugging cloud connectivity often means ignoring the VM entirely.

Because in the cloud, the firewall isn’t just a rule set.
It’s part of the infrastructure itself.

And if you understand that mental model, you will troubleshoot cloud systems faster than most engineers ever will.

## How Cloud Firewalls Are Different

When you move from on-premise or bare-metal systems into the cloud, one thing becomes clear very quickly:

A cloud firewall is not just `pf` or `iptables` running somewhere else.
It’s part of the infrastructure fabric itself — and that changes everything.

Traditional host firewalls (`pf`, `iptables`) enforce rules inside the OS kernel, directly on the network stack of the machine.
Cloud firewalls enforce rules before traffic ever touches your VM.

This shift introduces a few fundamental differences in how cloud firewalls behave, how they are applied, and how you diagnose problems.

### 1. Enforcement Happens Outside the VM

On a physical machine:

* pf filters packets in the FreeBSD kernel

* iptables/nftables filter packets in the Linux kernel

On the cloud:

* GCP VPC firewalls

* AWS Security Groups

* Azure NSGs

…filter packets in the virtual network fabric, often implemented inside the hypervisor or a distributed networking plane.

The result:
* Your VM never sees traffic the cloud firewall blocks.
* No logs, no SYN packets, nothing.

This is the #1 source of confusion for engineers who are used to on-host firewalls.

### 2. Firewalls Apply to NICs, Not Interfaces You Configure

In pf or iptables, you choose the interface:
```
pf:   pass in on em0 proto tcp to port 22
iptables: ..... -i eth0 -p tcp --dport 22 -j ACCEPT
```

Cloud firewalls attach to:

* network interfaces (AWS ENIs)

* VM instances (GCP instance-level)

* subnets or VPCs (Azure NSG / GCP VPC)

* tags or service accounts (GCP)

This means a firewall isn't tied to a kernel device — it’s tied to the resource identity.

For example, in GCP you can write:

```
Allow tcp:22
Source: service account = linux-admin@myproject.iam.gserviceaccount.com
```
You don’t even specify the VM name or network interface.
This abstraction is powerful, but different from traditional firewalls.

### 3. Most Cloud Firewalls Are Stateful by Default

pf is stateful.
iptables can be stateful, but the rules define it.
Cloud firewalls?

They’re automatically stateful — always.

* If you allow outbound 443, inbound return traffic is allowed automatically.

* If you allow inbound 22, outbound return packets are automatically permitted.

You don't write rules for “established” or “related” connections.

This simplifies policy, but also hides behavior.
It’s easy to forget that cloud firewalls track connection tables behind the scenes.

### 4. Traffic Direction Is Reversed Compared to On-Host Firewalls

On-host firewalls think in terms of:

* incoming traffic to the interface
* outgoing traffic from the interface

Cloud firewalls think in terms of:

* source → destination
* ingress to VM / egress from VM

For example, in GCP:

* “ingress” means traffic entering your VM from anywhere
* “egress” means traffic leaving your VM, before it reaches the Internet

And the rule evaluation is separate for each direction.

This directional split often confuses engineers who assume on-host semantics.

### 5. Cloud Firewalls Apply Before Routing

In `pf`/`iptables`, routing decisions happen before some firewall stages.

**In clouds, packets are filtered before:**

* DHCP is processed
* ARP/NDP discovery happens
* routing tables are evaluated
* your VM’s kernel sees anything

**This means:**

> **If the cloud firewall blocks it → the VM kernel has no clue traffic ever existed.**

This changes how you debug connectivity:

* tcpdump inside the VM shows nothing
* logs show nothing
* sysctl counters don’t increment

Most cloud networking outages are invisible from the OS perspective.

### 6. Policies Are Normalized Globally, Not Per-Host

`pf` rules belong to one host.
`iptables` rules belong to one host.

**Cloud firewalls belong to:**

* Entire VPC networks (GCP)
* Subnets (Azure)
* ENI attachments (AWS)
* Identity tags (GCP)
* Entire projects or resource groups

**This means:**

* policies are consistent across fleets
* there’s no “snowflake servers”
* you can secure 1000 machines with one rule

**But it also means:**

* one incorrect rule may break entire environments
* debugging can affect global traffic, not just one node

### 7. Cloud Firewalls Don’t Replace On-Host Firewalls — They Complement Them

A critical mindset shift:

> Cloud firewalls control north-south traffic (into/out of VMs).
> Host firewalls control east-west traffic (within the VM or between local apps).

You often still need `pf` or `iptables` for:

* microsegmentation inside a VM
* container isolation
* rate limiting
* advanced logging
* NAT tricks or port redirection
* blocking localhost → localhost traffic

Cloud firewalls aren’t the end of `pf` or `iptables` — they just come earlier in the chain.

### Why This Matters

Before you dive into AWS, GCP, or Azure specifics, you need the correct mental model:

Cloud firewalls don’t behave like pf or iptables.
They redefine the entire packet path.

Once you understand that, cloud networking stops feeling magical — and starts feeling predictable again.


## Packet Flow in GCP, AWS, and Azure — What Actually Happens Before Your VM Sees a Packet

Every cloud platform talks about “virtual networks,” but the real question for engineers is this:

What happens to an incoming packet before it reaches my VM?

In on-prem systems, the path is simple:
```
NIC → Kernel → pf/iptables → Application
```

In the cloud, that path becomes a distributed pipeline controlled by the provider — not your VM.
And each platform implements this pipeline differently.

Let’s break down what actually happens in Google Cloud (GCP), Amazon Web Services (AWS), and Microsoft Azure, focusing on the key security and routing decisions along the way.

### 1. Google Cloud Platform (GCP)
*Fabric-level enforcement, global VPC, predictable evaluation*

GCP has the cleanest and most transparent packet flow model of all major clouds.

**Inbound (Ingress) Packet Flow — GCP**
```
Internet  
   ↓  
Google Frontend (if using LB)  
   ↓  
VPC Firewall (ingress rules evaluate here)  
   ↓  
Subnetwork routing  
   ↓  
VM NIC (virtio)  
   ↓  
Guest OS (then pf/iptables if configured)
```

**Key Points**

* Firewall rules apply before routing — meaning a packet can be rejected before it knows the VM’s subnet.

* Firewall evaluation is global and distributed, not tied to a physical host.

**Rules can target:**

* IP ranges

* tags

* service accounts (super powerful)

**Outbound (Egress) Packet Flow — GCP**
```
Guest OS  
   ↓  
VM NIC  
   ↓  
VPC Firewall (egress rules evaluate here)  
   ↓  
Routing / NAT  
   ↓  
Internet / other VPCs
```
If you allow outbound 0.0.0.0/0, all return traffic is automatically allowed (stateful).

### 2. Amazon Web Services (AWS)
*Ingress controlled by Security Groups; subnet boundaries enforced by NACLs*

AWS has two layers of firewalling:

* **Security Groups (SG)** — stateful, attached to ENIs
* **Network ACLs (NACLs)** — stateless, attached to subnets

This dual model is more complex but extremely powerful when used correctly.

**Inbound (Ingress) Packet Flow — AWS**
```
Internet  
   ↓  
AWS Edge (NAT, ALB/ELB if used)  
   ↓  
NACL (subnet-level, stateless)  
   ↓  
Security Group (ENI-level, stateful)  
   ↓  
VM ENI  
   ↓  
Guest OS (iptables, nftables, pf for EC2 FreeBSD)
```

**Key Points**

* If NACL denies the packet → the instance never sees it.
* If SG denies the packet → instance never sees it.
* If SG allows inbound → outbound return is automatically allowed.
* SG logic is simple; NACL logic is complex because it’s stateless and evaluated in order.

**Outbound (Egress) Packet Flow — AWS**
```
Guest OS  
   ↓  
ENI  
   ↓  
Security Group (stateful)  
   ↓  
NACL (stateless)  
   ↓  
Routing / NAT  
   ↓  
Public Internet or VPC peering
```
AWS is the only cloud where outbound traffic may be blocked in two layers.

### 3. Microsoft Azure
*Distributed firewalls at multiple layers — NSGs, ASGs, and platform rules*

Azure uses:

* **NSGs (Network Security Groups)** — main firewall
* **ASGs (Application Security Groups)** — logical grouping
* **Platform-level rules** — sometimes implicit and non-removable
* **VNet-level routing** — affects evaluation order

Azure packet flow is the least intuitive, but still manageable with the right model.

**Inbound (Ingress) Packet Flow — Azure**
```
Internet  
   ↓  
Azure Load Balancer (if used)  
   ↓  
NSG (subnet-level)  
   ↓  
NSG (NIC-level)  
   ↓  
VM NIC  
   ↓  
Guest OS firewall
```

**Key Points**

* NIC-level NSG overrides subnet-level NSG when both exist.
* Azure applies implicit rules (e.g., deny inbound from Internet unless explicitly allowed).
* Azure is fully stateful — return traffic flows automatically.

**Outbound (Egress) Packet Flow — Azure**
```
Guest OS  
   ↓  
VM NIC  
   ↓  
NSG (NIC-level)  
   ↓  
NSG (subnet-level)  
   ↓  
Routing / NAT  
   ↓  
Internet / VNet peering
```
Azure has more hidden behaviors (e.g., platform traffic for health checks) than AWS/GCP.

### Comparing the Clouds (Summary)

**GCP**

* Simplest model
* One firewall layer (VPC firewall)
* Identity-based rules (tags, service accounts)
* Routing happens after firewall evaluation

**AWS**

* Two firewall layers (SG + NACL)
* SG: stateful, NIC-bound
* NACL: stateless, subnet-bound
* Most granular but most complex

**Azure**

* Two-layer NSG
* Implicit rules
* NIC-level NSG overrides subnet-level
* Most "enterprise-style" semantics

### Why This Matters**

When debugging cloud networking, you must know:

* Which layer blocks the packet?
* Does the platform log it?
* Does the VM even see it?
* Is the firewall stateful or stateless?
* Are there implicit allow/deny rules?
* Is routing evaluated before or after firewall?

Once you understand the packet path, cloud firewall debugging becomes deterministic instead of guesswork.

---

## Mapping `pf` Concepts to Cloud Firewalls (and Where the Mapping Breaks)

If you've used `pf`, `iptables`, or `nftables`, you already understand the basics of packet filtering.
Cloud firewalls reuse many of those ideas — but often in ways that don’t translate directly.

### 1. Rules and Anchors → Resource-Based Policies

In `pf`, you group rules into anchors:
```
anchor "web"
anchor "db"
```

Linux does something similar using tables and chains.
```
iptables -t filter -A INPUT ...
```

Cloud firewalls don’t use rule chains or anchors.
Instead, they rely on resource identity:

Cloud firewalls don’t group rules around functions.
They group them around resources, such as:

**GCP**

* tags
* service accounts
* instance names
* network/subnet scopes

**AWS**

* ENI attachments

* security group membership

**Azure**

* NIC-level NSGs
* subnet-level NSGs
* Application Security Groups (ASGs)

In practice:

* pf groups rules by function
* clouds group rules by resource

This shift is powerful but unfamiliar at first.

### 2. Interface Matching → Cloud Targeting

``pf/iptables`` attach rules to network interfaces:
```
pass in on em0 to port 22
```

**Cloud firewalls detach from the OS entirely and attach to:**

* instance tags (GCP)
* service accounts (GCP)
* ENIs (AWS)
* security groups (AWS)
* NIC-level or subnet-level NSGs (Azure)

There is no eth0 or em0 from the cloud’s perspective.
Everything is abstracted into resource identities.

### 3. State Tracking → Always Enabled

In `pf`:
```
pass in proto tcp keep state
```

In `iptables`:
```
iptables .... -m state --state ESTABLISHED,RELATED
```

**In the cloud:**

* GCP: always stateful
* AWS Security Groups: always stateful
* Azure NSGs: always stateful

You cannot disable connection tracking.
This makes configurations shorter, but it also hides behavior that `pf` exposes clearly.

### 4. Default Deny → Inconsistent Across Clouds

`pf` encourages the classic pattern:
```
block all
pass tcp to port 22
```

**Cloud defaults differ:**

**GCP**

* ingress: deny
* egress: allow

**AWS**

* SG inbound: deny
* SG outbound: allow
* NACLs: allow everything unless modified

**Azure**

* inbound: deny
* outbound: allow
* platform-level implicit rules also exist

If you’re used to pf’s strict block-everything model, these defaults may surprise you.

### 5. Logging → Flow Logs Instead of Per-Rule Logs

`pf` can log every rule:
```
pass in log on em0
```

`iptables` can log per rule as well.
```
iptables .... - j LOG
```

**Cloud firewalls use flow logs:**

* GCP VPC Flow Logs are excellent but lack per-rule detail.
* AWS Flow Logs show accept/deny but not which SG/NACL line matched.
* Azure NSG Flow Logs often lag and require diagnostic workspaces.

You lose visibility into which rule matched a packet.
You only see whether traffic was allowed or denied.


### 6. NAT and Redirection → Completely Different Behavior in Clouds

`pf` lets you create NAT and redirection rules manually:
```
nat on em0 from 10.0.0.0/24 to any
rdr on em0 proto tcp to port 80 -> 10.0.0.10 port 8080
```

In clouds:

> **Cloud NAT is a Gateway, not a rule.**

* GCP uses Cloud NAT (VPC-level)
* AWS uses NAT Gateway
* Azure uses NAT Gateway

You don't define NAT rules — you attach NAT appliances.
Port forwarding is done via load balancers
Not through per-host NAT like pf.
This is one of the areas where pf’s expressiveness has no direct cloud equivalent.


### 7. Rule Ordering → Sometimes Important, Sometimes Ignored

`pf` processes rules in order.
`iptables` processes chains in order.

Cloud platforms all behave differently:

**GCP**

* rules have priorities
* lower number = evaluated first

**AWS Security Groups**

* no priority

* first matching allow wins

* denies are implicit, not explicit

**AWS NACLs**

* ordered, evaluated line-by-line

**Azure NSGs**

* like GCP: priority numbers decide order

Where `pf` is explicit and deterministic, cloud behavior varies significantly.

### Where `pf` Maps Nicely to Cloud Firewalls

Many `pf` concepts translate well:

* allow/deny → allow/deny
* stateful rules → default statefulness
* grouping by purpose → grouping by tags/SGs
* interface groups → NIC groups / ASGs / service accounts
* global policies → VPC-wide firewalls

### Where pf Mapping Completely Breaks

Some pf features simply don’t exist in the cloud model:

* no rdr or port redirect (load balancers replace it)
* no per-rule logging
* no interface-specific rules
* no fine-grained packet normalization
* no mixing stateless + stateful rules (except AWS NACLs)
* no control over NAT behavior

If you’re used to pf’s precision, the cloud’s abstraction can feel blunt.

### Why This Matters

If you think in `pf` terms, cloud firewalls will look familiar.
If you understand cloud firewalls deeply, `pf` will feel elegant and powerful.

But they are **not interchangeable**.

**Cloud firewalls:**

* enforce policies earlier,
* hide OS-level visibility,
* simplify some decisions,
* and remove others entirely.

Understanding this difference is what makes troubleshooting cloud networking feel logical instead of mystical.

---

## Common Cloud Firewall Mistakes (and Why Debugging Is Harder Than On Unix)

When engineers move from pf or iptables into cloud environments, they often assume that debugging connectivity will feel the same.
It doesn’t.

Cloud firewalls create new failure modes that simply don’t exist on bare metal.
They hide information the OS would normally see, they enforce rules earlier in the packet path, and they introduce abstractions that confuse even experienced sysadmins.

Here are the most common cloud firewall mistakes — and the real reasons they’re so hard to troubleshoot.

### 1. Expecting the VM to See Blocked Packets

On FreeBSD or Linux, if your firewall drops a packet:

* you can see it with tcpdump,
* you can log it,
* counters increment,
* and the kernel knows something happened.

In the cloud:

**Blocked packets never reach the VM at all.
The guest OS has zero visibility.**

This leads to classic scenarios:

* “Why does my VM never receive SYN packets on port 80?”
* “The connection just hangs. No logs.”
* “It works locally but not from outside.”

When the cloud firewall drops traffic, the OS sees nothing, and tools like tcpdump, ss, or even pf counters stay silent.

> This is the biggest mindset shift.

### 2. Fixing the Wrong Firewall — Host vs Cloud

A very common mistake:

You open a port inside the VM (pf or iptables), but forget to open it in the cloud firewall.

Or the reverse:

You open it in the cloud firewall, but a host-level firewall still blocks it.

Cloud firewalls don’t replace pf or iptables.
They sit in front of them.

This leads to the “Double Black Hole” problem:

> The packet is blocked somewhere… but neither layer tells you where.

Cloud firewalls rarely log per-rule decisions, and host firewalls never see the dropped traffic.

### 3. Forgetting That Cloud Firewalls Are Directional

On Unix, you often write something like:
```
pass in on em0 proto tcp to port 22 keep state
```

That single rule defines both inbound and outbound behavior.

Cloud firewalls split these concerns:

* ingress rules

* egress rules

And they are evaluated separately.

Common mistake:

> “I allowed inbound 5432, but my VM can’t talk to the database.”

Why?

Because outbound traffic to port 5432 was blocked by an egress rule.

In the cloud, you must open the path both ways, even with stateful firewalls.

### 4. Using IPs Instead of Identities

In pf, you often define rules based on addresses:
```
pass in from 10.0.0.5 to any port 22
```

In the cloud, this approach is fragile.

**VMs:**

* get replaced
* get recreated
* get new IPs
* auto-scale
* move across zones

If you rely on IPs, your rules quickly become obsolete.

Cloud-native firewalling expects you to use:

* tags (GCP)
* security groups (AWS)
* ASGs (Azure)
* service accounts (GCP)

**Identity-based rules scale.
IP-based rules don’t.**

### 5. Overlooking Implicit Rules

`pf` and `iptables` are intentionally explicit.

Cloud firewalls include hidden defaults:

Examples

**GCP**

* allows outbound traffic unless explicitly denied
* blocks all inbound by default

**AWS**

* SG outbound is always allowed unless changed
* NACLs allow everything unless modified

**Azure**

* has platform-level rules you cannot remove
* NSGs have hard-coded deny/allow behavior

These implicit rules can override your expectations and create subtle bugs.

### 6. Not Understanding Multi-Layer Firewalls (AWS)

AWS uses:

* Security Groups (stateful, per-ENI)
* NACLs (stateless, per-subnet)

If either layer blocks traffic, your VM will see nothing.

Debugging AWS is hardest when:

* the SG allows it
* but the NACL silently denies it

Or the opposite:

* NACL allows it
* SG silently denies it

This duality is powerful, but confusing for engineers coming from pf (where there’s only one firewall).

### 7. Misunderstanding NAT in the Cloud

In `pf`, NAT is explicit and local:
```
nat on em0 from 10.0.0.0/24 to any
```

In cloud environments:

* NAT is a service, not a firewall rule
* It may live in a different availability zone
* You cannot configure NAT behavior from the VM

If NAT is misconfigured, the VM’s firewall looks fine — but the VM still has no outbound access.

This is one of the top causes of “my VM can’t reach the Internet” tickets.

### 8. Thinking Debugging Works the Same as On-Prem

On Unix, you debug by:

* watching interfaces
* reading logs
* checking counters
* inspecting rules
* using tcpdump at the NIC level

In the cloud, these tools don’t show anything when traffic is blocked “upstream” in the virtual network.

Debugging cloud firewalls requires:

* checking VPC rules
* checking routing tables
* verifying NAT gateways
* checking load balancer health
* examining IAM bindings (GCP)
* verifying SG/NACL combinations (AWS)
* verifying NSG priorities (Azure)

A simple connectivity issue can touch four or five layers of infrastructure.

### Why Cloud Debugging Feels Harder

Three reasons:

1. The VM has no visibility into dropped packets.

You can’t debug what you can’t observe.

2. The firewall is no longer part of the OS — it’s part of the infrastructure.

This changes where and how failures appear.

3. Multiple layers interact.

A single misconfiguration in any layer can stop traffic entirely.

Once you internalize these differences, cloud networking becomes predictable again — and you debug faster than most engineers.



## Do You Still Need pf or iptables in the Cloud? Yes — Here’s When.

At first glance, cloud firewalls look like a complete replacement for on-host firewalls.
They filter traffic earlier, they’re centrally managed, and they scale across entire VPCs or projects.

So a natural question appears:

“If the cloud already protects my VM, do I still need pf or iptables?”

The answer is yes — and in more cases than most engineers expect.

Cloud firewalls control *infrastructure perimeter*.
Host firewalls control what *happens inside the machine.*

They serve different purposes, and neither replaces the other.

Here’s when you still need `pf`, `iptables`, or `nftables` even in a fully cloud-native environment.

### 1. When You Need Micro-Segmentation Inside the VM

Cloud firewalls filter traffic before it reaches the VM, but once inside the machine, it’s open.

If you want to segment:

* specific applications
* containers
* services running on localhost
* internal ports used between daemons
* processes in different privilege domains

…then only host-level firewalls can enforce that.

Examples:

* block local MySQL except for one service
* prevent a compromised process from accessing internal APIs
* isolate containers without a service mesh

Cloud firewalls **cannot** see or filter internal VM traffic.

### 2. When You Want to Protect Against Lateral Movement

Imagine one VM gets compromised.

Cloud firewalls won’t stop lateral traffic originating inside that VM.
They only filter ingress/egress at the infrastructure level.

A host firewall can:

* block unnecessary east-west connections
* restrict outgoing traffic
* reduce the blast radius
* enforce strict segmentation inside the workload

This matters especially in:

* multi-tenant environments
* hybrid cloud setups
* older monolithic applications
* environments without a service mesh

### 3. When You Need Features Cloud Firewalls Don’t Provide

`pf` and `iptables` offer capabilities cloud providers do not expose, such as:

NAT and port redirection (rdr)

Useful for:

* local port forwarding
* transparent proxies
* redirecting traffic internally for testing
* migrating legacy applications

Cloud NAT and Load Balancers are not substitutes here.

**Traffic shaping and rate limiting**

Examples:
```
pf:   queue high, queue low
tc:   htb, fq_codel
```

Cloud firewalls cannot limit bandwidth or queue traffic.

**Packet normalization (scrub)**

`pf` can fix or sanitize malformed packets.
Cloud firewalls simply drop them.

**User-space filtering (UID/GID-based rules)**

Cloud firewalls cannot enforce rules like:
```
pass out on em0 proto tcp to any port 443 user www
```

These are local concerns that require local firewalls.

### 4. When You Want Better Visibility and Debugging

Cloud flow logs tell you:

* some packets were allowed or denied
* from which IP
* on which port
* at which time

But they do not tell you:

* which application accepted the connection

* which process used the port

* which user initiated the traffic

* why a packet was dropped

* which firewall rule matched

Host firewalls fill this gap.

With `pf`, `iptables`, or `nftables` you can:

* capture detailed logs
* count packets per rule
* observe the actual kernel behavior
* debug advanced network problems
* run tcpdump at multiple layers

Cloud firewalls hide too much for deep troubleshooting.

### 5. When You Run Kubernetes or Containers

Cloud firewalls sit outside Kubernetes.
Inside the cluster, communication is:

* pod → pod
* node → node
* sidecar → service
* service → external

Cloud firewalls cannot see these flows.

Kubernetes typically relies on:

* container-level iptables
* CNI plugin policies
* eBPF-based enforcement (Cilium, Calico)

So yes — iptables is still everywhere, even in modern clusters.

### 6. When You Run Services That Only Host Firewalls Understand

Some services require OS-level firewall rules:

* VPN servers (WireGuard, IPSec)
* NAT64/NAT46 translators
* reverse proxies with local redirect
* transparent proxies
* high-performance forwarding paths

Cloud firewalls don’t handle these use cases.
`pf`, `iptables`, or `nftables` do.

### 7. When You Want Defense-in-Depth

The strongest architecture uses:
* cloud firewalls for the outer perimeter
* host firewalls for inner segmentation
* service mesh or CNI policies for pod-level or microservice observability

This is how modern zero-trust networks are built.

Relying on cloud firewalls alone gives you coarse control.
Combining both gives you real defense.

### The Bottom Line

Cloud firewalls simplify networking and reduce operational burden.
But they don’t eliminate the need for host-level firewalls — not even close.

pf, iptables, and nftables remain essential when:

* traffic stays inside the VM,
* application-level control is required,
* advanced features are needed, or
* you want actual visibility into packet flow.

The cloud gives you guardrails.
Your host firewall gives you precision.

**Both matter.**



## The New Perimeter — Why Cloud Firewalls Matter More Than Ever

By now, one thing should be absolutely clear:

Cloud firewalls are not just a newer version of pf or iptables.
They are a shift in where security lives.

On traditional Unix systems, the firewall is part of the machine.
In the cloud, the firewall is part of the infrastructure fabric — the virtual network that all machines share.

And that shift changes everything:

* packets are filtered before the OS even sees them
* identity replaces IPs as the primary security primitive
* rules apply across entire fleets, not individual servers
* debugging moves from the kernel to the control plane
* security becomes centrally managed, not host-by-host

Cloud firewalls feel simpler on the surface, but beneath that simplicity is a powerful idea:

**Security enforcement no longer depends on the correctness of a single machine.**

You can destroy a VM, recreate it, autoscale it, or move it across zones — and your firewall rules stay intact.
You can apply policies globally without touching a single config file.

This is the kind of abstraction that makes modern cloud possible.

But abstractions have limits — and that’s where your traditional Unix knowledge still matters.