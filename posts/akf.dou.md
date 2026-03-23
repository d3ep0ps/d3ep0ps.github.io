# 3 Dimensions of System Scaling: Why Autoscaling is Only Part of the Picture

> **"Scaling isn't just about adding more servers; it's about how the system handles the weight of its own success."**

In previous **d3ep0ps** articles, we’ve discussed building clusters, automation, and data persistence. But one of the most critical skills for an architect is knowing how to scale a system correctly as the load begins to grow.

During technical interviews for DevOps and System Architect positions, I’ve noticed a pattern: when the conversation turns to scaling, it almost immediately narrows down to infrastructure.

*"We'll set up an Auto Scaling Group," "Add more pods to Kubernetes," "Provision a larger database instance."*

Candidates can spend hours passionately discussing Horizontal Pod Autoscaler (HPA) metrics, Terraform configurations, and stateless deployment strategies. But the conversation rarely touches upon the fundamental dimensions of scaling: dealing with state, the hard trade-offs of distributed systems, and real business requirements. We have begun to mistake simple infrastructure expansion for true architectural scaling.

## Why Do We Scale Systems at All?

Before discussing *how* to scale, we must remember *why* we do it. It doesn't start with CPU usage metrics; it starts with the business. Specifically, with the **SLA (Service Level Agreement)**—the contract between us and our clients.

The SLA defines what the client expects from our system. From these business agreements, specific engineering requirements emerge:

* **Availability**: The percentage of time the system is ready to process requests (the "nines").
* **Reliability**: The system’s ability to perform its functions without failure over a given period.
* **Data Integrity**: The guarantee that data will not be lost or corrupted.
* **Performance (Latency)**: The system's response speed. Even an "available" system is "broken" for a client if a page takes 10 seconds to load.
* **Scalability**: The system's ability to maintain performance as the load increases.
* **Security**: Protection of data and infrastructure, often strictly defined in enterprise contracts.

As the user base grows, maintaining these requirements (especially **scalability** and **availability**) within a single server becomes physically impossible. To fulfill the SLA under load, we are forced to distribute the system across multiple nodes.

## The CAP Theorem: Fundamental Constraints

The moment we move from a single server to a distributed system (a network of servers), scaling ceases to be just about "adding hardware." Fundamental laws of computing take over. The most famous of these is the **CAP Theorem**, which states that in any distributed system, it is impossible to simultaneously guarantee more than two out of three properties:

1. **Consistency**: Every client sees the same, most recent data (or gets an error).
2. **Availability**: Every request receives a successful response (without the guarantee that it contains the most recent data—e.g., the system might return a stale cache).
3. **Partition Tolerance**: The system continues to operate despite the loss of communication between its nodes.

The key point often missed: in the real world, networks *will* fail or lag. Therefore, **Partition Tolerance is not an option; it is a given.** Since network issues are unavoidable, engineers are always forced to make a difficult choice during a partition: do we stop the system to prevent serving stale data (choose Consistency), or do we keep responding to requests at the risk of serving outdated information (choose Availability)?

This Choice, driven by our SLA (is data accuracy or uptime more critical for the business right now?), dictates the entire architecture. This leads us to the main question: what tools can we use to achieve these goals?

## The Three Axes of Scaling: The AKF Scale Cube

To understand how we make these architectural decisions, we use a mental model known as the **AKF Scale Cube**. Let's look at it through a classic system design task: a URL shortening service like Bitly. I’m choosing a simple, familiar example so we can focus exclusively on the scaling axes.

Functionally, it's straightforward:

* User sends a long URL.
* System generates a short identifier (ID).
* Clicking the short link triggers a redirect.
* Analytics are collected.

At 10,000–20,000 requests per second (RPS), the architecture is trivial:

* Stateless API
* Load Balancer
* Multiple app instances
* A single relational database
* Cache (e.g., Redis) for popular links

### The X-Axis: Cloning (Horizontal Scaling)

When traffic increases, we simply change the replica count from 3 to 30 or set up autoscaling. This works as long as the bottleneck is CPU, RAM, or network at the application level.

By increasing the number of app instances behind a load balancer, we apply **Horizontal Scaling** (moving along the **X-axis**). This is our first solution for maintaining **Availability** and **Performance**. If one node fails, traffic goes to the others. If users increase, we add nodes to keep latency low.

This approach gives a quick result and requires minimal code changes. It’s the cheapest way to buy time. But eventually, as clones multiply, the system hits a new limit: all app instances begin to compete fiercely for the resources of the single database (writing new links, updating counters, transactions). Scaling the database itself (e.g., read replicas) only provides temporary relief.

### The Y-Axis: Functional Decomposition

When the load reaches hundreds of thousands of RPS and heavy analytics are added, a monolith with a single database can no longer survive. To fulfill the SLA, the system is logically divided into independent domains:

* **Redirect Service**: Extremely lightweight, optimized for read-only traffic.
* **Shortening Service**: A heavier component responsible for generating and validating new IDs.
* **Analytics Service**: Processes transition events (via an asynchronous pipeline or queue).

Now we isolate different types of load (high write vs. high read volumes) and can scale these components independently along the X-axis. From an SLA perspective, we protect **Scalability** and **Reliability**. For example, if the analytics service crashes or slows down, the core redirect functionality continues to work quickly and without failure.

Decomposing a service into smaller parts is movement along the **Y-axis**. It changes the application structure and team responsibilities. But there is a catch: often after decomposition, services continue to share a common database. The bottleneck doesn't disappear; it just becomes more obvious.

### The Z-Axis: Data Partitioning (Sharding)

As the service gains global popularity and load reaches millions of RPS, a single database physically cannot handle it.

The need arises for:

* Sharding by identifier hash (`hash(short_id)`).
* Regional distribution (Europe / USA / Asia).
* Isolating "hot" enterprise clients on dedicated clusters.
* Decoupled data storage (Data Warehouse) for analytics.

To process a request, the system must: determine which shard holds the data; route the request to the correct cluster; and account for different user latency profiles across regions (**Performance/Latency** requirement from SLA). This introduces entirely new engineering challenges: data rebalancing, complex cross-shard analytics, ensuring consistency, and disaster recovery at the shard level.

This is movement along the **Z-axis**. It radically changes the data model, and this is where we encounter the **CAP Theorem** in full. For instance, during regional sharding, global ID uniqueness becomes a problem. If connection between data centers in the USA and Europe is lost (Partition), we must choose:

* **Consistency**: Prohibit creating new links until the connection is restored (sacrificing availability but guaranteeing no conflicts).
* **Availability**: Allow creating links locally. we fulfill the availability SLA but are forced to introduce regional prefixes or complex reconciliation algorithms (eventual consistency), balancing latency against accuracy.

This is no longer just infrastructure scaling. it is a fundamental change in the system's topology.

## The Cost of Scaling: Infrastructure vs. Complexity

Architecture is always about trade-offs. Moving from the X-axis to the Y and Z axes isn't just a logical engineering evolution. It is a fundamental shift in focus: from increasing the number of servers to an exponential rise in engineering complexity, team coordination, and operational overhead.

X-axis movement (adding clones) is relatively simple: it mostly requires additional compute resources that are easy to provision or automate. In contrast, movement along the Y (microservices) and Z (sharding) axes brings exponential complexity. It involves painful refactoring of the monolith or database, solving fundamental consistency problems, and more complex deployment and monitoring. Ultimately, a distributed system can significantly slow down time-to-market.

This is why blindly implementing microservices or sharding just because "everyone else is doing it" is a mistake. A mature, high-load system almost always uses all three axes simultaneously, but it does so reasonably: scaling along a plane only evolutionarily, when simpler options are exhausted and maintaining the SLA with other tools is no longer physically possible.

## Summary

When we immediately propose "adding more pods" or "provisioning a larger DB instance" during an interview or a production "fire," we are acting reflexively, using only a fraction of our engineering toolkit.

The concept of the three axes, or the **AKF Scale Cube**, was proposed by Martin Abbott and Michael Fisher back in 2009. Since then, Kubernetes, serverless platforms, and powerful managed databases have emerged. Infrastructure has become much more convenient, but the fundamental constraints of distributed systems haven't gone anywhere.

Autoscaling doesn't solve data contention; microservices alone don't guarantee the absence of bottlenecks; and managed cloud services don't nullify the laws of physics, network latency, or the hard trade-offs of the CAP Theorem.

To make truly mature architectural decisions, follow a few basic principles:

1. **Start with the Business (SLA)**: Before scaling, understand the pain point. Are we saving Availability? Are we lacking Performance (latency)? Or can the system not ingest the new volume of data (Scalability)?
2. **Acknowledge Constraints (CAP)**: Accept that the network will fail (Partition Tolerance). Define with the business what you will sacrifice in that moment: data freshness (Consistency) or the ability to serve clients (Availability).
3. **Use the AKF Scale Cube to Choose a Strategy**: Ask yourself: *are we currently hitting compute limits (X-axis), database contention due to monolithic logic (Y-axis), or the limits of a single store at a global level (Z-axis)?*
4. **Don't Overcomplicate Prematurely**: For 80% of projects, smart movement along the X-axis will suffice for years. Move along the Y and Z axes only when the pain of implementation is less than the pain of being unable to meet the SLA with current tools.

For most products, X-axis scaling will be entirely sufficient throughout their lifecycle. and that’s okay. But as a system grows, understanding SLA requirements, CAP constraints, and other scaling dimensions allows for conscious, rather than reactive, engineering decisions.

Perhaps that is the key difference between an infrastructure mindset and an architectural one.
