# Caging the Beast: Implementing Jails and Containers

> **"A container is not a real thing. It is a lie we tell the kernel."**

Welcome back.
I’ve been quiet for a few weeks. Like many of you, I took the Christmas and New Year season to disconnect, recharge, and step away from the terminal. Sometimes, the best way to solve a complex architectural problem is to stop looking at it for a while.

But now, the holidays are over. The tree is down, the coffee is fresh, and we have a job to do.

In our last article, we discussed the **AKF Scale Cube**. We decided to dismantle our Monolithic server by moving up the **Y-Axis** (Functional Decomposition). We agreed that the Web Server and the Database should no longer live in the same room.

Today, we stop drawing diagrams. Today, we build the cages.

---

## 1. The FreeBSD Standard: Jails

We start with FreeBSD because it did this first, and it does it cleanest.

If you have been following this series, you might remember installing **BIND** (DNS). On FreeBSD, security-conscious admins have been putting BIND into a `chroot` or a Jail for decades. Why? Because BIND processes untrusted input from the internet. If it gets exploited, you want the attacker trapped in a box, not roaming your root filesystem.

### The Anatomy of a Jail

A Jail is **System Virtualization**. It shares the kernel with the host, but it has its own:

1. **Filesystem Root** (`/usr/jails/web01`)
2. **IP Address** (`10.0.0.2`)
3. **Process Table** (PID 1 inside the jail is `init`)

### The Implementation

We aren't going to use complex management tools yet. We will look at the raw config to understand the mechanism.

**1. Create the Filesystem**
We need a directory that will become the "root" of our new world.

```bash
# Fetch the base system for the jail
bsdinstall jail /usr/jails/web01

```

**2. Configure the Jail (`/etc/jail.conf`)**

```text
web01 {
    # The directory becomes the root
    path = "/usr/jails/web01";
    
    # Networking
    host.hostname = "web01.d3ep0ps.com";
    ip4.addr = 10.0.0.2;
    interface = em0;

    # Allow raw sockets (for ping/traceroute debugging)
    allow.raw_sockets = 1;

    # Start command
    exec.start = "/bin/sh /etc/rc";
    exec.stop = "/bin/sh /etc/rc.shutdown";
}

```

**3. Start the Beast**

```bash
service jail start web01
jexec web01 /bin/sh

```

You are now inside. If you run `ps aux`, you see only the processes inside this box.
Now, you simply install Apache:

```bash
# Inside the jail
pkg install apache24
sysrc apache24_enable="YES"
service apache24 start

```

From the Host's perspective, this is just a process. From Apache's perspective, it owns the machine.


---

## 2. The Linux Approach: Namespaces & Cgroups

Moving to Linux, things get messier but more granular.
Unlike FreeBSD, "Containers" do not exist in the Linux kernel. There is no system call named `create_container()`.

Instead, we use two separate technologies glued together:

1. **Namespaces:** Isolate *visibility* (What I can see).
2. **Cgroups:** Isolate *resources* (What I can use).

### The Hidden Danger: CPU Quotas and Multithreading

This is where many fail.
When you set a limit of "1 CPU" on a container, you are telling the **CFS (Completely Fair Scheduler)** to give that container 100ms of runtime every 100ms window.

**Scenario A: The Node.js App (Single Threaded)**
Node.js is single-threaded. If it works hard, it uses that 100ms. If it hits the limit, the kernel pauses it. The app slows down, but it works.

**Scenario B: The Java/JVM App (Multi-Threaded)**
This is the trap.
The JVM loves threads. It might spawn 4 Garbage Collection threads + 4 Worker threads.
If all 8 threads wake up at the same time, they burn through your "1 CPU" quota in **12.5 milliseconds**.
The kernel sees you've used your budget and **throttles** (pauses) the container for the remaining 87.5ms.
Your fancy Java app freezes for huge chunks of time, causing massive latency spikes, even though the CPU usage looks low on the dashboard.

**Architect's Lesson:** You must align your container limits with your application's threading model.

### The Implementation (LXC)

To replicate the "System Container" feel of a Jail on Linux, we use **LXC** (Linux Containers) or **LXD**. This gives us a full OS, not just a single application process (like Docker).

**1. Create the Container**

```bash
lxc launch ubuntu:22.04 web01

```

**2. Enter the Cage**

```bash
lxc exec web01 -- bash

```

**3. Install Services**

```bash
# Inside the container
apt update
apt install apache2 mysql-server

```

---

## 3. Networking: Virtual Cables and Bridges

How do these cages talk to the world? It isn't magic; it's Layer 2 switching.

To connect an isolated container to the real network, we use **Virtual Ethernet (veth)** pairs or **epairs** (FreeBSD). Think of these as a virtual patch cable with two ends:

* **End A** is plugged into the Container/Jail.
* **End B** is plugged into a virtual switch (Bridge) on the Host.

### FreeBSD: The Bridge and Epair

On FreeBSD, we create a bridge (`bridge0`) and attach the physical interface (`em0`) to it.
The Jail gets an `epair` interface. It feels like a real network card, but it is purely software.

**The Firewall Rule (NAT):**
Since the Jail has a private IP (`10.0.0.2`), it cannot talk to the internet directly. We need **NAT** (Network Address Translation).
We configure **PF** (`/etc/pf.conf`) to act as the router:

```text
# /etc/pf.conf
ext_if="em0"
jail_net="10.0.0.0/24"

# NAT: Masquerade traffic from the jail as the host IP
nat on $ext_if from $jail_net to any -> ($ext_if)

# Redirection: Forward Port 80 on Host to Port 80 in Jail
rdr on $ext_if proto tcp from any to any port 80 -> 10.0.0.2

```

### Linux: The Linux Bridge

On Linux, LXC/Docker creates a `br0` (or `docker0`) bridge.
When the container starts, it creates a `veth` pair. `veth1` stays in the host namespace (plugged into `br0`), and `veth2` moves into the container namespace (renamed to `eth0`).

The kernel then uses `iptables` (or `nftables`) to rewrite the packets, exactly like your home Wi-Fi router does.

---

## 4. Filesystems: Why Data Must Escape

Finally, we need to handle data storage. This is the single biggest cause of data loss in containerized environments.

### The Problem: Union Filesystems (Copy-on-Write)

Most modern Linux containers (Docker) use **OverlayFS** or **AuFS**.
This is a "layered" filesystem. You have a read-only base image (Ubuntu), and a writable top layer.
If you modify a file, the kernel copies it from the bottom layer to the top layer (**Copy-on-Write** or CoW).

**Why Databases Hate CoW:**
Databases (MySQL, PostgreSQL) are I/O intensive. They write to disk constantly.
If every write requires a "copy-up" operation and passes through a complex UnionFS driver, two things happen:

1. **Performance tanks:** You introduce massive latency.
2. **Ephemerality:** If you destroy the container, that top "writable" layer (and your database) vanishes forever.

### The Solution: Bind Mounts (The Wormhole)

To fix this, we punch a hole through the container wall.
We take a standard, high-performance directory on the **Host** (ext4, XFS, or ZFS) and mount it directly into the container.

* **FreeBSD (Nullfs):**
```bash
mount_nullfs /data/mysql /usr/jails/db01/var/db/mysql

```


* **Linux (Bind Mount):**
```bash
lxc config device add db01 mysqldata disk source=/data/mysql path=/var/lib/mysql

```



Now, the "brain" (MySQL process) is in the cage, but the "memory" (Data) sits safely on the host's robust filesystem, bypassing the container overlay entirely.

---

## Conclusion

We have successfully caged the beast.

* **Apache** is running in a Jail.
* **MySQL** is running in a Container.
* **Networking** uses virtual bridges and NAT to route traffic.
* **Storage** uses bind mounts to ensure performance and persistence.

We have achieved **Y-Axis Scalability**. We can now limit the MySQL container to 8GB of RAM, and let Apache run wild with the CPU, without them killing each other.

But there is a problem.
We are typing manual commands. `lxc launch`, `jail start`, `mount_nullfs`.
If this server dies, how do we recreate this complex web of cages?

We need automation. We need a way to describe this entire system in a text file.
In the next article, we will talk about **Orchestration**—not Kubernetes yet, but the concepts of defining infrastructure as code.

**Next Up: "The Conductor: Automating the Stack."**