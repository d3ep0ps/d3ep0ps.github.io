# Building the Web: LAMP vs FAMP — The Archetype of the Internet

> **Before we isolate, we must serve.**

In the previous article, we established why we are revisiting the "Old Stack." We hardened our server, secured the network, and prepared the ground. Now, it is time to build the engine.

Whether you run a massive WordPress cluster, a Magento e-commerce platform, or a custom API, the underlying architecture remains the undisputed king of the internet: **The 3-Tier Web Stack.**

  * **The OS:** Linux or FreeBSD
  * **The Web Server:** Apache (or Nginx)
  * **The Database:** MySQL (or MariaDB)
  * **The Language:** PHP (or Python/Perl)

This creates the famous acronyms: **LAMP** and **FAMP**. (There is also WAMP for Windows, but we are building servers here, not local dev environments).

In this guide, we won’t just run `apt-get install`. We are going to examine the fundamental differences in how Linux and FreeBSD approach storage, how they manage services, and how they maintain the system over time.



## 1\. The Stack Architecture

Before we install, we must understand the flow.

1.  **Apache** listens on TCP port 80/443. It handles the connection.
2.  If the request is for a static file (image, HTML), Apache serves it from the disk.
3.  If the request is for code (`index.php`), Apache passes it to the **PHP Interpreter**.
4.  PHP executes the logic, querying **MySQL** for data.
5.  MySQL returns the data, PHP renders HTML, and Apache sends it back to the user.

In modern microservices, these might be three different containers. In classic hosting, they are three services talking on `localhost`.



## 2\. Installation: Packages vs. Ports

Here is where the two operating systems diverge in philosophy.

### Linux (Debian/Ubuntu): The "Fast" Way

Linux distributions prioritize speed and convenience. They split software into many small, pre-compiled packages.

```bash
sudo apt update
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql
```

*Note: Linux automatically resolves dependencies and often starts the services for you immediately.*

### FreeBSD: The "Binary" Way (pkg)

FreeBSD offers a similar experience with its binary package manager. It is fast, clean, and reliable.

```bash
sudo pkg install apache24 mysql80-server php82 mod_php82
```

*Note: FreeBSD separates the Apache server (`apache24`) from the language module (`mod_php82`), giving you explicit control over versions.*

### FreeBSD: The "Architect" Way (Ports)

This is the superpower of BSD.
In my years at the hosting company, we often needed a specific version of Apache with a custom module, or a lightweight build of MySQL without embedded analytics. **Ports** let you compile software from source with a menu-driven interface.

```bash
# Update the ports tree
sudo git clone https://git.FreeBSD.org/ports.git /usr/ports

# Build Apache
cd /usr/ports/www/apache24
sudo make config
```

A blue TUI (Text User Interface) appears. You can check or uncheck features—enabling HTTP/2, disabling IPv6, stripping out debugging symbols.

```bash
sudo make install clean
```

**Why do this?**
Security and performance. If you don't install a feature, it can't be exploited. In high-performance hosting, stripping bloat matters.



## 3\. The Great Divide: Storage and Hierarchy

This is the most critical mental model to grasp. Linux and FreeBSD disagree fundamentally on two things: **How disks are managed** and **Where files go**.

### The Storage Layer: Layers vs. Integration

When you install a web server, you need to know where your data lives. If `/var/log` fills up, does your database crash? The answer depends on how you managed your storage.

#### Linux: The Layered Approach (LVM)

Linux follows the "Unix Philosophy" of small tools chained together. To get flexible storage, we typically use **LVM (Logical Volume Manager)**. It sits between your physical disk and your filesystem.

1.  **Physical Volume (PV):** The actual disk or partition (`/dev/sda1`).
2.  **Volume Group (VG):** A pool of storage created by combining PVs.
3.  **Logical Volume (LV):** The virtual "slice" you carve out of the group.
4.  **Filesystem:** Format the LV with `ext4` or `xfs`.

**Why this matters:**
If your database grows too large, you don't need to buy a bigger drive and copy everything. You simply plug in a new drive, add it to the Volume Group, and extend the Logical Volume on the fly.

#### FreeBSD: The Integrated Approach (ZFS)

FreeBSD (and modern systems generally) leans heavily on **ZFS**.
ZFS is a Volume Manager, RAID controller, and Filesystem all rolled into one. It doesn't have layers; it has knowledge of the entire stack.

Instead of partitions or volumes, ZFS uses **Datasets**.
In a standard FAMP stack, your layout often looks like this automatically:

  * `zroot/ROOT/default` (The Base OS)
  * `zroot/usr/local` (Your installed apps)
  * `zroot/var/log` (Your logs)
  * `zroot/var/db/mysql` (Your database)
  * `zroot/ust/home/d3ep0ps` (Your home directory)

**The Power of Datasets:**
Because ZFS manages all of this, you can apply policies per dataset instantly:

  * **Compression:** Turn on `lz4` compression just for `/var/log` (logs compress beautifully).
  * **Quotas:** Set a hard quota on `/zroot/var/db/mysql` so a runaway database cannot fill the root drive and crash the OS.
  * **Snapshots:** Take a snapshot of *just* the database dataset before an upgrade. If it breaks, `zfs rollback` is instant.

**A Note on "Slices":**
If you look at older FreeBSD documentation (or the underlying disk devices), you might see terms like `ada0s1` (Slice 1) and `ada0p2` (Partition 2).

  * **Slices** were FreeBSD's way of subdividing old MBR disks.
  * **Partitions** were subdivisions inside a slice.
    Today, with GPT (GUID Partition Table) and ZFS, we mostly treat the disk as a raw pool of storage, similar to how LVM treats a Physical Volume.

### The Filesystem Hierarchy

Because of this separation philosophy, FreeBSD enforces a strict boundary between the "Base System" and "Userland."

**Linux (The Blender)**

  * **Configs:** `/etc/apache2/`, `/etc/mysql/`
  * **Web Root:** `/var/www/html/`
  * **MySQL Data:** `/var/lib/mysql`
  * **Logs:** `/var/log/apache2/`
  * *Reality:* The OS configs (network, ssh) and Application configs (Apache, MySQL) are mixed in `/etc`.

**FreeBSD (The Separation)**

  * **Base OS Configs:** `/etc/` (SSH, PF, user accounts)
  * **Installed App Configs:** `/usr/local/etc/`
      * Apache: `/usr/local/etc/apache24/httpd.conf`
      * PHP: `/usr/local/etc/php.ini`
      * MySQL: `/usr/local/etc/mysql/my.cnf`
  * **Web Root:** `/usr/local/www/apache24/data/`
  * **MySQL Data:** `/var/db/mysql`
  * **Logs:** `/var/log/httpd-access.log`

**The Philosophy:** You could theoretically delete the entire `/usr/local` dataset, and your FreeBSD system would still boot, connect to the network, and allow SSH login. Your application layer never pollutes the kernel layer.



## 4\. Wiring It Together: Service Management (The Init Wars)

Installing software places the binaries on the disk, but the **Init System** is what breathes life into them. This is the first process the kernel starts (PID 1), and it is responsible for starting everything else.

Here, Linux and FreeBSD occupy two different worlds: **Revolution vs. Evolution.**

### Linux: From SysV to systemd (The Revolution)

For decades, Linux used **System V Init (SysV)**. If you are a veteran admin, you remember this well.

  * **The Scripts:** Services lived in `/etc/init.d/` as shell scripts.
  * **The Runlevels:** The OS had states called "Runlevels" (0–6).
      * `0`: Halt
      * `3`: Multi-user (Server mode)
      * `5`: Graphical (Desktop mode)
      * `6`: Reboot
  * **The Problem:** It was slow. Services started one by one (serial). If the network hung, everything behind it waited.

**Enter systemd (The Modern Standard)**
Around 2010–2015, most Linux distros switched to **systemd**. It was a massive paradigm shift.

  * **Parallelism:** It starts services simultaneously.
  * **Targets:** "Runlevels" were replaced by "Targets" (`multi-user.target`).
  * **Binary Logs:** Logs moved from text files to a binary format (`journalctl`).

<!-- end list -->

```bash
# Old Way (SysV)
service apache2 start
chkconfig apache2 on

# New Way (systemd)
systemctl enable apache2
systemctl start apache2
```

### FreeBSD: The rc.d System (The Evolution)

FreeBSD looked at the chaos of Init systems and decided to **evolve** rather than replace. It uses the **rc.d** system. It is still script-based (like SysV), but it solved the serialization problem intelligently.

  * **Dependency Based:** Instead of using numbers (`S01`, `S02`) to guess the order, every FreeBSD startup script contains metadata (`REQUIRE: NETWORKING`, `BEFORE: DAEMON`). The system calculates the correct order at every boot.
  * **Centralized Config (`rc.conf`):** Linux scatters "enabled" services. FreeBSD has a single source of truth: `/etc/rc.conf`.

**How We Configure It:**

**On FreeBSD (rc.conf):**
We edit the registry of truth. You can do this with `vi /etc/rc.conf` or use the helper:

```bash
sudo sysrc apache24_enable="YES"
sudo sysrc mysql_enable="YES"
sudo service apache24 start
```

*Note: On FreeBSD, you often need to perform one manual step for PHP. You must edit `/usr/local/etc/apache24/Includes/php.conf` (or create it) to tell Apache to handle `.php` files.*



## 5\. The Database: First-Time Security

Out of the box, MySQL is often insecure (empty root password, anonymous users allowed). Regardless of the OS, you must run the security script.

```bash
sudo mysql_secure_installation
```

Answer **Yes** to:

  * Set root password? (Make it strong)
  * Remove anonymous users?
  * Disallow root login remotely? (Critical)
  * Remove test database?
  * Reload privilege tables?

**Pro Tip:** Never make your application connect as `root`.

```sql
CREATE DATABASE d3ep0ps_web;
CREATE USER 'web_user'@'localhost' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON d3ep0ps_web.* TO 'web_user'@'localhost';
FLUSH PRIVILEGES;
```



## 6\. The Test: "Hello World"

Let’s verify the stack is talking. Create a file in your web root (`/var/www/html` on Linux, `/usr/local/www/apache24/data` on FreeBSD).

`index.php`:

```php
<?php
phpinfo();
?>
```

Open your browser and point it to your server's IP address: `http://192.0.2.10`.
You should see the purple PHP information page.

## 7\. Survival Skill: Storage Hygiene (Logs, Inodes, and Partitions)

In my hosting days, the \#1 cause of server crashes wasn't hackers—it was **full disks**. But "full" doesn't always mean what you think it means.

### The Hidden Killer: Inode Exhaustion

Most people check disk space with `df -h`. It says you have 50GB free.
But when you try to write a file, you get `No space left on device`. Why?

**You ran out of Inodes.**
Every file requires an **inode** (index node) to store its metadata. Standard filesystems create a fixed number of inodes when formatted.

**The Hosting Trap:**
A misconfigured PHP application starts creating session files in `/tmp` or `/var/lib/php/sessions`. It creates one small 4KB file for every visitor.

  * **Result:** You have 1 million tiny files. Your 1TB disk is only 1% full of data, but 100% full of inodes.
  * **The Check:** Run `df -i`. If it says 100%, you are dead in the water.

### Defensive Partitioning (Don't Mount / on Everything)

The worst partitioning scheme for a server is **one giant root (`/`) partition**.
If a user uploads too many files to `/home`, or Apache logs spam `/var`, they fill the **root filesystem**.
When `/` fills up, your shell can't create temp files, you can't hit `Tab` for autocomplete, and SSH might even reject you.

**The Fix:** Separate the volatile data from the OS.

#### Linux Strategy (LVM)

We use LVM to create separate logical volumes for the dangerous directories:

  * `/` (Root OS - Keep this clean\!)
  * `/var` (Logs and Databases - The most likely to explode)
  * `/tmp` (Temporary files - The Inode killer)
  * `/home` (User data)

If `/var` fills up, MySQL stops, **but the OS keeps running.** You can still SSH in to fix it.

#### FreeBSD Strategy (ZFS Datasets)

FreeBSD solves this automatically with ZFS datasets.

  * `zroot/tmp`
  * `zroot/var`
  * `zroot/var/log`

You can even set a **Quota** on `zroot/tmp`.

```bash
# Limit /tmp to 1GB so no script can crash the server
sudo zfs set quota=1G zroot/tmp
```

### Log Rotation

Finally, we must clean up the logs that are consuming that space.

**Linux (`logrotate`)**
Configured in `/etc/logrotate.d/apache2`. It uses `cron` to rename, compress, and delete old logs daily.

**FreeBSD (`newsyslog`)**
Configured in `/etc/newsyslog.conf`.

```text
# logfilename          [owner:group]    mode count size when  flags [/pid_file] [sig_num]
/var/log/httpd-access.log  root:wheel    644  7     * @T00  J     /var/run/httpd.pid 30
```

*Translation:* "Keep 7 compressed copies, rotate at midnight, and signal Apache to release the file handle."



## The Missing Link

We now have a robust Web, App, and Database stack running.
But right now, you can only visit it by typing an IP address.

To get `d3ep0ps.com` to point to this server, we need DNS.
And to host your *own* DNS on this server (creating `ns1.d3ep0ps.com`), we face a logical puzzle called the **Bootstrap Paradox**.

In the next article, we will tackle **BIND** and **Glue Records** to give our server a name.