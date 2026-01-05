
# Caging the Beast: Implementing Jails and Containers

> **"A container is not a real thing. It is a lie we tell the kernel."**

Welcome back.
I’ve been quiet for a few weeks. Like many of you, I took the Christmas and New Year season to disconnect, recharge, and step away from the terminal. Sometimes, the best way to solve a complex architectural problem is to stop looking at it for a while.

But now, the holidays are over. The tree is down, the coffee is fresh, and we have a job to do.

In our last article, we discussed the **AKF Scale Cube**. We decided to dismantle our Monolithic server by moving up the **Y-Axis** (Functional Decomposition). We agreed that the Web Server and the Database should no longer live in the same room.

Today, we stop drawing diagrams. Today, we build the cages.



## 1. The FreeBSD Standard: Jails

We start with FreeBSD because it did this first, and it does it cleanest.

If you have been following this series, you might remember installing **BIND** (DNS). On FreeBSD, security-conscious admins have been putting BIND into a `chroot` or a Jail for decades. Why? Because BIND processes untrusted input from the internet. If it gets exploited, you want the attacker trapped in a box, not roaming your root filesystem.

### The Anatomy of a Jail

A Jail is **System Virtualization**. It shares the kernel with the host, but it has its own:

1. **Filesystem Root** (`/usr/jails/web01`)
2. **Virtual Network Interface** (`epair0b`)
3. **Fully Isolated Network Stack** (Can run its own VPN, Firewall, etc.)

### 1.1 The "Root Filesystem" Problem

Before we can start a jail, we have a problem.
A Jail needs an environment. When you log in and type `ls`, the jail needs a `/bin/ls` binary to execute. It needs `/lib/libc.so` to run programs. It needs `/etc/passwd`.
This collection of files is called the **Userland** (or Base System).

**How to Build the "Gold Image" (Manual Method):**
We don't want to extract files every time we make a jail. We do it *once* to create a master template.

```bash
# 1. Create a dataset for our Gold Image
zfs create -o mountpoint=/usr/jails/base-13.2 zroot/jails/base-13.2

# 2. Download the official FreeBSD Userland (base.txz)
fetch https://download.freebsd.org/ftp/releases/amd64/13.2-RELEASE/base.txz

# 3. Extract it into the dataset
tar -xf base.txz -C /usr/jails/base-13.2

# 4. Lock it down (Snapshot)
zfs snapshot zroot/jails/base-13.2@gold

```

Now we have a pristine, read-only copy of the operating system.

### 1.2 The ZFS Superpower: Clones vs. Tarballs

Here is where FreeBSD leaves Linux in the dust. To create a new jail, we don't copy those files. We **Clone** them.

```bash
# Create a new jail from the snapshot
zfs clone zroot/jails/base-13.2@gold zroot/jails/web01

```

**Why this matters to an Architect:**

* 
**Speed:** The clone is created in milliseconds.


* **Efficiency:** It consumes **zero bytes** of storage until you write data. It points to the original blocks.


* **Safety:** Before a risky upgrade, `zfs snapshot zroot/jails/web01@pre-upgrade`. If it breaks, `zfs rollback` is instant.



### 1.3 The Implementation

Now that we have the filesystem, we configure the cage.

**1. Prepare the Host Networking (The Bridge)**
We need a virtual switch. Since we want to use private IPs (NAT), we give the bridge the Gateway IP.

```bash
ifconfig bridge0 create
ifconfig bridge0 10.0.0.1/24 up

```

**2. Configure the Jail (`/etc/jail.conf`)**
We use VNET (VIMAGE) to give the jail its own network stack.

```text
web01 {
    path = "/zroot/jails/web01";
    vnet;
    vnet.interface = "epair0b";

    # Create cable, plug into bridge
    exec.prestart += "ifconfig epair0 create up";
    exec.prestart += "ifconfig bridge0 addm epair0a";

    # Configure IP inside jail
    exec.start += "/sbin/ifconfig epair0b 10.0.0.2/24 up";
    exec.start += "/sbin/route add default 10.0.0.1";
    exec.start += "/bin/sh /etc/rc";

    # Cleanup
    exec.poststop += "ifconfig bridge0 deletem epair0a";
    exec.poststop += "ifconfig epair0a destroy";
}

```

**3. Start the Beast**

```bash
service jail start web01
jexec web01 /bin/sh

```

Inside, run `pkg install apache24`. From the host, it's just a process. From inside, it's a server.



## 2. The Linux Approach: Namespaces & Cgroups

Moving to Linux, things get messier but more granular.
Unlike FreeBSD, "Containers" do not exist in the Linux kernel. There is no system call named `create_container()`.

Instead, we use two separate technologies glued together:

1. 
**Namespaces:** Isolate *visibility* (What I can see).


2. 
**Cgroups:** Isolate *resources* (What I can use).



### 2.1 Building from Scratch (The Hard Way)

On FreeBSD, we used `fetch` and `tar`. On Linux, we use `debootstrap` (for Debian/Ubuntu) or extract a rootfs tarball (Alpine).

If you wanted to build a container **without** Docker or LXC, you would do this:

```bash
# 1. Download and unpack a minimal Debian system
sudo debootstrap --variant=minbase stable /var/lib/containers/web01 http://deb.debian.org/debian

# 2. Isolate it (Manual chroot/namespace entry)
sudo unshare --fork --pid --mount --net chroot /var/lib/containers/web01 /bin/bash

```

This command `debootstrap` is the Linux equivalent of extracting `base.txz`. It pulls down `apt`, `bash`, and `coreutils` to create a working environment.

### 2.2 The "Easy" Way (LXC)

Tools like **LXC** automate this. When you run `lxc launch`, it downloads a pre-built image (the result of a `debootstrap` run) and unpacks it for you.

```bash
# Downloads the image and creates the container
lxc launch ubuntu:22.04 web01

# Enter the container
lxc exec web01 -- bash

```

### 2.3 The Hidden Danger: CPU Quotas and Multithreading

This is where many fail.
When you set a limit of "1 CPU" on a container, you are telling the **CFS (Completely Fair Scheduler)** to give that container 100ms of runtime every 100ms window.

**Scenario A: The Node.js App (Single Threaded)**
Node.js works hard, hits the 100ms limit, gets paused, and resumes. It slows down, but works.

**Scenario B: The Java/JVM App (Multi-Threaded)**
The JVM spawns 8 threads. If they all wake up at once, they burn your "1 CPU" quota in **12.5 milliseconds**.
The kernel throttles the container for the remaining 87.5ms. Your app freezes, causing massive latency spikes.
**Architect's Lesson:** Align container limits with your threading model.



## 3. Networking: Virtual Cables and Bridges

How do these cages talk to the world? It isn't magic; it's Layer 2 switching.
We use **Virtual Ethernet (veth)** pairs or **epairs**. Think of these as a virtual patch cable:

* **End A** is plugged into the Container.
* 
**End B** is plugged into a virtual switch (Bridge) on the Host.



### FreeBSD: The Bridge and Epair

On FreeBSD, we create a bridge (`bridge0`) acting as the Gateway. The Jail gets an `epair` interface.
Since the Jail has a private IP (`10.0.0.2`), we use **PF** for NAT:

```text
# /etc/pf.conf
jail_net="10.0.0.0/24"
nat on em0 from $jail_net to any -> (em0)
rdr on em0 proto tcp from any to any port 80 -> 10.0.0.2

```

### Linux: The Linux Bridge

LXC/Docker creates a `br0`. When the container starts, it creates a `veth` pair. One end stays on the host (`veth123`), the other moves into the container namespace (`eth0`).



## 4. Filesystems: The Performance Trap

Finally, storage. This is where the architectural divergence is most painful.

### The Linux Problem: OverlayFS Overhead

Most Linux containers use **OverlayFS**. It stacks a read-only image and a writable top layer.
**The Trap:** If you modify a 1GB file, the kernel must **copy the entire 1GB file** from the lower layer to the upper layer before writing one byte. This "Copy-Up" latency kills database performance.

### The FreeBSD Solution: ZFS Datasets

FreeBSD Jails don't need OverlayFS because ZFS handles CoW at the **block level**.
Modifying one byte of a 1GB file allocates one new block. There is no massive file copy.

### The Universal Rule: Data Escapes

Regardless of OS, transient containers should not hold persistent data. We punch a hole through the container wall.

* **FreeBSD (Nullfs):**
```bash
mount_nullfs /data/mysql /usr/jails/db01/var/db/mysql

```


* **Linux (Bind Mount):**
```bash
lxc config device add db01 mysqldata disk source=/data/mysql path=/var/lib/mysql

```



Now, the "brain" (MySQL) is in the cage, but the "memory" (Data) sits safely on the host.



## Conclusion

We have successfully caged the beast.

* **Apache** is running in a Jail.
* **MySQL** is running in a Container.
* **Networking** uses virtual bridges and NAT.
* **Storage** uses bind mounts (and ZFS clones) for persistence.

We have achieved **Y-Axis Scalability**.
But we are typing manual commands. `lxc launch`, `jail start`. If this server dies, how do we recreate it?

We need automation. We need **Infrastructure as Code**.
**Next Up: "The Conductor: Automating the Stack."**