# Security Begins at the Network Layer

Firewalls are where that precision starts. They’re not flashy or modern, but they define the invisible rules of trust: who can talk, who can’t, and under what conditions.
Get it wrong, and everything above it — from SSH to Kubernetes — inherits your mistake.

In the Unix world, there are two main philosophies of control: `pf` on the BSD side, and `iptables` (and its successor, `nftables`) on Linux. Both protect systems, both trace their roots back decades, but they come from very different schools of thought.

This article is about understanding those differences — not in marketing terms, but in practice.
How each system approaches security, how configuration reflects its philosophy, and why, after all these years, FreeBSD’s `pf` still feels like the firewall that was designed by engineers, not committees.

Because good security isn’t about fear.
It’s about knowing exactly what your system will do — and why.

Unix Firewalls: Philosophy and Evolution

Unix systems have always shared a simple idea: everything that moves across a network should be observable and controllable.
But how that control is expressed — and how much complexity an engineer must accept — diverged sharply between the BSD and Linux worlds.

## The BSD Lineage — Clarity by Design

The BSD family has always favored explicit configuration over abstraction.
In the early days, `ipfw` handled packet filtering — a capable but low-level system that worked fine when networks were small.
Then came `pf`, short for Packet Filter, originally developed for OpenBSD in 2001 and later adopted by FreeBSD, NetBSD, and others.

`pf` wasn’t just a new firewall. It was a new way of thinking about policy.
Instead of managing chains or tables of cryptic rules, it read like a language — almost English in structure:
```
block in all
pass out quick on egress keep state
pass in on em0 proto tcp from any to any port 22 keep state
```

That simplicity made it easy to reason about what a system would do.
No hidden layers, no daemons rewriting your rules, no YAML or XML in sight — just a clean text file, `/etc/pf.conf`, that could describe your entire network stance in a handful of lines.

It reflected the BSD mindset perfectly: clarity is security.
When you can read your own firewall rules six months later and still understand them, you’re less likely to make catastrophic mistakes.

## The Linux Path — Power Through Abstraction

Linux took a different route.
Its early firewalls, `ipchains` and later `iptables`, were built for ultimate flexibility — letting administrators build almost any kind of filtering logic imaginable.
But flexibility came at the cost of readability.

Instead of sentences, iptables works with statements:

```
iptables -A INPUT -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

Powerful? Absolutely.
But verbose, hard to audit, and easy to get wrong — especially when you start combining dozens of chains and tables.

To its credit, Linux eventually evolved.
The modern `nftables` subsystem now replaces iptables with a cleaner syntax and better performance, but the conceptual model remains the same:
firewall as programmable logic, not policy description.

It’s a difference in philosophy that runs deep.

## Two Schools of Thought

Although both firewalls serve the same purpose — deciding what traffic to allow or block — they express that logic very differently.

`pf` (on FreeBSD) takes a declarative approach.
You describe what you want to happen in a simple, rule-based way. The configuration reads like a network policy document — clean, human-readable, and easy to reason about even months later. It’s designed for predictability and clarity.

`iptables` (and its successor `nftables`) on Linux follow a procedural model.
You build your firewall step by step — command by command — defining chains, tables, and matches. It’s powerful and extremely flexible, but it feels more like programming a filter engine than declaring a policy. That flexibility comes at the cost of complexity.

In short:

pf is written like a policy.

iptables is written like code.

Both work.
But only one can be confidently debugged at 2 a.m. without a manual open next to you.

## `pf` — The FreeBSD Way

When you open a FreeBSD system and look at `/etc/pf.conf`, there’s a sense of calm.
No chains, no tables named filter, nat, or mangle — just clean, declarative logic that reads more like a policy statement than a script.

`pf`, short for Packet Filter, is more than just a firewall. It’s a network policy language that favors precision over cleverness.
Every rule is evaluated top to bottom, and once you understand the flow, you can reason about your system’s behavior in seconds — even after months away.

### Structure of a pf Configuration

A basic `/etc/pf.conf` file often follows a simple, logical layout:
```sh
# Macros
ext_if = "em0"
tcp_services = "{ 22, 80, 443 }"

# Options
set skip on lo
set block-policy drop

# Default Deny
block in all
block out all

# Allow outbound and established traffic
pass out quick on $ext_if inet proto { tcp, udp } from any to any keep state
pass in quick on $ext_if proto tcp from any to any port $tcp_services keep state
```

Even without knowing pf syntax, you can probably read that out loud and understand it:

Define what “external” means (`em0`).

Decide which ports to open (`22, 80, 443`).

Drop everything by default.

Pass what you explicitly trust.

That’s it. No hidden rules, no implicit “allow all outbound” — you control every packet.

### Stateful by Default

One of pf’s most powerful features is its state tracking.
When you allow an outbound connection with keep state, pf automatically allows the return traffic.
There’s no need to write a second rule for the response path — the state table keeps track of active sessions.

You can inspect it anytime:
```sh
sudo pfctl -s state
```

That gives you real-time visibility into every connection — source, destination, protocol, and duration.
This is the kind of feedback loop that makes debugging and learning network behavior intuitive.

### Tables and Macros — pf’s Secret Weapon

`pf` also provides tables — dynamic lists of IP addresses that can be referenced by name in your rules.
They’re perfect for blocking large address ranges, whitelisting networks, or managing live blacklists.
```
table <badhosts> persist file "/etc/pf.badhosts"
block in quick from <badhosts> to any
```

You can update the table without reloading the entire firewall:
```
sudo pfctl -t badhosts -T add 203.0.113.5
```

This ability to modify behavior in real time without disrupting traffic is something even many modern Linux firewalls still struggle with cleanly.

### Anchors and Modularity

`pf` lets you include sections of rules from other files, known as anchors.
This makes complex configurations manageable and reusable — you can separate web server rules from VPN rules, for example, without creating separate rule sets.
```
anchor "web"
load anchor "web" from "/etc/pf.web.conf"
```

This modularity turns pf into something that scales — from a home lab to a multi-interface router.

### Clarity That Scales

`pf` works well because it doesn’t try to be clever.
It enforces the philosophy that a firewall should be auditable, predictable, and self-explanatory.
That’s why it remains the default on OpenBSD, the trusted choice for FreeBSD users, and the quiet backbone of countless routers and security appliances around the world.

You can trust `pf` not because it hides complexity — but because it exposes it, clearly.

## iptables and nftables — The Linux Way

If `pf` feels like policy, `iptables` feels like code.
It’s built around the idea that flexibility is everything — and that if you give engineers enough control, they’ll build whatever logic they need.

That philosophy has shaped Linux networking for over two decades.
It’s powerful, modular, and scriptable — but it also means there’s more to keep in your head.
A single typo or a rule in the wrong chain can silently turn a secure host into an open one.

### The `iptables` Model

`iptables` organizes its logic around tables (types of operations) and chains (execution paths).
Each table contains chains such as INPUT, OUTPUT, and FORWARD, and each chain holds rules that match packets and decide their fate.

A typical configuration might look like this:
```sh
# Flush old rules
sudo iptables -F

# Default policies
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Allow SSH and web
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow established responses
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

It’s explicit, granular, and extremely powerful.
But readability suffers — even a short configuration can span dozens of lines, and debugging becomes an exercise in tracing rule order across multiple chains.

Still, iptables built Linux’s reputation for flexibility: NAT, mangle rules, advanced matches, and user-defined chains let administrators fine-tune packet behavior at every stage.

### Enter nftables

By the 2010s, Linux networking needed a cleanup.
`iptables`, `ip6tables`, `arptables`, and `ebtables` all had separate syntax, binaries, and kernel hooks.
The solution came in the form of `nftables` — a unified firewall framework introduced in the Linux 3.13 kernel.

`nftables` keeps the same conceptual model but introduces a simpler, more consistent syntax and a central ruleset.
You define what you want once, and `nftables` handles both IPv4 and IPv6 traffic under the hood.

Example equivalent `nftables` rules:
```
table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;

    ct state established,related accept
    tcp dport { 22, 80, 443 } accept
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

It’s cleaner, more readable, and much easier to export or modify as a single unit:
```
sudo nft list ruleset
```

Underneath, `nftables` uses a pseudo-VM in the kernel to execute rules more efficiently than the linear chain traversal in iptables.
The performance gains are real, especially for large or dynamic rulesets — but the mental model remains procedural: packets flow through a sequence of evaluated conditions until one matches.

### Tools, Layers, and Abstractions

Linux’s flexibility means there’s rarely one right way to manage a firewall.
You can edit `iptables` directly, use `ufw` (Ubuntu), `firewalld` (RHEL/Fedora), or rely on container networking layers that inject their own rules dynamically (Docker, Kubernetes).

Each abstraction adds convenience but also opacity.
It’s easy to reach a point where even the system administrator can’t quite tell why a specific port is open.

That’s the tradeoff:

On FreeBSD, you own the rules.

On Linux, you often own the outcome, but the path there can be surprisingly crowded.

### Strengths and Weaknesses

The Linux firewall stack remains unmatched in breadth and integration — it supports container isolation, traffic shaping, connection tracking, and NAT, all in one framework.
But that same integration can make it harder to maintain and reason about in production.

`pf` gives you confidence by showing you exactly what it’s doing.
`iptables` and `nftables` give you possibilities — and it’s your job to make sure they behave.

## pf vs iptables: Simplicity vs Flexibility (Real-World Comparison)

Let’s move beyond the textbook examples.
Imagine you’re securing a small public-facing server — it runs a few services:

* SSH (22) for admin access
* HTTP (80) and HTTPS (443) for a website
* NTP (123) for time sync

and it must block all inbound SMTP (25) because this isn’t a mail server.

On top of that, you want logging for dropped packets and rate-limiting SSH to prevent brute-force attacks.

That’s where the philosophies of pf and iptables start to diverge in a way you can feel.

### pf — concise, policy-driven, readable
```sh
# /etc/pf.conf
ext_if = "em0"
web_ports = "{ 80, 443 }"
ssh_guard = "{ 22 }"

set block-policy drop
set skip on lo
set loginterface $ext_if

# Default deny
block all

# Allow established sessions
pass in quick on $ext_if proto { tcp, udp } from any to any keep state

# Web services
pass in on $ext_if proto tcp from any to any port $web_ports keep state

# SSH with rate-limit (max 5 connections in 30 seconds)
pass in on $ext_if proto tcp from any to any port $ssh_guard \
    flags S/SA keep state (max-src-conn-rate 5/30, overload <ssh-abuse> flush global)

# Time sync (outgoing only)
pass out on $ext_if proto udp from any to any port 123 keep state

# Block mail explicitly
block in log on $ext_if proto tcp from any to any port 25
```

Readable. Declarative. Self-contained.
Every rule says what you want, not how to achieve it.

You can audit this file line by line and instantly know what traffic is allowed, logged, and rate-limited.
And if something breaks, you check one file — not a dozen chains or scripts.

pf’s rate-limiting and dynamic tables (<ssh-abuse>) are built-in.
You can blacklist aggressive IPs automatically — no external daemon required.

### iptables — explicit, procedural, layered

Now let’s try to express the same logic on Linux:
```sh
# Flush and reset
iptables -F
iptables -X
iptables -t nat -F
iptables -t mangle -F

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established/related sessions
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Web services
iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack \
    --ctstate NEW,ESTABLISHED -j ACCEPT

# SSH with rate limit (requires hashlimit module)
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
    -m hashlimit --hashlimit 5/minute --hashlimit-burst 3 \
    --hashlimit-mode srcip --hashlimit-name ssh_guard -j ACCEPT

# NTP (outbound only — allow replies)
iptables -A INPUT -p udp --sport 123 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# Block mail and log
iptables -A INPUT -p tcp --dport 25 -j LOG --log-prefix "SMTP_DROP: "
iptables -A INPUT -p tcp --dport 25 -j DROP
```

The result: it works — but you can feel the difference.

The logic is spread across multiple layers, and every small change means touching several commands.
To make it permanent, you’ll also need to export and re-import your rules:
```sh
sudo iptables-save > /etc/iptables/rules.v4
```

Even experienced administrators often rely on wrapper tools (`ufw`, `firewalld`, `nft`) because manually maintaining iptables becomes a chore over time.

### What the difference feels like

`pf`:

* feels like reading a security policy document.
* rule order and evaluation are intuitive.
* easy to apply, audit, and share between systems.

`iptables`:

* feels like writing a mini program that controls packet flow.
* incredibly powerful, but verbose and fragile.
* one typo away from locking yourself out.

pf assumes you understand your intent.
`iptables` assumes you’ll build it yourself — one packet at a time.

Both are tools of control, but pf favors clarity of expression,
while `iptables` favors granularity of control.

## Performance, Stability, and Integration

Performance in firewalls isn’t just about packet-per-second numbers — it’s about consistency.
When traffic spikes, rules change, or logs rotate, you want a system that stays predictable.
That’s where the design choices behind `pf` and `iptables/nftables` start to matter most.

### pf — Built for Predictable Load

The Packet Filter was designed for stability under pressure.
From its OpenBSD roots, `pf` has always favored determinism over dynamic trickery.
It runs in kernel space, uses a single-pass rule evaluation model, and keeps all state tracking in a centralized table that’s both efficient and easy to inspect.

Even on modest hardware, `pf` can handle thousands of active connections without jitter or packet loss.
You can literally watch it scale:

```sh
sudo pfctl -s info
sudo pfctl -s state | wc -l
```

Because rules are compiled once and loaded atomically, `pf` avoids partial reloads or transient states — something that can happen in Linux when chains are re-applied on a busy system.

The same design discipline makes `pf` incredibly stable for long-running servers, routers, or VPN gateways.
Many embedded network appliances — from commercial firewalls to OpenBSD-based routers — rely on pf precisely because it’s boringly reliable.

### `iptables` — Flexible, but Stateful Complexity

`iptables` was built for flexibility, and that flexibility comes with overhead.
Every rule belongs to a chain, every chain belongs to a table, and packets traverse those chains in order.
The more rules you add, the more lookups the kernel performs — linearly — until a match is found.

For small setups, the difference is negligible.
For large or dynamically managed networks, it can become measurable, especially when rules are frequently inserted or removed by automation tools like firewalld, Docker, or Kubernetes.

Still, Linux’s connection tracking engine is robust — it can track millions of connections efficiently, provided you tune nf_conntrack_max and related parameters.
But tuning is part of the job.
pf, in contrast, rarely needs it.

### nftables — The Modern Middle Ground

`nftables` addresses many of these performance challenges.
Instead of linear rule traversal, it uses an internal bytecode engine that evaluates rules more efficiently and can represent multiple protocols (IPv4, IPv6, ARP, etc.) in a single unified rule set.

Reloading rules is atomic and doesn’t disrupt existing connections — finally matching pf’s behavior.
In many ways, `nftables` brings Linux closer to the simplicity pf users have had for years.

But `nftables` also inherits Linux’s biggest challenge: integration sprawl.
Containers, orchestration tools, and cloud platforms all inject their own firewall logic.
By the time you list your rules, you might see chains that belong to Docker, Podman, or NetworkManager — each managing its own subset of traffic.

That complexity makes performance troubleshooting harder, even when raw throughput is excellent.

### Practical Reality

In everyday use:

`pf` feels stable because it is — rule reloads are atomic, syntax is deterministic, and its state management is transparent.

`iptables` and `nftables` can match or exceed pf’s throughput in benchmarks, but maintaining them in a live, evolving environment often introduces human and procedural fragility.

And as anyone who’s operated real systems knows:
a stable firewall that never surprises you is faster than one that needs constant tuning.

## Integration and Tooling

Firewalls don’t live in isolation.
They’re part of a system — connected to logs, interfaces, monitoring agents, and automation tools.
And while both pf and iptables guard packets, their ecosystem integration tells two very different stories.

### `pf` — The Engineer’s Toolkit

FreeBSD’s firewall stack is built around the idea that one tool should do one thing well — and together, they form a clean, composable workflow.

``pfctl`` — the control utility.
It loads, flushes, and inspects the active ruleset:
```sh
sudo pfctl -sr        # Show loaded rules
sudo pfctl -si        # Show interface stats
sudo pfctl -ss        # Show active state table
sudo pfctl -nf /etc/pf.conf   # Dry-run configuration check
```

Every command gives you human-readable output — no need to parse JSON or XML.

``pflog`` — built-in packet logging.
Logged packets go into a pseudo-interface (pflog0), which you can monitor in real time:
```
sudo tcpdump -n -e -ttt -i pflog0
```

That single line lets you trace what the firewall is doing, packet by packet — a luxury when debugging complex policies.

Integration with rc(8).
`pf` starts and stops like any other system service.
Rules are declared in `/etc/pf.conf`, and startup is managed declaratively through` /etc/rc.conf`:
```
pf_enable="YES"
pflog_enable="YES"
```

There are no daemons rewriting your configuration behind the scenes — which means no surprises.

### Extensibility.
`pf` integrates naturally with other BSD subsystems: `altq` for traffic shaping, `carp` for redundancy, `pfsync` for state synchronization between firewalls.
The same philosophy extends from a single laptop to an enterprise-grade HA cluster.

In short, `pf` gives you direct control with minimal moving parts — perfect for administrators who prefer clarity to automation magic.

### `iptables/nftables` — Layers Upon Layers

Linux, on the other hand, sits at the heart of modern infrastructure — from laptops to hyperscale clouds — and its firewall tools evolved accordingly.
That evolution brought both power and fragmentation.

Core tools:

`iptables` — the legacy user-space CLI for managing kernel chains and tables.

`nft` — the modern replacement that unifies IPv4, IPv6, ARP, and bridging rules.

Both provide complete control, but neither are truly user-friendly.
```
sudo iptables -L -v -n
sudo nft list ruleset
```

Reading these outputs feels more like reading source code than a configuration file.

Wrapper tools:
To simplify administration, distributions introduced abstraction layers:

`ufw` (Ubuntu): beginner-friendly, declarative syntax.
`firewalld` (RHEL/Fedora): D-Bus–based daemon with zones and runtime reloads.
`iptables-persistent`: saves rules for system startup.

Each abstraction helps in one environment — but complicates another.
`ufw` ignores `nftables`, `firewalld` injects chains dynamically, and containers add their own.

Container and Cloud Integration:
Docker, Kubernetes, and CNI plugins generate firewall rules dynamically in `iptables/nftables` to enforce network policies.
While this integration is powerful, it also means your rule set is no longer entirely yours.
One misbehaving container can rewrite or reorder chains without you realizing it.

### Monitoring and Logging:
Linux supports logging via `LOG` targets or `NFLOG`, integrated with `journalctl` or external tools.
It works — but it lacks the elegance of `pflog`.
Tracing packets across dynamically inserted rules requires skill, patience, and a scroll buffer that can stretch for pages.

### Two Toolchains, Two Mindsets

`pf` assumes you are the system’s brain — it provides tools, not frameworks.
Linux assumes the system will evolve around you — it provides layers for coexistence and orchestration.

`pf’`s minimalism makes it a joy to debug.
iptables and nftables’ flexibility makes them indispensable in distributed systems — but also harder to trust completely, because too many layers may modify behavior before the packets ever reach your rules.

Both ecosystems are mature and capable.
The choice isn’t about which one is “better” — it’s about how much abstraction you can tolerate before you start losing visibility.

## Why FreeBSD Still Wins

When you’ve managed systems long enough, you realize that security isn’t about locking everything down — it’s about controlling complexity.
Every rule, every daemon, every automation layer adds a little more unpredictability.
And when something breaks at 3 a.m., predictability is the only thing that matters.

That’s where FreeBSD’s pf earns its reputation.

It’s not that pf is faster, flashier, or newer.
It’s that pf is composed, understandable, and consistent — qualities that scale far better than raw performance ever will.

1. Declarative Simplicity

`pf` speaks in complete thoughts.
Every line in `/etc/pf.conf` expresses intent, not mechanics:
```
block in all
pass in on em0 proto tcp to port { 22, 443 } keep state
```

That’s not syntax sugar — it’s philosophy.
You’re describing policy, not micro-managing chains and tables.
When someone reads your configuration, they see what you meant, not just how you implemented it.
And because rules are atomic and readable, configuration drift is almost nonexistent.

2. Predictable Operations

Reloading a `pf` ruleset doesn’t interrupt traffic.
It happens atomically — either the new rules load successfully, or they don’t load at all.
There’s no “half-applied” firewall state, no invisible race conditions.

When you inspect your rules with `pfctl -sr`, you’re seeing exactly what’s running.
No hidden chains from Docker, no system daemons rewriting policies in the background.

That predictability is gold when your uptime — or your reputation — depends on consistency.

3. Integration Without Entanglement

`pf` integrates cleanly with the rest of the FreeBSD ecosystem —
`rc.conf`, `pflog`, `altq`, `carp`, and `pfsync` — without needing a management daemon or plugin layer.
It respects the Unix philosophy: small, composable tools, each doing one job well.

The result is a system where you always know where configuration lives, how it’s applied, and how to reproduce it on another host.
You don’t need to “manage the manager.”

4. Security by Design, Not Reaction

`pf`’s development model comes from the OpenBSD lineage — where correctness and security are cultural defaults, not afterthoughts.
That mindset carries over to FreeBSD.
While Linux continuously evolves around new frameworks and container abstractions, FreeBSD tends to evolve inwards — making what already exists more robust.

That’s not stagnation. That’s maturity.
You get fewer regressions, fewer “breaking changes,” and more time to focus on what actually matters: system behavior.

5. The Calmness Factor

When you open a FreeBSD system after a year, it behaves the same way it did the day you left it.
`pf` rules still load the same.
Logs still look the same.
There’s something quietly powerful about that — a stability that earns trust the longer you use it.

In contrast, on Linux, even minor version upgrades can change how `nftables`, `firewalld`, or `NetworkManager` interact.
You can still achieve reliability — but it takes more work, more testing, and more documentation.

### Not About Winning — About Knowing Why

pf isn’t “better” because it’s from BSD.
It’s better because it’s honest: it shows you exactly what it’s doing, with no hidden layers.
And when your security depends on transparency, that honesty is worth more than any new feature.

FreeBSD’s `pf` represents the kind of engineering that lasts — minimal, predictable, and designed for humans who need to understand systems, not just configure them.

>In the end, “FreeBSD still wins” isn’t a slogan — it’s a quiet statement about reliability.
>You can’t automate trust. You have to build it, one packet at a time.