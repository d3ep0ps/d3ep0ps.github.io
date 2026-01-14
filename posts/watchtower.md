# The Watchtower: Observability in a Fragmented System

> **"You cannot scale what you cannot see."**

In our previous articles, we did something remarkable.
We took a fragile Monolith and transformed it into a robust, isolated, automated system.

* **Apache** lives in a FreeBSD Jail defined by a `Bastillefile`.
* **MySQL** lives in a Linux Container defined by a `Dockerfile`.

But in doing so, we introduced a dangerous blind spot.
We broke `top`. We broke `tail -f /var/log/syslog`.

### The Regression

To be clear: **Centralized logging and monitoring didn't begin with containers.**
In a well-architected Monolith, you should already be shipping logs to a central server and monitoring CPU spikes. But let's be honest—most of us didn't. We relied on the fact that if the site was slow, we could just SSH in, run `top`, and check the local logs. The "Localhost" was our dashboard.

**With containers, that luxury is gone.**
We have built "Black Boxes."

* If a container restarts (which it is designed to do), the local logs inside it are **destroyed**.
* If you run `top` on the host, you see a mess of generic process IDs, not a clear view of "Web Server Health."

We have built a stealth bomber. It is fast, isolated, and powerful. But we are flying it blind.
To fix this, we need to build a **Watchtower**.



## 1. The Philosophy: Logs are Streams, Not Files

The first mental shift an Architect must make is about the nature of logs.

**The Old Way (Files):**
In the legacy world, applications wrote to files like `/var/log/nginx/access.log`. You managed these with `logrotate`.

**The New Way (Streams):**
In a distributed system, **containers have no persistent disk.** If your strategy relies on writing to a file inside the container, you have already lost data.
We must follow the **12-Factor App** principle: **Logs are a stream.**

* **The Application:** Should not care where logs go. It should just print to `stdout` (Standard Output) and `stderr` (Standard Error).
* **The Infrastructure:** Should capture that stream and ship it to a central vault.

This decouples the application from the storage. Apache doesn't need to know if logs are going to a file, a database, or the cloud. It just speaks; the infrastructure listens.



## 2. The Evolution of Monitoring

Before we pick a tool, we need to understand how monitoring has changed.

**Phase 1: Status Checks (Nagios/Zabbix)**
For decades, we used tools like **Nagios** or **Zabbix**. They asked binary questions: *"Is the server up?"* or *"Is CPU > 90%?"*
This worked great for static servers. But in a dynamic container environment, "Up/Down" isn't enough. We need to know *trends*.

**Phase 2: Graphs (RRDtool/Cacti)**
We started using **RRDtool** (Round Robin Database) to draw graphs. We could see CPU usage over time. But the data was static, resolution was lost over time, and it was hard to query complex relationships.

**Phase 3: Time-Series Databases (The Modern Era)**
Cloud-native systems produce millions of data points. We moved to **Time-Series Databases (TSDB)**. Instead of just asking "Is it up?", we stream numeric data constantly.



## 3. The Anatomy of a Stack: Collect, Store, Visualize, Alert

A modern monitoring system isn't just a database. It is a pipeline.
Regardless of the brand, every stack must perform four distinct actions:

1. **Collect** (Grab the data).
2. **Store** (Keep it efficiently).
3. **Visualize** (Show it to humans).
4. **Alert** (Wake up the on-call engineer).

In the wild, we usually run two parallel stacks: one for **Metrics** (Numbers) and one for **Logs** (Text).

### The Metrics Stack (Prometheus or TICK)

For numbers (CPU usage, RAM, Request Count), we need high-speed Time-Series Databases.

* **Option A (Pull): The Prometheus Ecosystem.**
* **Collect:** Exporters (e.g., `node_exporter` for OS, `mysqld_exporter` for DB). They expose metrics via HTTP.
* **Store:** Prometheus Server. It scrapes (downloads) metrics every 15s.
* **Visualize:** Grafana.
* **Alert:** Alertmanager.


* **Option B (Push): The TICK Stack (InfluxDB).**
* **Collect:** Telegraf. An agent that pushes metrics to the DB.
* **Store:** InfluxDB.
* **Visualize:** Chronograf.
* **Alert:** Kapacitor.



### The Logging Stack (The ELK Stack)

For text (Stack traces, Error messages, Access logs), Prometheus is useless. We need a search engine. The industry standard is the **ELK Stack** (Elasticsearch, Logstash, Kibana).

* **Collect (Beats):** Lightweight agents like **Filebeat** sit on the container host, read the JSON log streams, and ship them out.
* **Buffer/Transform (Logstash):** An optional processing layer that parses messy logs (e.g., turning a raw Apache log line into structured JSON fields like `client_ip` and `status_code`).
* **Store (Elasticsearch):** A massive search engine that indexes every word. This lets you ask questions like: *"Show me all logs containing 'NullPointerException' from 'web01' in the last 15 minutes."*
* **Visualize (Kibana):** The dashboard where you browse logs and investigate incidents.



## 4. Implementing on FreeBSD (The Syslog Roots)

FreeBSD has been doing "Streams" since the 1980s via `syslog`. We simply lean into this native capability.

**Logs (The Solution):**
We configure the Jail to forward everything to the Host or a Log Server.

1. **Inside the Jail:** We configure `syslogd` to send logs to a remote IP.
```text
# /etc/syslog.conf inside Jail
*.* @10.0.0.1:514

```


2. **On the Host:** We run a collector (like **syslog-ng** or **Fluentd**) to catch these packets and ship them to Elasticsearch.

**Metrics:**
We run the **Node Exporter** (Prometheus) or **Telegraf** (Influx) *on the Host*. Since FreeBSD Jails are just processes sharing the host kernel, the Host sees the CPU usage of every Jail perfectly. We get instant visibility without installing agents inside every single Jail.



## 5. Implementing on Linux (The Docker Driver)

On Linux, Docker makes this easier because it was built for this workflow.

**The Challenge:**
By default, Docker writes logs to JSON files on the host disk. This fills up the disk and requires cleanup.

**The Solution:**
We change the **Docker Logging Driver**.
Instead of writing to disk, we tell the Docker Daemon to ship logs directly to our aggregator.

```bash
# running a container that ships logs elsewhere
docker run \
  --log-driver=syslog \
  --log-opt syslog-address=udp://10.0.0.1:514 \
  my-app

```

Now, even if the container crashes and is deleted, the "last words" of the application are safe in our central database.



## 6. The Dashboard: Grafana

We have metrics flowing into our TSDB (Prometheus/Influx). We have logs flowing into our Aggregator (Elasticsearch).
Now we build the **Watchtower**.

We use **Grafana** to visualize this data.
Instead of typing `top`, we look at a dashboard.

* **Row 1:** Global Health (Traffic, Error Rates).
* **Row 2:** Infrastructure Health (CPU/RAM of the Host).
* **Row 3:** Service Health (MySQL Latency, Apache Active Connections).

This is the feedback loop.
When we deploy a new `Dockerfile` via our CI/CD pipeline, we watch the dashboard.

* Did memory usage spike? **Rollback.**
* Did error rates drop? **Success.**



## Conclusion

We have restored our vision.
We are no longer flying blind. We have **decoupled** the monitoring from the application.

* **The App** speaks to `stdout`.
* **The Container** isolates the process.
* **The Infrastructure** captures the stream (Collects, Stores, Visualizes, Alerts).

Now we have a system that is:

1. **Decoupled** (Architecture).
2. **Isolated** (Jails/Containers).
3. **Automated** (IaC).
4. **Observable** (Prometheus/ELK).

We are finally ready.
We have a robust, observable building block. But right now, it all runs on **one single server**.
What happens if the hard drive fails? What happens if we get 10,000 users and one server isn't enough?

We need to take this architecture and spread it across multiple machines.
We need to talk about **Orchestration**.

**Next Up: "The Swarm: Moving from One Server to a Cluster."**