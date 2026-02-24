# The Vault: Persistent Data and Database Migrations

> **"Servers are cattle, but Data is a family heirloom."**

In the previous chapters of **d3ep0ps**, we built a self-healing cluster (**The Swarm**) and a high-velocity deployment pipeline (**The Forge**).

We have achieved a beautiful, highly automated architecture where code is ephemeral. If a container dies, the scheduler replaces it. If a hypervisor fails, the workloads migrate. But we have been avoiding a difficult truth: microservices are easy because they are stateless.

What happens when the workload involves a Database? If the scheduler moves your PostgreSQL container from Node A to Node B, the container spins up perfectly, but the local disk is gone. Your data vanished. Code is easy to move because it is stateless, but data has **gravity**. When your scheduler moves a database, the data must follow, or you are left with an empty shell.

To truly achieve a "Zero-Touch" infrastructure, we need to solve the two biggest challenges of state: **Space (Persistence)** and **Time (Evolution)**.

---

## 1. The Gravity of Data: Storage in a Cluster

As I noted in **The Swarm**, my experience with cloud providers taught me a critical lesson: storage must be decoupled from compute. We don't store data on the server’s local hard drive; we use **Network Attached Storage (NAS)** or **Distributed Filesystems**.

### The Container Storage Interface (CSI)

In a modern cluster, whether you use Kubernetes or Nomad, this decoupling is achieved using the **Container Storage Interface (CSI)**. CSI allows the orchestration tool to dynamically attach and detach storage volumes to containers as they move across physical boundaries.

For example, in our D3ep0ps lab powered by Nomad, we use a CSI plugin (like NFS, Ceph, or a cloud-provider specific driver). Our application doesn't care *where* it is running; it simply asks the scheduler for a volume.

**The D3ep0ps Way (Nomad CSI snippet):**

```hcl
job "database" {
  group "postgres" {
    # Request a pre-existing CSI volume
    volume "db-data" {
      type      = "csi"
      read_only = false
      source    = "postgres-prod-vol"
    }

    task "server" {
      driver = "docker"
      config {
        image = "postgres:15"
      }
      # Mount the network storage inside the container
      volume_mount {
        volume      = "db-data"
        destination = "/var/lib/postgresql/data"
      }
    }
  }
}
```

If Node 1 loses power, Nomad automatically reschedules the database task to Node 2, and the CSI plugin seamlessly unmounts the storage from Node 1 and attaches it to Node 2. The data follows the container.

### The POSIX Challenge and Split-Brain

CSI solves the mechanical problem of *attaching* storage, but what happens when the container actually tries to *use* it?

Most legacy applications expect a **POSIX-compliant** filesystem—they want to read and write files directly as if they own the block device entirely. While distributed filesystems (like GlusterFS or Ceph FS) can aggregate hard drives across a network and present a POSIX interface, they introduce the terrifying "Split-Brain" risk.

Network partitions *will* happen. When they do, if two nodes lose contact with each other but both think they are the "Primary," they will both blindly write to the shared disk simultaneously. This results in irrecoverable data corruption in milliseconds.

To prevent this, we must enforce a strict, cluster-wide rule: **One Writer at a time.** While distributed systems are excellent at scaling *read* operations (where data is replicated safely across nodes), having multiple cluster nodes writing to the same filesystem simultaneously without a distributed lock manager is architectural suicide. We gladly trade a bit of write-latency (waiting for a lock) for absolute data integrity.

---

## 2. Scaling the Core: Replicas and Sharding

When our application grows beyond the capabilities of a single shared disk, we move the logic from the filesystem layer up to the **Database layer**. The database engine itself becomes responsible for managing the cluster.

* **Master-Replica (Active-Passive):** The most common approach. We designate one instance as the "Primary" (the only one allowed to Write) and multiple "Read Replicas." This allows us to scale read-heavy workloads infinitely. The Primary handles the "Source of Truth," while the replicas stay synchronized to handle the query load.
* **Multi-Master (Active-Active):** A more complex setup where multiple nodes can accept writes. Tools like Galera Cluster or specialized distributed SQL databases provide this. It solves the single point of failure for writes but introduces heavy latency due to synchronous replication requirements.
* **Data Sharding:** When the dataset becomes too massive for one server to even *hold*, or when we strictly must overcome the single-writer bottleneck without the penalty of synchronous multi-master replication, we shard. We split the data horizontally (e.g., European Users on Server A, Asian Users on Server B). This pushes complexity into the application layer but removes the physical ceiling on scale. By sharding, every shard has its own dedicated writer, effectively parallelizing the write throughput. *Note: Modern cloud-native databases (like CockroachDB or TiDB) and sharding middlewares (like Vitess for MySQL) now automate this abstraction, allowing the developer to treat a massive, sharded cluster as a single logical database.*

---

## 3. Evolution: Schema Migrations in the Pipeline

Even if the network storage is safe and highly available, the data **structure** is constantly changing. The schema is a living thing.

If a human has to manually log into a database console and run an `ALTER TABLE` command before a deployment, your automation is broken. Worse, it creates a massive synchronization risk between the deployed code and the active database schema.

**The Strategy: Schema as Code.**

We treat our database schema exactly like we treat our application logic. Every change is a numbered, version-controlled script (e.g., `001_create_users.sql`, `002_add_email_index.sql`) stored alongside the source code in Git.

In **The Forge**, we use tools like **Flyway**, **Liquibase**, or **dbmate** to apply these changes automatically during deployment.

**The Handshake (Nomad `prestart` hook):**

Before the new version of your application is allowed to start, a "Migration Task" runs. This is defined as a `prestart` lifecycle hook in Nomad (or an `initContainer` in Kubernetes).

```hcl
task "migrate-db" {
  driver = "docker"
  
  # Run this task to completion before the main app starts
  lifecycle {
    hook    = "prestart"
    sidecar = false
  }

  config {
    image   = "dbmate/dbmate"
    command = ["dbmate", "up"]
  }
}
```

1. **The Logic:** The tool checks a dedicated `schema_version` table in the database.
2. **The Execution:** It applies only the missing scripts in sequential order.
3. **The Gate:** Only when the migration succeeds does the scheduler launch the main application container.

This guarantees that the **Schema** and the **Code** are always perfectly in sync, permanently eliminating "Column not found" errors in production.

---

## 4. The Last Line of Defense: Snapshots and Backups

High availability is not a backup system. A replicated database will happily replicate a `DROP TABLE users;` command to every node in milliseconds.

Persistence is only as good as your restore time. In the **d3ep0ps** lab, we use a layered, defense-in-depth approach:

* **The Time Machine (ZFS & LVM Snapshots):** If our underlying storage layer uses ZFS on FreeBSD (or OpenZFS on Linux), we can take instantaneous, atomic, block-level snapshots. If we use traditional Linux filesystems, **LVM (Logical Volume Manager)** provides similar snapshot capabilities. They cost almost nothing in performance or space. If an automated migration goes catastrophically wrong, we can "roll back" the entire volume to the exact second before the deployment occurred.
* **Database-Native Backups:** Snapshots capture the disk state, but sometimes we need application-aware backups.
  * *Logical Dumps:* Tools like `pg_dump` or `mysqldump` export the database as plain SQL text. They are slow to restore but incredibly useful for moving data between different database engine versions or for auditing.
  * *Physical/Point-in-Time:* Enterprise environments use tools like `pgBackRest` or `wal-g`. These continuously ship transaction logs (WAL) to object storage (like AWS S3 or a local MinIO cluster), allowing us to restore the database to any specific millisecond in time.
* **The Simplest Approach (rsync over SSH):** The most universally supported backup method for general files. We periodically copy data to another server. It is simple to debug and requires nothing more than secure SSH keys.
* **Incremental Binary Shipping (ZFS Send/Receive):** For advanced setups, we don't copy entire disks every night. We use the extreme efficiency of `zfs send | zfs receive` to stream only the changed blocks over SSH to an off-site TrueNAS server, or to reliable cloud storage.
* **The "3-2-1" Rule:** 3 copies of data, 2 different media types, 1 copy entirely off-site.

---

## Conclusion

We have finally completed the architecture.

* **The Swarm** provides the muscle (Compute).
* **The Gateway** provides the entrance (Ingress).
* **The Forge** provides the velocity (CI/CD).
* **The Vault** provides the memory (Persistence).

You are no longer a sysadmin fighting fires or manually connecting cables in a datacenter. You are a **System Architect** who has built a living, breathing, self-correcting organism.

You can deploy updates without fear, knowing the migrations will run automatically. You can sleep through a hard drive failure or a dead node, knowing the scheduler will move your compute tasks, and the storage interface will ensure the data follows. And, thanks to block-level snapshots and ship-away WAL archives, you can even sleep through a developer accidentally running `DROP TABLE users;` on a Friday afternoon.

**Next Up: "The Horizon: Scaling to the Edge and Beyond."**
