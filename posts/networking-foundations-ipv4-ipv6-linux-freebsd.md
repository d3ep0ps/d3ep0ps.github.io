# Networking Foundations: IPv4, IPv6, and Basic Configuration on Linux and FreeBSD
> Before you can secure a system, you must make it accessible.

## Where It All Started — Six Years at an ISP
I spent six years working as a system engineer at an Internet Service Provider (ISP) — the kind of place where networking isn’t just theory, it’s *survival*.

Routers, BGP sessions, fiber cuts at 2 a.m., and the quiet satisfaction when a link comes back online — that was daily life.

Those years taught me more about how the Internet truly works than any textbook ever could.

You see, at an ISP, networking isn’t a service — it’s the **heartbeat** of everything else. Every packet matters, every prefix has a story, and every configuration line can bring silence or stability.

That background shaped how I approach systems today. Whether it’s Linux, FreeBSD, or cloud platforms, I start from the same principle I learned back then:

> “If you don’t understand the network, you don’t understand the system.”

This post is about that foundation. We’ll take a close, practical look at:

* how IP addressing actually works (IPv4 and IPv6),
* the difference between static and dynamic configuration,
* and how to bring up networking on both *Linux* and *FreeBSD* — step by step.

This isn’t a deep dive into theory — it’s the kind of knowledge every engineer should have before touching firewalls or cloud networks.

Because everything else depends on one thing: **your system being able to talk**.

---

## What an IP Address Really Is
At its core, networking is about identity and direction — who you are and how to reach others.

Every device on a network needs a unique address, a kind of digital coordinate that tells packets where to go and how to come back.

That coordinate is your IP address.

There are two versions of this language in use today: IPv4 and IPv6. They do the same job, but they speak in very different scales.

### IPv4 — The Classic Standard
IPv4 uses a 32-bit number written as four decimal values separated by dots:

`192.168.1.10`

Each section (called an *octet*) can range from 0 to 255, giving a total of about 4.3 billion possible addresses — which used to sound infinite.

Every IPv4 address includes two parts:

* The **network portion** — defining which subnet the device belongs to.
* The **host portion** — defining the specific device within that subnet.

We often describe this using CIDR notation (Classless Inter-Domain Routing), like `/24`, which means “the first 24 bits identify the network.”

**Example:**
* `192.168.1.0/24` → network
* `192.168.1.10` → host inside that network

It’s compact, reliable, and deeply familiar — but also limited. The world simply ran out of IPv4 space.

### IPv6 — The Next Generation
IPv6 was designed to solve that limitation — and then some.

It uses 128 bits, written in hexadecimal and separated by colons:
```sh
fd42:abcd:1234::10 ### local address
2a00:1450:4025:802::64 ### google.com
```
That single format allows for **340 undecillion** addresses — enough for every grain of sand on Earth (and then a few more).

But the structure remains simple:

* The **prefix** defines the network.
* The **interface ID** defines the device.

IPv6 also introduces several address “types” — link-local, global, and unique-local — that make it more flexible for complex environments.

If IPv4 feels like a phone directory, IPv6 is a *global GPS system*: you can define, locate, and connect anything — without translation or NAT.

### The Core Idea
Whether it’s IPv4 or IPv6, every address defines the same relationship:

> “This system belongs to this network — and it can reach others through that gateway.”

Everything else — routing, firewalls, VPNs, cloud connectivity — builds on that foundation.


## How Systems Get Their IPs — Static, Dynamic, and the Cloud Way
Once you understand what an IP address is, the next question becomes: **how does your system actually get one?**

There are two main ways this happens — either the machine is *told* what address to use, or it *asks* for one automatically.

### Dynamic Allocation — DHCP
When a device joins a network without knowing its identity, it requests assistance from the network.

That’s the job of **DHCP (Dynamic Host Configuration Protocol)**. The process is simple but elegant:

1.  The device broadcasts a request: “Can someone assign me an IP?”
2.  A DHCP server responds with an offer: “Take `192.168.1.50`, use `192.168.1.1` as your gateway, and here’s your DNS.”
3.  The device accepts, and the network handshake completes.

DHCP makes sense when flexibility is more important than control — for example:

* personal computers and laptops,
* virtual machines that spin up and down frequently,
* or cloud environments where the provider manages the networking layer on your behalf.

In most home or enterprise setups, DHCP quietly does its job in the background — assigning, renewing, and recycling IPs so you don’t have to think about it.

### Static Allocation — Predictability and Control
For servers and routers, *consistency* matters more than convenience.

That’s why these systems use **static IPs** — addresses manually configured and written into their network settings.

A static setup typically includes:

* IP address and subnet mask (or prefix)
* default gateway
* DNS servers

This ensures your system always comes up on the same address, which is essential for:

* web servers,
* load balancers,
* VPN endpoints,
* and anything other systems must reliably reach.

In short, DHCP is about flexibility. Static configuration is about stability.

### The Cloud’s Hybrid Approach
Modern cloud environments automate all this, but the same principles still apply.

Under the surface, cloud networking is just *DHCP with policy*.

**Internal IPs** (like `10.x.x.x` or `172.16.x.x`) are usually assigned dynamically when an instance is created — but the cloud control plane can “reserve” them, making them effectively static.

**External or public IPs** are mapped using NAT — exactly how routers and ISPs have done it for decades.

When you “promote” an ephemeral IP to a static one in GCP or AWS, what you’re really doing is *locking the lease* — telling DHCP never to reassign it.

Even in the cloud era, the fundamentals haven’t changed:

> The cloud didn’t reinvent networking — it just automated what engineers used to do by hand.

### How Cloud VMs Get Their IPs — Behind the Scenes on GCP
When you create a VM on Google Cloud, the networking configuration seems to “just work” — your instance boots, `eth0` or `en0` comes online, and you already have both IPv4 and IPv6 ready.

But under the hood, the process is handled by a few simple but elegant mechanisms.

Here’s how it happens on *Debian-based* GCP images:

1.  **udev brings the interface online**
    When the VM boots and `eth0` or `en0` is detected, a udev rule fires automatically from:
    `/etc/udev/rules.d/75-cloud-ifupdown.rules`

2.  **The helper applies a cloud-specific template**
    The rule triggers the script:
    `/etc/network/cloud-ifupdown-helper`

    Instead of manually configuring IPs or querying metadata, this helper simply *applies a pre-defined template* called `cloud-interfaces-template`, which looks like this:
    ```ini
    auto $INTERFACE
    allow-hotplug $INTERFACE
    
    iface $INTERFACE inet dhcp
    
    iface $INTERFACE inet6 manual
      try_dhcp 1
    ```
    This tells the system to obtain its IPv4 address dynamically via DHCP and to attempt DHCPv6 for IPv6 — exactly how cloud networking is meant to behave.

3.  **DHCP happens behind the scenes**
    The GCP hypervisor provides a *virtual DHCP server* on the local network. When the VM requests an address, the DHCP server responds with the correct internal IP, subnet, and gateway — derived from the instance metadata you see in the console.

4.  **The result: clean, declarative networking**
    From the OS’s perspective, it’s all just standard DHCP — no hardcoded IPs, no special binaries, no hidden cloud magic. If you ever disable or replace these udev rules, the interface won’t come up automatically, because nothing will apply that template during boot.

This design keeps GCP images clean and maintainable:

* It stays 100% compatible with traditional Debian networking tools.
* It delegates configuration to cloud infrastructure via DHCP, not via system scripts.
* It avoids embedding any cloud-specific logic directly into `/etc/network/interfaces`.

In short:
> Google Cloud makes Linux networking “cloud-native” by automation, not by rewriting how Linux works.

### Why It Matters
When something breaks, understanding *how* your system got its IP is the key to fixing it.

Perhaps the DHCP lease has expired, the gateway is missing, or a static IP address overlaps with another system.

These are classic problems that appear everywhere — from bare metal to virtual networks.

So before touching firewalls or load balancers, always make sure of one thing:
> Your system knows who it is — and how to find the rest of the network.

---

## How the Internet Stays Organized — RFCs and Private Networks
The Internet might look like an endless mesh of routers and cables, but underneath it runs on order — a system of agreements and standards that keep billions of devices from stepping on each other’s toes.

That order comes from **RFCs — Request for Comments** — the open documents that define how every part of the Internet behaves.

One of the most important areas these RFCs define is **address space**: which IPs are public and can be routed globally, and which ones are private — safe to use inside your own networks, labs, or datacenters.

These **private address spaces** form the invisible backbone of every internal network, from your home router to hyperscale cloud platforms.

### IPv4 — RFC 1918: Private Address Space
RFC 1918 defines three IPv4 blocks that are reserved for local networks. They’ll never be routed on the public Internet — meaning you can use them freely without risk of collisions.

* **Network:** `10.0.0.0/8`
    **Range:** `10.0.0.0` – `10.255.255.255`
    **Common Use:** Large enterprise or ISP internal networks

* **Network:** `172.16.0.0/12`
    **Range:** `172.16.0.0` – `172.31.255.255`
    **Common Use:** Corporate and datacenter environment

* **Network:** `192.168.0.0/16`
    **Range:** `192.168.0.0` – `192.168.255.255`
    **Common Use:** Home routers and small LANs

For your *d3ep0ps* lab, a simple and safe choice is the `10.0.2.x` range. It’s clean, consistent, and avoids conflicts with most consumer networks.

### IPv6 — RFC 4193: Unique Local Addresses (ULA)
IPv6 has its own concept of private networks, known as **Unique Local Addresses (ULA)**, defined in RFC 4193.

* **Range:** `fd00::/8`
* **Description:** Internal use only, not routed globally
* **Example:** `fd42:abcd:1234::10/64`

ULAs are like private IPv6 networks that you can safely design yourself. Unlike IPv4’s static ranges, ULAs include a random component to avoid collisions when two private networks connect.

They’re especially useful in labs and hybrid environments — giving you the realism of IPv6 without touching the public Internet.

### Link-Local Addresses — The Built-In Fallback
Even if you don’t assign any IPs manually, your system still generates a **link-local address**.

These are used for neighbor discovery and local diagnostics — they only work within the same physical or virtual link.

* **IPv4 (RFC 3927):** `169.254.0.0/16`
* **IPv6 (RFC 4291):** `fe80::/10`

If you’ve ever seen one of those pop up on boot when DHCP failed, that’s your system saying:
> “I can’t reach the Internet, but I can still talk locally.”

### Why This Matters
Understanding which networks are private, reserved, or link-local helps you design safe and predictable environments.

It’s how you avoid routing conflicts, keep lab traffic isolated, and build configurations that scale — from your home VM setup to multi-tenant cloud networks.

In short:
> Public IPs connect the world. Private IPs let you build your own.

---

## Configuring Network Interfaces — Linux vs FreeBSD
At some point, theory meets reality — your machine needs a real IP address and a working gateway.

This is where Linux and FreeBSD begin to reveal their distinct personalities.

Both achieve the same outcome — a live, networked system — but the way you tell them *who they are* is quite different.

### FreeBSD — Configuration Lives in /etc/rc.conf
FreeBSD follows a clean and predictable philosophy:
System configuration belongs in one place — `/etc/rc.conf`.

This file defines what your system does at boot: network interfaces, services, routing, and more. You edit it once, and it remains consistent — every restart, every time.

Example of a static configuration:
```sh
# /etc/rc.conf
# Static IPv4
ifconfig_em0="inet 10.0.2.11 netmask 255.255.255.0"
defaultrouter="10.0.2.1"
# Static IPv6
ifconfig_em0_ipv6="inet6 fd00:2::11 prefixlen 64"
ipv6_defaultrouter="fd00:2::1"
```
After saving, apply your changes:
```
service netif restart && service routing restart
```
Everything is declarative — no daemons or complex YAML layers, just a plain-text file that describes how your system should look.

That simplicity is one of FreeBSD’s biggest strengths: when something goes wrong, you know exactly where to look.

## Temporary Configuration on FreeBSD (ifconfig)
If you just want to test something — or configure a lab quickly — you can use the classic `ifconfig` command directly:
```sh
# Temp IPv4
sudo ifconfig em0 inet 10.0.2.11 netmask 255.255.255.0 up
sudo route add default 10.0.2.1
# or with CIDR notation
sudo ifconfig em0 inet 10.0.2.11/24 up
sudo route add default 10.0.2.1

# Temp IPv6
sudo ifconfig em0 inet6 fd00:2::11 prefixlen 64
sudo route add -inet6 default fd00:2::1
#or with CIDR notation
sudo ifconfig em0 inet6 fd00:2::11/64
sudo route add -inet6 default fd00:2::1
```

These commands are temporary — they’ll reset after reboot — but perfect for experimentation. It’s fast, transparent, and close to the metal.

## Linux — Many Paths to the Same Goal
Linux offers more flexibility — which can be both empowering and confusing. There’s more than one way to bring up a network interface.

### 1.  Temporary Configuration (Modern Tools)
The ip command is the standard way to manage interfaces on Linux:
```sh
# Temp IPv4
sudo ip addr add 10.0.2.10/24 dev eth0
sudo ip route add default via 10.0.2.1

# Temp IPv6
sudo ip -6 addr add fd00:2::10/64 dev eth0
sudo ip -6 route add default via fd00:2::1
```
Like FreeBSD’s `ifconfig`, these changes disappear after a reboot — great for testing or lab work.

Check your configuration anytime:
```sh
ip a
ip r
```
### 2. Persistent Configuration (Distribution-Specific)
Permanent network settings depend on your Linux distribution:

**Debian / Ubuntu (legacy):** `/etc/network/interfaces`
```sh
auto eth0
iface eth0 inet static
    address 10.0.2.10
    netmask 255.255.255.0
    gateway 10.0.2.1
```

**Modern Ubuntu (Netplan):** `/etc/netplan/01-eth0-config.yaml`
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.0.2.10/24]
      gateway4: 10.0.2.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```
Apply it: `sudo netplan apply`

**Fedora/RHEL/CentOS(`nmcli`): Use `nmcli` (NetworkManager)**

```sh
sudo nmcli con add type ethernet ifname eth0 con-name static-ip \
  ipv4.addresses 10.0.2.10/24 \
  ipv4.gateway 10.0.2.1 \
  ipv4.dns "8.8.8.8" \
  ipv4.method manual
```


Linux distributions give you options — from simple scripts to full network management frameworks — which is both its power and its complexity.

I personally prefer the Debian style.

## Two Philosophies, One Purpose
* FreeBSD (Persistent)
  Philosophy: Declarative, simple, unified
  Config Method: /etc/rc.conf

* FreeBSD (Temporary Lab)
  Philosophy: Manual & quick
  Config Method: ifconfig, route

* Linux (Flexible)
  Philosophy: Flexible, layered
  Config Method: ip, netplan, nmcli, etc.
  Persistence: Depends on the method

FreeBSD keeps configuration centralized and consistent. Linux offers modularity and choice, adapting to different environments — from cloud VMs to embedded systems.

Both are valid — it just depends on whether you value clarity or flexibility more.

## Ready to Try It Yourself?
The full hands-on lab for this article — including `ip` and `ifconfig` exercises — is available here:

👉 d3ep0ps Networking Foundations Lab on GitHub

You’ll find:

* step-by-step missions for Linux and FreeBSD
* troubleshooting and verification tips
* and a bonus IPv6 challenge

## Coming Next on d3ep0ps
In the next post, we’ll build on this foundation and start talking about firewalls:

    “Securing Unix Systems: pf vs iptables — Why FreeBSD Still Wins.”

We’ll explore how each system protects itself, compare firewall philosophies, and learn how to lock down our lab safely.

🧠 Follow d3ep0ps on Medium to keep learning Unix, FreeBSD, and Cloud networking — one command at a time.

Connect on LinkedIn — Let’s make engineering calm again.
